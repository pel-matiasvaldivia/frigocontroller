# FrigoController — Gateway IoT para cámaras frigoríficas con controladores Copeland/Dixell XR06 y XR75

Dossier de diseño de producto: electrónica, mecánica, firmware, protocolo y app para
monitoreo y control remoto de equipos de refrigeración comandados por combistatos
**Copeland / Emerson (Dixell)**. El cliente tiene parque mixto: **XR06CH** (no comunicable) y
**XR75CX** (familia con ModBUS).

> ## Parque del cliente: dos modelos, dos caminos
>
> | Modelo declarado | Firmware | Camino |
> |---|---|---|
> | **XR06CH-5GACILS** | V1.1 | **A — envolvente FC-A.** 3 relés, 2 NTC, 1 entrada digital, sin ModBUS. Código de 7 caracteres: **variante OEM**, así que **no se pueden asumir los parámetros por defecto** |
> | **XR75CX-5N7I3** | V2.6b | **B — ModBUS.** Familia con 4 sondas, 4 relés, 2 entradas digitales y red ModBUS-RTU. **Es la familia que ya integramos en el proyecto SmartPath M3**: estas cámaras pueden ir a nivel 2 **sin reemplazar el controlador** |
>
> Detalle y verificación pendiente (si esta variante trae RS-485 a bordo o requiere XJ485):
> **[docs/13-inventario-controladores-cliente.md](docs/13-inventario-controladores-cliente.md)**

> ## Rev. 3 — replanteo a partir del manual oficial
>
> El manual `1594020160 XR06CX SP r1.2` confirma que **el XR06CX no tiene ModBUS ni puerto
> serie de proceso**: no hay parámetro de dirección de esclavo en toda la lista de
> parámetros, los datos técnicos no declaran salida serie y el único conector es la **Hot
> Key**, un canal de programación que exige apagar y encender el instrumento.
>
> Las rev. 1 y 2 se apoyaban en un bus que no existe. **La rev. 3 implementa una envolvente
> física:** el gateway mide con su propia sonda, lee las **tres salidas de relé** del
> controlador (compresor, deshielo, ventiladores) y corta la alimentación del XR06 para el
> ON/OFF remoto, **sin intervenir el controlador ni el cableado de potencia**.
>
> Análisis completo, evidencia y los tres caminos evaluados:
> **[docs/10-hallazgos-manual-xr06cx.md](docs/10-hallazgos-manual-xr06cx.md)**

## 1. Cobertura de los requerimientos

| Requerimiento del cliente | Estado | Cómo |
|---|---|---|
| Transmitir temperatura | ✅ | Sonda NTC propia (cámara) + opcional de evaporador |
| Indicación de funcionamiento del compresor | ✅ | Detección de tensión en la salida de relé de compresor |
| Indicación de deshielo en la App | ✅ | Salida de deshielo + salida de ventiladores: reconstruye deshielo, goteo (`dt`) y retardo de ventiladores (`Fd`) |
| Registro de aperturas de puerta | ✅ | Fin de carrera propio (la única entrada digital del XR06 no se toca) |
| Alarmas de alta y baja temperatura | ✅ | Calculadas sobre la sonda propia, replicando `AU`/`AL`/`Ad` del controlador |
| Alerta de error de comunicación | ✅ | Watchdog del gateway + LWT MQTT |
| ON / OFF remoto | ✅ | Relé **NC** que corta los 230 Vac del XR06; con `od` ≥ 3 min para proteger el compresor al reponer |
| Alertas por falla de sondas | ⚠️ **Inferencial** | El XR06CX no tiene relé de alarma ni registro legible. Se detecta por la **firma de ciclado `Cy`/`Cn`** (15 min ON / 30 min OFF por defecto) sin relación con la temperatura: **detección en ~2 h, no inmediata** |
| **Setup de temperatura remoto** | ❌ **No sobre el XR06CX** | No existe canal de escritura en línea. Tres salidas: consigna virtual **solo hacia arriba** por compuerta, **reemplazo del controlador** (upgrade de nivel 2, USD 290-390/cámara), o emulación de Hot Key (I+D, no se cotiza) |
| **Humedad** | ❌ **salvo accesorio** | El XR06CX no mide HR. Sonda T/HR accesorio (+USD 45-60), conector ya previsto en la PCB |

