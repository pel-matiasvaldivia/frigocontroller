# 02 — Diseño de hardware (revisión 2: optimizado en costo)

> **Rev. 2.** La rev. 1 usaba sonda T/HR propia, RS-485 aislado integrado, ESP32-S3, 2 relés,
> 4 entradas y transformador de corriente: USD ~64/u. a 500 u. **más** el módulo XJ485
> (USD 35-55/punto). Esta revisión toma todos los datos de proceso del XR06 por Modbus,
> conecta directo al puerto TTL del controlador y baja a **USD ~25/u. a 500 u. instalado**.
> El registro de decisiones y lo que se cede en cada recorte está en
> [09-optimizacion-de-costos.md](09-optimizacion-de-costos.md).

## 1. Diagrama de bloques

```
  100–240 VAC ──[F 500mA T]──[MOV 275V]──┬─▶ AC/DC 5 V 3 W (HLK-PM05)
   ó 12–24 VDC del tablero ──[TVS]───────┘        │
                                                  ├─▶ Buck 3V3 (MP2451) ─▶ ESP32-C3
                                                  ├─▶ 5 V ─▶ bobina relé, wetting DI
                                                  └─▶ supercap 0.47 F ─▶ RTC + flush de log
 ┌───────────────────── ESP32-C3-MINI-1U-N4 (antena externa) ─────────────────────┐
 │ UART0 ─▶ aislador digital ADuM1201 ─▶ cable TTL ─▶ puerto TTL del XR06         │
 │        └ (población alternativa: SP3485 ─▶ bornera A/B ─▶ XJ485/bus RS-485)    │
 │ I²C   ─▶ RTC PCF85063A  ·  conector de 4 p para SONDA T/HR OPCIONAL            │
 │ GPIO  ─▶ 2× DI opto-aisladas (NA/NC por software)                              │
 │ GPIO  ─▶ 1× relé SPDT 16 A  (población alternativa: 2º relé)                   │
 │ GPIO  ─▶ LED RGB · buzzer · pulsador        ADC ─▶ entrada CT opcional          │
 │ UART/USB ─▶ pads de servicio (sin conector: se flashea en cama de agujas)      │
 └────────────────────────────────────────────────────────────────────────────────┘
```

**Una PCB, cuatro poblaciones.** El mismo circuito impreso cubre FC-E (TTL directo, Wi-Fi,
1 relé), FC-R (RS-485 para bus/XJ485 o otros combistatos), y las opciones de sonda HR y CT.
Un solo juego de herramientas, un solo stock de PCB, cuatro SKU. Es la decisión de costo
más rentable del proyecto después de eliminar el XJ485.

## 2. Componentes: qué cambió y por qué

