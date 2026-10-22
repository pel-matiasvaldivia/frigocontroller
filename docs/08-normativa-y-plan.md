# 08 — Normativa, ensayos y plan de trabajo

## 1. Normativa aplicable (Argentina y exportación)

| Ámbito | Requisito | Impacto en el diseño |
|---|---|---|
| Seguridad eléctrica (AR) | Resolución SIC **169/2018** y concordantes: certificación obligatoria para equipos eléctricos de uso domiciliario/comercial. Ensayo según **IEC/EN 60730-1** (controles automáticos) o 61010-1 según clasificación | Clearances ≥ 3 mm, fuente certificada, borneras aptas, marcado, ensayo en organismo reconocido (INTI, IRAM, UL-AR). **Verificar el alcance exacto con el certificador antes de congelar el gabinete** |
| Radio (AR) | Homologación **ENACOM** del equipo con radio | Usar módulo **pre-certificado** (ESP32-S3-WROOM) reduce el trámite a documentación + ensayos reducidos. Diseñar la antena como accesorio homologado junto al equipo |
| EMC | **EN 61000-6-2** (inmunidad industrial) y **EN 61000-6-4** (emisión). Crítico: EN 61000-4-4/-4-5 (transitorios y sobretensiones) por los contactores | Filtro de entrada, TVS en todas las borneras de campo, RS-485 aislado, snubbers en relés. **Diseñado desde el día 1 para pasar, no para corregir** |
| Alimentos / trazabilidad | **Código Alimentario Argentino** y planes HACCP: registro continuo de temperatura, retención de registros, alarma documentada y acción correctiva registrada | Agregados de 15 min con retención de 2 años, reportes firmados, bitácora de auditoría inalterable, reconocimiento de alarma con nota |
| Exportación (opcional) | CE: Directiva de Baja Tensión + EMC + RED; FCC parte 15B/15C | Módulo con FCC ID y CE ya obtenidos; el diseño no agrega radio propia |
| RoHS/REACH | Componentes y proceso sin plomo | Declaración de los proveedores y BOM marcada |

**No hay lazo de control de seguridad en el gateway**, lo que mantiene la clasificación
normativa y la responsabilidad civil acotada: el XR06, ya certificado, sigue siendo el
control. Esto debe quedar explícito en el manual y en los términos del servicio.

## 2. Plan de ensayos antes del piloto

| Ensayo | Criterio de aceptación |
|---|---|
| Fidelidad de lectura: valor leído por Modbus vs. display del XR06 | Coincidencia exacta en los 3 modos (`CF` en °C/°F, `rES` entero/décimas), ×20 puntos incluido negativo |
| **Pinout, niveles y consumo del puerto TTL** en 3 unidades de XR06 | Documentado y repetible; sin degradación del controlador tras 72 h conectado |
| **Equivalencia TTL directo vs. XJ485**: 24 h de polling continuo | Resultados idénticos, < 0.1 % de tramas con error |
| Precisión de temperatura y HR de la **sonda accesorio** vs. patrón trazable | ≤ ±0.5 °C y ≤ ±3 %HR (solo SKU +HR) |
| Robustez Modbus: 200 conmutaciones de contactor real | < 0.1 % de tramas con error, cero alarmas espurias |
| Ensayo de red: 100 ciclos de caída y retorno de Wi-Fi, cortes de 1 min a 48 h | Cero pérdida de registros, backlog reenviado completo y ordenado |
| Corte de alimentación: 50 ciclos, incluidos cortes durante escritura en flash | Cero corrupción de datos, estampa de tiempo correcta al volver |
| Punta a punta de alarma (temperatura, puerta, sonda, compresor) | Notificación en el teléfono en < 60 s, ×30 repeticiones |
| Desgaste de EEPROM del XR06: 1000 cambios de consigna con el rate-limit activo | Sin degradación, rate-limit respetado y reportado a la app |
| **Comportamiento con el enlace Modbus caído** (desconexión en caliente del cable TTL, ×50) | `comm_fail` en < 4 s, app muestra "sin comunicación" con la lectura tachada, recuperación automática sin reinicio |
| Burn-in de la fuente HLK-PM05: 100 % de las unidades, 4 h a 55 °C con carga | Cero fallas; es la contrapartida de haber elegido una fuente de menor gama |
| Térmico: 72 h a 55 °C y a −20 °C en funcionamiento | Sin reinicios, deriva de medición dentro de especificación |
| IP65 (variante FC-E IP65): chorro de agua según IEC 60529 | Sin ingreso |
| Precertificación EMC en laboratorio | Márgenes ≥ 6 dB en emisión; inmunidad sin pérdida de función |

## 3. Roadmap