**Las tres filas no verdes requieren decisión del cliente antes del piloto**, no durante.

## 2. Arquitectura en una pantalla

```
   CÁMARA FRIGORÍFICA              TABLERO                          NUBE            USUARIO
 ┌────────────────────┐   ┌──────────────────────────────┐
 │ Evaporador         │   │  XR06CX   (intacto)          │
 │ NTC del XR06 ──────┼───┤  3 relés: compresor ─┐       │
 │ NTC propia ────────┼─┐ │           deshielo ──┼─┐     │
 │ [T/HR opcional] ───┼┐│ │           ventil. ───┼┐│     │
 │ Fin de carrera ────┼┼┼─┤  230 Vac ◀── corte ──┼┼┼──┐  │
 └────────────────────┘│││ └──────────────────────┼┼┼──┼──┘
                       │││   detección de tensión │││  │ relé NC
                       ▼▼▼            ┌───────────▼▼▼──┴──┐  MQTT/TLS  ┌─────────┐  ┌──────────┐
                                      │ FRIGOCONTROLLER   ├─ Wi-Fi ───▶│ Broker  │─▶│ App móvil│
                                      │ FC-A  ESP32-C3    │            │ + TSDB  │  │ + Web    │
                                      │ 2 NTC · 3 AC · DI │            │ + Reglas│  │ Push/WA  │
                                      └───────────────────┘            └─────────┘  └──────────┘
```

**Principio de diseño rector:** el XR06 conserva la autoridad de control y de seguridad — el
propio manual advierte que el instrumento *"no debe ser usado como un dispositivo de
seguridad"*. El gateway supervisa y, para el ON/OFF, **habilita o deshabilita** al
controlador; nunca sustituye sus enclavamientos. Fail-safe: relé sin energía = XR06
alimentado = cámara refrigerando.

## 3. Documentos

| Doc | Contenido |
|---|---|
| **[docs/10-hallazgos-manual-xr06cx.md](docs/10-hallazgos-manual-xr06cx.md)** | **Leer primero.** Hallazgos del manual, los 3 caminos, matriz de cumplimiento, corrección de nomenclatura |
| [docs/01-arquitectura.md](docs/01-arquitectura.md) | Arquitectura de sistema, reparto de responsabilidades, FMEA |
| [docs/02-hardware.md](docs/02-hardware.md) | **Electrónica rev. 3:** detección de salidas de CA, entradas NTC, relé de corte, una PCB con 4 poblaciones |
| [docs/03-interfaz-controlador.md](docs/03-interfaz-controlador.md) | Camino B: ModBUS RS-485 para controladores comunicables, y validación de mapas de registros |
| [docs/04-firmware.md](docs/04-firmware.md) | Tareas, motor de alarmas, buffer offline, OTA, seguridad |
| [docs/05-nube-y-app.md](docs/05-nube-y-app.md) | Topics MQTT, payloads, backend, UX de la app, reportes HACCP |
| [docs/06-diseno-industrial.md](docs/06-diseno-industrial.md) | Gabinete, sonda, instalación, serigrafía, LEDs, usabilidad en obra |
| [docs/07-bom-y-costos.md](docs/07-bom-y-costos.md) | **BOM rev. 3**, accesorios, costeo y catálogo de dos niveles |
| [docs/08-normativa-y-plan.md](docs/08-normativa-y-plan.md) | Normativa AR/CE, ensayos, roadmap, riesgos, relevamiento previo |
| [docs/09-optimizacion-de-costos.md](docs/09-optimizacion-de-costos.md) | Registro de recortes: qué se ahorró, qué se cede, qué se rechazó |
| **[docs/12-plataforma-smartpath-m3.md](docs/12-plataforma-smartpath-m3.md)** | **El proyecto SmartPath M3 y el XR75CH:** el nivel 2 ya está construido. Decisión de plataforma |
| [docs/11-controladores-nivel2.md](docs/11-controladores-nivel2.md) | Relevamiento de controladores con ModBUS: alternativas, criterios de compra, fuentes |
| [config/perfil-xr06cx-envolvente.yaml](config/perfil-xr06cx-envolvente.yaml) | **Perfil del camino A:** entradas, salidas, máquina de estados y alarmas |
| [config/modbus-generico-rs485.yaml](config/modbus-generico-rs485.yaml) | Plantilla del camino B para controladores con ModBUS |
| [hardware/diagrama-conexionado.md](hardware/diagrama-conexionado.md) | Diagrama de borneras y conexionado de campo |
| **[docs/13-inventario-controladores-cliente.md](docs/13-inventario-controladores-cliente.md)** | **Los dos modelos que el cliente tiene instalados**, qué camino le toca a cada uno y la verificación de 10 minutos |
| [docs/manuales/](docs/manuales/) | Manual oficial del XR06CX (fuente de la rev. 3) y manual de la RTU SmartPath M3 |

