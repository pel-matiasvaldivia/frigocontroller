# 14 — Qué hace falta para conectar cada combistato a la RTU

Respuesta operativa, por modelo. El cliente es **revendedor/integrador**: compra estos dos
controladores y quiere ofrecer el monitoreo como servicio adicional a *sus* clientes. Por eso
esto está armado como **kits estandarizados**, no como soluciones a medida.

## Resumen en una tabla

| | **XR75CX-5N7I3** | **XR06CH-5GACILS** |
|---|---|---|
| ¿Habla ModBUS? | **Sí** (familia XR75, la que ya integra la RTU) | **No.** Sin puerto serie de proceso ni parámetro de dirección |
| ¿Se conecta a la RTU? | **Sí, directo** | **No se puede, tal cual está** |
| Qué hace falta | 2 hilos de RS-485, fuente de 12 V para la RTU, fijar `Adr` y velocidad — **y el XJ485 solo si esa variante no trae la bornera RS-485** | Una de tres opciones (§3): reemplazar el controlador, o un adaptador que lo haga hablar ModBUS, o monitoreo sin RTU |
| Costo de materiales (sin RTU) | **USD 20-28** · ó USD 60-80 si necesita XJ485 | USD 60-120 (reemplazo) ó ≈ USD 26 de costo (adaptador FC-A) |
| Tiempo de instalación | 30-40 min | 45 min (reemplazo) · 25 min (adaptador) |

---

## 1. XR75CX — conexión directa

### 1.1 Primero, la verificación que define el kit (10 minutos, sin costo)

Dentro de la familia XR75 hay variantes **con bornera RS-485** y variantes **solo con el
puerto TTL de la Hot Key**. El sufijo del código decide y la evidencia pública es
contradictoria ([doc 13 §2](13-inventario-controladores-cliente.md)), así que se verifica en
el equipo:

| Qué mirar | Resultado |
|---|---|
| **Bornes 16 y 17 rotulados `RS485+ / RS485−`** en la bornera de señal | **Kit A.** ModBUS nativo, sin adaptador |
| Solo conector **Hot Key / X-Rep**, sin bornes 16-18 | **Kit B.** Hace falta XJ485-CX |
| **¿Existe el parámetro `Adr` en el menú?** | Si existe, es esclavo ModBUS. Si no aparece, no lo es |

Una foto de la bornera y una del menú de parámetros resuelven las dos preguntas.

### 1.2 Kit A — XR75CX con bornera RS-485

| # | Ítem | Cant. | USD |
|---|---|---|---|
| 1 | **Fuente 12 VDC riel DIN**, 15 W (la RTU es de 12 V y el combistato de 230 V) | 1 | 12-18 |
| 2 | Cable **apantallado 2 × 0,5 mm² + malla** para el bus (A/B) | 2-5 m | 2-4 |
| 3 | Resistencia de terminación **120 Ω** (si el bus queda largo o con varios nodos) | 1-2 | ~0 |
| 4 | Bornes, prensacables, riel y canaleta | — | 3-5 |
| 5 | **SIM M2M** (la provee la RTU) + verificación de señal | 1 | — |
| | **Total materiales, sin RTU** | | **≈ 20-28** |

**Conexionado:**

```
   XR75CX                              RTU SmartPath M3
 ┌──────────┐                        ┌──────────────────┐
 │ 16 ──────┼── A (+) ───────────────┤ RS485 +          │
 │ 17 ──────┼── B (−) ───────────────┤ RS485 −          │
 │ 18 ──────┼── GND/malla ─ ─ ─ ─ ─ ─┤ (malla a tierra  │
 │          │                        │  SOLO de un lado)│
 └──────────┘                        │ 12V  GND ────────┼── Fuente 12 V DIN
   230 Vac                           └──────────────────┘
```

> Según el diagrama oficial del XR75CH, **borne 16 = `RS485 +` y borne 17 = `RS485 −`**
> ([doc 15 §2](15-verificacion-borneras.md)). **Confirmar la serigrafía del equipo antes de
> cablear:** la numeración varía entre sub-variantes del mismo modelo, y circula por ahí
> material con pinouts incorrectos (doc 15 §3). Invertir + y − no rompe nada, pero no
> comunica, y es la causa nº 1 de "no enlaza".

**Parámetros a fijar en el controlador** (menú del frente):

| Parámetro | Valor | Por qué |
|---|---|---|
| `Adr` (dirección) | **1** | Es lo que el manual de la RTU indica para el enlace |
| Velocidad | **9600** | Ídem. En esta familia suele ser fija 9600 8N1 |

