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
control — y su propio manual advierte que *"no debe ser usado como un dispositivo de
seguridad"*, frase que conviene citar textualmente en los términos del servicio.

**Dos puntos nuevos de la rev. 3 para discutir con el certificador antes de congelar el
diseño:**

1. **Las tres entradas de detección** están permanentemente a potencial de red y entran al
   gabinete por borneras de campo: hay que declararlas como circuito de red (no SELV) y
   ensayarlas como tal, con su aislación de 4 kV y creepage de 5,5 mm.
2. **El relé que corta la alimentación de otro aparato** puede cambiar la clasificación del
   equipo según IEC 60730 (tipo de acción, clase de software). El XR06 declara "Tipo de acción
   1B" y "Software clase A": el gateway debe declarar su propia clasificación y **no puede
   presentarse como función de seguridad**. Si se habilita la consigna virtual por compuerta
   (doc 10 §7), el gateway pasa a ser un control automático y la conversación con el
   certificador cambia: conviene mantener esa función deshabilitada por defecto.

## 2. Plan de ensayos antes del piloto

| Ensayo | Criterio de aceptación |
|---|---|
| Precisión de la sonda propia vs. patrón trazable (−30, −18, 0, +5, +25 °C) | Error ≤ ±0,5 °C; y ≤ ±0,8 °C de diferencia contra el display del XR06 con `ot` aplicado |
| **Detección de las 3 salidas**: 500 conmutaciones de cada relé, con cargas de 1 A y de 16 A | 100 % de transiciones detectadas, latencia < 1 s, cero falsos positivos por acoplamiento entre canales |
| **Reconstrucción de la máquina de estados** contra registro manual de 48 h de operación real (`td=EL` y `td=in`, `FC` en los 4 modos) | Estado correcto ≥ 99 % del tiempo; fases de goteo y retardo de ventiladores identificadas |
| **Detección inferida de falla de sonda**: desconectar la NTC del XR06 ×20 | `probe_fail_inferred` en ≤ 3 × (`Cy`+`Cn`), cero falsos positivos en 30 días de operación normal |
| **Corte y reposición del XR06 por el relé**: 200 ciclos con `od` = 3 min | El compresor nunca arranca antes de `od`; el gateway confirma el estado por las salidas; cero alarmas espurias |
| **Watchdog de hardware del relé de corte**: colgar el firmware a propósito ×20 | El relé se desenergiza y el XR06 queda alimentado en < 60 s, **siempre** |
| Precisión de temperatura y HR de la **sonda accesorio** vs. patrón trazable | ≤ ±0.5 °C y ≤ ±3 %HR (solo SKU +HR) |
| Robustez Modbus: 200 conmutaciones de contactor real | < 0.1 % de tramas con error, cero alarmas espurias |
| Ensayo de red: 100 ciclos de caída y retorno de Wi-Fi, cortes de 1 min a 48 h | Cero pérdida de registros, backlog reenviado completo y ordenado |
| Corte de alimentación: 50 ciclos, incluidos cortes durante escritura en flash | Cero corrupción de datos, estampa de tiempo correcta al volver |
| Punta a punta de alarma (temperatura, puerta, sonda, compresor) | Notificación en el teléfono en < 60 s, ×30 repeticiones |
| Desgaste de EEPROM del controlador (solo camino B): 1000 cambios de consigna con el rate-limit activo | Sin degradación, rate-limit respetado y reportado a la app |
| **Comportamiento con la sonda propia en falla** (desconexión en caliente, corto, ×50) | `own_probe_fail` en < 10 s, app muestra la lectura tachada, recuperación automática sin reinicio |
| **Aislación de las entradas de detección de CA**: ensayo dieléctrico 4 kV, 1 min | Sin ruptura; y cero acoplamiento a la lógica con las 3 entradas a 264 Vac |
| Burn-in de la fuente HLK-PM05: 100 % de las unidades, 4 h a 55 °C con carga | Cero fallas; es la contrapartida de haber elegido una fuente de menor gama |
| Térmico: 72 h a 55 °C y a −20 °C en funcionamiento | Sin reinicios, deriva de medición dentro de especificación |
| IP65 (variante FC-E IP65): chorro de agua según IEC 60529 | Sin ingreso |
| Precertificación EMC en laboratorio | Márgenes ≥ 6 dB en emisión; inmunidad sin pérdida de función |

## 3. Roadmap

