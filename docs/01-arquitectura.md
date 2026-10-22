# 01 — Arquitectura de sistema (rev. 3)

## 1. Reparto de responsabilidades

| Capa | Responsable | Qué hace | Qué NO hace |
|---|---|---|---|
| Control y seguridad | **XR06CX** | Lazo de temperatura, ciclos de deshielo, protección anticiclos (`AC`), retardo al arranque (`od`), relés de potencia | Comunicar nada: no tiene puerto serie de proceso |
| Medición y supervisión | **Gateway FrigoController** | Mide temperatura con sonda propia, **observa las tres salidas de relé** del controlador, lee la puerta, genera alarmas, almacena y reenvía, y **habilita o deshabilita** al XR06 para el ON/OFF remoto | No conmuta cargas, no sustituye los enclavamientos del XR06, no escribe parámetros |
| Persistencia y reglas | **Backend** | Series temporales, escalado de alarmas, reportes, multi-sitio, usuarios | Control en tiempo real |
| Operación | **App** | Visualización, comando, reconocimiento de alarmas, reportes | — |

El manual del XR06CX dice, en su primera página, que el instrumento *"no debe ser usado como
un dispositivo de seguridad"*. Esa frase define la arquitectura: el controlador es el lazo de
control, el gateway es el testigo que avisa, y **ninguno de los dos es el dispositivo de
seguridad de la instalación** — ese sigue siendo el presostato, el térmico y el monitor de
fase, que el gateway puede vigilar por su entrada DI2.

## 2. Flujo de datos

```
NTC1 cámara (propia) ──5 s──────────────▶┐
NTC2 evaporador (propia, opc.) ──5 s────▶┤
Salida compresor del XR06 ──20 ms───────▶┤
Salida deshielo del XR06 ──20 ms────────▶├──▶ reconstrucción de  ──▶ motor de  ──▶ eventos ──▶ cola en flash
Salida ventiladores del XR06 ──20 ms────▶┤    estado del equipo       alarmas       (30 días)
DI1 puerta (propia) ──10 ms─────────────▶┤    (enfriando/deshielo/     (local,           │
DI2 auxiliar ──10 ms────────────────────▶┤     goteo/retardo vent./     offline)          │
[SHT45 / CT accesorios] ────────────────▶┘     parado/apagado)              │             │
                                                                           │             │
   parámetros del XR06 cargados a mano ──▶ umbrales y lógica ──────────────▶┤             │
   en la puesta en marcha (AU/AL/Ad/dA/                                     │             │
   Cy/Cn/td/Md/dt/Fd/ot/oE)                     telemetría 60 s ───────────┤             │
                                                                           ▼             ▼
                                                       MQTT/TLS ──▶ broker ──▶ reglas ──▶ push / WhatsApp / mail
                                                            ▲
App ──▶ backend ──▶ comando MQTT (QoS1) ────────────────────┘
```

| Dato | Período | Motivo |
|---|---|---|
| Detección de las 3 salidas de CA | 20 ms de scan, confirmación sobre 3 ciclos de red | Capturar transiciones de relé sin leer el parpadeo de 50 Hz |
| Sondas NTC propias | 5 s, mediana de 5 | Inercia térmica alta; filtra ruido del tablero |
| Entradas digitales | 10 ms + antirrebote (puerta: 2 s) | No perder un pulso de puerta |
| Telemetría a la nube | 60 s (configurable 15 s–15 min) | Costo de datos vs. resolución |
| Eventos y alarmas | Inmediato (< 2 s) | Es el valor del producto |
| Registro para trazabilidad | 15 min (promedio, mín, máx) | Requisito típico de auditoría |

**Las tres salidas de relé son la telemetría que el controlador no sabe transmitir.** Con
ellas y los parámetros del XR06 (`td`, `dt`, `Fd`, `FC`) se reconstruye la máquina de estados
completa, con resolución de 1 s y sin tocar el cableado de potencia.

## 3. ON/OFF remoto: habilitar o deshabilitar al controlador