## 4. Catálogo: dos niveles en lugar de una promesa incumplida

| | **Nivel 1 — FC-A** | **Nivel 2 — upgrade "control total"** |
|---|---|---|
| Qué hace | Monitoreo completo, alarmas, deshielo, compresor, puerta, **ON/OFF remoto** | Todo lo anterior **+ setup de temperatura remoto** |
| Cómo | Envolvente del XR06CX existente | **Dixell XR75CH + RTU SmartPath M3**, que ya existe y ya se presentó a este cliente ([doc 12](docs/12-plataforma-smartpath-m3.md)). Alternativa: XR75CH + FC-R |
| Toca el controlador | **No.** Sin corte de servicio | Sí: reemplazo, 1-2 h por cámara |
| Costo @500 u. | **USD 26.34** | Controlador USD 60-120 + RTU + instalación. **Desarrollo: cero** |
| Precio sugerido | **USD 105-140** + abono USD 6-8 | USD 290-390 + abono USD 8-10 |
| Instalación | **15 min** | 1-2 h |

**El nivel 2 ya está construido:** la RTU SmartPath M3 opera consigna remota y ON/OFF sobre el
XR75CH desde el año pasado. El esfuerzo de ingeniería va al nivel 1, que es donde no hay nada
y donde está la mayoría de las cámaras del cliente.

Se vende el nivel 1 en todo el parque y el nivel 2 solo en las cámaras donde la consigna
remota haga falta. Misma app, mismo backend, mismo gateway: el cliente no nota más
diferencia que la aparición del control de consigna.

Accesorios: **+P2** (sonda de evaporador) · **+HR** (sonda T/HR, única forma de medir
humedad) · **+CT** (clamp de compresor) · **+2R** (sirena) · **IP65** · **LTE**.

## 5. Próximo paso: 3 datos del cliente, sin banco ni material

**Lo más rentable de esta semana: el conteo cámara por cámara.** Cada cámara con **XR75CX** es
una venta de nivel 2 **sin reemplazar el controlador**; cada una con **XR06CH** es un FC-A.
Alcanza con una foto de la bornera del XR75CX (para ver si tiene los bornes 16-17-18 `RS485`)
y el conteo de cada modelo — [doc 13 §4](docs/13-inventario-controladores-cliente.md).

De cada XR06CH instalado hacen falta además:

1. **`Pt` (mapa de códigos) y `rL` (versión de software)** — ambos parámetros de solo lectura,
   visibles en el display. Identifican la variante exacta de firmware y son el dato que hay
   que llevarle a Emerson para la única pregunta que podría reabrir el camino ModBUS.
2. **`iF`, `iP`, `dC`, `rd`** — define si el fin de carrera de puerta está en uso, y cómo.
3. **`ot`, `oE`** — calibración de sondas. Si no son 0, la sonda propia del gateway va a
   diferir del display y hay que replicar el offset en la app.

Más el protocolo completo de relevamiento en
[docs/08 §5](docs/08-normativa-y-plan.md#5-lo-que-hay-que-relevar-en-el-sitio-del-cliente-antes-de-cotizar).
