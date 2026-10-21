# 04 — Arquitectura de firmware

**Plataforma:** ESP-IDF 5.x (C), FreeRTOS. Se descarta Arduino-ESP32 para producción:
se necesitan particiones OTA A/B, secure boot v2, flash encryption y control fino de
tareas y watchdogs.

## 1. Tareas

| Tarea | Núcleo | Prio | Período | Responsabilidad |
|---|---|---|---|---|
| `modbus_master` | 1 | 6 | 1 s | Polling por grupos según el perfil YAML, cola de escrituras con read-back, autodetección de baudrate, contador de errores CRC |
| `io_scan` | 1 | 7 | 10 ms | Muestreo de DI, antirrebote, conteo de aperturas de puerta, RMS del CT (ventana de 1 s) |
| `sensors` | 0 | 4 | 5 s | SHT45 (con calentador periódico anticondensación), RTC, filtro mediana-de-5 |
| `alarm_engine` | 0 | 5 | 1 s | Evaluación de reglas locales, retardos, histéresis, dedup, salida a buzzer/relé 2 |
| `logger` | 0 | 3 | evento / 60 s | Ring buffer en flash (registro binario de 32 B), agregados de 15 min para HACCP |
| `net_mgr` | 0 | 3 | — | Wi-Fi con backoff exponencial, SNTP, provisioning, (LTE en FC-300) |
| `mqtt_client` | 0 | 4 | — | Publicación, suscripción a comandos, reenvío del backlog con límite de ancho de banda |
| `ota` | 0 | 2 | — | Descarga firmada, A/B, rollback automático si no hay MQTT a los 10 min |

Watchdog de tarea habilitado en todas; `modbus_master` e `io_scan` aisladas en el núcleo 1
para que una tormenta de red nunca retrase la detección de una apertura de puerta.

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

- Partición dedicada de 3 MB, ring buffer de registros binarios de 32 B
  (ts u32, temp_xr06 i16, temp_evap i16, temp_own i16, rh i16, setpoint i16, flags u16,
  corriente u16, contador de puerta u16, CRC16).
- ≈ 95 000 registros/MB → **> 60 días a 1 muestra/min** con 3 MB. Se especifica 30 días
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
│   ├── sensors/                     # SHT45, DS3231, plausibilidad
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
