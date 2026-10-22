# 12 — La plataforma SmartPath M3 y el XR75CH

Análisis del proyecto **SmartPath M3**, la RTU ModBUS/celular presentada a este mismo cliente
el año pasado ([manual](manuales/SmartPath-M3-manual-usuario.pdf)), y qué cambia en el diseño.

## 1. Dos hallazgos, y el segundo es el importante

### 1.1 El XR75CH tiene RS-485 ModBUS **a bordo**

El diagrama de bornes del manual lo muestra sin ambigüedad: **los bornes 16, 17 y 18 están
rotulados `RS485`**, y el conector **Hot Key / X-Rep está aparte**, abajo a la derecha. No hay
XJ485 en el diagrama ni en el conexionado, y el texto del manual confirma el enlace directo:

> *"La RTU necesita solo alimentacion de 12V, y el conexionado al dispositivo (combistato) a
> traves de su interface RS485. […] el dispositivo debe ser programado con la direccion o
> ADDRESS en 1, y la velocidad de comunicación de 9600 baudios."*

Es decir: **dentro de la misma familia Emerson/Dixell que el cliente ya usa existe un modelo
comunicable, con RS-485 nativo, sin adaptador y sin los USD 35-55 por punto.** Y tiene
parámetro de dirección — la prueba que le faltaba al XR06CX.

El resto del diagrama describe un controlador bastante más capaz que el XR06CX:
**4 relés** (Fan, Comp, Light, Def), **2 entradas digitales** (D.I.1, D.I.2), **4 sondas**
(Room, Evap., Pb3, Pb4), alimentación 230 V, contactos de 16/20 A y 8(3) A, y salida X-Rep
para display remoto. En la foto del frente se ve además **una tecla de encendido/apagado
(⏻)**, lo que explica que el ON/OFF remoto estuviera en el alcance de aquel proyecto.

### 1.2 Ya tenemos construido el nivel 2

El M3 no es un prototipo: es un producto con manual de usuario, certificado de garantía,
app Android/iOS y versión web. Según su manual, ya entrega sobre el XR75CH:

| Del combistato | De la propia RTU |
|---|---|
| Temperatura sonda 1 | Alimentación principal (conexión/desconexión) |
| Conexión/desconexión de sonda 1 y de sonda 2 | Batería baja y nivel de batería |
| Buzzer | Nivel de señal |
| Estado de entrada digital 1 | **Alarma por alta y baja temperatura** |
| **Set point de temperatura** | Reporte periódico de status y temperaturas |
| **Encendido/apagado del combistato (ON/OFF)** | |
| Conexión/desconexión al BUS ModBUS | |

**Escritura de consigna y ON/OFF remoto ya funcionan.** Son exactamente las dos funciones que
el XR06CX no puede dar y que motivaron todo el camino B del doc 10.

Características técnicas del M3: 12 VDC, **respaldo LiPo 3,7 V con 4-6 h de autonomía**,
100 mA promedio (160 mA en pico de transmisión), módulo **GSM 2G / LTE-M pentabanda**,
NanoSIM, antena interna, RS-485, −20 a 65 °C, formato riel DIN.

## 2. Corrección al doc 11: Dixell vuelve, y primero

En [doc 11 §3.4](11-controladores-nivel2.md) descarté la familia Dixell para el nivel 2, con
dos argumentos públicos: que necesita XJ485 y que Emerson no publica los registros. **Los dos
caen con esta evidencia:**

| Argumento del doc 11 | Qué muestra el M3 |
|---|---|
| "Necesita XJ485-CX, +USD 35-55" | **El XR75CH tiene RS-485 en los bornes 16/17/18.** El modelo importa: el XR06CX y el XR60CH no lo tienen, el XR75CH sí. La búsqueda pública me llevó a los modelos equivocados |
| "Los registros son información propietaria" | Sigue siendo cierto *públicamente*, pero **nosotros ya los tenemos**: el M3 lee y escribe esos registros hace un año. El mapa es un activo interno, no un riesgo |

**Ranking corregido para el nivel 2:**