**Puesta en marcha:** energizar, y el **LED MODBUS de la RTU debe quedar encendido fijo**
(parpadeando = no encuentra el dispositivo). Después, LED COM según el manual: 4 parpadeos =
sesión de datos OK; encendido = 2G; encendido con un parpadeo = 4G.

### 1.3 Kit B — XR75CX sin bornera RS-485 (solo TTL)

Lo mismo que el Kit A, más:

| # | Ítem | USD |
|---|---|---|
| 6 | **Adaptador Dixell XJ485-CX** + cable **CAB/RS** | 35-55 |

El XJ485 va entre el puerto TTL del controlador y el bus RS-485 de la RTU. Es el módulo
oficial de la marca, así que no hay riesgo de garantía.

### 1.4 Lo que falta del lado del software (no del cableado)

Esto es lo que hay que resolver antes de prometer la fecha, y no se ve en el tablero:

1. **El mapa de registros para el firmware V2.6b.** Los mapas cambian entre versiones.
   **Primera pregunta al equipo de la RTU: ¿el mapa que usan hoy cubre esta variante?** Si la
   RTU ya habla con un XR75, lo más probable es que sí o casi.
2. **Si no lo cubre**, la validación es de media hora en banco con `mbpoll`, con dos
   hipótesis documentadas para esta familia: temperatura de sonda 1 en el **holding register
   256**, y tabla de parámetros desde el **HR768 en el mismo orden del manual de usuario**.
3. **Deshielo y compresor en la app.** Hoy la RTU no los muestra ([doc 12 §3](12-plataforma-smartpath-m3.md)),
   pero el XR75CX los tiene en sus registros de estado: es trabajo de firmware y pantalla, no
   de hardware.

---

## 2. XR06CH — no se puede conectar a la RTU tal cual está

Hay que decirlo sin vueltas: **el XR06CH no tiene puerto serie de proceso ni parámetro de
dirección.** No existe cable, adaptador ni configuración que lo haga hablar ModBUS. El XJ485
tampoco sirve: convierte TTL a RS-485, pero lo que corre por el TTL de este equipo es la
transferencia de la Hot Key, no ModBUS.

---

## 3. Tres opciones para las cámaras con XR06CH

### Opción 1 — Reemplazar el controlador por un XR75CX *(la más simple comercialmente)*

**El cliente ya compra los dos modelos.** Para las cámaras donde el usuario final contrate el
servicio, se instala directamente el XR75CX en lugar del XR06CH.

| | |
|---|---|
| Costo | Controlador USD 60-120 + Kit A (USD 20-28) |
| Instalación | **≈ 45 min.** Los dos son **formato 32 × 74 con recorte 71 × 29 mm**: se reusa el agujero del tablero. Las sondas son **NTC en los dos**: se reusan. Los relés del XR75CX tienen igual o mayor capacidad |
| Ventaja | Cero desarrollo, el instalador no aprende nada nuevo, y el usuario final queda con un controlador mejor (4 sondas, 4 relés, 2 entradas digitales, tecla de ON/OFF) |
| Desventaja | Hay que cortar el servicio un rato y tocar el cableado de carga. Y hay que reconfigurar los parámetros: **el XR06CH es variante OEM (`5GACILS`), así que hay que copiar su configuración actual antes de desmontarlo** |

**Recomendación comercial, y es la más importante de este documento:** que el cliente
**estandarice en XR75CX toda compra nueva** donde el usuario final pueda querer el servicio.
La diferencia de precio entre los dos modelos es mucho menor que el costo de resolver después
la falta de comunicación, y evita el trabajo de reemplazo.

### Opción 2 — FC-A como adaptador ModBUS *(la que escala, y la que da producto propio)*

El gateway FC-A que diseñamos ([doc 02](02-hardware.md)) mide el XR06CH por fuera: sonda
propia, detección de tensión en las tres salidas de relé y relé de corte para el ON/OFF.
**La idea nueva es que publique esos datos en el bus RS-485 como un esclavo ModBUS, usando el
mismo mapa de registros del XR75 que la RTU ya sabe leer.**

```
 XR06CH ──[sonda propia]──▶┌────────┐
        ──[3 salidas CA]──▶│  FC-A  │── A/B ──▶ RTU SmartPath ──▶ app (sin cambios)
        ◀─[relé de corte]──│ esclavo│
                           │ ModBUS │
                           └────────┘
```

