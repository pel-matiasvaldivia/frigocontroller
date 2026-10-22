# 13 — Inventario de controladores del cliente

Dos modelos declarados por el cliente. **Son dos casos distintos y cada uno va por un camino
distinto**, lo que confirma que la estrategia de dos niveles era la correcta.

| | **Controlador A** | **Controlador B** |
|---|---|---|
| Marca | Copeland | Emerson / Copeland |
| Modelo | **XR06CH-5GACILS** | **XR75CX-5N7I3** |
| Versión de firmware (`rL`) | **V1.1** | **V2.6b** |
| Camino | **A — envolvente** (FC-A) | **B — ModBUS**, muy probablemente |
| Producto | FrigoController FC-A | **XR75CX + RTU SmartPath M3** (ya existe) |

## 1. XR06CH-5GACILS — envolvente, sin ModBUS

Es el caso que ya veníamos diseñando. El `CH` en lugar de `CX` es una variante de la misma
familia; según la ficha de un distribuidor, el **XR06CH** es 230 V, recorte de panel
**71 × 29 mm**, sonda NTC, rango −40 a +110 °C, **3 relés**, display de 3½ dígitos,
**frontal IP54**, y **sin reloj ni zumbador interno**. Mismo esquema funcional que el XR06CX
cuyo manual analizamos: 3 relés (compresor, ventiladores, deshielo), 2 sondas NTC y **una**
entrada digital configurable.

**No hay evidencia de RS-485 ni de parámetro de dirección.** Todo el análisis del
[doc 10](10-hallazgos-manual-xr06cx.md) aplica tal cual, y la solución es el FC-A: sonda
propia, detección de tensión en las tres salidas de relé y relé NC para el ON/OFF.

### El detalle que hay que mirar: `5GACILS` no es un código estándar

Los códigos de pedido de esta familia tienen 5 caracteres (`5N0C1`, `4N1F1`, `5R0C1`).
**`5GACILS` tiene siete**, lo que apunta a una **variante OEM con juego de parámetros
personalizado de fábrica** — algo habitual cuando el controlador viene dentro de un equipo de
marca. Consecuencia práctica, y es importante:

> **No se pueden dar por buenos los valores por defecto del manual.** `AU`, `AL`, `Ad`, `dA`,
> `Cy`, `Cn`, `iF`, `iP`, `dC`, `td`, `id`, `Md` pueden venir de fábrica con otros valores.
> El asistente de instalación ya los pide uno por uno leyéndolos del display
> ([doc 06 §9](06-diseno-industrial.md)); con esta variante **eso deja de ser una buena
> práctica y pasa a ser obligatorio**. Conviene además anotar `Pt` (mapa de códigos), que
> identifica el juego de parámetros, y `rL` = V1.1.

Dos consecuencias menores más: el **frontal IP54** (no IP65) manda cuidado si la cámara se
lava con manguera cerca del tablero, y la **falta de zumbador interno** refuerza el valor del
buzzer del FC-A y de la opción **+2R** de sirena: en estas cámaras, hoy, **una alarma no hace
ruido en ningún lado.**

## 2. XR75CX-5N7I3 — el candidato a nivel 2, con una verificación pendiente

Esta es la buena noticia: **el cliente ya tiene instalado un controlador de la familia que
integramos en el proyecto SmartPath M3** ([doc 12](12-plataforma-smartpath-m3.md)).

La familia XR75CX es claramente superior al XR06: hasta **4 sondas NTC**, **4 relés**,
**2 entradas digitales**, y fichas de distribuidor de varias variantes la describen como
conectable a redes **"MODBUS-RTU compatible, such as the monitoring units of XWEB family"**.
Es, en esencia, el hermano `CX` del **XR75CH** cuyo diagrama de bornes (en el manual del M3)
muestra **RS485 en los bornes 16/17/18**.

### Lo que falta confirmar: si esta variante trae RS-485 a bordo o solo TTL

Dentro de la misma familia, **el sufijo decide**, y la evidencia pública es contradictoria:

| Código | Cómo lo describe el distribuidor |
|---|---|
| XR75CX-5N6**1**3, XR75CX-4S7**C**3 | "RS-485 serial output… ModBUS-RTU compatible (XWEB)" |
| XR70CX-5N0**I**3 | **"salida serie TTL"** |
| **XR75CX-5N7I3** (el del cliente) | **sin ficha encontrada** |

El carácter en esa posición del código del cliente es **`I`**, el mismo que en el único
listado que aparece como **TTL**. No alcanza para concluir —son modelos y distribuidores
distintos, y los datos de catálogo de este rubro ya nos fallaron una vez— pero **hay que
verificarlo antes de prometer nada.**

### Verificación de 10 minutos, sin herramientas ni costo

1. **Mirar la bornera del controlador.** Si tiene **bornes 16 y 17 rotulados `RS485+ / −`**
   (como el XR75CH del manual del M3), **tiene ModBUS nativo y esa cámara entra directo al
   nivel 2**. Si solo está el conector **Hot Key / X-Rep**, hace falta el **XJ485-CX**
   (+USD 35-55). **Una foto de la bornera y de la etiqueta resuelve la duda.**
