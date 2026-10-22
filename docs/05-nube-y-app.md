# 05 — Nube y aplicación

## 1. Topics MQTT

Namespace: `fc/{tenant}/{site}/{device_id}/...`

| Topic | Dir | QoS | Retain | Contenido |
|---|---|---|---|---|
| `.../tel` | ↑ | 0 | no | Telemetría periódica (60 s) |
| `.../evt` | ↑ | 1 | no | Eventos: alarmas, puerta, deshielo, comandos aplicados, cortes |
| `.../state` | ↑ | 1 | **sí** | Estado completo + configuración; la app lo lee al abrir |
| `.../hist` | ↑ | 1 | no | Lotes del backlog offline |
| `.../cmd` | ↓ | 1 | no | Comandos (`power`, `ack_alarm`, `set_config`, `ota`; **`set_setpoint` y `force_defrost` solo en el camino B**) |
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
Resultados posibles: `applied` · `rejected_unsupported` (**camino A: la consigna no es
escribible**) · `rejected_out_of_range` (con `limit`) · `rejected_rate_limit` (con
`retry_after_s`) · `failed_readback` (con el estado real observado en las salidas) ·
`failed_comm`.
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

La humedad aparece como segundo dato grande **solo si la cámara tiene la sonda accesorio
instalada**; en el equipo base, ese espacio lo ocupa la temperatura de evaporador. La app
no muestra un recuadro de humedad vacío ni un guion: una métrica que el equipo no puede
medir simplemente no existe en la interfaz de esa cámara.

```
┌──────────────────────────────────────────┐
│ ‹ Cámara 2 — Congelados    ● En línea    │
│                                          │
│            −18.4 °C                      │   ← sonda propia del gateway, legible a 2 m
│         evaporador −24.1 °C              │   ← sonda NTC2 propia, si se cotizó (+P2)
│         consigna −18.0 °C  ⓘ             │   ← cargada en la puesta en marcha; la ⓘ
│                                          │      explica que no es editable (nivel 1)
│                                          │
│   [ ❄ ENFRIANDO ]  compresor ● en marcha │   ← tensión en la salida de relé del XR06
│                                          │      (con +CT se agrega el consumo en A)
│                                          │
│   Puerta  ○ cerrada   ·  23 aperturas hoy│   ← entrada digital del XR06, o DI1 propia
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
   no se muestra un número viejo como si fuera actual.
   **Caso propio de la rev. 3:** si la sonda propia falla, no hay temperatura que mostrar. La
   tarjeta pasa a ámbar con *"sonda en falla — última lectura −18.4 °C hace 6 min"*, el número
   tachado y el botón de diagnóstico. El vacío de dato se hace visible, no se disimula con el
   último valor: es la diferencia entre un operador que va a mirar la cámara y uno que se
   queda tranquilo sin motivo.
3. **La consigna se muestra, pero en el nivel 1 no se edita.** El XR06CX no tiene canal de
   escritura: el control de consigna aparece **deshabilitado con una explicación en una línea**
   ("tu controlador no admite cambio remoto de consigna") y un enlace a la oferta de upgrade.
   Lo que **no** se hace: esconder la función, ni mostrar un control que falle al tocarlo.
   En el nivel 2 (camino B) el mismo control se habilita solo, con rueda de 0.5 °C, los
   límites `LS`/`uS` visibles como topes, confirmación y envío al soltar (debounce 1.5 s) para
   proteger la EEPROM del controlador.
4. **Apagar es una acción peligrosa.** Confirmación con texto escrito ("APAGAR"), aviso de
   consecuencia ("la cámara dejará de refrigerar") y **recordatorio automático a los 60 min**
   si sigue apagada. En la lista, una cámara apagada se muestra en ámbar, no en gris: apagada
   no es lo mismo que sin novedad.
5. **Deshielo visible, no alarmante.** Chip azul "en deshielo, 12 min · próximo a las 10:10"
   con el contexto de que el ascenso de temperatura es normal, y las fases de goteo (`dt`) y
   retardo de ventiladores (`Fd`) que el gateway reconstruye de las tres salidas. Es la
   pregunta nº 1 de soporte y se responde con un chip.
   **Detalle que evita el llamado nº 2:** con `dF = it` (valor por defecto) el display del
   XR06 **congela** la temperatura de inicio durante el deshielo, mientras la app muestra la
   subida real. El chip incluye la nota *"el display del controlador muestra un valor fijo
   durante el deshielo: no es un error"*.
6. **Centro de alarmas** con severidad por color + icono + texto (no solo color: hay
   operarios daltónicos), reconocimiento con nota, y **causa probable y qué revisar** en cada
   alarma. `compressor_no_current` no dice "sin corriente": dice "el controlador pidió
   compresor y no hay consumo — revisar térmico, fases y contactor".
7. **Alarma de sonda inferida, explicada.** `probe_fail_inferred` no dice "falla de sonda":
   dice *"el controlador está ciclando por tiempo (15 min on / 30 min off) sin relación con la
   temperatura — probable falla de la sonda del controlador. Revisar P1 en el display"*, con
   el gráfico de los ciclos detectados al lado. La inferencia se muestra con su evidencia, no
   como un veredicto.
8. **Reportes HACCP** en un toque: PDF/CSV por rango de fechas con mín/máx/promedio,
   tiempo fuera de rango, lista de aperturas de puerta, deshielos y alarmas con su
   reconocimiento y la firma del operador. Programable por email semanal.
9. **Widget y notificaciones críticas**: canal de notificación de alta prioridad que suena
   con el teléfono en silencio (crítica únicamente), más widget de pantalla de inicio con
   las temperaturas de todas las cámaras.
10. **Modo instalador** (QR del equipo): asistente de puesta en marcha (doc 06 §9), carga de
    los parámetros del controlador, comparación con el display, **prueba de las tres
    derivaciones con un deshielo manual**, prueba del relé de corte, nivel de señal Wi-Fi con
    semáforo para ubicar la antena, y test de alarma de punta a punta antes de firmar.
    Incluye el **recordatorio de revalidar los parámetros cada 6 meses**, porque un cambio en
    el frente del controlador el gateway no lo ve.

### Roles

| Rol | Ve | Comanda | Configura |
|---|---|---|---|
| Operador | su sitio | reconoce alarmas | — |
| Encargado | su sitio | consigna, ON/OFF | umbrales, contactos |
| Técnico / instalador | sitios asignados | todo | parámetros del XR06 (carga manual), E/S, perfil del controlador |
| Administrador | toda la organización | todo | usuarios, guardias, facturación |

Toda escritura queda en **bitácora de auditoría** (quién, cuándo, de qué valor a qué valor,
desde qué IP). En un reclamo por producto perdido, esa bitácora es el entregable que
protege al cliente.
