# 10 — Hallazgos del manual del XR06CX y replanteo del diseño

**Fuente:** manual oficial `1594020160 XR06CX SP r1.2 27.07.2015`
([docs/manuales/](manuales/XR06CX-SP-r1.2-27.07.2015.pdf)).

## 1. Hallazgo que invalida la premisa de las rev. 1 y 2

> **El XR06CX no es un controlador comunicable. No tiene ModBUS ni puerto serie de proceso.**

Evidencia en el propio manual:

| Evidencia | Dónde | Consecuencia |
|---|---|---|
| **No existe parámetro de dirección serie** (`Adr`, `dEA`, `FAA`…) en la lista completa de parámetros, ni en el menú secreto (§7 y §15) | Lista de parámetros y tabla de valores por defecto | **Decisivo:** un esclavo ModBUS necesita dirección. Si no hay parámetro para fijarla, no hay esclavo ModBUS |
| Datos técnicos no mencionan salida serie, RS-485 ni TTL de proceso | §13: "Conexiones: Terminal con tornillos. Cables ≤ 2,5 mm²" | No hay bus de campo |
| El único puerto es la **Hot Key**, y es un canal de *programación*, no de lectura | §11: para cargar o descargar hay que **apagar el instrumento, insertar la llave y volver a encender** | No sirve para polling de 1 s. No es un canal de telemetría |
| No se menciona el módulo XJ485 ni ningún accesorio de comunicación | Todo el manual | El camino "XJ485" de la rev. 1 y el "TTL directo" de la rev. 2 **no aplican a este modelo** |

El conector TTL de la Hot Key existe físicamente, pero lo que corre por ahí es la
transferencia de la lista de parámetros, no ModBUS-RTU.

**Qué hay que verificar con Emerson antes de cerrar nada:** si el part number exacto del
cliente tiene una variante de firmware con salida serie ModBUS y parámetro de dirección
(algunas familias XR lo agregan). El dato está en el display del propio equipo:
**`Pt` (mapa de códigos) y `rL` (versión de software)**, ambos parámetros de solo lectura
(§7, apartado OTROS). **Esos dos valores hay que leerlos en cada controlador del cliente y
pasárselos al soporte de Emerson**: es la consulta de 5 minutos que decide el camino.

Mientras no se confirme lo contrario, el diseño asume **que no hay ModBUS**.

## 2. Qué se puede y qué no se puede hacer con este controlador

| Requerimiento del cliente | ¿Alcanzable sobre un XR06CX existente? | Cómo |
|---|---|---|
| Transmitir temperatura | **Sí** | Sonda propia del gateway (vuelve al diseño, ahora es obligatoria) |
| Indicación de funcionamiento del compresor | **Sí, mejor que por ModBUS** | Detección de tensión en la **salida de relé de compresor** (dato físico, no un bit) |
| Indicación de deshielo en la App | **Sí** | Detección en la **salida de deshielo** (y la de ventiladores, que para durante el deshielo con `FC=cn/on`) |
| Registro de aperturas de puerta | **Sí** | Sensor propio en la puerta (ver §4: la entrada digital del XR06 es **una sola** y conviene no gastarla) |
| Alarmas de alta y baja temperatura | **Sí** | Calculadas por el gateway sobre su sonda, replicando `AU`/`AL`/`Ad` del controlador |
| Alerta de error de comunicación | **Sí** | Watchdog del gateway + LWT MQTT (no depende del XR06) |
| **ON / OFF remoto** | **Sí**, por dos caminos (§5) | Relé que corta la alimentación del XR06, **ó** entrada digital en modo `bA` |
| **Alertas por falla de sondas (P1/P2)** | **Parcial: inferencial, no directa** (§6) | El XR06 no tiene contacto de alarma ni registro legible. Se detecta por la **firma de ciclado `Cy`/`Cn`** |
| **Setup de temperatura remoto** | **NO sobre el XR06CX tal cual** (§7) | Solo "consigna virtual hacia arriba" por compuerta, reemplazo de controlador, o emulación de Hot Key (I+D) |