2. **Buscar el parámetro de dirección** en el menú (`Adr`). Si existe, es esclavo ModBUS.
   Si no aparece, no lo es — es la misma prueba que descartó al XR06CX.
3. **Anotar `Pt` y `rL`** (ya sabemos: V2.6b).

### Prueba de banco, media hora, apenas confirmado lo anterior

Con `Adr = 1` y **9600 baudios**, igual que en la puesta en marcha del M3. Dos hipótesis a
probar con `mbpoll`, tomadas de documentación de terceros sobre esta familia (no de Emerson,
así que son hipótesis, no verdades):

- **Temperatura de sonda 1 en el holding register 256** (función 0x03).
- **Tabla de parámetros a partir del holding register 768**, en **el mismo orden en que los
  lista el manual de usuario** — si se cumple, el mapa sale de leer el manual en orden, sin
  descubrir nada.
- El formato serie suele ser **9600 8N1** en esta familia (el M3 usa 9600).

> Y antes que nada: **preguntarle al equipo del M3 si su mapa del XR75 ya cubre esta
> variante.** V2.6b es un firmware concreto y los mapas cambian entre versiones, pero si el
> M3 habla con un XR75 hoy, el 90 % del trabajo está hecho.

## 3. Qué cambia en la propuesta

| Antes de conocer los modelos | Ahora |
|---|---|
| "El parque es XR06CX, no comunica: envolvente para todo" | **Parque mixto.** Las cámaras con **XR75CX** son candidatas a nivel 2 **con la RTU que ya existe**; las de **XR06CH** van con FC-A |
| El nivel 2 requería **reemplazar** el controlador (USD 60-120 + 1-2 h de obra) | En las cámaras con XR75CX **no hay que reemplazar nada**: el controlador ya está. El upgrade es solo la RTU y, si hiciera falta, el XJ485 |
| Nivel 2 = USD 290-390 por cámara | **Nivel 2 sobre XR75CX existente ≈ RTU + 0 a 55 de adaptador**, sin corte de servicio y sin cambiar el controlador. Es **mucho** más barato y más fácil de vender |

**El relevamiento cámara por cámara pasa a ser lo más rentable que podemos hacer esta
semana.** Cada cámara con XR75CX es una venta de nivel 2 sin reemplazo de controlador; cada
una con XR06CH es un FC-A. Hasta no tener ese conteo, no se puede cotizar el proyecto.

## 4. Lo que hay que pedirle al cliente (y entra en un mensaje)

1. **Foto de la bornera del XR75CX** (parte de atrás, con los números de borne visibles) y
   **foto de la etiqueta** con el código completo.
2. **Cuántas cámaras tienen cada modelo**, y en cuáles se necesita consigna remota.
3. En el XR75CX, **si existe el parámetro `Adr`** y qué valor tiene.
4. En ambos, el valor de **`Pt`** y la lista de parámetros tal como están configurados hoy
   (la del XR06CH especialmente, por ser variante OEM).
5. **Si alguna cámara ya tiene cableado el fin de carrera de puerta**, y a qué entrada.

Con los puntos 1 y 2 ya se puede armar la cotización: definen cuántos FC-A, cuántas RTU y
cuántos XJ485.

## 4.1 Diagramas de bornera verificados

Los diagramas reales de ambas familias, transcritos de la documentación oficial, están en
**[doc 15](15-verificacion-borneras.md)** — junto con la advertencia sobre un esquema de
pinout incorrecto que conviene no usar.

## 5. Fuentes

- [XR06CH, ficha de distribuidor](https://www.gev-online.com/en/webshop/product/378753) (230 V, 71×29 mm, NTC, 3 relés, IP54, sin zumbador)
- [XR06CX-5N0C1, ficha de distribuidor](https://www.ricambi.com.au/xr06cx-5n0c1-digital-controller-dixell) (3 relés, 2 NTC, 1 entrada digital)
- [XR75CX-5N613](https://www.ricambi.com.au/xr75cx-5n613-digital-controller-dixell) y [XR75CX-4S7C3](https://www.ricambi.com.au/xr75cx-4s7c3-digital-controller-dixell): hasta 4 sondas NTC, red ModBUS-RTU compatible con XWEB
- [XR70CX-5N0I3, descripto con salida serie TTL](https://www.ricambi.com.au/xr70cx-5n0i3-digital-controller-dixell)
- [Notas de integración ModBUS de la familia Dixell (terceros)](https://docs.webhmi.com.ua/dixell): dirección en el parámetro `Adr`, formato fijo 9600 8N1, tabla de parámetros desde **HR768** en el orden del manual, y advertencia de que varios módulos **no** traen driver RS-485 integrado
- [Hilo sobre registros ModBUS del XR75CX (terceros)](https://control.com/forums/threads/emerson-dixell-xr75cx-modbus-registers.50566/post-211553): lectura de temperatura P1 en el **holding register 256**
- Manual de la RTU SmartPath M3 y diagrama de bornes del XR75CH (RS485 en bornes 16/17/18): [docs/manuales/](manuales/SmartPath-M3-manual-usuario.pdf)
