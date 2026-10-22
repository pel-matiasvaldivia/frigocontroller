# 02 — Diseño de hardware (rev. 3: envolvente no invasiva del XR06CX)

> **Por qué rev. 3.** El manual del XR06CX confirma que el controlador **no tiene ModBUS ni
> puerto serie de proceso** ([doc 10](10-hallazgos-manual-xr06cx.md)). Las rev. 1 y 2 se
> apoyaban en un bus que no existe. Esta revisión implementa el **camino A**: el gateway
> rodea al controlador sin intervenirlo — mide con su propia sonda, lee las tres salidas de
> relé y corta la alimentación del XR06 para el ON/OFF remoto.
>
> | | Rev. 1 | Rev. 2 | **Rev. 3** |
> |---|---|---|---|
> | Premisa | ModBUS vía XJ485 | ModBUS vía TTL directo | **Sin ModBUS: envolvente física** |
> | Viable sobre un XR06CX | **No** | **No** | **Sí** |
> | Costo @500 u. | 63.88 + XJ485 45.00 | 24.79 | **26.34** |

## 1. Diagrama de bloques

```
  230 VAC ──[F 500mA T]──[MOV 275V]──┬─▶ AC/DC 5 V 3 W (HLK-PM05)
                                     │        ├─▶ Buck 3V3 ─▶ ESP32-C3
                                     │        └─▶ supercap ─▶ RTC + flush de log
                                     └─▶ RELÉ 1 (NC) ─▶ alimentación del XR06  ← ON/OFF remoto
 ┌───────────────────── ESP32-C3-MINI-1U-N4 (antena externa) ─────────────────────┐
 │ I²C   ─▶ ADS1115 ─▶ 2× entrada NTC 10k (cámara + evaporador)                   │
 │        └▶ RTC PCF85063A  ·  conector sonda T/HR opcional (SHT45)               │
 │ GPIO  ─▶ 3× DETECCIÓN DE TENSIÓN 230 V en las salidas del XR06:                │
 │            · compresor   · deshielo   · ventiladores     (opto H11AA1)         │
 │ GPIO  ─▶ 2× DI opto para contacto limpio: puerta (propia) + libre              │
 │ GPIO  ─▶ RELÉ 1 (corte del XR06) · [RELÉ 2 opcional: sirena]                   │
 │ GPIO  ─▶ LED RGB · buzzer · pulsador     ADC ─▶ CT opcional                    │
 │ UART  ─▶ [población FC-R: SP3485 ─▶ A/B, para el camino B y otros combistatos] │
 └────────────────────────────────────────────────────────────────────────────────┘
```

**La idea central:** las tres salidas de relé del XR06 son la telemetría que el controlador
no sabe transmitir. Detectando **tensión** (no corriente) en cada una se obtiene estado de
compresor, de deshielo y de ventiladores con resolución de 1 s, sin tocar el cableado de
potencia ni depender del tamaño de la carga.

## 2. Componentes

| Bloque | Elección | Justificación |
|---|---|---|
| MCU + Wi-Fi | **ESP32-C3-MINI-1U-N4** (antena externa) | Módulo pre-certificado; un núcleo alcanza (la carga es scan de I/O y MQTT). Variante **U**: imprescindible en tablero metálico |
| **Detección de salidas** | **3 × H11AA1** (opto de entrada CA bidireccional) + 2 × 100 kΩ 1/2 W + RC | Opto específico para CA de red: detecta presencia de 230 V con 1 mA y aislación de 4 kV. Mide **tensión en la salida del relé**, así que funciona igual con un contactor de 1 A o de 20 A, y no requiere abrir ningún cable |
| **Entradas de temperatura** | **ADS1115** (16 bit, I²C) + 2 × NTC 10 k B3435 + divisor con resistencia 0,1 % | Dos canales: cámara y evaporador, replicando P1/P2 del controlador. El ADC interno del ESP32 no alcanza la precisión objetivo (±0,5 °C). Usar el **mismo tipo de NTC que el XR06** permite comparar lecturas sin discutir tecnologías |
| Sonda de cámara | NTC 10 k en cápsula IP67, cable 3 m FEP | Igual que la del controlador: −40÷110 °C, apta congelados. Montaje con el bulbo **hacia arriba** y lejos de flujos de aire, como indica el manual del XR06 (§10.1) |
| Sonda T/HR (opcional) | **SHT45** sobre el mismo conector | Única forma de cubrir humedad: el XR06 no la mide |
| Relé de corte | **1 × SPDT 8 A**, contacto **NC**, snubber RC + MOV | El XR06 consume 3,5 VA: un relé chico sobra. **NC** = relé sin energía deja el controlador alimentado |
| Entradas digitales | 2 × PC817 con wetting de 12 V | Puerta propia (contacto limpio) + una libre. **No se comparte la entrada digital del XR06** (doc 10 §4) |
| Fuente | **HLK-PM05**, 100–264 Vac | Toma de los mismos bornes de alimentación del XR06 |
| RTC | **PCF85063A** + supercap | Estampa de tiempo de los registros offline |
| Interfaz serie (población FC-R) | **SP3485** + bornera A/B + jumper de 120 Ω, **y un puerto TTL con driver de ≥ 50 mA** | Para el **camino B** y para extender el producto a otras marcas sin rediseñar. **Las dos salidas:** el relevamiento de controladores (doc 11 §1) mostró que la mayoría del rubro usa TTL + adaptador externo, y el Danfoss ERC 213 exige más de 50 mA en su línea TTL RX. Preverlo ahora cuesta ~USD 0.40; descubrirlo después es un rediseño de PCB |

**Una PCB, cuatro poblaciones:** FC-A (envolvente, línea base) · FC-R (RS-485, camino B y
otros controladores) · +HR (sonda de humedad) · +CT (clamp de corriente). Un stock, una
herramienta, cuatro SKU — y el camino B sale sin ingeniería nueva.

