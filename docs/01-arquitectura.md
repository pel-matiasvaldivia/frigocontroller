# 01 — Arquitectura de sistema (rev. 2)

## 1. Reparto de responsabilidades

| Capa | Responsable | Qué hace | Qué NO hace |
|---|---|---|---|
| Control, seguridad **y medición** | **XR06CX** | Lazo de temperatura, ciclos de deshielo, protección de compresor, relés de potencia, **sondas NTC P1/P2**, estado de puerta | Nada que dependa de la red |
| Supervisión y comando | **Gateway FrigoController** | Lee estado y temperaturas por Modbus, comanda consigna y ON/OFF, genera alarmas locales, almacena y reenvía | Jamás conmuta el compresor ni sustituye los enclavamientos del XR06 |
| Persistencia y reglas | **Backend** | Series temporales, escalado de alarmas, reportes, multi-sitio, usuarios | Control en tiempo real |
| Operación | **App** | Visualización, comando, reconocimiento de alarmas, reportes | — |

**Cambio clave de la rev. 2:** la medición vuelve al XR06. Esto tiene una ventaja real más
allá del costo — la app muestra **exactamente** el número del display del controlador, así
que el operador nunca ve dos temperaturas distintas para la misma cámara. Era una fuente
previsible de llamados a soporte.

Y tiene una consecuencia que hay que asumir: **el enlace Modbus pasa a ser crítico.** Sin
él no hay dato de proceso, por lo que `comm_fail` sube de severidad media a **alta**, y la
calidad de la interfaz física al XR06 (aislación, protecciones, cable) deja de ser un lujo.
Por eso la aislación galvánica se mantuvo pese al recorte de costos.

## 2. Flujo de datos

```
XR06 ──Modbus 1 s──▶ cache de estado ──┐
  · temperaturas P1 / P2                │
  · bits de relés y deshielo            │
  · flags de alarma AH1/AL1/P1/P2       ├──▶ motor de     ──▶ eventos ──▶ cola en flash
  · estado de entrada de puerta         │    alarmas           (ring buffer, 30 días)
DI1/DI2 ──scan 10 ms───────────────────▶┤   (local, offline)        │
[sonda T/HR accesorio] ──I²C 30 s──────▶┤                           │
[CT accesorio] ──RMS 1 s───────────────▶┘──▶ telemetría 60 s ───────┤
                                                                    ▼
                                                    MQTT/TLS ──▶ broker ──▶ reglas ──▶ push / WhatsApp / mail
                                                         ▲
App ──▶ backend ──▶ comando MQTT (QoS1, idempotente) ────┘
```

| Dato | Período | Motivo |
|---|---|---|
| Polling Modbus de estado y temperaturas | 1 s | Capturar transiciones de relés y deshielo |
| Lectura de parámetros (`SET`, umbrales, `idF`, `MdF`…) | 10 s / 5 min | Cambian poco; ahorra tráfico en el bus |
| Scan de entradas digitales propias | 10 ms + antirrebote (puerta: 2 s) | No perder un pulso de puerta |
| Telemetría a la nube | 60 s (configurable 15 s–15 min) | Costo de datos vs. resolución |
| Eventos y alarmas | Inmediato (< 2 s) | Es el valor del producto |
| Registro para trazabilidad | 15 min (promedio, mín, máx) | Requisito típico de auditoría |

## 3. Comando de ON/OFF: dos caminos, uno confiable

Sin cambios respecto de la rev. 1 — es el punto más delicado del requerimiento, porque un
ON/OFF que falla deja la cámara apagada sin que nadie lo sepa.

**Camino A — Modbus (primario):** escritura del bit de stand-by. Requiere que el parámetro
`ONF` esté habilitado. El XR06 queda en stand-by "consciente", con display y alarmas.

**Camino B — Relé sobre entrada digital (respaldo):** una entrada digital del XR06
configurada como ON/OFF (`i1F = oFF`), comandada por el **Relé 1** con contacto **NC**.
Funciona aun si el puerto serie falla.

**Verificación cruzada:** tras cada comando el gateway relee el estado por Modbus. Si en
10 s no coincide, usa el camino alternativo; si tampoco, emite `cmd_fail` con el estado
real. **Nunca se reporta "apagado" u "encendido" por haber enviado el comando: solo por
haberlo leído.**

