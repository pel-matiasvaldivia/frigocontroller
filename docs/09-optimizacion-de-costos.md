# 09 — Registro de optimización de costos (rev. 1 → rev. 2)

Qué se recortó, cuánto ahorró y qué se cede. Un recorte sin su contrapartida explícita no
es una decisión de ingeniería: es una sorpresa diferida.

## 1. Resumen

| | Rev. 1 | **Rev. 2** | Δ |
|---|---|---|---|
| BOM + ensamblado @500 u. | USD 63.88 | **USD 24.79** | −61 % |
| Interfaz al XR06 (XJ485) | USD 45.00 | **USD 0** (cable TTL incluido) | −45.00 |
| **Costo instalado por punto @500 u.** | **USD 108.88** | **USD 24.79** | **−77 %** |
| Precio sugerido de equipo | USD 210–260 (+60–80 XJ485) | **USD 95–130** | −55 % |

El 54 % del ahorro viene de **una sola decisión** (eliminar el XJ485), el 27 % de eliminar
los sensores propios y el 19 % restante de 10 recortes menores en la electrónica.

## 2. Palancas, ordenadas por rendimiento

| # | Palanca | Ahorro @500 | Qué se cede | Veredicto |
|---|---|---|---|---|
| 1 | **Conexión directa al puerto TTL** en lugar del módulo XJ485 | **45.00** | Pinout a relevar; sin soporte del fabricante; posible efecto en la garantía del XR06 | **Tomar**, con plan B poblado en la misma PCB (doc 02 §4) |
| 2 | **Eliminar sonda T/HR propia**; temperatura por Modbus | 10.60 | **Se pierde la humedad** (el XR06 no la mide) y la verificación cruzada de sonda; sin dato de temperatura si cae el Modbus | **Tomar**, con la sonda como accesorio (doc 02 §5 opción B) |
| 3 | **Eliminar el CT**; estado de compresor por el bit del relé | 2.60 | Sin detección de "pidió compresor y no hay corriente", contactor pegado y ciclado corto | **Tomar**, como accesorio para clientes con mantenimiento predictivo |
| 4 | Fuente HLK-PM05 en lugar de Mornsun | 2.40 | Menor gama: exige ensayo propio y 100 % de burn-in | Tomar, con control de calidad en producción |
| 5 | Borneras a tornillo en lugar de enchufables | 1.60 | +5 min de servicio en obra | Tomar |
| 6 | PCB de 2 capas, 75×55 mm | 1.75 | Disciplina de ruteo obligatoria | Tomar |
| 7 | Gabinete DIN 4M genérico con etiqueta impresa | 1.80 (**+ USD 8-15 k de herramienta no gastada**) | Menos identidad de marca | **Tomar.** El molde propio se evalúa recién a partir de 2 000 u./año |
| 8 | ESP32-C3 en lugar de ESP32-S3 | 1.50 | Un solo núcleo; sin PSRAM | Tomar: la carga real es 1 trama Modbus/s |
| 9 | Un solo relé | 1.48 | Sirena externa pasa a opción | Tomar |
| 10 | RTC PCF85063A en lugar de DS3231 | 0.93 | Deriva ±2 min/mes sin red | Tomar: SNTP la corrige en cada reconexión |
| 11 | 2 entradas digitales en lugar de 4 | 0.17 | Menos contactos auxiliares | Tomar |
| 12 | Sin conector USB (flasheo por pads) | 0.55 | Diagnóstico en obra solo por Wi-Fi/app | Tomar |

**Palancas rechazadas:**

| Palanca rechazada | Ahorro | Por qué no |
|---|---|---|
| RS-485/TTL **sin aislación galvánica** | 2.50 | Los transitorios de los contactores y los lazos de masa del tablero son la falla de campo nº 1. Una visita técnica cuesta USD 150: el ahorro se pierde con 1 falla cada 60 equipos |
| Antena de PCB (módulo sin variante U) | 1.10 | Dentro de un tablero metálico no conecta. Es el defecto que hace que el equipo ande en el banco y no en obra |
| Sin RTC | 0.93 | Los registros offline sin estampa de tiempo confiable no sirven para auditoría, que es parte del valor vendido |
| Sin TVS ni PTC en borneras de campo | 0.90 | Ídem aislación: falla de campo, no de banco |
| Sensor de HR a bordo del gabinete (AHT20) | — (sumaba 1.20) | Mediría la humedad del tablero. **Un dato falso es peor que ningún dato** |
| Relé de 10 A en lugar de 16 A | 0.35 | Deja de poder comandar luz de cámara o marco de puerta: se pierde una función vendible por 35 centavos |
| Módulo ESP sin certificación (clon) | 0.80 | Rompe la vía corta de homologación ENACOM: costaría meses y miles de dólares en ensayos de radio |

## 3. Vía de prototipado sin herramienta (fases 0 y 1)

Antes de pagar una PCB y un lote, el piloto se puede armar con material de estante. Esto
valida lo único realmente incierto —el mapa Modbus y el pinout TTL— por menos de USD 150
en total:

| Ítem | Precio unit. | Nota |
|---|---|---|
| ESP32-C3 SuperMini | 2.50 | Misma familia que el módulo de producción: el firmware migra sin cambios |
| Módulo TTL aislado (ADuM1201 breakout) ó módulo MAX485 | 2.00 | Según camino TTL o RS-485 |
| Módulo de 1 relé optoacoplado | 1.20 | — |
| Fuente HLK-PM05 en placa | 2.50 | — |
| Caja DIN genérica + bornera | 3.00 | — |
| **Total por unidad de piloto** | **≈ 11.20** | 10 unidades ≈ USD 112 |

**Regla de inversión:** no se compra herramienta ni se lanza PCB rev. B hasta que el mapa
Modbus esté validado contra el firmware real del XR06 del cliente (fase 0) y 10 equipos
hayan sobrevivido 6 semanas en sus cámaras (fase 2). Es la forma más barata de no gastar
USD 15 000 en un gabinete para un producto cuyo protocolo todavía no se confirmó.

## 4. Dónde está el próximo dólar (rev. 3, a partir de 2 000 u./año)

| Idea | Ahorro estimado | Condición |
|---|---|---|
| ESP32-C2/C3 en chip desnudo + antena impresa calificada | 1.20 | Requiere homologación de radio propia: solo se justifica a gran volumen |
| Gabinete con molde propio | 1.00–1.50 | A partir de ~2 000 u./año amortiza la herramienta, y recupera identidad de marca |
| Fuente discreta (no módulo) | 1.30 | Suma trabajo de certificación y riesgo de seguridad; evaluar al final |
| Alimentación tomada del propio XR06 (modelos de 12 V) | 2.40 | Elimina la fuente AC/DC. Verificar margen de la fuente del controlador — **no se hace sin medir** |
| Compra directa de fábrica (no distribuidor) | 8–12 % de la BOM | A partir de pedidos de 2 000 u. |