**Dos requerimientos de los ocho cambian de naturaleza, y hay que decirlo al cliente ahora,
no en el piloto:** la falla de sondas pasa de alarma directa a detección inferencial, y el
setup remoto de temperatura **no es posible** sin una de las tres salidas del §7.

## 3. Tres caminos, con su costo

| | **A — Envolvente no invasiva** | **B — Reemplazo del controlador** | **C — Emulación de Hot Key** |
|---|---|---|---|
| Qué es | Se deja el XR06CX intacto y el gateway lo rodea: sonda propia, detección de las 3 salidas, relé de corte | Se cambia el XR06CX por un controlador con RS-485 ModBUS nativo (Full Gauge MT-530/TC-900 con Sitrad, Dixell XW, Carel IR33, Eliwell con RS-485) | El gateway emula la memoria de la Hot Key y fuerza una descarga de parámetros con un ciclo de alimentación |
| Setup de temperatura remoto | **No** (solo consigna virtual hacia arriba) | **Sí, nativo** | **Sí**, si la ingeniería inversa resulta |
| Falla de sondas P1/P2 | Inferencial | **Directa, por registro** | Directa |
| Costo de equipo @500 u. | **USD 26.44** | USD 24.20 **+ controlador USD 60–120** | A + I+D no acotada |
| Riesgo | **Bajo.** Nada se interviene del controlador | Bajo técnicamente; alto comercialmente (hay que convencer de cambiar un equipo que funciona, y re-cablear) | **Alto.** Protocolo propietario, desgaste de EEPROM, ~15 s de corte por escritura, riesgo de dejar el controlador sin parámetros |
| Instalación | 15 min, sin tocar el cableado de potencia | 1–2 h por cámara, con corte de servicio | Como A |
| Veredicto | **Línea base.** Cumple 7 de 8 requerimientos sin tocar la instalación | **Opción premium**, para el cliente que exija consigna remota | **No se cotiza.** Solo I+D de fase 4, con el cliente informado |

**Recomendación:** vender **A** como producto, con **B** como *upgrade* ofrecido por cámara
para quien necesite consigna remota. Esto convierte la limitación en catálogo de dos
niveles, en lugar de en una promesa incumplida.

## 4. La entrada digital es **una sola**: hay que decidir en qué se gasta

El XR06CX tiene **una** entrada digital (contacto limpio), configurable con `iF`:

| `iF` | Función | Efecto en las salidas |
|---|---|---|
| `EA` | Alarma genérica (muestra `EA`) | **Sin cambios** |
| `bA` | Alarma grave (muestra `CA`) | **Salidas apagadas** ← utilizable como OFF remoto |
| `do` | Switch de puerta (muestra `dA`) | Según `dC`: `no`/`Fn`/`cP`/`Fc` |
| `dF` | Inicia deshielo | Deshielo |
| `Au` | Inutilizado | — |
| `Hc` | Invierte la acción frío/calor | — |

Y `iP` define la polaridad (`oP` = activa al abrir, `cL` = activa al cerrar; **por defecto
`cL`**). El valor de fábrica de `iF` es **`EA`**, no `do`: en una instalación cualquiera, la
entrada digital probablemente **no** esté configurada como puerta aunque haya un fin de
carrera cableado. **Hay que leer `iF`, `iP`, `dC` y `rd` en cada controlador.**

**Decisión de diseño:** el gateway **aporta su propio sensor de puerta** y deja la entrada
digital del XR06 libre. Esto es mejor que compartirla por dos razones: no hay que inyectar
tensión en una entrada de contacto limpio ajena (riesgo real de dañar el controlador), y la
entrada queda disponible para el OFF remoto por `bA` si se elige ese camino.

## 5. ON/OFF remoto: lo que el manual habilita

