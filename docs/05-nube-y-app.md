# 05 — Nube y aplicación

## 1. Topics MQTT

Namespace: `fc/{tenant}/{site}/{device_id}/...`

| Topic | Dir | QoS | Retain | Contenido |
|---|---|---|---|---|
| `.../tel` | ↑ | 0 | no | Telemetría periódica (60 s) |
| `.../evt` | ↑ | 1 | no | Eventos: alarmas, puerta, deshielo, comandos aplicados, cortes |
| `.../state` | ↑ | 1 | **sí** | Estado completo + configuración; la app lo lee al abrir |
| `.../hist` | ↑ | 1 | no | Lotes del backlog offline |
| `.../cmd` | ↓ | 1 | no | Comandos (`set_setpoint`, `power`, `force_defrost`, `ack_alarm`, `set_config`, `ota`) |
| `.../cmd/ack` | ↑ | 1 | no | Resultado del comando con read-back |
| `.../lwt` | ↑ | 1 | sí | `offline` publicado por el broker al caer la sesión |

**Comando (descendente):**
```json
{ "id":"c-7f3a","ts":"2026-10-21T14:02:00Z","nonce":"9d2b","op":"set_setpoint",
  "args":{"value":-18.5},"by":"user:142" }
```
**ACK (ascendente):**
```json
{ "id":"c-7f3a","result":"applied","readback":-18.5,"applied_at":"2026-10-21T14:02:03Z" }
```
Resultados posibles: `applied` · `rejected_out_of_range` (con `limit`) · `rejected_rate_limit`
(con `retry_after_s`) · `failed_readback` (con el valor real) · `failed_comm`.
**La app nunca muestra un cambio como hecho antes del ACK**; mientras espera, muestra el
valor pendiente en gris con un spinner, y si llega `failed_*` revierte y explica por qué.

## 2. Backend

```
Dispositivos ──MQTT/TLS──▶ EMQX / AWS IoT Core
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
             Ingesta (Go/Node)      Shadow / estado
                    │                     │
          ┌─────────┼─────────┬───────────┴──────┐
          ▼         ▼         ▼                  ▼
    TimescaleDB  Postgres  Motor de reglas   API REST/WS
    (series)    (config,   (escalado,        (app móvil y web)
                 usuarios,  deduplicación,
                 eventos)   guardias)
                              │
                 ┌────────────┼────────────┬─────────────┐
                 ▼            ▼            ▼             ▼
            Push FCM/APNs  WhatsApp      Email      Webhook/ERP
                           (Cloud API)   (+PDF)
```

**Escalado de alarmas (configurable por sitio):** t=0 push a todos los operadores → t=5 min
sin reconocer, WhatsApp al encargado → t=15 min, WhatsApp + llamada al responsable técnico
→ t=30 min, email al dueño. Con **calendario de guardias** y horario comercial: una alarma
a las 3 AM de un sábado tiene que llegar a quien está de guardia, no al grupo.

**Retención:** crudo 1 min por 90 días; agregados de 15 min por 2 años (requisito de
auditoría); eventos y alarmas, 5 años.

## 3. App móvil y web

Flutter (una base, iOS + Android + web). Jerarquía: **Organización → Sitio → Cámara**.

### Pantalla de cámara (la que mira el operador 20 veces al día)

```
┌──────────────────────────────────────────┐
│ ‹ Cámara 2 — Congelados    ● En línea    │
│                                          │
│            −18.4 °C          86 %HR      │   ← dato grande, legible a 2 m
│         consigna −18.0 °C                │
│                                          │
│   [ ❄ ENFRIANDO ]  compresor ● 7.3 A     │   ← chip de estado + confirmación real
│                                          │
│   Puerta  ○ cerrada   ·  23 aperturas hoy│
│   Deshielo  último 04:10 · próximo 10:10 │
│   ───────── 24 h ─────────────────────── │
│   ╱╲    ╱╲    ╱╲   ← curva con bandas de │
│  ╱  ╲__╱  ╲__╱  ╲    alarma sombreadas,  │
│  ▮      ▮       ▮    marcas de deshielo  │
│  └ deshielos      └ aperturas de puerta  │
│                                          │
│   [ Cambiar consigna ]   [ Apagar ]      │
└──────────────────────────────────────────┘
```

Decisiones de UX que importan en este dominio:

1. **Un dato dominante.** La temperatura ocupa la pantalla; el resto es contexto. Se lee
   desde lejos, con guantes, en un pasillo oscuro.
2. **Nunca mentir sobre el estado.** Cada valor muestra su antigüedad ("hace 12 s"). Si el
   equipo está offline, la tarjeta se pone en gris con el último valor conocido y su hora;
   no se muestra un número viejo como si fuera actual. Si el Modbus falla pero la sonda
   propia responde, se indica explícitamente "sin comunicación con el controlador —
   midiendo con sonda propia".
3. **Consigna con fricción deliberada.** Rueda o stepper de 0.5 °C con los límites `LS`/`US`
   del controlador visibles como topes, resumen "de −18.0 a −18.5 °C" y confirmación.
   Se envía al soltar (debounce 1.5 s), no en cada movimiento: protege la EEPROM del XR06.
4. **Apagar es una acción peligrosa.** Confirmación con texto escrito ("APAGAR"), aviso de
   consecuencia ("la cámara dejará de refrigerar") y **recordatorio automático a los 60 min**
   si sigue apagada. En la lista, una cámara apagada se muestra en ámbar, no en gris: apagada
   no es lo mismo que sin novedad.
5. **Deshielo visible, no alarmante.** Chip azul "en deshielo, 12 min" con el contexto de que
   el ascenso de temperatura es normal. Es la pregunta nº 1 de soporte y se responde con un
   chip.
6. **Centro de alarmas** con severidad por color + icono + texto (no solo color: hay
   operarios daltónicos), reconocimiento con nota, y **causa probable y qué revisar** en cada
   alarma. `compressor_no_current` no dice "sin corriente": dice "el controlador pidió
   compresor y no hay consumo — revisar térmico, fases y contactor".
7. **Reportes HACCP** en un toque: PDF/CSV por rango de fechas con mín/máx/promedio,
   tiempo fuera de rango, lista de aperturas de puerta, deshielos y alarmas con su
   reconocimiento y la firma del operador. Programable por email semanal.
8. **Widget y notificaciones críticas**: canal de notificación de alta prioridad que suena
   con el teléfono en silencio (crítica únicamente), más widget de pantalla de inicio con
   las temperaturas de todas las cámaras.
9. **Modo instalador** (QR del equipo): asistente de puesta en marcha, prueba de relés y
   entradas, lectura cruda de Modbus, nivel de señal Wi-Fi con semáforo para decidir dónde
   poner la antena, y test de alarma de punta a punta antes de firmar la instalación.

### Roles

| Rol | Ve | Comanda | Configura |
|---|---|---|---|
| Operador | su sitio | reconoce alarmas | — |
| Encargado | su sitio | consigna, ON/OFF | umbrales, contactos |
| Técnico / instalador | sitios asignados | todo + forzar deshielo | parámetros del XR06, E/S, perfil Modbus |
| Administrador | toda la organización | todo | usuarios, guardias, facturación |

Toda escritura queda en **bitácora de auditoría** (quién, cuándo, de qué valor a qué valor,
desde qué IP). En un reclamo por producto perdido, esa bitácora es el entregable que
protege al cliente.
