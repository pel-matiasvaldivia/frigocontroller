# 11 — Controladores para el upgrade de nivel 2 (camino B)

Relevamiento de controladores con ModBUS para reemplazar al XR06CX donde el cliente exija
**setup de temperatura remoto**, que el XR06CX no puede dar
([doc 10 §7](10-hallazgos-manual-xr06cx.md#7-setup-de-temperatura-remoto-las-tres-salidas-honestas)).

> ## Corrección posterior — leer primero [doc 12](12-plataforma-smartpath-m3.md)
>
> Este relevamiento se hizo **antes** de conocer el proyecto SmartPath M3 que ya presentamos a
> este cliente. Ese proyecto demuestra que el **Dixell XR75CH tiene RS-485 ModBUS a bordo**
> (bornes 16/17/18, sin XJ485) y que **ya tenemos su mapa de registros funcionando en
> producción**. Los dos argumentos con los que acá descarto a la familia Dixell (§3.4) caen:
> **el XR75CH pasa a ser el candidato nº 1 del nivel 2.**
>
> Lo que sigue conserva su valor como relevamiento de alternativas y, sobre todo, como
> **criterios de compra** (§4) — pero el orden de preferencia está corregido en el doc 12 §2.

> **Estado de la información.** Lo que sigue se armó con fichas de fabricante, listados de
> distribuidor y documentación de terceros. Los fabricantes de este rubro publican poco y
> cambian de línea seguido, así que **nada de esto reemplaza una cotización firme con el
> número de parte exacto y la confirmación por escrito de los 6 puntos de §4.** Donde la
> fuente es un distribuidor o un tercero, está dicho.

## 1. El hallazgo más importante: "ModBUS nativo" casi no existe en este rubro

En esta clase de producto la arquitectura dominante es **puerto TTL a bordo + adaptador
TTL→RS-485 externo**, exactamente el patrón del XJ485 que ya conocíamos:

| Controlador | Interfaz física | ¿Adaptador externo? |
|---|---|---|
| Dixell XR / XW (Copeland) | TTL (Hot Key) | **Sí**, XJ485-CX |
| Danfoss ERC 213 | TTL a bordo | **Sí**, adaptador TTL-485 |
| Eliwell IDNext 974 P | "un puerto serie de tipo TTL" (ficha oficial) | **Sí** |
| Carel IR33 (original) | — | **Sí**, placa serie IROPZSER30 |
| **Carel IR33+** | **RS-485 a bordo** (según distribuidor) | **No** |
| Eliwell/EVCO clase EV3294 | "puerto ttl modbus esclavo **o** puerto rs-485 modbus esclavo (según el modelo)" | **Según modelo** |

**Consecuencia de costeo:** si el controlador elegido necesita adaptador, al precio del
controlador hay que sumarle **USD 35-55 por punto**, y el upgrade de nivel 2 sube de
USD 290-390 a USD 340-450. Es el mismo costo que nos hizo descartar el XJ485 en la rev. 2,
reapareciendo por la puerta de atrás. **Preferir controladores con RS-485 a bordo no es una
cuestión de elegancia: son USD 45 por cámara.**

## 2. Corrección: Full Gauge queda fuera

En los documentos anteriores nombré a **Full Gauge (MT-530 Super, TC-900)** como candidato
de ModBUS nativo. **Es incorrecto y lo corrijo.**

Full Gauge comunica por RS-485 pero con **protocolo propietario para su software Sitrad**, no
con ModBUS RTU. El convertidor TCP-485 de la propia marca está descripto como diseñado para
funcionar **únicamente con instrumentos Full Gauge**, y no encontré ninguna fuente que
documente ModBUS RTU en la línea MT/TC. Las fichas solo dicen "comunicación serial para
conexión con SITRAD".

Es una lástima desde lo comercial: Full Gauge es probablemente la marca más disponible y más
barata en Argentina. **Quedan dos salidas, y conviene preguntar antes de descartarla del
todo:**

1. **Consultar a Full Gauge** si alguna línea actual expone ModBUS RTU (es la clase de
   función que los fabricantes agregan sin anunciarla fuerte). Si la respuesta es sí, vuelve
   a ser el candidato nº 1 por precio y disponibilidad.
2. **Implementar el protocolo Sitrad** en el gateway. Es ingeniería inversa de un protocolo
   propietario: **no se cotiza como desarrollo**, pero si el cliente tiene parque Full Gauge
   grande, es una conversación distinta — se evalúa con el volumen sobre la mesa, no antes.

## 3. Candidatos, por orden de preferencia

### 3.1 Danfoss ERC 213 — **recomendado para el piloto**

| | |
|---|---|
| ModBUS | **Sí, ModBUS RTU desde el firmware 4.16** (fuente: documentación de terceros) |
| Interfaz | TTL a bordo → **requiere adaptador TTL-485** |
| Serie | 9600, 8 bits, paridad **par**, 1 stop (fijo) |
| **Mapa de registros** | **Publicado por Danfoss** en su guía de programación ModBUS de la familia ERC 21x: p. ej. consigna en PNU 100, diferencial en PNU 101 |
| Hardware | 3 relés; relé de alta performance de 16 A capaz de conmutar directamente cargas de hasta 2 HP |
| Código | 080G3265 (la tienda de Danfoss advierte que **hay una versión más nueva** y recomienda consultarla) |

**Por qué primero:** es el único de la lista cuyo **mapa de registros es público y
documentado**. Eso elimina de un golpe el riesgo que nos comimos con el XR06CX: no hay que
descubrir direcciones por diff de estímulos, están en un PDF del fabricante. Para un producto
que se va a vender con abono y soporte, un protocolo documentado vale más que USD 20 de
diferencia de precio.

**Dos cuidados concretos:**

- **El TTL RX del ERC 213 exige más de 50 mA de capacidad de drive**, y hay reportes de que
  "algunos adaptadores de terceros pueden no funcionar". Nuestra población FC-R tiene que
  dimensionar el driver para eso, o usar el adaptador de Danfoss. **Es un dato de diseño, no
  una anécdota**: hay que verificarlo en banco antes de comprometer el upgrade.
- El mapa publicado que encontré está fechado en 2015 y cubre firmware 4.13/4.16: hay que
  validarlo contra el firmware que venga en la unidad comprada.

### 3.2 Carel IR33+ — **el único con RS-485 a bordo**

| | |
|---|---|
| ModBUS | **Sí**, y según el distribuidor **reconoce automáticamente ModBUS y protocolo CAREL** |
| Interfaz | **RS-485 a bordo** → **sin adaptador: USD 35-55 menos por punto** |
| Alimentación | 12-24 Vac/dc ó 115-230 Vac |
| Sondas | NTC y PTC |
| Salidas | compresor de 16 A |
| Frente | IP65 |
| Códigos | IREVS0LN00 (230 V, 2 entradas, on/off) · IREVC7HN00 (termocupla) |
| **Alerta** | El listado de RS marca el producto como **en fin de producción / no disponible**. **Hay que pedirle a Carel el reemplazo vigente de la línea** antes de diseñar sobre él |

**Por qué segundo a pesar de ser técnicamente el mejor:** el riesgo de obsolescencia. No se
puede construir un upgrade sobre un producto que está saliendo de catálogo. Si Carel confirma
el sucesor con RS-485 a bordo y ModBUS, **pasa a ser el candidato nº 1** por los USD 45 de
adaptador que ahorra. El IR33 **original** no sirve: necesita la placa serie IROPZSER30.

### 3.3 Eliwell / EVCO, línea con RS-485 a bordo

| | |
|---|---|
| ModBUS | **Sí**, esclavo ModBUS |
| Interfaz | Según el modelo: hay variantes con **puerto TTL ModBUS** y variantes con **puerto RS-485 ModBUS** — un catálogo de distribuidor lo dice así textualmente para la clase EV3294 |
| Hardware típico | 12-24 V, sonda de cámara y de evaporador |
| **A verificar** | **El número de parte exacto que trae RS-485 a bordo.** Es toda la diferencia de costo |

El **IDNext 974 P** de Eliwell, que a primera vista parecía candidato, **no sirve tal cual**:
su ficha oficial declara "un puerto serie de tipo TTL". Sirve la familia ID (ID 971/974) por
formato y por ser un estándar de la industria, pero **solo en la variante con RS-485**.

### 3.4 Dixell XW / XR (quedarse en la familia Copeland) — ~~no recomendado~~ **→ ver doc 12**

> **Corregido.** El **XR75CH** tiene RS-485 a bordo y su integración ya está probada por
> nosotros. Lo de abajo sigue valiendo para los modelos **sin** bornera RS-485 (XR06CX,
> XR60CH, XW60K): dentro de la familia, **el sufijo del part number decide**.

Para los modelos sin RS-485 a bordo:

- **Necesita XJ485-CX** (TTL→RS-485): +USD 35-55 por punto. El listado de un distribuidor del
  XW60K muestra el campo "interface RS485" **vacío**, y la ficha del XW60L dice que la salida
  **HOT KEY** permite conectarse a una red ModBUS-RTU **mediante el módulo externo XJ485-CX**.
- **Y el problema de fondo:** en un foro de Emerson, un contacto de la propia empresa afirma
  que **"los registros ModBUS son información propietaria y no se entregan libremente"**. Si
  el fabricante no publica el mapa, el upgrade depende de volver a descubrir direcciones por
  ingeniería inversa — el mismo pozo del que acabamos de salir.

**En los modelos sin bornera RS-485, quedarse en Dixell cuesta más y documenta menos.** Pero
con el XR75CH no aplica ninguno de los dos problemas: tiene RS-485 nativo y el mapa de
registros lo tenemos en casa (doc 12 §2).

### 3.5 Controladores genéricos de importación con ModBUS

Existen controladores de cámara con RS-485 ModBUS a **USD 45-60** en canales de importación
directa. Para un producto que se vende **con abono y con SLA de notificación**, no los
recomiendo como línea base: sin certificación local, sin soporte, sin garantía de continuidad
del mapa de registros entre lotes, y el riesgo de que dos unidades "iguales" respondan
distinto. Sí son útiles para **el banco de desarrollo**: permiten probar la población FC-R y
el perfil ModBUS por poco dinero antes de comprar un controlador de marca.

## 4. Criterios de selección: lo que hay que confirmar por escrito

Antes de comprometer el upgrade con un cliente, de cada candidato:

| # | A confirmar | Por qué |
|---|---|---|
| 1 | **ModBUS RTU documentado, con mapa de registros entregable** (no "soporta ModBUS" a secas) | Es la lección del XR06CX. Sin mapa, el desarrollo es ingeniería inversa de duración desconocida |
| 2 | **RS-485 a bordo, o costo real del adaptador** | USD 35-55 por punto decide el precio del upgrade |
| 3 | **Registro de escritura de la consigna, con límites** | Es *la* función que justifica el upgrade. Que se pueda leer no significa que se pueda escribir |
| 4 | **ON/OFF o stand-by por ModBUS** | Si lo tiene, el relé de corte del gateway queda como respaldo en lugar de como camino principal |
| 5 | **Formato de panel 71 × 29 mm de recorte** (frente 32 × 74 mm) | **Es el formato del XR06CX** (§9 de su manual) y un estándar de la industria compartido por Eliwell ID, AKO, Sporlan y LAE. Si el reemplazo lo respeta, **se reusa el agujero del tablero**: nada de mecanizado |
| 6 | **Sondas NTC compatibles y relés de capacidad igual o mayor** | El XR06CX usa NTC de −40÷110 °C y relés de 8/16/20 A. Si coinciden, **se reusan las sondas y el cableado de potencia existentes** |

Los puntos 5 y 6 son los que convierten el reemplazo en un trabajo de **45 minutos en lugar
de 2 horas**: mismo agujero, mismas sondas, mismos cables de carga, y solo hay que pasar los
dos hilos del RS-485. Conviene hacerlos requisito de compra, no un hallazgo afortunado.

Además, y vale para todos: **certificación local** (seguridad eléctrica según Res. 169/2018 y
concordantes) y disponibilidad real de stock y repuesto en Argentina. Un controlador excelente
con 12 semanas de lead time no sirve para un upgrade que se vende por cámara.

## 5. Impacto en nuestro diseño

| Hallazgo | Qué cambia en el producto |
|---|---|
| La mayoría de los candidatos usa TTL + adaptador | La población **FC-R** debe prever **las dos conexiones**: bornera A/B para RS-485 a bordo, **y** un puerto TTL con driver capaz de ≥ 50 mA para los controladores que lo exijan. Es el mismo circuito aislado, con dos salidas de conector: **costo marginal cercano a cero si se decide ahora**, rediseño de PCB si se decide después |
| Los mapas de registros difieren por marca, modelo y firmware | Se confirma la decisión de la rev. 1: **el mapa vive en un perfil YAML**, uno por modelo y versión. Con el ERC 213 el primer perfil se arma leyendo un PDF, no haciendo diffs en el banco |
| Los serie van a 9600 8E1 (Danfoss) o 9600 8N1/8N2 (otros) | La autodetección de velocidad y formato del firmware ya lo cubre; hay que incluir **8E1** en la lista de candidatos |
| Full Gauge usa protocolo propietario | Si el cliente tiene parque Full Gauge, el camino A (envolvente) es la **única** opción razonable hoy. Refuerza que el nivel 1 sea el producto y el nivel 2 el upgrade, y no al revés |
| Formato 71 × 29 mm estándar | Se documenta como **requisito de compra** del controlador de upgrade, y el procedimiento de reemplazo se escribe asumiendo reuso de agujero, sondas y cableado de potencia |

## 6. Próximos pasos concretos

0. **Confirmar con Emerson el part number del XR75CH** con RS-485 a bordo, precio y stock en
   Argentina. **Es el paso que cierra el nivel 2**; los de abajo pasan a ser plan de
   contingencia y comparación de precio.
1. **Pedir a Danfoss** la guía de programación ModBUS vigente del ERC 21x y el código del
   adaptador TTL-485 recomendado, más precio y stock en Argentina del modelo nuevo que la
   propia tienda menciona.
2. **Pedir a Carel** el reemplazo vigente del IR33+ y confirmar RS-485 a bordo + ModBUS en el
   número de parte concreto. Si existe y está disponible, es el candidato nº 1.
3. **Preguntar a Full Gauge** por ModBUS RTU en su línea actual. Es la consulta más rentable
   de las tres: si la respuesta es sí, cambia el costo y la logística del upgrade entero.
4. **Pedir a Eliwell/EVCO** el número de parte con RS-485 a bordo de la clase EV3294.
5. Comprar **un** controlador genérico con ModBUS (USD 45-60) para el banco, y validar con él
   la población FC-R y el parser de perfiles mientras llegan las respuestas.

Con las cuatro respuestas se elige un único controlador de upgrade, se arma su perfil YAML y
se cierra el precio del nivel 2. Hasta entonces, el nivel 2 se cotiza **"a confirmar"**: es
más barato decir eso que repetir el error de prometer una interfaz sin haber leído el manual.

## 7. Fuentes

Todas consultadas en octubre de 2026. Las marcadas como *distribuidor* o *terceros* requieren
confirmación del fabricante.

- Danfoss ERC 213: [ficha de producto 080G3265](https://store.danfoss.com/en/Product-Type/Electronic-controls/ERC-213%2C-Electronic-Controls/Electronic-refrigerat-control%2C-ERC-213/p/080G3265) · [guía de programación ModBUS ERC 21x](https://assets.danfoss.com/documents/283596/AM446512660651en-000101.pdf) · [guía de instalación](https://assets.danfoss.com/documents/latest/34791/AN194386421321de-000401.pdf) · [notas de integración (terceros)](https://docs.webhmi.com.ua/erc-213)
- Danfoss adaptadores TTL/RS-485: [EKA 206 084B4088](https://store.danfoss.com/sg/en/p/084B4088) · [EKA 206 084B4188](https://store.danfoss.com/sg/en/p/084B4188)
- Carel IR33+: [listado RS Online IREVS0LN00 (distribuidor)](https://uk.rs-online.com/web/p/temperature-controllers/8090966) · [IREVC7HN00 (distribuidor)](https://za.rs-online.com/web/p/temperature-controllers/8090975) · [folleto IR33 original](https://digitalassets.reecegroup.com.au/m/0547b5de58e1fd7c/original/Brochure-CAREL-IR33-Display-Suit-DN33-IR00XGD000.pdf) · [placa serie IROPZSER30 (distribuidor)](https://stagingtwo.i-acs.co.uk/iropzser30-carel-rs485-serial-card-for-ir33-din-rail-version.html)
- Eliwell: [IDNext 974 P, ficha oficial (puerto TTL)](https://www.eliwell.com/en/Partnumber/IDN974PED303000.html) · [IDNext 978 P](https://www.eliwell.com/en/Partnumber/IDN978P4D307000.html) · [catálogo con clase EV3294 y "rs-485 modbus esclavo según el modelo"](https://wrs01.salvadorescoda.com/DocTec/10055510.pdf) · [ID974, formato 71×29 mm (distribuidor)](https://www.gev-online.com/en/webshop/product/378135) · [ID971 (distribuidor)](https://www.ricambi.com.au/digital-temperature-controller-id971)
- Dixell / Copeland: [XW60K-5N2C0, campo "interface RS485" vacío (distribuidor)](https://www.gev-online.com/en/webshop/product/378555) · [XW60L-5N1C1N, ModBUS vía XJ485-CX (distribuidor)](https://www.ricambi.com.au/xw60l-5n1c1n-digital-controller-dixell) · [adaptador XJ485CX+CAB/RS (distribuidor)](https://schiessl-kaelte.com/en_DE/Shop/Refrigeration-components/Regulators/Electronic/Accessories-/-spare-parts/Dixell-interface-adapter-XJ485CX-CAB-RS~p52888) · [notas de integración Dixell (terceros)](https://docs.webhmi.com.ua/dixell) · [foro Emerson: "los registros ModBUS son información propietaria"](https://emersonexchange365.com/services/f/services-forum/5556/interlink-dixell-xjp60d-to-deltav) · [XW60K material de capacitación (Copeland)](https://webapps2.copeland.com/ci/dixell-xw60k/XW60K%20Training%20PPT%20050619.pdf)
- Full Gauge: [manual del convertidor TCP-485](https://fullgauge.com/public/uploads/files/products/manual-de-produto-79-158.pdf) · [ficha TCP-485 (distribuidor)](https://iprpartesyrepuestos.com/wp-content/uploads/2018/07/fichas/INTERFACE-MODULO-TCP485-FULL-GAUGE.pdf) · [ficha MT-530 en español (distribuidor)](https://repuestos.refrigeracion-omega.com/importador/fichas/server/php/files/T-006-025.pdf) · [MT-530E Super con comunicación serial (distribuidor)](https://www.keepsupply.com/mt-530e-super-robertshaw-115-230v-humidity-and-temperature-controller-with-serial-communication)
- Formato de panel 71 × 29 mm en otras marcas: [Sporlan PSK231 (distribuidor)](https://www.supplyhouse.com/Sporlan-PSK231N7EXBS-Refrigeration-Controller-for-Medium-Temperatures-30A-Compressor-230-VAC)
- Genérico con ModBUS, referencia de precio: [controlador RS485 para cámara, USD 44.51](https://www.walmart.com/ip/5182461664)