| Bloque | Rev. 1 | **Rev. 2** | Ahorro @500 | Qué se cede |
|---|---|---|---|---|
| MCU + Wi-Fi | ESP32-S3-WROOM-1U (2 núcleos, 8 MB, PSRAM) | **ESP32-C3-MINI-1U-N4** (1 núcleo RISC-V, 4 MB) | 1.50 | Un solo núcleo. Suficiente: el polling Modbus es 1 trama/s y MQTT es ocioso. Se compensa con prioridades de tarea y watchdog (doc 04 §1). Sigue siendo módulo pre-certificado |
| Interfaz al controlador | ADM2587E (RS-485 aislado) **+ XJ485 externo** | **ADuM1201 + DC/DC 1 W + cable TTL armado** | 3.65 **+ 45.00 del XJ485** | Pinout del puerto TTL a relevar en banco; conexión no soportada por el fabricante. **Es el riesgo que paga todo el ahorro** — ver §4 |
| Sonda T/HR | SHT45 + cápsula IP67 + 3 m FEP (fija) | **Eliminada. Temperatura por Modbus. Conector de 4 p para sonda opcional** | 10.60 | **Sin humedad** salvo accesorio, y sin verificación cruzada de sonda (se pierde la alarma `probe_mismatch`). Ver §5 |
| Confirmación de compresor | CT clamp fijo | **Bit del relé del XR06 por Modbus. CT como accesorio** | 2.60 | Se pierde "pidió compresor y no hay corriente" y "contactor pegado" salvo accesorio |
| Entradas digitales | 4 × TLP291-4 | **2 × PC817** (puerta + 1 libre) | 0.17 | Menos contactos auxiliares; el estado de la entrada de puerta del XR06 también llega por Modbus |
| Salidas | 2 × relé 16 A | **1 × relé 16 A** (ON/OFF). 2º relé como población | 1.48 | La sirena externa pasa a opción; el buzzer interno queda de serie |
| Fuente | Mornsun LS03 | **HLK-PM05** (ó entrada 12-24 VDC, aún más barata) | 2.40 | Marca de menor gama: exige ensayo de seguridad propio y 100 % de burn-in en producción |
| RTC | DS3231SN (compensado) | **PCF85063A + supercap** | 0.93 | Deriva ~±2 min/mes sin red (vs. ±1 min/año). Irrelevante: se corrige por SNTP en cada reconexión |
| Borneras | Enchufables | **A tornillo, fijas** | 1.60 | +5 min de servicio si hay que bajar el equipo |
| PCB | 4 capas, 100×80 | **2 capas, 75×55** | 1.75 | Layout más cuidadoso; sin plano dedicado. Obliga disciplina de ruteo (§3) |
| Gabinete | DIN 6M custom serigrafiado | **DIN 4M genérico + etiqueta impresa** | 1.80 | Menos identidad de marca en el frente; sin molde propio (y **sin USD 8-15 k de herramienta**) |
| Antena | Dipolo + pigtail | **Sin cambio** | — | **No se recorta.** Es el componente que decide si el equipo funciona en obra |

**Lo que NO se recortó, a propósito:** aislación galvánica en la interfaz al controlador,
antena externa, protecciones en las borneras de campo y RTC. Cada uno de los cuatro es
causa directa de devolución en campo; ahorrar USD 2 ahí cuesta USD 150 de visita técnica.

## 3. Reglas de layout en 2 capas

Pasar de 4 a 2 capas ahorra dinero y se paga con disciplina:

1. **Capa inferior reservada como plano de masa continuo** en la zona de lógica; el ruteo
   cruza por la capa superior. No se permite partir el plano bajo el módulo Wi-Fi ni bajo el
   cristal.
2. **Tres zonas con barrera física:** red/relé · lógica · lado aislado del ADuM1201, con
   ranura (slot) fresada en cada barrera de aislación.
3. **Creepage/clearance ≥ 3.0 mm** red↔lógica (IEC 60664-1, grado de polución 2), ≥ 5.5 mm
   bajo el relé y la fuente.
4. **TVS y PTC en todas las borneras de campo**; serie de 100 Ω en las líneas TTL y
   TVS de baja capacidad, porque el cable al XR06 corre junto a potencia dentro del tablero.
5. Módulo variante **U** con pigtail IPEX→SMA al frente. Nunca antena de PCB dentro de un
   tablero metálico.
6. Fuente AC/DC y relé lejos del módulo de radio; retorno de la bobina del relé por una
   pista propia al borne de la fuente, no por el plano de señal.
7. Pads de test para cama de agujas: 3V3, 5V, TX/RX, bobina, DI. **Sin conector USB**: se
   flashea y testea por los pads en producción (ahorra el conector y un mecanizado).

## 4. El riesgo del TTL directo, y cómo se cubre

El puerto TTL del XR06 es el mismo que usa la Hot Key de copia de parámetros. No está
documentado como interfaz de usuario y la conexión directa no cuenta con soporte del
fabricante.

| Mitigación | Detalle |
|---|---|
| Aislación galvánica | ADuM1201 + DC/DC de 1 W: el XR06 nunca comparte masa con la red del gateway ni con el cable de campo |
| Limitación de corriente | Serie de 100 Ω y clamp a 5 V en cada línea: una falla del gateway no puede inyectar energía al controlador |
| Relevamiento en banco | Pinout, niveles y consumo medidos en fase 0 con un XR06 de sacrificio, antes de cualquier instalación en cliente |
| **Camino de salida** | Si el relevamiento o el cliente lo desaconsejan, **la misma PCB se puebla como FC-R** (SP3485, +USD 0.90) y se usa el XJ485 oficial. Cero rediseño, cero herramienta perdida |
| Acuerdo comercial | La conexión al puerto TTL se documenta y se acuerda por escrito con el cliente antes del piloto (efecto sobre la garantía del XR06) |