| Fase | Duración | Entregable | Riesgo que cierra |
|---|---|---|---|
| **0 — Banco de envolvente** | 2 sem | XR06CX en banco con 2 NTC, contactor real y simulador de puerta: detección de las 3 salidas validada, máquina de estados reconstruida, corte y reposición con `od`, firma de falla de sonda reproducida. **Y la consulta a Emerson con `Pt`/`rL`** | **Confirma que el camino A cumple lo prometido** y cierra la única duda que podría reabrir el camino ModBUS. < USD 150 de material de estante (doc 09 §3) |
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
| **El cliente exige setup de temperatura remoto sobre sus XR06CX existentes** | **Alta** | **Alto** | No es posible: el controlador no tiene canal de escritura (doc 10 §7). **Se resuelve en la reunión previa, no en el piloto**, con el catálogo de dos niveles: nivel 1 sobre el parque y upgrade de nivel 2 (reemplazo de controlador) en las cámaras que lo necesiten |
| **Humedad fuera de alcance del equipo base** | **Alta** | Medio | El cliente la pidió explícitamente: sonda T/HR accesorio cotizada por cámara, o exclusión aceptada por escrito |
| Falla de sonda del controlador detectada 2 h tarde (inferencia) | Media | Medio | Declararlo en la propuesta. Mientras tanto la alarma de temperatura propia vigila el resultado, que es lo que protege la mercadería |
| **Error de cableado de las tres derivaciones** | Media | Medio | Kit pre-armado con colores fijos; el asistente de instalación lo verifica pidiendo un deshielo manual. Entradas eléctricamente idénticas: un cruce no daña nada |
| Alguien cambia parámetros en el frente del XR06 y el gateway no se entera | **Alta** | Bajo-Medio | La app muestra la antigüedad de la última verificación y recuerda revalidar cada 6 meses; el acta de puesta en marcha deja constancia de los valores originales |
| Variante de firmware del XR06 con ModBUS que nadie detectó | Baja | **Favorable** | Leer `Pt` y `rL` y consultar a Emerson: si existiera, se habilita el camino B sin cambiar de hardware (población FC-R) |
| Fuente de menor gama con falla temprana | Media | Medio | Burn-in 100 % en producción, ensayo de seguridad propio, segundo proveedor calificado |
| Wi-Fi inexistente o inestable en la cámara | Alta | Medio | Antena externa obligatoria, medición de señal en el asistente de instalación, variante LTE, operación offline completa |
| Falsas alarmas → el cliente deja de confiar | **Alta** | **Alto** | Enmascarado por deshielo y puerta, retardos, histéresis, agrupación, revisión de cada falsa alarma del piloto como defecto |
| Ruido eléctrico del tablero | Media | Alto | RS-485 aislado, TVS, ensayo con contactor real desde el prototipo |
| Desgaste de EEPROM por escrituras de consigna | Media | Medio | Rate-limit en firmware **y** debounce en la app |
| Otro maestro ya en el bus RS-485 (XWEB del cliente) | Media | Medio | Relevamiento previo; modo solo-lectura; puerto alternativo |
| Demora de certificación | Media | Medio | Módulo pre-certificado; iniciar fase 4 en paralelo con la 3 |
| Reclamo por producto perdido atribuido al sistema | Baja | **Alto** | El gateway no controla; términos de servicio claros; bitácora de auditoría; SLA de notificación explícito y medido |

## 5. Lo que hay que relevar en el sitio del cliente antes de cotizar

1. Modelo y **versión de firmware** exacta de cada XR06 (foto del frente y de la etiqueta), y
   los parámetros de solo lectura **`Pt` (mapa de códigos) y `rL` (versión de software)**
   leídos del display. Son el dato que hay que llevarle a Emerson para preguntar si esa
   variante admite ModBUS.
2. **Parámetros del controlador, leídos del display y anotados** (la app los pide uno por uno
   en la puesta en marcha, pero conviene tenerlos antes de cotizar):
   `SET`, `Hy`, `LS`, `uS`, **`ot`, `oE`** (si no son 0, la sonda propia va a diferir del
   display), `AU`, `AL`, `Ad`, `dA`, `Cy`, `Cn`, `td`, `id`, `Md`, `dt`, `dP`, `FC`, `Fd`,
   **`iF`, `iP`, `dC`, `rd`**, `od`, `AC`, `P2`.
3. **Espacio en las borneras de salida del XR06** para derivar los tres hilos de detección, y
   si el tablero permite intercalar el relé de corte en su alimentación.
4. Si existe un supervisor (tipo XWEB) o un bus RS-485 instalado, y si hay otro maestro.
5. Si el fin de carrera de puerta existe y **a dónde está cableado**. El gateway usa su propia
   DI1: si ya hay un contacto libre o un segundo polo, se aprovecha; si no, hay que llevar un
   hilo o agregar un reed.
6. **En qué cámaras se necesita humedad** (define cuántas sondas +HR se cotizan) y en cuántas
   hace falta **consigna remota** (define cuántos upgrades de nivel 2).
7. Espacio libre en riel DIN y disponibilidad de 220 V en el tablero.
8. Cobertura Wi-Fi medida **en el tablero** (no en la oficina), y política de red del cliente
   (hay que salir a 8883/TCP).
9. Cantidad de cámaras, tipo (conservación/congelados), y qué mercadería hay dentro — define
   los umbrales y el valor que justifica el precio.
10. Quién recibe las alarmas de noche y los fines de semana (define el escalado y las guardias).
