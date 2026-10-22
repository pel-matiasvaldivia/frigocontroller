# 04 — Arquitectura de firmware

**Plataforma:** ESP-IDF 5.x (C), FreeRTOS, sobre **ESP32-C3 (un solo núcleo RISC-V)**. Se
descarta Arduino-ESP32 para producción: se necesitan particiones OTA A/B, secure boot v2,
flash encryption y control fino de tareas y watchdogs.

> **Rev. 3:** no hay maestro ModBUS (el XR06CX no comunica). La tarea crítica pasa a ser el
> **scan de las tres entradas de detección de CA** y de las sondas. Un solo núcleo alcanza con
> holgura; la garantía de tiempos vive en las prioridades y en reglas de bloqueo explícitas.
> La tarea `modbus_master` se compila solo para la población FC-R (camino B).

## 1. Tareas

| Tarea | Prio | Período | Responsabilidad |
|---|---|---|---|
| `io_scan` | 7 | 10 ms | Muestreo de las **3 entradas de detección de CA** (integración sobre 3 ciclos de red) y de DI1/DI2, antirrebote, conteo de aperturas de puerta, RMS del CT si está presente |
| `state_rebuild` | 6 | 200 ms | Reconstruye la máquina de estados del equipo (enfriando / deshielo / goteo `dt` / retardo de ventiladores `Fd` / parado / apagado) a partir de las 3 salidas y los parámetros del controlador |
| `sensors_ntc` | 6 | 5 s | ADS1115, linealización de NTC (Steinhart-Hart), mediana de 5, aplicación del offset `ot`/`oE`, validación de plausibilidad |
| `modbus_master` | 6 | 1 s | **Solo población FC-R (camino B):** polling por grupos según el perfil, cola de escrituras con read-back, autodetección de velocidad |
| `alarm_engine` | 5 | 1 s | Evaluación de reglas locales, retardos, histéresis, dedup, salida a buzzer y relé |
| `mqtt_client` | 4 | — | Publicación, suscripción a comandos, reenvío del backlog con límite de ancho de banda |
| `sensors_opt` | 4 | 30 s | **Solo si hay accesorios:** SHT45 por I²C (con calentador anticondensación periódico) y RTC |
| `logger` | 3 | evento / 60 s | Ring buffer en flash (registro binario de 24 B), agregados de 15 min para HACCP |
| `net_mgr` | 3 | — | Wi-Fi con backoff exponencial, SNTP, provisioning, (LTE en la variante FC-E LTE) |
| `ota` | 2 | — | Descarga firmada, A/B, rollback automático si no reconecta a MQTT en 10 min |

**Reglas de tiempo real en un solo núcleo** (sustituyen al aislamiento por núcleo de la
rev. 1):

1. `io_scan`, `state_rebuild` y `sensors_ntc` **nunca** llaman a funciones de red, de TLS ni
   de flash;
   publican en colas y siguen. Todo lo que puede bloquear vive en prioridades ≤ 4.
2. Las escrituras en flash se hacen en bloques de un sector, siempre desde `logger`, y no
   suspenden el bus de instrucciones más de 10 ms.
3. La negociación TLS y la descarga OTA ceden explícitamente cada 20 ms (`vTaskDelay`), y
   durante la OTA el scan de entradas **no se suspende nunca**: el equipo no deja de vigilar
   la cámara por estar actualizándose, y menos aún de poder liberar el relé de corte.
4. **El relé de corte tiene watchdog de hardware propio** (temporizador retriggerable): si la
   tarea que lo sostiene deja de refrescarlo, el relé se desenergiza y el XR06 vuelve a estar
   alimentado. El fail-safe no depende de que el firmware se dé cuenta de que falló.
5. Watchdog de tarea habilitado en todas, con `io_scan` como testigo: si se retrasa más de
   500 ms se registra el evento con la tarea culpable — dato de campo para ajustar
   prioridades, no solo un reinicio silencioso.

## 2. Máquina de estados del equipo

```
        ┌──────────┐  credenciales   ┌──────────────┐  conectado   ┌─────────┐
  ──────▶ FACTORY  ├────────────────▶│ PROVISIONING ├─────────────▶│ ONLINE  │
        └──────────┘  (SoftAP/BLE)   └──────────────┘              └──┬───┬──┘
                                            ▲  sin red / 3 reint.    │   │
                                            │                        │   │ pierde red
                                            │                        ▼   │
   botón 10 s ─▶ FACTORY_RESET              └──────────────── OFFLINE_LOGGING
                                             (alarmas locales activas, buffer en flash,
                                              reintentos 5s→10s→30s→60s→300s)
```

El estado del **control de la cámara** no es parte de esta máquina: vive en el XR06. Esa es
la decisión de diseño que mantiene el producto seguro.

## 3. Motor de alarmas local

```c
typedef struct {
    alarm_id_t id;
    bool  (*condition)(const state_t *s);   // evaluación pura sobre el estado
    uint16_t confirm_s;                     // retardo de confirmación
    uint16_t clear_s;                       // retardo de reposición
    float    hysteresis;                    // unidades de ingeniería
    severity_t severity;                    // info | media | alta | critica
    bool  local_siren;                      // dispara relé 2 / buzzer
} alarm_rule_t;
```

Reglas de higiene de notificaciones, para que la app siga siendo creíble:

- **Confirmación e histéresis** en toda alarma analógica (§4 del doc 03).
- **Fuente única de temperatura.** Al evaluar reglas sobre la sonda propia, si el dato tiene
  más de 10 s de antigüedad la regla **no se evalúa** y el período se marca como "sin dato" en
  el registro. Nunca se alarma —ni se deja de alarmar— sobre una lectura vieja.
- **Umbrales heredados del controlador.** `AU`, `AL`, `Ad` y `dA` se cargan en la puesta en
  marcha desde el display del XR06: las alarmas del gateway y las del controlador tienen que
  disparar en el mismo momento, o el cliente ve dos criterios y deja de creerle a uno.
- **Ventana `dA` tras un corte de energía.** El XR06 excluye las alarmas de temperatura
  durante `dA` minutos al encender (90 por defecto). El gateway aplica la misma ventana al
  volver la luz; sin esto, cada corte generaría una ráfaga de alarmas falsas.
- **Exclusión de deshielos y de paradas por puerta** en el análisis de duty cycle y de
  ciclado corto: con `td = in` el compresor funciona durante el deshielo, y con `dC = Fc` se
  para al abrir la puerta. Ninguna de las dos es una falla.
- **Enmascarado durante deshielo:** la alta temperatura se inhibe durante el deshielo y
  durante el tiempo de recuperación configurado (parámetro propio, por defecto `MdF` +
  30 min). Sin esto, cada deshielo genera una falsa alarma — el error clásico.
- **Enmascarado por puerta abierta:** alta temperatura inhibida mientras la puerta esté
  abierta y 10 min después, pero con escalado si la puerta queda abierta > 10 min.
- **Agrupación:** varias alarmas correlacionadas en < 60 s se notifican como un incidente
  único con causa raíz probable (p. ej. `comm_fail` + `probe_fail` → "revisar controlador").
- **Repetición acotada:** recordatorio cada 30 min mientras la alarma crítica no sea
  reconocida, máximo 6; después, escalado al siguiente contacto.
- Toda alarma se **registra siempre**, incluso enmascarada, con el motivo del enmascarado:
  la auditoría necesita el dato crudo.

## 4. Registro y almacenamiento

- Partición dedicada de **1.5 MB** (flash de 4 MB en el ESP32-C3), ring buffer de registros
  binarios de **24 B** (ts u32, ntc1 i16, ntc2 i16, rh i16, flags de las 3 salidas + puerta +
  estado u16, corriente u16, contador de puerta u16, CRC16).
- ≈ 43 700 registros/MB → **> 45 días a 1 muestra/min** con 1.5 MB. Se especifica 30 días
  para dejar margen de desgaste de flash y de agregados.
- Agregados de 15 min (mín/máx/promedio) en partición aparte, **retención 2 años**: es el
  formato que piden las auditorías de cadena de frío y pesa 60× menos.
- Reenvío al reconectar: lotes de 100 registros, con límite de 50 kB/min para no saturar el
  enlace del cliente, intercalado con telemetría en vivo (lo que pasa ahora tiene prioridad
  sobre lo que pasó ayer).
- Escritura con wear leveling y verificación de CRC en lectura; sector corrupto se saltea y
  se reporta.

## 5. Seguridad del dispositivo

| Medida | Implementación |
|---|---|
| Identidad única | Certificado X.509 por dispositivo, clave privada generada **en el chip** y nunca exportada; provisionada en fábrica |
| Transporte | TLS 1.2+ mutuo, puerto 8883, validación de CA y de hostname |
| Arranque | Secure Boot v2 + Flash Encryption; JTAG deshabilitado en producción |
| OTA | Imagen firmada, particiones A/B, rollback automático si no reconecta a MQTT en 10 min; despliegue por canarios (1 % → 10 % → 100 %) |
| Credenciales Wi-Fi | NVS encriptado |
| Provisioning | SoftAP con portal cautivo + BLE; la clave del AP es única por equipo (impresa en la etiqueta y en el QR), nunca una genérica |
| Superficie de ataque | Sin servidor HTTP en operación normal; USB solo diagnóstico de lectura; sin credenciales embebidas en el binario |
| Autorización de comandos | El dispositivo acepta comandos **solo** por el topic `cmd` de su propio ID, firmado por el backend, con `nonce` + ventana de 60 s (anti-replay) |

## 6. Estructura del proyecto propuesta

```
firmware/
├── main/
│   ├── app_main.c
│   ├── state.h / state.c            # estado compartido, un solo escritor por campo
│   └── Kconfig.projbuild
├── components/
│   ├── modbus_profile/              # parser del YAML -> tabla de puntos
│   ├── xr06_driver/                 # polling, escrituras con read-back, autodetección
│   ├── io_drivers/                  # DI con antirrebote, relés, CT RMS
│   ├── sensors/                     # opcional: SHT45 accesorio, RTC, plausibilidad
│   ├── alarm_engine/
│   ├── datalog/                     # ring buffer + agregados
│   ├── cloud/                       # MQTT, backlog, comandos, OTA
│   └── provisioning/
├── test/                            # unit tests host-side del motor de alarmas y escalas
└── simulator/                       # esclavo Modbus XR06 simulado (pytest + pymodbus)
```

**El simulador es parte del entregable**, no un extra: permite desarrollar la app y el
backend sin el hardware, y correr regresión del motor de alarmas en CI inyectando
escenarios (deshielo, puerta abierta, falla de sonda, compresor sin corriente).