| Fase | Duración | Entregable | Riesgo que cierra |
|---|---|---|---|
| **0 — Banco, mapa Modbus y puerto TTL** | 2 sem | XR06 de sacrificio + XJ485, mapa de registros validado y volcado al perfil YAML, prueba de concepto de ON/OFF y consigna, **y relevamiento del puerto TTL** | **Los dos riesgos técnicos reales del proyecto, y de los que depende todo el ahorro de la rev. 2.** Se cierran primero, con < USD 150 de material de estante (doc 09 §3) |
| **1 — Prototipo funcional** | 4 sem | PCB rev. A (2 unidades) **o placas de estante si el banco todavía deja dudas**, firmware con Modbus + sonda + E/S + MQTT, backend mínimo, app con una pantalla | Viabilidad integral |
| **2 — Piloto de campo** | 6 sem | 10 equipos en 3 sitios del cliente, app completa, motor de alarmas, reportes | Falsas alarmas, Wi-Fi real, usabilidad de instalación |
| **3 — Industrialización** | 8 sem | PCB rev. B con DFM, gabinete con herramienta/serigrafía final, banco de test de producción, manual, precertificación EMC | Costo, calidad, repetibilidad |
| **4 — Certificación** | 8–12 sem (en paralelo) | Seguridad eléctrica + ENACOM, documentación técnica | Venta legal |
| **5 — Lanzamiento** | — | Primer lote de 100 u., onboarding de instaladores, soporte | — |

Hitos de decisión: al final de la fase 0 se sabe si el ON/OFF y la consigna por Modbus son
viables en el firmware real del cliente (si no, el modo degradado + relé sigue cumpliendo 6
de los 8 requerimientos y se replantea el alcance explícitamente con el cliente). Al final
de la fase 2 se congela el alcance de la v1.

## 4. Riesgos y mitigaciones

| Riesgo | Prob. | Impacto | Mitigación |
|---|---|---|---|
| Mapa Modbus del XR06 indocumentado o distinto por versión | **Alta** | **Alto** | Fase 0 primero; perfiles versionados en YAML. **En rev. 2 este riesgo pesa más**: sin Modbus no hay temperatura, el modo degradado ya no sostiene el producto |
| **Pinout del puerto TTL no relevable, o el cliente no acepta la conexión directa** | Media | Medio | Población FC-R poblada en la misma PCB: se pasa a XJ485 sin rediseño, +USD 45/punto. El precio de lista se cotiza con esa contingencia declarada |
| **Humedad fuera de alcance por haber eliminado los sensores propios** | **Alta** | Medio | El cliente la pidió explícitamente: **se resuelve antes del piloto**, con la sonda accesorio cotizada por cámara o con la exclusión aceptada por escrito (doc 02 §5) |
| Fuente de menor gama con falla temprana | Media | Medio | Burn-in 100 % en producción, ensayo de seguridad propio, segundo proveedor calificado |
| Wi-Fi inexistente o inestable en la cámara | Alta | Medio | Antena externa obligatoria, medición de señal en el asistente de instalación, variante LTE, operación offline completa |
| Falsas alarmas → el cliente deja de confiar | **Alta** | **Alto** | Enmascarado por deshielo y puerta, retardos, histéresis, agrupación, revisión de cada falsa alarma del piloto como defecto |
| Ruido eléctrico del tablero | Media | Alto | RS-485 aislado, TVS, ensayo con contactor real desde el prototipo |
| Desgaste de EEPROM por escrituras de consigna | Media | Medio | Rate-limit en firmware **y** debounce en la app |
| Otro maestro ya en el bus RS-485 (XWEB del cliente) | Media | Medio | Relevamiento previo; modo solo-lectura; puerto alternativo |
| Demora de certificación | Media | Medio | Módulo pre-certificado; iniciar fase 4 en paralelo con la 3 |
| Reclamo por producto perdido atribuido al sistema | Baja | **Alto** | El gateway no controla; términos de servicio claros; bitácora de auditoría; SLA de notificación explícito y medido |

## 5. Lo que hay que relevar en el sitio del cliente antes de cotizar

1. Modelo y **versión de firmware** exacta de cada XR06 (foto del frente y de la etiqueta).
   **Y foto del puerto TTL** con el conector a la vista: define si el camino económico es viable.
2. Si ya existe XJ485 o supervisor Dixell (XWEB) instalado, y si hay otro maestro en el bus.
3. Qué parámetros usa hoy cada controlador: `SET`, `Hy`, `ALU`/`ALL`, `ALd`, `idF`, `MdF`,
   `ONF`, `i1F` (copiarlos: son la configuración base de las alarmas).
4. Si el fin de carrera de puerta existe, y **a dónde está cableado**: si va a una entrada
   digital del XR06, las aperturas se leen por Modbus y no hay que tender ningún hilo; si no
   está cableado, hay que llevarlo a la DI1 del gateway. Cambia el tiempo de instalación.
5. En qué cámaras se necesita **humedad** (define cuántas sondas accesorio se cotizan).
6. Espacio libre en riel DIN y disponibilidad de 220 V en el tablero.
7. Cobertura Wi-Fi medida **en el tablero** (no en la oficina), y política de red del cliente
   (hay que salir a 8883/TCP).
8. Cantidad de cámaras, tipo (conservación/congelados), y qué mercadería hay dentro — define
   los umbrales y el valor que justifica el precio.
9. Quién recibe las alarmas de noche y los fines de semana (define el escalado y las guardias).