| | Controlador | Por qué |
|---|---|---|
| **1** | **Dixell XR75CH** | RS-485 a bordo · misma marca que el parque instalado · **integración ya probada y en producción** · el instalador no aprende nada nuevo · mapa de registros en casa |
| 2 | Danfoss ERC 213 | Mapa publicado, pero requiere adaptador TTL (+USD 35-55) y driver de ≥ 50 mA |
| 3 | Carel IR33+ | RS-485 a bordo, pero sale de catálogo |
| — | Full Gauge | Fuera: protocolo propietario Sitrad |

## 3. Qué le falta al M3 frente a los 8 requerimientos de hoy

| Requerimiento | M3 hoy | Qué hace falta |
|---|---|---|
| ON/OFF remoto | ✅ | — |
| Setup de temperatura | ✅ | — |
| Alarmas de alta y baja | ✅ | — |
| Error de comunicación | ✅ (pérdida de BUS) | — |
| Falla de sondas | ✅ (conexión/desconexión de sonda 1 y 2) | — |
| Aperturas de puerta | ⚠️ parcial | El M3 informa el **estado** de la entrada digital 1. Falta **contar** aperturas y acumular tiempo abierto, y que la DI esté configurada como puerta (`iF = do`) |
| **Indicación de deshielo** | ❌ no está en la app | **Está en el controlador:** el XR75CH tiene relé de deshielo y registros de estado. Es trabajo de firmware y app, **no de hardware** |
| **Funcionamiento del compresor** | ❌ no está en la app | Ídem: relé de compresor en el mismo registro de estado |
| **Humedad** | ❌ | Ningún XR mide HR. Hace falta sonda propia en la RTU — el M3 no tiene entrada de sonda |

**La conclusión práctica es muy favorable:** de los tres faltantes, **dos se resuelven leyendo
registros que el XR75CH ya expone y mostrándolos en la app.** Son horas de firmware y de
pantalla, no un producto nuevo. Solo la humedad necesita hardware.

## 4. Preguntas al equipo del M3 antes de decidir

Son las que convierten este hallazgo en un plan:

1. **El mapa de registros del XR75 que usa el M3.** ¿Incluye la palabra de estado con los bits
   de compresor, deshielo y ventiladores? Si sí, deshielo y compresor son una tarde de trabajo.
2. **¿El M3 guarda registro local cuando se cae la red?** El manual no menciona buffer offline.
   Para HACCP hacen falta 15 min × 2 años, y sin eso no se puede vender trazabilidad.
3. **¿Qué son los bornes `C`, `IN`, `S1`, `S2`** de la bornera? Si hay una entrada digital y
   salidas libres, el sensor de puerta propio y la sirena ya estarían resueltos.
4. **¿Hay entrada analógica o I²C libre** para la sonda de humedad, o hay que hacer revisión de
   hardware?
5. **Costo del equipo, costo del abono de datos y stock disponible.**
6. **Estado de la relación con SmartPath S.A.S.** (el certificado de garantía está a su
   nombre): ¿es producto propio, de un socio, o licenciado? Define si podemos evolucionar el
   firmware o solo integrarlo.

## 5. Observaciones de diseño sobre el M3

Dichas como insumo para la próxima versión, no como crítica del producto:

| Observación | Por qué importa | Sugerencia |
|---|---|---|
| **Antena interna** | Es la falla de campo nº 1 en tableros metálicos y cámaras. El manual aclara que hace falta "disponibilidad de red de telefonía celular en la zona", pero el problema suele ser el gabinete, no la cobertura del barrio | Opción de antena externa SMA; el relevamiento debe medir señal **dentro del tablero cerrado** |
| **Batería LiPo en un tablero** | El equipo declara operación hasta 65 °C. Una LiPo a 60 °C permanentes pierde capacidad rápido y es el componente que más envejece | Verificar la química y la estrategia de carga; evaluar LiFePO₄ o supercapacitor + apagado ordenado si solo se busca sobrevivir al corte y avisar |
| **Solo celular** | USD ~28 de módem + abono de datos por equipo. Donde hay Wi-Fi confiable, es costo evitable | Variante Wi-Fi para instalaciones con red, manteniendo LTE para sitios sin ella (es exactamente la variante FC-E LTE de nuestro diseño, al revés) |
| **Garantía de 6 meses** | Para un producto vendido con abono mensual, 6 meses es corto: el cliente paga un servicio y el equipo que lo sostiene queda sin cobertura | Extender la garantía mientras el abono esté activo. Es un argumento de venta, no un costo: el equipo ya está desplegado y monitoreado |
| **No se menciona trazabilidad / reportes** | Es la mitad del valor en alimentos (auditoría, HACCP) | Agregar agregados de 15 min con retención de 2 años y reporte PDF/CSV (doc 05) |
| **Dirección ModBUS fija en 1** | Un bus, un controlador. Si en un tablero hay dos o tres cámaras, hay una RTU por cámara | Soportar varias direcciones por RTU: un solo equipo para 2-4 controladores baja mucho el costo por cámara |