| | |
|---|---|
| Qué logra | **La RTU y la app funcionan sin ningún cambio.** Para ellas, la cámara con XR06CH es indistinguible de una con XR75CX |
| Costo | ≈ USD 26 de costo de equipo (doc 07), sin reemplazar el controlador ni cortar el servicio |
| Instalación | ≈ 25 min, sin tocar el cableado de potencia |
| Desarrollo | Firmware del FC-A **+ pila de esclavo ModBUS** (es poco trabajo: la parte difícil, el mapa, ya existe del lado de la RTU) |
| Límites | Sin consigna remota (el XR06CH no se puede escribir) y falla de sonda por inferencia, no por flag. Las dos limitaciones son del controlador, no del adaptador |

**Es la opción que convierte el problema en producto.** El cliente podría ofrecer el servicio
sobre **todo el parque instalado de sus propios clientes**, sin pedirles cambiar el
controlador — que es, justamente, la objeción que mata este tipo de venta.

### Opción 3 — FC-A con nuestra propia nube, sin RTU

El FC-A tiene Wi-Fi y su propio backend y app ([doc 05](05-nube-y-app.md)). Sirve donde hay
Wi-Fi confiable y evita el módem celular y el abono de datos. **Desventaja:** son dos
plataformas y dos apps conviviendo, lo que el cliente va a sufrir en soporte. Lo dejo como
alternativa, no como recomendación: si la RTU ya existe y el cliente ya la presentó a sus
clientes, **conviene una sola app**, y para eso están las opciones 1 y 2.

---

## 4. Lo que hace falta en los dos casos (y se suele olvidar)

| Ítem | Detalle |
|---|---|
| **Alimentación de 12 V para la RTU** | Los combistatos son de 230 V y la RTU de 12 V: hace falta **una fuente DIN de 12 V por tablero**. No está en el kit de la RTU y es el ítem que más veces falta en la caja |
| **Cobertura celular medida dentro del tablero** | La RTU tiene **antena interna**. Medir señal con la puerta del tablero **cerrada**, no en la vereda. Si el tablero es metálico, prever antena externa o reubicar la RTU |
| **Bus RS-485 bien hecho** | Par trenzado apantallado, malla a tierra **de un solo lado**, lejos de cables de potencia y cruzándolos a 90°, 120 Ω en los extremos si el bus es largo |
| **Una RTU por controlador, o varias direcciones** | El manual de la RTU indica fijar la dirección en **1**, lo que sugiere un dispositivo por RTU. **Pregunta clave al equipo de la RTU: ¿puede consultar varias direcciones?** Si puede, un tablero con 3 cámaras se resuelve con **una** RTU y tres controladores en el mismo bus — divide por tres el costo por cámara |
| **Sensor de puerta** | El XR75CX tiene **2 entradas digitales**: una puede ser la puerta (`iF = do`). En el XR06CH hay **una sola**, por eso el FC-A aporta la suya |
| **Parámetros actuales del controlador** | Anotarlos **antes** de tocar nada, sobre todo en el XR06CH por ser variante OEM. Son la base de los umbrales de alarma |

---

## 5. Decisiones que dependen del cliente (y que no puedo cerrar sin él)

1. **¿La variante XR75CX-5N7I3 trae bornera RS-485?** → define Kit A o Kit B (USD 45 de
   diferencia por cámara). **Una foto lo resuelve.**
2. **¿El mapa de registros de la RTU cubre el firmware V2.6b?** → define si hay media hora de
   banco o nada de trabajo.
3. **¿La RTU puede consultar varias direcciones ModBUS?** → define si es una RTU por cámara o
   una por tablero. Es la variable que más mueve el costo por cámara.
4. **Para el parque XR06CH: ¿opción 1 o 2?** → reemplazo (sin desarrollo, con corte de
   servicio y costo de controlador) o adaptador FC-A (con desarrollo, sin tocar nada). Mi
   recomendación: **las dos**, porque resuelven casos distintos — reemplazo en instalaciones
   nuevas y en equipos a renovar, adaptador para el parque ya instalado de sus clientes.

## 6. Lo que yo haría esta semana

1. Pedir **la foto de la bornera del XR75CX** y cerrar la pregunta del RS-485.
2. Preguntarle al equipo de la RTU por el **mapa V2.6b** y por el **multi-dirección**.
3. Armar **un Kit A completo** sobre un XR75CX de banco y hacerlo enlazar con la RTU de punta
   a punta, hasta ver el dato en la app. Es la prueba que convierte todo esto en una
   cotización firme, y con un controlador que el cliente ya tiene comprado.
4. En paralelo, **cotizarle al cliente la estandarización en XR75CX** para toda venta nueva:
   es la decisión que más le simplifica el servicio a futuro.