**Importante: `ONF` no existe en este modelo.** No hay parámetro de habilitación de ON/OFF
en la lista (§7/§15) y el teclado documentado (§6) tiene solo `SET`, deshielo, ▲ y ▼. La
mención de una "tecla ON/OFF" en §11 (procedimiento de Hot Key) es texto de plantilla de
otro modelo de la familia — **a verificar en el equipo físico**. Todo el camino "bit de
stand-by por ModBUS" de las rev. 1 y 2 queda sin base.

| Camino | Cómo | Pro | Contra |
|---|---|---|---|
| **A1 (recomendado): cortar la alimentación del XR06** | Relé del gateway en serie con los 230 Vac del controlador (3,5 VA, §13). Sin tensión, los 3 relés abren y todas las cargas quedan OFF | OFF real y completo; deja la entrada digital libre para el sensor de puerta; eléctricamente simple | Se apaga el display; al restituir, hay que proteger el compresor con **`od` ≥ 3 min** (retardo de activación de salidas al arranque, §7) y tener en cuenta `dP` (deshielo al encendido) y `dA` (exclusión de alarma al arranque, 90 min por defecto) |
| **A2: entrada digital en `bA`** | El relé del gateway activa la entrada; el XR06 apaga las salidas y muestra `CA` | El controlador sigue alimentado, con display y medición | Consume la única entrada digital; el equipo queda mostrando una **alarma** (`CA`) mientras está "apagado", lo que confunde al operario; no se puede usar también como puerta |
| **Descartado: relé en serie con la bobina del contactor** | — | — | El XR06 sigue creyendo que el compresor está en marcha: al reponer, su relé ya está cerrado y el compresor arranca **sin el retardo anticiclos `AC`**. Riesgo de rotura de compresor |

**Fail-safe (sin cambios y ahora más importante):** contacto **NC**, relé sin energía =
XR06 alimentado = cámara refrigerando. Un gateway muerto nunca apaga una cámara.

**Parámetros que hay que dejar configurados en el XR06 al instalar (camino A1):**
`od` = 3 min (protección de compresor al reponer tensión), y registrar `dP` y `dA` para que
el gateway replique el enmascarado de alarmas tras un corte.

## 6. Falla de sondas: detección por firma de ciclado

El XR06CX no tiene relé de alarma, ni registro legible, así que `P1`/`P2` no se pueden leer.
Pero el manual dice exactamente qué hace el controlador cuando pierde la sonda:

> §3: "En caso de fallo en la sonda termostática el arranque y detención del compresor se
> calcula conforme lo programado en los parámetros `Cy` y `Cn`."
> §15: `Cy` = 15 min ON, `Cn` = 30 min OFF (valores por defecto).

Eso deja una **firma inconfundible en la salida de compresor**: un ciclado perfectamente
periódico de `Cy` ON / `Cn` OFF, **sin ninguna correlación con la temperatura** que mide la
sonda del gateway. La regla de detección es:

```
si  (ciclos de compresor con período ≈ Cy+Cn, tolerancia ±10 %, durante ≥ 3 ciclos)
y   (la temperatura propia no explica los arranques: no cruza SET±Hy)
→   alarma `probe_fail_inferida`, severidad crítica
```

Es más lenta que un flag (necesita ~2 horas con los valores por defecto) pero igual de
accionable, y no requiere ningún hardware extra: sale de la misma detección de salidas que
ya se usa para informar el estado del compresor. Lo mismo aplica a `P2`: con la sonda de
evaporador en falla, **el deshielo termina siempre por tiempo** (`Md` exacto, 30 min por
defecto) en lugar de por temperatura — un deshielo que *siempre* dura exactamente `Md` es la
firma de `P2` en falla.

Hay que ser explícito con el cliente: **detección en ~2 h en lugar de inmediata.** Y que
mientras tanto el producto está protegido igual, porque la alarma de temperatura propia sigue
vigilando el resultado.

## 7. Setup de temperatura remoto: las tres salidas honestas

