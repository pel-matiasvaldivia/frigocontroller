# 03 — Interfaz Modbus con el Copeland/Dixell XR06

> **Advertencia de ingeniería, leer antes de cotizar:** la línea XR expone un **puerto TTL**
> (el mismo que usa la Hot Key de copia de parámetros), *no* un RS-485 nativo. El mapa de
> registros Modbus **depende de la versión de firmware** del controlador. Todas las
> direcciones de este documento están marcadas **`[VERIFICAR]`** y deben confirmarse con
> (a) el anexo "RS485 ModBUS-RTU" del manual del XR06 instalado y (b) el banco de pruebas
> de §5. **No se libera firmware a producción con direcciones no validadas.**
> Esta es la única incógnita real del proyecto y se cierra en una tarde de banco.

## 1. Capa física: tres opciones

| Opción | Cómo | Pro | Contra | Recomendación |
|---|---|---|---|---|
| **A. XJ485-CX** (módulo oficial Dixell) | TTL del XR06 → XJ485 → RS-485 → gateway | Soportado por el fabricante, aislado, cable estándar, ya homologado | +USD 35–55 por punto, una caja más en el tablero | **Baseline para el piloto y para instalaciones donde el cliente exige soporte oficial** |
| **B. Interfaz TTL propia** | Conector del puerto TTL del XR06 directo a un puerto del gateway (nivel 5 V, aislado con ADuM1201) | Elimina el costo del XJ485, un solo equipo | Pinout del conector a relevar; no soportado por el fabricante; riesgo de garantía del XR06 | **Objetivo de costo para la v2**, previa validación eléctrica y acuerdo con el cliente |
| **C. Sin Modbus (degradado)** | Solo sonda propia + DI (contacto de alarma y de deshielo del XR06) + relé ON/OFF | Universal, funciona con cualquier combistato | Sin setpoint remoto ni diagnóstico fino | **Modo de respaldo del firmware**, y argumento comercial para equipos viejos |

El gateway implementa **A y C en la v1**; el firmware ya contempla B por tener la capa
física abstraída.

**Parámetros serie del XR06 (a confirmar):** 9600 bps, 8 bits de datos, 2 bits de stop,
sin paridad `[VERIFICAR]`. La dirección de esclavo se fija en el parámetro de dirección
del propio controlador (`Adr` / familia-dispositivo según modelo) `[VERIFICAR]`.
El firmware prueba automáticamente 9600/19200 × {8N2, 8E1, 8N1} en el arranque y persiste
la combinación que responda.

## 2. Modelo de datos que expone el gateway (estable, independiente del XR06)

El firmware **no** expone direcciones crudas: normaliza todo a este modelo. Cambiar de
controlador implica cambiar `config/modbus-xr06.yaml`, no el firmware ni la app.

```json
{
  "temp_xr06": -18.4,          // °C, sonda de regulación del controlador
  "temp_evap": -24.1,          // °C, sonda de evaporador (si P2 está habilitada)
  "temp_own": -18.1,           // °C, sonda propia del gateway
  "rh_own": 86.2,              // %HR, sonda propia
  "setpoint": -18.0,           // °C, consigna leída del XR06
  "setpoint_min": -25.0,       // límite LS
  "setpoint_max": -10.0,       // límite US
  "state": "cooling",          // off | cooling | idle | defrost | drip | fan_delay | standby
  "relays": { "compressor": true, "defrost": false, "fans": true, "light": false },
  "compressor_current_a": 7.3, // medición propia por CT
  "door_open": false,
  "door_openings_today": 23,
  "door_open_seconds_today": 412,
  "alarms": ["AH1"],           // ver §4
  "comm_ok": true,
  "ts": "2026-10-21T14:03:12Z"
}
```

## 3. Mapa de registros (plantilla a validar)

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

### 3.2 Lectura/escritura de parámetros

| Parámetro XR06 | Significado | Función | Dirección | Escala |
|---|---|---|---|---|
| `SET` | Consigna de temperatura | 0x03 / 0x06 | `0x0100` `[VERIFICAR]` | ×0.1 °C, int16 |
| `Hy` | Diferencial (histéresis) | 0x03 / 0x06 | `[VERIFICAR]` | ×0.1 °C |
| `LS` / `US` | Límite inferior / superior de consigna | 0x03 | `[VERIFICAR]` | ×0.1 °C — **el gateway clampea todo comando a este rango** |
| `ALU` / `ALL` (ó `AU`/`AL`) | Umbral de alarma de alta / baja | 0x03 / 0x06 | `[VERIFICAR]` | ×0.1 °C |
| `ALd` | Retardo de alarma de temperatura | 0x03 / 0x06 | `[VERIFICAR]` | min |
| `idF` | Intervalo entre deshielos | 0x03 / 0x06 | `[VERIFICAR]` | h |
| `MdF` | Duración máxima de deshielo | 0x03 / 0x06 | `[VERIFICAR]` | min |
| `dOA` | Retardo de alarma de puerta abierta | 0x03 / 0x06 | `[VERIFICAR]` | min |
| `ONF` | Habilitación de ON/OFF (tecla / entrada digital) | 0x03 | `[VERIFICAR]` | enum — **debe estar habilitado para el ON/OFF remoto** |
| `i1F` | Función de la entrada digital 1 | 0x03 / 0x06 | `[VERIFICAR]` | enum: puerta / alarma externa / ON-OFF / deshielo |