Que el plan B sea una población alternativa de la misma placa es lo que permite apostar al
camino económico sin arriesgar el proyecto.

## 5. Consecuencia de eliminar los sensores propios — requiere decisión del cliente

| Requerimiento original | Estado en rev. 2 |
|---|---|
| Temperatura | **Cubierto por Modbus** (sondas P1/P2 del XR06). Mejor incluso: es *la misma* temperatura que ve el controlador, así que la app y el display nunca se contradicen |
| **Humedad** | **NO cubierto.** El XR06 no tiene sensor de HR ni registro donde leerla. Sin sensor propio, la humedad sale del alcance |
| Verificación cruzada de sonda | Se pierde la alarma predictiva `probe_mismatch` (hielo o deriva en la NTC de regulación). Queda a cargo del mantenimiento preventivo |
| Medición en zona de producto | La sonda del XR06 está en el retorno de aire del evaporador, no junto a la mercadería. Si una auditoría exige medición en la zona de producto, hace falta la sonda accesorio |
| Operación con el serie caído | Rev. 1 seguía midiendo y alarmando sin Modbus. **Rev. 2, sin Modbus, se queda sin dato de temperatura**: solo alarma `comm_fail` y lo que llegue por las entradas digitales |

**Opciones para la humedad, por costo:**

| Opción | Costo | Cuándo |
|---|---|---|
| **A. Fuera de alcance** | 0 | Si el cliente solo necesita cadena de frío de congelados, donde la HR no se controla ni se audita. **Requiere que lo acepte por escrito: lo pidió explícitamente** |
| **B. Accesorio (recomendada)** | +USD 0.10 en el equipo (conector de 4 p) y +USD 7-9 de BOM solo en las unidades que lo lleven | Se cotiza la sonda T/HR únicamente en cámaras de frescos, maduración, o donde haya auditoría. Es la salida económica que **no pierde el requerimiento** |
| **C. Sensor de bajo costo a bordo** | +USD 1.20 (AHT20/SHT30) | Descartada: dentro del gabinete, en el tablero, mide la humedad del tablero y no la de la cámara. **Un dato falso es peor que ningún dato** |

La opción B deja la decisión en la mano comercial, sin castigar el costo del equipo base.

## 6. Asignación de E/S (rev. 2)

| Canal | Función por defecto | Notas |
|---|---|---|
| TTL / A-B | Interfaz al XR06 | Población FC-E (TTL) o FC-R (RS-485) |
| DI1 | Fin de carrera de puerta | Solo necesario si el contacto **no** está cableado al XR06; si lo está, el estado llega por Modbus |
| DI2 | Libre: contacto de alarma del XR06, térmico, presostato o monitor de fase | Respaldo que funciona aun con el Modbus caído |
| RELÉ 1 | ON/OFF remoto → entrada digital del XR06 | Contacto **NC**: relé sin energía = cámara encendida |
| Conector sonda | Sonda T/HR opcional (I²C) o sonda de producto DS18B20 | Sin poblar en el equipo base |
| ADC | CT de compresor opcional | Sin poblar en el equipo base |

**Fail-safe sin cambios:** relé desenergizado = cámara refrigerando.

## 7. Especificación eléctrica (rev. 2)

| Parámetro | Valor |
|---|---|
| Alimentación | 100–264 VAC 50/60 Hz, ó **12–24 VDC** (variante más barata si el tablero ya tiene fuente) |
| Consumo | < 1.0 W reposo, < 2.2 W con relé y Wi-Fi TX |
| Temperatura de operación | −20 a +60 °C |
| Aislación interfaz del controlador ↔ lógica | 2.5 kV RMS (ADuM1201) |
| Aislación red/relé ↔ lógica | 3 kV |
| Retención de datos sin red | ≥ 30 días a 1 muestra/min (ring buffer en 1.5 MB de la flash de 4 MB) |
| Respaldo de reloj | 72 h (supercap) + resincronización SNTP |