## 3. Detección de salidas: el detalle que la hace confiable

```
   XR06 relé compresor ──┬───────────▶ bobina del contactor
                         │
                    ┌────┴────┐
                    │ 100k ½W │   (×2, uno por línea: tolera 230 V continuos)
                    └────┬────┘
                      ┌──┴──┐
                      │H11AA1│  opto CA bidireccional, aislación 4 kV
                      └──┬──┘
                         └──▶ GPIO con RC (filtro de 20 ms) + pull-up
```

1. **Se detecta tensión en paralelo, nunca en serie:** el gateway no interrumpe ni carga el
   circuito de mando. Si el gateway se quema, el contactor sigue funcionando.
2. **Filtro de 20 ms** sobre el semiciclo de red: el opto conduce en picos a 50 Hz, así que
   el firmware integra sobre 3 ciclos para no leer parpadeo.
3. **Tres canales dan más que tres bits.** Combinando compresor + deshielo + ventiladores
   con los parámetros del controlador (`td`, `FC`, `Fd`) se reconstruye la máquina de estados
   completa: enfriando, deshielo, goteo (`dt`), retardo de ventiladores (`Fd`), parado.
4. **Resistencias de 1/2 W y creepage de 5,5 mm** en las tres entradas: están a potencial de
   red de forma permanente, es la zona más crítica de la placa.
5. El canal de deshielo se conecta al **común del relé SPDT de deshielo**, para que funcione
   tanto con `td=EL` (resistencia) como con `td=in` (gas caliente).

## 4. ON/OFF remoto por corte de alimentación del controlador

Camino **A1** del [doc 10 §5](10-hallazgos-manual-xr06cx.md#5-onoff-remoto-lo-que-el-manual-habilita):

| Regla de diseño | Motivo |
|---|---|
| Contacto **NC** en serie con los 230 Vac del XR06 | Relé sin energía = controlador alimentado = cámara refrigerando. Un gateway muerto no apaga nada |
| El relé **nunca** va en serie con la bobina del contactor | El XR06 no se enteraría del corte y, al reponer, cerraría sin el retardo anticiclos `AC`: riesgo de rotura de compresor |
| Al instalar se configura **`od` ≥ 3 min** en el XR06 | `od` retarda la activación de las salidas al arranque: es la protección del compresor cuando el gateway restituye la tensión |
| Tiempo mínimo entre conmutaciones: **5 min**, impuesto en firmware | Protege compresor y relé aun si la app pide ON/OFF repetidamente |
| Watchdog de compuerta en hardware | Si el firmware se cuelga con el relé energizado, el watchdog lo libera: la cámara vuelve a enfriar sola |

## 5. Lo que este diseño NO puede hacer, por hardware

| Limitación | Causa | Mitigación |
|---|---|---|
| **Setup de temperatura remoto** | El XR06CX no tiene ningún canal de escritura en línea | Consigna virtual hacia arriba por compuerta, o camino B (doc 10 §7) |
| **Flags `P1`/`P2` de falla de sonda** | No hay relé de alarma ni registro legible | Detección inferencial por firma de ciclado `Cy`/`Cn` (doc 10 §6): ~2 h en lugar de inmediato |
| **Leer los parámetros del controlador** (`SET`, `AU`, `AL`, `Hy`…) | Ídem | Se **cargan a mano en la app** durante la puesta en marcha, leyéndolos del display. El asistente de instalación los pide uno por uno y queda registrado en el acta |

La tercera fila tiene una consecuencia operativa que hay que aceptar: **si alguien cambia un
parámetro en el frente del controlador, el gateway no se entera.** La app muestra la fecha de
la última verificación de parámetros y recuerda revalidarlos cada 6 meses.

## 6. Asignación de E/S (rev. 3)

| Canal | Función | Notas |
|---|---|---|
| AC1 | Salida de **compresor** del XR06 | Detección de tensión, en paralelo |
| AC2 | Salida de **deshielo** del XR06 | Común del SPDT |
| AC3 | Salida de **ventiladores** del XR06 | Completa la máquina de estados |
| NTC1 | Sonda de cámara (propia) | Obligatoria: es la única fuente de temperatura |
| NTC2 | Sonda de evaporador (propia, opcional) | Habilita `defrost_ineffective` |
| DI1 | Fin de carrera de puerta (propio) | Contacto limpio; **no se comparte el del XR06** |
| DI2 | Libre: térmico, presostato, monitor de fase | — |
| RELÉ 1 | **Corte de alimentación del XR06** (NC) | ON/OFF remoto |
| RELÉ 2 | Sirena / baliza (opcional) | — |
| Conector sonda | SHT45 (humedad) o DS18B20 (producto) | Opcional |

## 7. Especificación eléctrica

| Parámetro | Valor |
|---|---|
| Alimentación | 100–264 Vac 50/60 Hz (de los mismos bornes del XR06) |
| Consumo | < 1.2 W reposo, < 2.5 W con relé y Wi-Fi TX |
| Entradas de detección de CA | 3 × 90–264 Vac, aislación 4 kV, consumo < 1 mA |
| Precisión de temperatura | ±0,5 °C de −40 a +25 °C (objetivo: no quedar peor que el ±0,1 °C ±1 dígito del XR06) |
| Corte de alimentación del controlador | Relé SPDT 8 A, contacto NC, 10⁵ ciclos |
| Temperatura de operación | −20 a +60 °C (el XR06 exige 0÷60 °C: el tablero ya cumple) |
| Aislación red ↔ lógica | 3 kV (relé y fuente), 4 kV (detección de CA) |
| Retención de datos sin red | ≥ 30 días a 1 muestra/min |