> **No recomendado:** cortar la alimentación del compresor con un relé del gateway. Se
> pierden los tiempos mínimos de protección y los enclavamientos del XR06.

## 4. Matriz de modos de falla (FMEA, rev. 2)

| Falla | Detección | Comportamiento del gateway | Comportamiento de la cámara |
|---|---|---|---|
| Caída de Wi-Fi | Sin ACK MQTT | Sigue leyendo el XR06 y registrando en flash; alarmas locales por buzzer; reconexión con backoff | Normal |
| Caída del backend | LWT / sin ACK | Ídem; reenvía backlog al volver | Normal |
| Gateway sin alimentación | Nube: LWT + sin heartbeat > 5 min → "equipo offline" | — | **Normal (fail-safe)** |
| **Falla del enlace al XR06** (cable TTL, aislador, puerto) | 3 timeouts Modbus consecutivos (3 s) | **Alarma `comm_fail` de severidad alta.** Sin temperatura: el equipo queda ciego al proceso y lo dice explícitamente en la app. Sigue funcionando DI1/DI2 y el relé de ON/OFF | Normal (el XR06 controla solo) |
| Falla de sonda del XR06 (P1/P2) | Flags `P1`/`P2` | Alarma `probe_fail` crítica | XR06 pasa a ciclos por tiempo — requiere visita |
| Deshielo que no termina | Duración > `MdF` + 20 % | Alarma `defrost_timeout` | XR06 corta por tiempo máximo |
| Puerta abierta | Estado de la DI del XR06 por Modbus, o DI1 propia | Alarma a los 90 s (configurable) | XR06 puede inhibir compresor/ventiladores |
| Relé del gateway pegado | Verificación cruzada de estado | Alarma `cmd_fail` | El XR06 conserva su lógica |
| Compresor que no arranca | **Solo con accesorio CT** (relé activo y < 0.5 A por 60 s). Sin CT: se detecta indirectamente por la alarma de alta temperatura, más tarde | `compressor_no_current` ó `temp_high` | Sube la temperatura |
| Hielo o deriva en la sonda de regulación | **No detectable sin sonda accesorio** (se perdió `probe_mismatch`) | — | Queda a cargo del mantenimiento preventivo |
| Ciclado corto | Conteo de transiciones del bit de relé por Modbus (sin CT, con menor precisión) | `short_cycling` | Normal pero degradado |

Las dos filas de "no detectable sin accesorio" son el precio explícito del recorte de
costos: están listadas aquí, y no escondidas, para que la decisión se tome con el dato.

## 5. Decisiones de arquitectura y por qué (actualizado)

| Decisión | Alternativa descartada | Razón |
|---|---|---|
| **Temperatura tomada del XR06 por Modbus** | Sonda propia de serie | −USD 10.60/u. y un solo número para el operador y el display. Costo: `comm_fail` pasa a crítica y se pierde la verificación cruzada |
| **Sonda T/HR como accesorio, no de serie** | Sensor barato a bordo del gabinete | El XR06 no mide HR y un sensor dentro del tablero mediría la humedad del tablero. Se cotiza solo donde la HR importa |
| **Conexión directa al puerto TTL** | Módulo XJ485 oficial | −USD 45/punto, el 54 % del ahorro total. Riesgo cubierto con población alternativa RS-485 en la misma PCB |
| **Aislación galvánica mantenida** | TTL/RS-485 sin aislar (−USD 2.50) | Transitorios de contactores y lazos de masa; y ahora el enlace es crítico. Una visita técnica cuesta USD 150 |
| ESP32-C3 de un solo núcleo | ESP32-S3 | La carga real es 1 trama Modbus/s; se compensa con prioridades y watchdog |
| Una PCB con cuatro poblaciones | Un diseño por variante | Un stock, una herramienta, cuatro SKU — y el plan B del TTL sale gratis |
| Motor de alarmas en el equipo | Reglas solo en la nube | Alarma con la red caída, que es cuando suelen fallar las cosas |
| Mapa Modbus en archivo de configuración | Direcciones compiladas | Variantes de firmware del XR06 y extensión futura a otros combistatos sin tocar el firmware |
| MQTT sobre TLS mutuo | HTTP polling | Comandos descendentes, bajo consumo de datos, detección de offline con LWT |