| Salida | Qué da | Costo | Riesgo |
|---|---|---|---|
| **Consigna virtual hacia arriba** (por compuerta del camino A1) | Se configura el XR06 en el valor **más frío** requerido y el gateway, cortando y restituyendo su alimentación, mantiene la cámara a una temperatura **más alta** que la del controlador. Permite modo nocturno, cambio de producto y ahorro de energía | 0 | **El gateway pasa a ser parte del lazo de control.** Exige reglas de seguridad propias: tiempo mínimo ON/OFF, tope de tiempo de compuerta, watchdog local que libera la compuerta si el firmware se cuelga, y fail-safe NC. Es un cambio de principio arquitectónico y hay que asumirlo por escrito |
| **Reemplazo por controlador con RS-485** (camino B) | Consigna remota nativa, bidireccional, con todos los parámetros | +USD 60–120 por cámara + 1-2 h de instalación | Bajo. Es la opción correcta para quien exija esta función |
| **Emulación de Hot Key** (camino C) | Escritura de `SET` y de cualquier parámetro | I+D | Alto: protocolo propietario, reescribe **toda** la lista de parámetros, ~15 s de corte, desgaste de EEPROM, riesgo de dejar el controlador mal configurado. **No se vende hasta validarlo** |

> Se puede implementar la consigna virtual **solo hacia arriba**: cortando la alimentación
> del controlador se puede dejar de enfriar, nunca enfriar más de lo que su propio `SET`
> permite. Decirlo así, en una línea, evita una expectativa equivocada en la demo.

## 8. Otros datos del manual que mejoran el diseño

| Hallazgo | Dónde | Qué cambia |
|---|---|---|
| **`ot` y `oE`: calibración (offset) de sondas**, rango ±9,9 °C | §7 | Si el cliente tiene `ot ≠ 0`, el display muestra un valor corregido y **nuestra sonda independiente va a diferir**. **Hay que leer `ot` e introducirlo en el gateway** para que la app y el display coincidan. Sin esto, llamados a soporte garantizados |
| **El XR06 deshabilita las alarmas de alta y baja con la puerta abierta** | §8.1 | Confirma el enmascarado por puerta que ya tenía el diseño; ahora replica *exactamente* la lógica del controlador en lugar de inventar una |
| **Las alarmas `HA`/`LA` cesan cuando empieza un deshielo** | §12.1 | Ídem para el enmascarado por deshielo |
| **`dA` = 90 min: exclusión de alarma de temperatura al encendido** | §7, §15 | El gateway debe aplicar la misma ventana tras un corte de energía, o va a disparar alarmas falsas cada vez que vuelve la luz |
| **`Ad` = 15 min: retardo de alarma de temperatura** | §15 | El umbral y el retardo del gateway se **inicializan leyendo los del controlador**, para que no alarmen en momentos distintos |
| **`dF` = `it` por defecto: durante el deshielo el display congela la temperatura de inicio** | §7 | La app (con sonda propia) va a mostrar la subida real mientras el display muestra un valor fijo. **No es un error**: hay que explicarlo con el chip "en deshielo" y ponerlo en la guía rápida |
| **`td = in` (gas caliente): el compresor funciona durante el deshielo** | §4 | El análisis de duty cycle y de ciclado corto debe excluir los deshielos. `td` es dato de configuración obligatorio del gateway |
| **`dC` = `Fc` por defecto: con la puerta abierta se paran compresor y ventiladores** | §8.1, §15 | Las paradas por apertura de puerta no son fallas: hay que excluirlas del conteo de arranques |
| **`AC` = 1 min: retardo anticiclos cortos** | §15 | Es el valor que hace peligroso cortar la bobina del contactor (§5) y el que define el umbral real de "ciclado corto" |
| **`id` = 6 h, `Md` = 30 min, `dt` = 0, `Fd` = 10 min** | §15 | Valores base para las alarmas de `defrost_timeout` y `defrost_ineffective`, y para predecir el próximo deshielo en la app |
| **Sonda NTC −40÷110 °C, resolución 0,1 °C, precisión ±0,1 °C ±1 dígito** | §13 | Especificación objetivo de la sonda propia: NTC 10 k equivalente o sensor digital con ≤ ±0,5 °C, para no quedar peor que el controlador |
| **Relés: compresor 8(3) / 16(6) / 20(8) A; deshielo SPDT 8(3) A; ventiladores 8(3) o 5(2) A** | §13 | La detección de salidas se diseña para 230 Vac con corrientes de hasta 20 A en el circuito vigilado (se detecta **tensión**, no corriente: inmune al tamaño de la carga) |
| **Alimentación 230 Vac ±10 %, 3,5 VA** | §13 | Un relé chico sobra para el corte del camino A1, y el gateway puede tomar su alimentación de los mismos bornes |
| **Temperatura de operación del XR06: 0÷60 °C** | §13 | El controlador no puede ir dentro de la cámara. Confirma que el gateway va en el tablero y la sonda afuera |
| **"No debe ser usado como un dispositivo de seguridad"** | §1.1 | Lo dice el fabricante: sostiene la arquitectura (el gateway no es lazo de seguridad) y conviene citarlo en los términos del servicio |
| **"En aplicaciones industriales, el uso de filtros (FT1) en paralelo con cargas inductivas puede resultar útil"** | §1.2 | Respalda los snubbers RC del diseño y se puede ofrecer el filtro en la instalación |
| **`Pt` (mapa de códigos) y `rL` (versión de software), solo lectura** | §7 | **Van al protocolo de relevamiento:** identifican la variante exacta de firmware, y son el dato que hay que darle a Emerson para preguntar por ModBUS |
| **Montaje en panel 71×29 mm, frontal IP65, caja IP20** | §9, §13 | El controlador vive en la puerta del tablero; el gateway en riel, detrás. Confirma el formato del producto |

