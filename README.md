# FrigoController — Gateway IoT para cámaras frigoríficas con controlador Copeland/Dixell XR06

Dossier de diseño de producto: electrónica, mecánica, firmware, protocolo y app para
monitoreo y control remoto de equipos de refrigeración comandados por un combistato
**Copeland (Dixell) XR06CX**.

> **Nota de marca:** la línea XR (XR02/XR03/XR06CX) es de Dixell, hoy parte de
> Emerson / Copeland. En este dossier se la nombra **XR06** indistintamente.

## 1. Qué resuelve

| Requerimiento del cliente | Cómo se implementa |
|---|---|
| ON / OFF remoto | Comando Modbus (bit stand-by) **+ camino redundante** por relé sobre entrada digital del XR06 configurada como ON/OFF |
| Setup de temperatura | Escritura del registro `SET` por Modbus, con límites `LS`/`US` y rate-limit anti-desgaste de EEPROM |
| Alarmas de alta y baja temperatura | Lectura de flags `AH1`/`AL1` del XR06 + motor de alarmas propio redundante sobre sonda propia |
| Alerta de error de comunicación | Watchdog de polling Modbus (3 timeouts) + LWT MQTT para detectar caída del gateway |
| Alertas por falla de sondas | Flags `P1`/`P2` del XR06 + validación de plausibilidad de la sonda propia |
| Registro de aperturas de puerta | Entrada digital propia (fin de carrera / reed) con antirrebote, contador y tiempo acumulado por día |
| Indicación de deshielo en la App | Bit de estado de deshielo + fases (deshielo / goteo / retardo de ventiladores) y duración |
| Indicación de funcionamiento del compresor | Bit de relé de compresor + **confirmación real por transformador de corriente** (detecta contactor pegado o no actuando) |
| Humedad | Sensor propio SHT45 (el XR06 no mide humedad) |

## 2. Arquitectura en una pantalla

```
   CÁMARA FRIGORÍFICA                 TABLERO                      NUBE              USUARIO
 ┌────────────────────┐        ┌──────────────────────┐
 │ Evaporador         │        │  XR06CX (combistato) │
 │ Sonda NTC (XR06) ──┼────────┤  control + seguridad │
 │ Fin de carrera     │        │  puerto TTL          │
 │  puerta ───────────┼──┐     └──────┬───────────────┘
 │ Sonda T/HR propia  │  │            │ TTL ↔ RS-485 (XJ485)
 │  (SHT45, IP67) ────┼─┐│            │
 └────────────────────┘ ││     ┌──────┴───────────────┐      MQTT/TLS      ┌─────────┐    ┌──────────┐
                        │└─────┤ FRIGOCONTROLLER GW   ├───── Wi-Fi 2.4G ──▶│ Broker  │───▶│ App móvil│
                        └──────┤ ESP32-S3 + RS485 iso │     (LTE opcional) │ + TSDB  │    │ + Web    │
                               │ 4 DI (NA/NC) 2 RELÉ  │                    │ + Reglas│    │ Push/WA  │
                               └──────────────────────┘                    └─────────┘    └──────────┘
```

**Principio de diseño rector:** el XR06 conserva *siempre* la autoridad de control y de
seguridad. El gateway es un **supervisor y comandante de consigna**, nunca un lazo de
control crítico. Todo falla en modo seguro: si el gateway se apaga, se cuelga o pierde
red, la cámara sigue refrigerando con la última configuración válida.

## 3. Documentos

| Doc | Contenido |
|---|---|
| [docs/01-arquitectura.md](docs/01-arquitectura.md) | Arquitectura de sistema, variantes de producto, modos de falla |
| [docs/02-hardware.md](docs/02-hardware.md) | Diseño electrónico: ESP32-S3, RS-485 aislado, DI opto, relés, fuente, protecciones |
| [docs/03-modbus-xr06.md](docs/03-modbus-xr06.md) | Interfaz con el XR06: cableado TTL/RS-485, mapa de registros, procedimiento de descubrimiento |
| [docs/04-firmware.md](docs/04-firmware.md) | Tareas FreeRTOS, motor de alarmas, buffer offline, OTA, seguridad |
| [docs/05-nube-y-app.md](docs/05-nube-y-app.md) | Topics MQTT, payloads, backend, UX de la app, reportes HACCP |
| [docs/06-diseno-industrial.md](docs/06-diseno-industrial.md) | Gabinete, sonda, instalación, serigrafía, LEDs, usabilidad en obra |
| [docs/07-bom-y-costos.md](docs/07-bom-y-costos.md) | BOM con alternativas, costeo a 100/500/2000 u., modelo de precio |
| [docs/08-normativa-y-plan.md](docs/08-normativa-y-plan.md) | Normativa AR/CE, ensayos, roadmap y plan de validación |
| [config/modbus-xr06.yaml](config/modbus-xr06.yaml) | Mapa de registros **data-driven** (editable sin recompilar firmware) |
| [hardware/diagrama-conexionado.md](hardware/diagrama-conexionado.md) | Diagrama de borneras y conexionado de campo |

## 4. Variantes de producto

| | **FC-100 Panel** | **FC-200 IP65** | **FC-300 LTE** |
|---|---|---|---|
| Montaje | Riel DIN 6M, dentro del tablero | Caja IP65 a pared | IP65 + módem |
| Conectividad | Wi-Fi 2.4 GHz | Wi-Fi 2.4 GHz | Wi-Fi + LTE Cat-1 |
| Caso de uso | Instalación nueva, tablero con espacio | Retrofit, tablero lleno | Sitio sin Wi-Fi confiable |

## 5. Estado

Dossier de diseño. Los valores marcados **`[VERIFICAR]`** requieren confirmación contra
el manual del firmware específico del XR06 instalado y/o banco de pruebas — están
listados en [docs/03-modbus-xr06.md](docs/03-modbus-xr06.md#5-procedimiento-de-descubrimiento-y-validación).