El XR06CX **no tiene parámetro `ONF`** ni tecla de ON/OFF documentada, y su única entrada
digital no ofrece una función de ON/OFF
([doc 10 §5](10-hallazgos-manual-xr06cx.md#5-onoff-remoto-lo-que-el-manual-habilita)). El
camino implementado es cortar su alimentación:

| Regla | Motivo |
|---|---|
| Relé **NC** en serie con los 230 Vac del XR06 (3,5 VA) | Relé sin energía = controlador alimentado = cámara refrigerando. Un gateway muerto no apaga nada |
| Configurar **`od` ≥ 3 min** en el XR06 al instalar | Retarda la activación de salidas al reponer tensión: es la protección del compresor |
| Mínimo 5 min entre conmutaciones, impuesto en firmware | Protege compresor y relé aunque la app insista |
| Watchdog de hardware sobre el relé | Si el firmware se cuelga con el relé energizado, se libera solo: la cámara vuelve a enfriar |
| **Nunca** el relé en serie con la bobina del contactor | El XR06 no se enteraría del corte y, al reponer, cerraría sin el retardo anticiclos `AC`: riesgo de rotura de compresor |
| Verificación por observación, no por comando | Tras cortar, el gateway **confirma que las tres salidas quedaron sin tensión**; tras reponer, que el controlador volvió a operar. Si no coincide → `cmd_fail` con el estado real |

**Nunca se reporta "apagado" u "encendido" por haber enviado el comando: solo por haber
observado el efecto en las salidas.** Esto es más fuerte que el read-back de un registro: se
verifica el mundo físico, no lo que un controlador dice de sí mismo.

## 4. Matriz de modos de falla (FMEA, rev. 3)

| Falla | Detección | Comportamiento del gateway | Comportamiento de la cámara |
|---|---|---|---|
| Caída de Wi-Fi | Sin ACK MQTT | Sigue midiendo, observando y registrando en flash; alarmas locales por buzzer; reconexión con backoff | Normal |
| Caída del backend | LWT / sin ACK | Ídem; reenvía backlog al volver | Normal |
| Gateway sin alimentación | Nube: LWT + sin heartbeat > 5 min → "equipo offline" | — | **Normal (fail-safe):** el relé NC deja el XR06 alimentado |
| **Falla de la sonda propia** | Circuito abierto o en corto, valor fuera de −50…60 °C, derivada > 5 °C/min | **`own_probe_fail` crítica.** Sin ella no hay temperatura: la app lo declara explícitamente. Siguen el estado de salidas, la puerta y el ON/OFF | Normal (el XR06 controla con su propia sonda) |
| Falla de la sonda del XR06 (`P1`) | **Inferencial:** ciclado periódico `Cy`/`Cn` (15/30 min por defecto) sin relación con la temperatura medida | `probe_fail_inferred` crítica, **en ~2 h** | XR06 cicla por tiempo: riesgo real de producto, requiere visita |
| Falla de la sonda de evaporador (`P2`) | **Inferencial:** todos los deshielos duran exactamente `Md` | `probe_fail_inferred` (P2) | Deshielo por tiempo, menos eficiente |
| Deshielo que no termina | Salida de deshielo activa > `Md` + 20 % | `defrost_timeout` | El XR06 corta por `Md` |
| Deshielo inefectivo | Con NTC2: tras 3 deshielos el evaporador no supera 0 °C | `defrost_ineffective` (predictiva: evaporador bloqueado) | Pérdida de capacidad |
| Puerta abierta | DI1 propia | Alarma a los 90 s; alarmas de temperatura enmascaradas (igual que hace el XR06 con `dC`/`rd`) | Según `dC`: por defecto `Fc`, compresor y ventiladores paran |
| Relé de corte pegado | Las salidas no cambian tras el comando | `cmd_fail` | El XR06 conserva su lógica |
| Compresor que no arranca | **Con accesorio CT:** salida con tensión y < 0,5 A por 60 s. **Sin CT:** más tarde, por `temp_high` | `compressor_no_current` ó `temp_high` | Sube la temperatura |
| Ciclado corto | > 10 arranques/hora en la salida de compresor, **excluyendo deshielos (`td=in`) y paradas por puerta (`dC`)** | `short_cycling` (predictiva) | Normal pero degradado |
| Corte de energía general | Brown-out + RTC | `power_loss` con estampa de tiempo; **se aplica la ventana `dA` (90 min) de exclusión de alarma**, igual que el XR06 | Arranca con `od`, y `dP` decide si deshiela |
| **Alguien cambia un parámetro en el frente del XR06** | **No detectable** | La app muestra la antigüedad de la última verificación y recuerda revalidar cada 6 meses | Normal |

Las filas inferenciales y la última son el precio de trabajar con un controlador no
comunicable. Están listadas acá, y en la matriz de cobertura del README, para que la
decisión se tome con el dato a la vista.

## 5. Decisiones de arquitectura y por qué

| Decisión | Alternativa descartada | Razón |
|---|---|---|
| **Envolvente física del XR06CX** | ModBUS vía XJ485 (rev. 1) o TTL directo (rev. 2) | **El XR06CX no tiene ModBUS** (doc 10 §1). Las dos revisiones anteriores no eran instalables |
| **Detección de tensión en las 3 salidas** | Transformadores de corriente en las 3 líneas | La detección de tensión es inmune al tamaño de la carga, cuesta un tercio y no requiere abrir ningún cable. El CT queda como accesorio para *confirmar* marcha |
| **Sonda propia obligatoria** | — | No hay alternativa: es la única fuente de temperatura |
| **NTC del mismo tipo que la del controlador** | Sensor digital de otra tecnología | Permite comparar lecturas con el display sin discutir tecnologías, y replicar el offset `ot` |
| **ON/OFF por corte de alimentación del XR06** | Entrada digital en `bA`, o relé en serie con el contactor | `bA` gasta la única entrada digital y deja el equipo mostrando una alarma `CA` mientras está "apagado". El relé en serie con el contactor pierde la protección `AC` |
| **Sensor de puerta propio** | Compartir la entrada digital del XR06 | Inyectar tensión en una entrada de contacto limpio ajena puede dañar el controlador, y la entrada queda libre |
| **Parámetros del XR06 cargados a mano** | Asumir valores por defecto | Los umbrales del gateway tienen que coincidir con los del controlador, o alarman en momentos distintos y el cliente pierde confianza |
| **Aislación de 4 kV en la detección de CA** | Divisor resistivo directo | Tres entradas permanentemente a potencial de red: es la zona más crítica de la placa |
| Motor de alarmas en el equipo | Reglas solo en la nube | Alarma con la red caída, que es cuando suelen fallar las cosas |
| Perfiles en archivo de configuración | Lógica compilada | Un perfil por estrategia (`envolvente` / `modbus`) y por modelo de controlador, sin tocar el firmware |
| Una PCB con cuatro poblaciones | Un diseño por variante | El camino B (ModBUS, upgrade de nivel 2) sale sin ingeniería nueva |
