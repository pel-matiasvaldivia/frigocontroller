# FrigoController — Gateway IoT para cámaras frigoríficas con controlador Copeland/Dixell XR06

Dossier de diseño de producto: electrónica, mecánica, firmware, protocolo y app para
monitoreo y control remoto de equipos de refrigeración comandados por un combistato
**Copeland (Dixell) XR06CX**.

> **Rev. 2 — diseño optimizado en costo.** Todos los datos de proceso se toman del XR06 por
> Modbus (sin sensores propios) y el gateway se conecta **directo al puerto TTL** del
> controlador, sin el módulo XJ485. Costo instalado por punto: **USD 24.79 @500 u.**
> (rev. 1: USD 108.88). Ver [docs/09-optimizacion-de-costos.md](docs/09-optimizacion-de-costos.md).

> **Nota de marca:** la línea XR (XR02/XR03/XR06CX) es de Dixell, hoy parte de
> Emerson / Copeland. En este dossier se la nombra **XR06** indistintamente.

## 1. Qué resuelve

| Requerimiento del cliente | Cómo se implementa | Fuente |
|---|---|---|
| ON / OFF remoto | Comando Modbus (bit stand-by) **+ camino redundante** por relé sobre entrada digital del XR06 | Modbus + relé |
| Setup de temperatura | Escritura del registro `SET`, con clamp a `LS`/`US` y rate-limit anti-desgaste de EEPROM | Modbus |
| Alarmas de alta y baja temperatura | Flags `AH1`/`AL1` del XR06, con retardo, histéresis y enmascarado por deshielo y puerta | Modbus |
| Alerta de error de comunicación | Watchdog de polling (3 timeouts) + LWT MQTT para detectar caída del gateway | Gateway + nube |
| Alertas por falla de sondas | Flags `P1`/`P2` del XR06 | Modbus |
| Registro de aperturas de puerta | Estado de la entrada digital del XR06 **ó** entrada digital propia si el contacto no está cableado al controlador | Modbus / DI1 |
| Indicación de deshielo en la App | Bit de estado + fases (deshielo / goteo / retardo de ventiladores) y duración | Modbus |
| Indicación de funcionamiento del compresor | Bit del relé de compresor (opcional: clamp de corriente para confirmar marcha real) | Modbus (+CT opc.) |
| **Humedad** | **El XR06 no mide HR: no existe registro Modbus que leer.** Se cubre con la **sonda T/HR accesorio** solo en las cámaras que la requieran (+USD 7.80 de BOM, conector ya previsto en la PCB). **Requiere decisión del cliente** — ver [docs/02 §5](docs/02-hardware.md#5-consecuencia-de-eliminar-los-sensores-propios--requiere-decisión-del-cliente) | Accesorio |

## 2. Arquitectura en una pantalla

```
   CÁMARA FRIGORÍFICA              TABLERO                        NUBE              USUARIO
 ┌────────────────────┐    ┌──────────────────────────┐
 │ Evaporador         │    │  XR06CX (combistato)     │
 │ Sonda NTC P1 ──────┼────┤  control + seguridad     │
 │ Sonda NTC P2 ──────┼────┤  puerto TTL ─────┐       │
 │ Fin de carrera ────┼────┤  entrada digital │       │
 │  de puerta         │    └──────────────────┼───────┘
 └────────────────────┘                       │ TTL aislado (ADuM1201), 1 m
                                              │ (opción: RS-485 + XJ485)
                              ┌───────────────┴──────────┐   MQTT/TLS    ┌─────────┐   ┌──────────┐
                              │ FRIGOCONTROLLER  FC-E    ├── Wi-Fi 2.4 ─▶│ Broker  │──▶│ App móvil│
                              │ ESP32-C3 · 2 DI · 1 relé │               │ + TSDB  │   │ + Web    │
                              │ [ conector sonda opc. ]  │               │ + Reglas│   │ Push/WA  │
                              └──────────────────────────┘               └─────────┘   └──────────┘
```

**Principio de diseño rector:** el XR06 conserva *siempre* la autoridad de control y de
seguridad. El gateway es un **supervisor y comandante de consigna**, nunca un lazo de
control crítico. Todo falla en modo seguro: si el gateway se apaga, se cuelga o pierde
red, la cámara sigue refrigerando con la última configuración válida.

## 3. Documentos

| Doc | Contenido |
|---|---|
| [docs/01-arquitectura.md](docs/01-arquitectura.md) | Arquitectura de sistema, variantes, modos de falla |
| [docs/02-hardware.md](docs/02-hardware.md) | **Electrónica rev. 2:** ESP32-C3, interfaz TTL aislada, 2 DI, 1 relé, una PCB con 4 poblaciones |
| [docs/03-modbus-xr06.md](docs/03-modbus-xr06.md) | Interfaz con el XR06: capa física, mapa de registros, procedimiento de descubrimiento |
| [docs/04-firmware.md](docs/04-firmware.md) | Tareas, motor de alarmas, buffer offline, OTA, seguridad |
| [docs/05-nube-y-app.md](docs/05-nube-y-app.md) | Topics MQTT, payloads, backend, UX de la app, reportes HACCP |
| [docs/06-diseno-industrial.md](docs/06-diseno-industrial.md) | Gabinete, instalación, serigrafía, LEDs, usabilidad en obra |
| [docs/07-bom-y-costos.md](docs/07-bom-y-costos.md) | **BOM rev. 2**, poblaciones y accesorios, costeo y precio |
| [docs/08-normativa-y-plan.md](docs/08-normativa-y-plan.md) | Normativa AR/CE, ensayos, roadmap, riesgos, relevamiento previo |
| [docs/09-optimizacion-de-costos.md](docs/09-optimizacion-de-costos.md) | **Registro de recortes:** qué se ahorró, qué se cede, qué se rechazó y por qué |
| [config/modbus-xr06.yaml](config/modbus-xr06.yaml) | Mapa de registros **data-driven** (editable sin recompilar firmware) |
| [hardware/diagrama-conexionado.md](hardware/diagrama-conexionado.md) | Diagrama de borneras y conexionado de campo |

## 4. Variantes (una sola PCB, cuatro poblaciones)

| | **FC-E** (base) | **FC-R** | **FC-E IP65** | **FC-E LTE** |
|---|---|---|---|---|
| Interfaz al XR06 | Cable TTL directo | RS-485 (XJ485 o bus existente) | TTL o RS-485 | TTL o RS-485 |
| Montaje | Riel DIN 4M | Riel DIN 4M | Caja IP65 a pared | Caja IP65 |
| Conectividad | Wi-Fi 2.4 GHz | Wi-Fi 2.4 GHz | Wi-Fi 2.4 GHz | Wi-Fi + LTE Cat-1 |
| Costo @500 u. | USD 24.79 | +0.90 (+ XJ485) | +9.50 | +28.00 |
| Caso de uso | Instalación estándar | Cliente que exige soporte oficial, u otro combistato | Zona de lavado, montaje a la vista | Sitio sin Wi-Fi |

Accesorios comunes a todas: **+HR** (sonda T/HR), **+CT** (clamp de compresor), **+2R**
(segundo relé), **DC-in** (alimentación 12-24 V, −USD 1.70).

## 5. Estado y próximo paso

Dossier de diseño. Hay **dos incógnitas que se cierran en la misma sesión de banco** y de
las que depende todo el ahorro de la rev. 2:

1. El **mapa de registros Modbus** del firmware específico del XR06 — todos los valores
   marcados `[VERIFICAR]` en [docs/03](docs/03-modbus-xr06.md#5-procedimiento-de-descubrimiento-y-validación).
2. El **pinout y los niveles del puerto TTL** del XR06, para la conexión directa — ver
   [docs/02 §4](docs/02-hardware.md#4-el-riesgo-del-ttl-directo-y-cómo-se-cubre).

Fase 0 del roadmap: 2 semanas, un XR06 de sacrificio y menos de USD 150 en material de
estante ([docs/09 §3](docs/09-optimizacion-de-costos.md#3-vía-de-prototipado-sin-herramienta-fases-0-y-1)).
**No se compra herramienta ni se lanza PCB hasta cerrarlas.**