## 6. Decisión de plataforma

El cliente tiene dos realidades distintas y cada una tiene su respuesta:

| Escenario | Solución | Desarrollo |
|---|---|---|
| **Parque XR06CX existente, sin cambiar el controlador** | **FrigoController FC-A** (envolvente, doc 02) | Producto nuevo, el del resto de este dossier |
| **Cámaras que necesiten consigna remota** | **XR75CH + SmartPath M3** | **Ninguno.** Ya existe y ya se presentó a este cliente |
| Cámaras con consigna remota **y** Wi-Fi disponible | XR75CH + FC-R (población RS-485 de nuestra PCB) | Bajo: es la población que ya está prevista |

**Recomendación: no desarrollar de cero lo que ya está hecho.** El nivel 2 se cotiza con
XR75CH + M3 desde la semana uno, y el esfuerzo de ingeniería se concentra donde no hay nada:
la envolvente del parque XR06CX, que es además la mayoría de las cámaras del cliente.

Las tres mejoras del M3 que conviene hacer igual, por orden de rentabilidad:
**(1)** deshielo y compresor en la app — dos registros que ya se leen;
**(2)** buffer offline y reportes HACCP — habilita vender trazabilidad;
**(3)** varias direcciones ModBUS por RTU — divide el costo por cámara en tableros con
varias cámaras.

## 7. Lo que esto cambia en la propuesta comercial

El cliente ya vio el M3 el año pasado. La propuesta de ahora no lo contradice: lo **completa**,
y conviene presentarlo así.

> *"Lo que les mostramos el año pasado resuelve control remoto completo, y sigue siendo la
> respuesta correcta — pero requiere el XR75CH. Sus cámaras tienen XR06CX, que no comunica:
> por eso desarrollamos un equipo que monitorea y apaga/enciende sin cambiar el controlador.
> El resultado es un parque completo monitoreado desde el primer mes, y el control de consigna
> donde realmente haga falta, con el equipo que ya conocen."*

Es una posición mucho más fuerte que la de volver con un producto nuevo que ignora el
anterior: **explica por qué el proyecto del año pasado no se podía aplicar tal cual a su
instalación**, que es probablemente la razón por la que no avanzó.

## 8. A verificar antes de cotizar

1. **Modelo exacto de cada controlador del cliente.** Si alguna cámara ya tiene un **XR75CH**
   (o cualquier XR con bornes RS-485), **esa cámara entra directo al nivel 2 sin cambiar
   nada**: solo hay que fijarle `Adr` y la velocidad y colgarle la RTU. Vale la pena revisarlo
   cámara por cámara antes de cotizar envolventes de más.
2. **Part number del XR75CH** y confirmación con Emerson de que la variante cotizada trae
   RS-485 a bordo: dentro de la misma familia hay variantes con y sin (el XR60CH-5N0C1, por
   ejemplo, figura **sin** interfaz RS485 en el catálogo de un distribuidor). El sufijo manda.
3. **Formato de panel**: el XR75CH es formato 32 × 74 mm con recorte **71 × 29 mm**, el mismo
   del XR06CX — el reemplazo reusa el agujero, y con 4 sondas y 4 relés admite el cableado
   existente con margen.
