# 01 — Arquitectura de sistema

## 1. Reparto de responsabilidades

| Capa | Responsable | Qué hace | Qué NO hace |
|---|---|---|---|
| Control y seguridad | **XR06CX** | Lazo de temperatura, ciclos de deshielo, protección de compresor (tiempos mínimos), relés de potencia | Nada que dependa de la red |
| Supervisión y comando | **Gateway FrigoController** | Lee estado, mide T/HR propia, lee puerta y contactos auxiliares, comanda consigna y ON/OFF, genera alarmas locales, almacena y reenvía | Jamás conmuta el compresor directamente ni sustituye los enclavamientos del XR06 |
| Persistencia y reglas | **Backend** | Series temporales, escalado de alarmas, reportes, multi-sitio, usuarios | Control en tiempo real |
| Operación | **App** | Visualización, comando, reconocimiento de alarmas, reportes | — |

Esta separación es lo que hace al producto vendible en un contexto de alimentos: ninguna
decisión que afecte la cadena de frío depende de la nube ni del enlace Wi-Fi.

## 2. Flujo de datos

```
XR06 ──Modbus RTU 1 s──▶ cache de estado ──┐
SHT45 ──I²C 5 s─────────▶ filtro + plausib.─┤
DI (puerta, HP, térmico) ──scan 10 ms──────▶┤──▶ motor de     ──▶ eventos ──▶ cola NVS/flash
CT compresor ──RMS 1 s─────────────────────▶┤    alarmas           (ring buffer, 30 días)
                                            │   (local, offline)        │
                                            └──▶ telemetría 60 s ───────┤
                                                                        ▼
                                                        MQTT/TLS ──▶ broker ──▶ TSDB + reglas ──▶ push / WhatsApp / mail
                                                             ▲
App ──▶ backend ──▶ comando MQTT (QoS1, idempotente) ────────┘
```

**Cadencias de diseño**

| Dato | Período | Motivo |
|---|---|---|
| Polling Modbus de estado | 1 s | Capturar transiciones de relés y deshielo |
| Lectura T/HR propia | 5 s | Inercia térmica alta; ahorra ruido y consumo |
| Scan de entradas digitales | 10 ms + antirrebote 50 ms (puerta: 2 s de confirmación) | No perder un pulso de puerta |
| Telemetría a la nube | 60 s (configurable 15 s–15 min) | Compromiso costo de datos / resolución HACCP |
| Eventos y alarmas | Inmediato (< 2 s) | Es el valor del producto |
| Registro para trazabilidad | 15 min (promedio, mín, máx del intervalo) | Requisito típico de auditoría |

## 3. Comando de ON/OFF: dos caminos, uno confiable

Es el punto más delicado del requerimiento, porque un ON/OFF que falla deja la cámara
apagada sin que nadie lo sepa.

**Camino A — Modbus (primario):** escritura del bit de stand-by del XR06. Requiere que el
parámetro `ONF` esté habilitado. Ventaja: el XR06 queda en stand-by "consciente",
mantiene display, alarmas de puerta y comunicación.

**Camino B — Relé sobre entrada digital (respaldo y verificación):** se configura una
entrada digital del XR06 (`i1F = oFF` o equivalente) como ON/OFF remoto y se la comanda
con el **Relé 1** del gateway. Ventaja: funciona aun si el puerto serie falla.

**Regla de verificación cruzada:** tras cada comando, el gateway relee el estado por
Modbus. Si en 10 s el estado no coincide con lo comandado, usa el camino alternativo y, si
tampoco coincide, emite la alarma `cmd_fail` a la app con el estado real. **Nunca se
reporta "apagado" u "encendido" por haber enviado el comando: solo por haberlo leído.**

> **No recomendado:** cortar la alimentación del compresor con un relé del gateway. Se
> pierden los tiempos mínimos de protección del compresor y los enclavamientos del XR06.

## 4. Matriz de modos de falla (FMEA resumido)

| Falla | Detección | Comportamiento del gateway | Comportamiento de la cámara |
|---|---|---|---|
| Caída de Wi-Fi | Sin ACK MQTT / sin AP | Sigue registrando en flash, alarmas locales por buzzer y relé 2, reconexión con backoff | Normal |
| Caída del backend | LWT / sin ACK | Igual al anterior; reenvía backlog al volver | Normal |
| Gateway sin alimentación | Nube: LWT + ausencia de heartbeat > 5 min → alerta "equipo offline" | — | **Normal (fail-safe)** |
| Falla de puerto serie / XJ485 | 3 timeouts Modbus consecutivos | Alarma `comm_fail`; **sigue midiendo con sonda propia y reportando T/HR** | Normal |
| Falla de sonda del XR06 (P1/P2) | Flag del XR06 | Alarma `probe_fail` crítica | XR06 pasa a modo degradado (ciclos por tiempo) — alarma crítica, requiere visita |
| Falla de sonda propia | Sin ACK I²C, valor fuera de rango o derivada imposible | Alarma `own_probe_fail`, usa la del XR06 como fuente única | Normal |
| Relé del gateway pegado | Verificación cruzada de estado | Alarma `cmd_fail` | Según el camino: el XR06 conserva su lógica |
| Deshielo que no termina | Duración > `MdF` + margen | Alarma `defrost_timeout` | XR06 corta deshielo por tiempo máximo |
| Compresor que no arranca | Bit de relé activo pero CT sin corriente > 60 s | Alarma `compressor_no_current` (**térmico, fase o contactor**) | Sube temperatura → alarma de alta |
| Ciclado corto | > N arranques/hora medidos por CT | Alarma `short_cycling` (predictiva: carga de gas, presostato) | Normal pero degradado |
| Puerta abierta | DI + `dOA` del XR06 | Alarma a los 90 s (configurable) | XR06 puede inhibir compresor/ventiladores |

Las tres filas marcadas en negrita son las que justifican el producto frente a un simple
datalogger: detectan el problema **antes** de que el producto se pierda.

## 5. Decisiones de arquitectura y por qué

| Decisión | Alternativa descartada | Razón |
|---|---|---|
| Sonda T/HR propia, independiente del XR06 | Leer solo los registros del XR06 | Permite detectar discrepancia sonda-vs-sonda (deriva/hielo en la NTC) y seguir reportando con el serie caído. Además el XR06 no mide HR |
| RS-485 aislado galvánicamente | RS-485 no aislado (más barato) | Lazos de masa y transitorios de contactores en tableros de refrigeración; es la falla de campo nº 1 en gateways industriales |
| Motor de alarmas en el equipo | Reglas solo en la nube | Alarma con la red caída, que es exactamente cuando suelen fallar las cosas |
| Mapa Modbus en archivo de configuración | Direcciones compiladas | La línea XR tiene variantes de firmware y el producto debe extenderse luego a otros combistatos (XR02/XR03, Eliwell, Full Gauge) sin tocar el firmware |
| Wi-Fi 2.4 GHz + LTE opcional | Solo LTE | Costo y alcance: la mayoría de los comercios tiene Wi-Fi; LTE se reserva para la variante FC-300 |
| MQTT sobre TLS mutuo | HTTP polling | Comandos descendentes, bajo consumo de datos, detección de offline con LWT |