## 9. Corrección de nomenclatura (las rev. 1 y 2 usaban nombres equivocados)

Los nombres de parámetros de los documentos anteriores venían de otras familias Dixell. Los
reales del XR06CX son:

| Usado antes (**incorrecto**) | Real en XR06CX | Qué es |
|---|---|---|
| `SET` | `SET` ✔ | Consigna (por teclado, **no por comunicación**) |
| `Hy` | `Hy` ✔ | Diferencial |
| `LS` / `US` | `LS` / **`uS`** | Límites de consigna |
| `ALU` / `ALL` | **`AU`** / **`AL`** | Alarma de máxima / mínima temperatura |
| `ALd` | **`Ad`** | Retardo de alarma de temperatura (15 min) |
| — | **`dA`** | Exclusión de alarma al encendido (90 min) |
| `idF` | **`id`** | Intervalo entre deshielos (6 h) |
| `MdF` | **`Md`** | Duración máxima de deshielo (30 min) |
| `dOA` | **`di`** | Retardo de alarma de la entrada digital (5 min) |
| `i1F` | **`iF`** | Configuración de la entrada digital (`EA`/`bA`/`do`/`dF`/`Au`/`Hc`) |
| — | **`iP`** | Polaridad de la entrada digital (`cL` por defecto) |
| `ONF` | **no existe** | — |
| `P2P` | **`P2`** | Presencia de sonda de evaporador |
| `AH1` / `AL1` | **`HA`** / **`LA`** | Mensajes de alarma de alta / baja temperatura |
| — | **`CA`** / `EA` / `dA` | Alarma grave / genérica / puerta abierta |
| — | **`Cy`** / **`Cn`** | Tiempos ON/OFF del compresor con sonda en falla (15/30 min) |
| — | **`ot`** / **`oE`** | Calibración de sondas |
| — | **`td`**, `dE`, `dd`, `dt`, `dP`, `dF` | Parámetros de deshielo |
| — | **`FC`**, `Fd`, `FS` | Control de ventiladores |
| — | **`dC`**, `rd` | Comportamiento con puerta abierta |
| — | **`Pt`**, `rL`, `d1`, `d2` | Solo lectura: mapa de códigos, versión, sondas |

Los documentos 01 a 08 y el perfil YAML quedaron actualizados con esta nomenclatura.
