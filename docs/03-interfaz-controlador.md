# 03 — Interfaz con el controlador

> ## El XR06CX no tiene ModBUS
>
> El manual oficial lo confirma: **no hay puerto serie de proceso, no hay parámetro de
> dirección de esclavo y el único conector es la Hot Key**, que es un canal de programación
> que exige apagar y encender el instrumento ([doc 10 §1](10-hallazgos-manual-xr06cx.md#1-hallazgo-que-invalida-la-premisa-de-las-rev-1-y-2)).
>
> Por lo tanto:
> - **Camino A (línea base):** envolvente física. El gateway mide con su propia sonda y
>   detecta tensión en las tres salidas de relé del XR06. Se especifica en
>   [doc 02](02-hardware.md) y en `config/perfil-xr06cx-envolvente.yaml`.
> - **Camino B:** ModBUS RTU sobre RS-485, para controladores que sí lo tienen — ya sea
>   porque el cliente los tiene o porque se reemplaza el XR06CX para habilitar consigna
>   remota. Candidatos relevados, con sus fuentes, en
>   **[doc 11](11-controladores-nivel2.md)**: **Danfoss ERC 213** (mapa de registros
>   publicado por el fabricante) y **Carel IR33+** (RS-485 a bordo, sin adaptador). **Full
>   Gauge queda fuera:** su RS-485 usa protocolo propietario Sitrad, no ModBUS.
>
> **Este documento especifica el camino B** y el procedimiento de validación de cualquier
> mapa de registros. Sigue siendo necesario: es la base del *upgrade* de nivel 2 y de la
> extensión del producto a otras marcas, con la población FC-R de la misma PCB.

## 1. Capa física del camino B

| Opción | Cómo | Cuándo |
|---|---|---|
| **RS-485 nativo del controlador** | Bornera A/B del controlador → bornera A/B del gateway (población FC-R, SP3485 + jumper de 120 Ω) | Controlador con ModBUS de fábrica. **Es el camino B puro** |
| **Bus existente del cliente** | El gateway se cuelga de un RS-485 ya instalado | Hay que verificar que **no haya otro maestro** (un supervisor tipo XWEB se pisa con el gateway). Si lo hay: modo solo-lectura pasiva, u otro puerto |
| **Adaptador TTL→RS-485 del fabricante** | Para familias que sí lo soportan y lo documentan | **Es el caso más común del rubro** (Dixell XJ485-CX, Danfoss EKA 206, Eliwell, Carel IR33 original): +USD 35-55 por punto. **No aplica al XR06CX**, que no habla ModBUS por ningún medio |

**Parámetros serie:** los fija el manual del controlador elegido. El firmware autodetecta
probando 9600/19200 × {8N2, 8E1, 8N1} en el arranque y persiste la combinación que responda.
La dirección de esclavo se fija en el parámetro correspondiente del controlador — **si un
controlador no tiene ese parámetro, no es un esclavo ModBUS**, que es exactamente el
argumento que descartó al XR06CX.

## 2. Modelo de datos que expone el gateway (idéntico en los dos caminos)

El firmware **no** expone direcciones crudas: normaliza todo a este modelo, y **la app no
sabe de qué camino viene cada dato**. Es lo que permite vender el nivel 1 (envolvente) y el
nivel 2 (ModBUS) con la misma aplicación, y que un cliente que haga el upgrade no note más
diferencia que la aparición del control de consigna.

Cambiar de controlador implica cambiar el perfil en `config/`, no el firmware ni la app.

```json
{
  "temp_xr06": -18.4,          // °C, sonda de regulación del controlador
  "temp_evap": -24.1,          // °C, sonda de evaporador (si P2 está habilitada)
  "temp_probe": -18.1,         // °C, sonda accesorio (ausente en el equipo base)
  "rh_probe": 86.2,            // %HR, sonda accesorio (ausente en el equipo base)
  "setpoint": -18.0,           // °C; en el camino A se carga a mano en la puesta en marcha
  "setpoint_min": -25.0,       // límite LS
  "setpoint_max": -10.0,       // límite US
  "state": "cooling",          // off | cooling | idle | defrost | drip | fan_delay | standby
  "relays": { "compressor": true, "defrost": false, "fans": true, "light": false },
  "compressor_current_a": 7.3, // solo con accesorio CT; ausente en el equipo base
  "door_open": false,          // de la entrada digital del XR06, o de DI1 propia
  "door_openings_today": 23,
  "door_open_seconds_today": 412,
  "alarms": ["AH1"],           // ver §4
  "comm_ok": true,             // camino B: enlace ModBUS. Camino A: siempre true
  "source": "wrapper",         // wrapper | modbus — la app lo usa para habilitar la consigna
  "ts": "2026-10-21T14:03:12Z"
}
```

## 3. Mapa de registros — plantilla del camino B

> Las direcciones de abajo quedaron de la plantilla Dixell de las rev. 1 y 2. **No
> corresponden al XR06CX** (que no tiene ModBUS) y deben rellenarse con el manual del
> controlador que se elija para el camino B. Se conservan como estructura: los nombres de
> parámetros ya están corregidos a la nomenclatura real (`AU`/`AL`/`Ad`/`id`/`Md`/`di`/`iF`),
> que es la que usa toda la familia XR y sirve de referencia.

Convención: direcciones **0-based de protocolo**; se indica la función Modbus.
`[VERIFICAR]` = confirmar contra manual + banco.

### 3.1 Lectura de estado

| Señal | Función | Dirección | Escala | Notas |
|---|---|---|---|---|
| Temperatura sonda 1 (regulación) | 0x04 Read Input Reg | `0x0000` `[VERIFICAR]` | ×0.1 °C, int16 con signo | Respetar `CF` (°C/°F) y `rES` (resolución) del XR06 |
| Temperatura sonda 2 (evaporador) | 0x04 | `0x0001` `[VERIFICAR]` | ×0.1 °C, int16 | Puede estar deshabilitada (`P2P = n`) |
| Palabra de estado / flags | 0x04 ó 0x03 | `0x0010` `[VERIFICAR]` | bitfield | Ver §3.3 |
| Estado de salidas (relés) | 0x01 Read Coils ó bits de la palabra de estado | `[VERIFICAR]` | bit | Compresor / deshielo / ventiladores / luz |
| Estado de entradas digitales del XR06 | 0x02 Read Discrete Inputs | `[VERIFICAR]` | bit | Incluye el fin de carrera de puerta si está cableado al XR06 |

**Puerta y compresor sin hardware adicional.** Dos requerimientos del cliente se cubren
leyendo registros, sin cablear nada propio:

- **Aperturas de puerta:** si el fin de carrera ya está cableado a una entrada digital del
  XR06 (`i1F = dor`), su estado se lee por Modbus y el gateway cuenta las aperturas y el
  tiempo acumulado. La entrada DI1 propia queda como respaldo para las instalaciones donde
  el contacto no está cableado al controlador — hay que **verificarlo en el relevamiento**,
  porque es la diferencia entre instalar un hilo o ninguno.
- **Funcionamiento del compresor:** el bit del relé de compresor da arranques, paradas y
  duty cycle con resolución de 1 s, que es exactamente lo que pide el requerimiento. El
  clamp de corriente accesorio solo agrega *confirmación física* de la marcha (contactor
  pegado, térmico abierto), que es diagnóstico predictivo, no indicación de estado.

### 3.2 Lectura/escritura de parámetros

| Parámetro XR06 | Significado | Función | Dirección | Escala |
|---|---|---|---|---|
| `SET` | Consigna de temperatura | 0x03 / 0x06 | `0x0100` `[VERIFICAR]` | ×0.1 °C, int16 |
| `Hy` | Diferencial (histéresis) | 0x03 / 0x06 | `[VERIFICAR]` | ×0.1 °C |
| `LS` / `uS` | Límite inferior / superior de consigna | 0x03 | `[VERIFICAR]` | ×0.1 °C — **el gateway clampea todo comando a este rango** |
| `AU` / `AL` | Umbral de alarma de alta / baja | 0x03 / 0x06 | `[VERIFICAR]` | ×0.1 °C |
| `Ad` | Retardo de alarma de temperatura | 0x03 / 0x06 | `[VERIFICAR]` | min |
| `id` | Intervalo entre deshielos | 0x03 / 0x06 | `[VERIFICAR]` | h |
| `Md` | Duración máxima de deshielo | 0x03 / 0x06 | `[VERIFICAR]` | min |
| `di` | Retardo de alarma de puerta abierta | 0x03 / 0x06 | `[VERIFICAR]` | min |
| `iP` | Polaridad de la entrada digital (`oP`/`cL`) | 0x03 / 0x06 | `[VERIFICAR]` | enum. **`ONF` no existe en el XR06CX** (doc 10 §5) |
| `iF` | Configuración de la entrada digital (`EA`/`bA`/`do`/`dF`/`Au`/`Hc`) | 0x03 / 0x06 | `[VERIFICAR]` | enum, según el controlador |

### 3.3 Comandos

| Comando | Implementación | Dirección | Notas |
|---|---|---|---|
| Stand-by ON/OFF | 0x05 Write Single Coil ó bit en registro de comando | `[VERIFICAR]` | Solo camino B. **En el camino A el ON/OFF se hace cortando la alimentación del controlador** (doc 10 §5). Verificación cruzada obligatoria en los dos casos |
| Forzar deshielo | 0x05 Write Single Coil | `[VERIFICAR]` | Útil para mantenimiento remoto; limitar a 1 cada 30 min |
| Reconocer / silenciar alarma | 0x05 | `[VERIFICAR]` | No borra el evento en el historial del gateway |

### 3.4 Cuidados obligatorios en la escritura

1. **Desgaste de EEPROM.** Los parámetros del XR06 se guardan en EEPROM (~100 k ciclos).
   El firmware: (a) escribe solo si el valor cambia, (b) aplica un mínimo de **60 s entre
   escrituras** del mismo parámetro, (c) limita a **20 escrituras/día** por parámetro y
   (d) rechaza con error explícito a la app si se excede, en lugar de fallar silenciosamente.
   Un termostato "deslizable" en la app que escriba en cada movimiento del dedo destruye el
   controlador: la app **envía la consigna al soltar**, con debounce de 1.5 s.
2. **Clamp a `LS`/`US`.** Toda consigna se recorta al rango del propio XR06; si el usuario
   pide fuera de rango, la app explica el límite y quién lo configuró.
3. **Escritura atómica y confirmada.** Todo write va seguido de un read-back del mismo
   registro antes de responder "aplicado" a la app.
4. **Un solo maestro en el bus.** Si el cliente ya tiene un supervisor Dixell (XWEB), el
   gateway debe configurarse en **modo solo-lectura pasiva** o conectarse a otro puerto:
   dos maestros en el mismo RS-485 se pisan. Verificar en el relevamiento.

## 4. Normalización de alarmas

> En el **camino A** estas mismas alarmas las calcula el gateway sobre su propia sonda y la
> detección de salidas; los códigos y la severidad son idénticos, así que la app y el
> backend no cambian. Las diferencias están en `config/perfil-xr06cx-envolvente.yaml` y
> resumidas en [doc 10 §2](10-hallazgos-manual-xr06cx.md#2-qué-se-puede-y-qué-no-se-puede-hacer-con-este-controlador).

| Código gateway | Origen | Severidad | Disparo |
|---|---|---|---|
| `temp_high` | Flag `HA` del controlador, y evaluación propia sobre `temp_p1` contra el umbral configurado en la app | Crítica | Doble evaluación sobre el mismo dato: alarma incluso si el controlador tiene `AU` mal configurado |
| `temp_low` | Flag `LA`, ídem | Crítica | Ídem |
| `probe_fail_p1` / `probe_fail_p2` | **Camino B:** flags del controlador. **Camino A:** `probe_fail_inferred` por firma de ciclado `Cy`/`Cn` (doc 10 §6), ~2 h | Crítica | El controlador pasa a ciclos por tiempo: hay riesgo real de producto |
| `probe_mismatch` | \|temp_p1 − temp_probe\| > 3 °C por > 15 min | Media | **Solo con sonda accesorio.** Predictiva: hielo o deriva en la NTC de regulación |
| `comm_fail` | 3 timeouts Modbus consecutivos (3 s) | Media | **Solo camino B.** En el camino A no hay enlace que caer: la temperatura la mide el propio gateway, y la alarma equivalente es `own_probe_fail` |
| `door_open` | **Camino A:** DI1 propia (la entrada del XR06 no se comparte, doc 10 §4). **Camino B:** registro del controlador | Media | > 90 s configurable; se registran *todas* las aperturas, no solo las alarmadas |
| `defrost_timeout` | Duración de deshielo > `Md` + 20 % | Media | Resistencia quemada o válvula de gas caliente trabada |
| `defrost_ineffective` | Tras N deshielos, la temperatura de evaporador no sube de 0 °C | Media | Predictiva: evaporador bloqueado por hielo |
| `compressor_no_current` | Relé activo y CT < 0.5 A por > 60 s | Crítica | **Solo con accesorio CT.** Térmico, fase o contactor. Sin CT, la falla se detecta más tarde por `temp_high` |
| `short_cycling` | > 10 transiciones del bit de relé por hora | Media | Predictiva: carga de refrigerante, presostato o histéresis mal configurada. Funciona sin CT |
| `cmd_fail` | Read-back ≠ comando en ambos caminos | Alta | El usuario se enteró de que su orden NO se aplicó |
| `device_offline` | Nube: LWT o sin heartbeat > 5 min | Alta | Se genera en el backend, no en el equipo |
| `power_loss` | Brown-out detectado (supercap) / arranque en frío | Media | Con estampa de tiempo del RTC: permite reconstruir cortes |

Todas las alarmas tienen **retardo de confirmación** y **histéresis de reposición** (3 %
del rango o 1 °C, lo que sea mayor) para no generar ráfagas de notificaciones en el borde
del umbral — la causa nº 1 de que un cliente silencie la app y deje de usar el producto.

## 5. Procedimiento de descubrimiento y validación de un mapa de registros

Vale para cualquier controlador del camino B, y es lo que permite agregar una marca nueva al
producto sin tocar el firmware.

Banco: controlador + fuente + 2 NTC (una en vaso con hielo) + simulador de contacto de puerta
+ gateway (población FC-R) + PC con `modpoll`/`mbpoll`.

1. **Leer el manual del controlador concreto** y pedir el mapa de registros **por escrito**.
   Si no declara ModBUS, no tiene parámetro de dirección de esclavo, o el fabricante no
   entrega el mapa, **no es candidato** — es la verificación que descartó al XR06CX y la que
   descarta a la familia Dixell para el camino B (doc 11 §3.4). Hacerla primero ahorra
   semanas.
2. **Enumerar.** `mbpoll -m rtu -a <addr> -b 9600 -P none -s 2 -t 4 -r 1 -c 125 /dev/ttyUSB0`
   para funciones 0x03 y 0x04, barriendo el rango que indique el manual. Guardar el dump como
   línea base.
3. **Diferenciar por estímulo.** Cambiar *una* cosa a la vez y volver a volcar: sumergir la
   sonda en hielo (→ registro de temperatura), cambiar la consigna desde el frente (→ registro
   de consigna), forzar un deshielo, abrir el contacto de puerta, provocar una alarma de alta.
   El diff identifica cada dirección sin ambigüedad.
4. **Validar escalas y signo** con al menos tres puntos, incluyendo uno negativo (−18 °C) y
   uno positivo, en °C y en °F, y con la resolución en entero y en décimas.
5. **Validar escritura:** escribir la consigna, read-back, verificar en el display físico,
   cortar y restituir alimentación y comprobar persistencia.
6. **Validar el ON/OFF** por el camino que ofrezca el controlador, con verificación cruzada, y
   desconectando el RS-485 en caliente.
7. **Ensayo de ruido:** conmutar un contactor real 200 veces en el mismo tablero y medir la
   tasa de error CRC. Criterio de aceptación: **< 0,1 % de tramas con error y cero alarmas
   espurias**.
8. **Volcar el resultado** a un perfil nuevo en `config/`, versionado con el modelo y la
   versión de firmware ensayada. Un perfil por versión de controlador.

El paso 3 es el que convierte una incógnita de documentación en un dato duro, y es replicable
por el cliente para cualquier controlador que quiera incorporar después.
