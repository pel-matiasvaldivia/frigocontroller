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
| Precisión de temperatura vs. patrón trazable (−30, −18, 0, +5, +25 °C) | Error ≤ ±0.5 °C en todo el rango |
| Precisión de HR vs. cámara climática (40/60/85 %HR a 5 °C y −18 °C) | Error ≤ ±3 %HR |
| Escarchado/condensación de la sonda: 30 ciclos cámara↔ambiente | Recuperación de la lectura de HR en < 15 min con el calentador |
| Robustez Modbus: 200 conmutaciones de contactor real | < 0.1 % de tramas con error, cero alarmas espurias |
| Ensayo de red: 100 ciclos de caída y retorno de Wi-Fi, cortes de 1 min a 48 h | Cero pérdida de registros, backlog reenviado completo y ordenado |
| Corte de alimentación: 50 ciclos, incluidos cortes durante escritura en flash | Cero corrupción de datos, estampa de tiempo correcta al volver |
| Punta a punta de alarma (temperatura, puerta, sonda, compresor) | Notificación en el teléfono en < 60 s, ×30 repeticiones |
| Desgaste de EEPROM del XR06: 1000 cambios de consigna con el rate-limit activo | Sin degradación, rate-limit respetado y reportado a la app |
| Térmico: 72 h a 55 °C y a −20 °C en funcionamiento | Sin reinicios, deriva de medición dentro de especificación |
| IP65 (FC-200): chorro de agua según IEC 60529 | Sin ingreso |
| Precertificación EMC en laboratorio | Márgenes ≥ 6 dB en emisión; inmunidad sin pérdida de función |

## 3. Roadmap

| Fase | Duración | Entregable | Riesgo que cierra |
|---|---|---|---|
| **0 — Banco y mapa Modbus** | 2 sem | XR06 + XJ485 en banco, mapa de registros validado y volcado al perfil YAML, prueba de concepto de ON/OFF y consigna | **El único riesgo técnico real del proyecto.** Se cierra primero y barato |
| **1 — Prototipo funcional** | 4 sem | PCB rev. A (2 unidades), firmware con Modbus + sonda + E/S + MQTT, backend mínimo, app con una pantalla | Viabilidad integral |
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
| Mapa Modbus del XR06 indocumentado o distinto por versión | **Alta** | Alto | Fase 0 primero; perfiles versionados en YAML; modo degradado que cumple los requisitos de monitoreo sin Modbus |
| Wi-Fi inexistente o inestable en la cámara | Alta | Medio | Antena externa obligatoria, medición de señal en el asistente de instalación, variante LTE, operación offline completa |
| Falsas alarmas → el cliente deja de confiar | **Alta** | **Alto** | Enmascarado por deshielo y puerta, retardos, histéresis, agrupación, revisión de cada falsa alarma del piloto como defecto |
| Ruido eléctrico del tablero | Media | Alto | RS-485 aislado, TVS, ensayo con contactor real desde el prototipo |
| Desgaste de EEPROM por escrituras de consigna | Media | Medio | Rate-limit en firmware **y** debounce en la app |
| Otro maestro ya en el bus RS-485 (XWEB del cliente) | Media | Medio | Relevamiento previo; modo solo-lectura; puerto alternativo |
| Demora de certificación | Media | Medio | Módulo pre-certificado; iniciar fase 4 en paralelo con la 3 |
| Reclamo por producto perdido atribuido al sistema | Baja | **Alto** | El gateway no controla; términos de servicio claros; bitácora de auditoría; SLA de notificación explícito y medido |

## 5. Lo que hay que relevar en el sitio del cliente antes de cotizar

1. Modelo y **versión de firmware** exacta de cada XR06 (foto del frente y de la etiqueta).
2. Si ya existe XJ485 o supervisor Dixell (XWEB) instalado, y si hay otro maestro en el bus.
3. Qué parámetros usa hoy cada controlador: `SET`, `Hy`, `ALU`/`ALL`, `ALd`, `idF`, `MdF`,
   `ONF`, `i1F` (copiarlos: son la configuración base de las alarmas).
4. Si el fin de carrera de puerta existe y está cableado, y a dónde.
5. Espacio libre en riel DIN y disponibilidad de 220 V en el tablero.
6. Cobertura Wi-Fi medida **en el tablero** (no en la oficina), y política de red del cliente
   (hay que salir a 8883/TCP).
7. Cantidad de cámaras, tipo (conservación/congelados), y qué mercadería hay dentro — define
   los umbrales y el valor que justifica el precio.
8. Quién recibe las alarmas de noche y los fines de semana (define el escalado y las guardias).