### 3.3 Comandos

| Comando | Implementación | Dirección | Notas |
|---|---|---|---|
| Stand-by ON/OFF | 0x05 Write Single Coil ó bit en registro de comando | `[VERIFICAR]` | Camino A de §3 del doc 01. Verificación cruzada obligatoria |
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

| Código gateway | Origen | Severidad | Disparo |
|---|---|---|---|
| `temp_high` | Flag `AH1` del XR06 **y/o** sonda propia > umbral por > `ALd` | Crítica | Doble fuente: si la sonda propia detecta alta y el XR06 no, se alarma igual y se marca `probe_mismatch` |
| `temp_low` | Flag `AL1` / sonda propia | Crítica | Ídem |
| `probe_fail_p1` / `probe_fail_p2` | Flags `P1`/`P2` del XR06 | Crítica | El XR06 pasa a ciclos por tiempo: hay riesgo real de producto |
| `own_probe_fail` | Sin ACK I²C, CRC inválido, valor fuera de rango o derivada > 5 °C/min | Media | Se degrada a fuente única |
| `probe_mismatch` | \|temp_xr06 − temp_own\| > 3 °C por > 15 min | Media | **Predictiva:** típicamente hielo sobre la sonda de regulación o deriva — se descubre antes de perder producto |
| `comm_fail` | 3 timeouts Modbus consecutivos (3 s) | Media | Se mantiene el resto de las funciones |
| `door_open` | DI1 y/o `dOA` del XR06 | Media | > 90 s configurable; se registran *todas* las aperturas, no solo las alarmadas |
| `defrost_timeout` | Duración de deshielo > `MdF` + 20 % | Media | Resistencia quemada o válvula de gas caliente trabada |
| `defrost_ineffective` | Tras N deshielos, la temperatura de evaporador no sube de 0 °C | Media | Predictiva: evaporador bloqueado por hielo |
| `compressor_no_current` | Relé activo y CT < 0.5 A por > 60 s | Crítica | Térmico, fase o contactor |
| `short_cycling` | > 10 arranques/hora | Media | Predictiva: carga de refrigerante, presostato o histéresis mal configurada |
| `cmd_fail` | Read-back ≠ comando en ambos caminos | Alta | El usuario se enteró de que su orden NO se aplicó |
| `device_offline` | Nube: LWT o sin heartbeat > 5 min | Alta | Se genera en el backend, no en el equipo |
| `power_loss` | Brown-out detectado (supercap) / arranque en frío | Media | Con estampa de tiempo del RTC: permite reconstruir cortes |

Todas las alarmas tienen **retardo de confirmación** y **histéresis de reposición** (3 %
del rango o 1 °C, lo que sea mayor) para no generar ráfagas de notificaciones en el borde
del umbral — la causa nº 1 de que un cliente silencie la app y deje de usar el producto.

## 5. Procedimiento de descubrimiento y validación

Banco: XR06CX + XJ485 + fuente + 2 NTC (una en vaso con hielo) + simulador de contacto de
puerta + gateway + PC con `modpoll`/`mbpoll`.

1. **Enumerar.** `mbpoll -m rtu -a <addr> -b 9600 -P none -s 2 -t 4 -r 1 -c 125 /dev/ttyUSB0`
   para funciones 0x03 y 0x04, barriendo `0x0000–0x01FF`. Guardar el dump como línea base.
2. **Diferenciar por estímulo.** Cambiar *una* cosa a la vez y volver a volcar:
   sumergir la sonda en hielo (→ registro de temperatura), cambiar `SET` desde el frente
   (→ registro de consigna), forzar deshielo (→ bit de estado), abrir el contacto de puerta,
   provocar alarma de alta subiendo `ALU`. El diff identifica cada dirección sin ambigüedad.
3. **Validar escalas y signo** con al menos tres puntos, incluyendo uno negativo
   (−18 °C) y uno positivo, en °C y en °F, y con `rES` en entero y en décimas.
4. **Validar escritura:** escribir `SET`, read-back, verificar en el display físico, cortar
   y restituir alimentación y comprobar persistencia.
5. **Validar ON/OFF** por ambos caminos (Modbus y relé sobre entrada digital) y la
   verificación cruzada, incluyendo el caso de desconectar el RS-485 en caliente.
6. **Ensayo de ruido:** conmutar un contactor real 200 veces en el mismo tablero y medir
   la tasa de error CRC Modbus. Criterio de aceptación: **< 0.1 % de tramas con error y
   cero alarmas espurias**.
7. **Volcar el resultado** a `config/modbus-xr06.yaml` y versionarlo con el número de
   firmware del XR06 ensayado. Un perfil por versión de controlador.

El paso 2 es el que convierte una incógnita de documentación en un dato duro, y es
replicable por el cliente para cualquier otro combistato que quiera incorporar después.
