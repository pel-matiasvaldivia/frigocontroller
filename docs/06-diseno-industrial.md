# 06 — Diseño industrial y mecánico (rev. 2)

El equipo vive en un lugar hostil y poco glamoroso: un tablero de refrigeración, instalado
por un técnico apurado, sobre una instalación existente. El diseño industrial se juzga por
tres cosas: **que entre, que se instale en 15 minutos y que el estado se entienda sin abrir
la app.**

> **Rev. 2.** Al tomar la medición del XR06 desaparece la sonda de serie: el producto pasa a
> ser **un solo cuerpo**, más chico, más barato de inyectar y más rápido de instalar. El
> gabinete pasa de 6M custom a **4M genérico con etiqueta impresa**, lo que además evita
> USD 8–15 k de herramienta de molde hasta tener volumen que la justifique.

## 1. Arquitectura de producto

```
  ┌────────────────────────────┐
  │  FRIGOCONTROLLER FC-E      │   cable TTL 1 m ──▶ puerto TTL del XR06
  │  riel DIN 4M (71×90×58)    │   (va dentro del mismo tablero)
  │                            │
  │  ◦ conector de sonda ──────┼─ ─ ─ ▶ [ SONDA T/HR ACCESORIO, solo si se cotiza ]
  └────────────────────────────┘
```

Un solo cuerpo, dentro del tablero, al lado del controlador que supervisa. Es el formato más
barato y el que menos tiempo de instalación consume: no hay nada que amurar ni cable que
tender hacia la cámara, salvo que se pida la sonda de humedad.

## 2. FC-E (riel DIN) — formato base

| | |
|---|---|
| Formato | Caja modular DIN **4M, 71 × 90 × 58 mm**, genérica de catálogo (sin molde propio) |
| Material | Policarbonato V-0, gris RAL 7035 (gris estándar de tablero) |
| Protección | IP20 (va dentro del tablero) |
| Borneras | A tornillo. Inferior: red + relé, paso **5.08 mm**. Superior: señales (TTL/RS-485, DI, sonda), paso **3.5 mm** |
| Frente | Ventana difusora de LED RGB + pulsador recesado + **etiqueta impresa de policarbonato** con logotipo, modelo, serie, QR y leyenda de LED |
| Antena | SMA hembra en el lateral superior, whip 2 dBi rebatible |
| Servicio | Pads de test internos; sin conector USB (se flashea en cama de agujas en producción) |

**Etiqueta impresa en lugar de frente serigrafiado:** se imprime por lote en policarbonato
retro-impreso con adhesivo 3M 467, cuesta USD 0.25 contra USD 1.80 de un frente con molde y
serigrafía, y permite cambiar modelo, versión y QR **sin tocar la herramienta**. Para un
producto que va a iterar tres veces en dos años, es la decisión correcta — y en un tablero,
nadie compra por el frente.

**Por qué riel DIN:** es el lenguaje del tablerista. Se instala en el espacio que ya existe
al lado del XR06, con la misma herramienta, sin perforar nada. 4M ocupa menos de la mitad
del ancho de un interruptor tetrapolar: entra incluso en tableros llenos.

## 3. FC-E IP65 (a pared) — variante

| | |
|---|---|
| Formato | Caja ABS/PC **130 × 80 × 70 mm** de catálogo, tapa con 4 tornillos cautivos y junta de poliuretano |
| Protección | **IP65** (lavados con hidrolavadora: requisito real en frigoríficos) |
| Pasacables | 3 × prensacables M16 IP68 + 1 × M12 para la sonda accesorio; posiciones ciegas pre-moldeadas |
| Frente | Policarbonato traslúcido con difusor para el LED; etiqueta impresa resistente a alcohol y detergente alcalino |
| Montaje | 4 bridas externas (los tornillos **nunca** atraviesan la zona estanca) |
| Antena | SMA con antena tipo puck magnética sobre cable RG-174 de 2 m |

Detalle de campo: los prensacables van **hacia abajo** y las borneras ocupan la mitad
inferior, para que la condensación escurra y no entre por el cable (efecto mecha). El lazo
de goteo está en las instrucciones y en la etiqueta.

## 4. Comunicación de estado sin app: el LED y el buzzer

Un LED RGB difuso, con un lenguaje que el técnico aprende en una visita y que está impreso
en la etiqueta del frente:

| Patrón | Significado |
|---|---|
| Verde fijo | Todo normal, conectado |
| Verde latido lento | Normal, trabajando offline (registrando en memoria) |
| Celeste pulsante | En deshielo |
| Azul intermitente rápido | Modo configuración / SoftAP esperando aparear |
| **Ámbar intermitente** | **Sin comunicación con el controlador** (en rev. 2 es la falla que deja al equipo ciego: tiene su propio color y es lo primero que mira el técnico) |
| Rojo intermitente + buzzer | Alarma crítica activa (alta/baja temperatura, falla de sonda del XR06) |
| Rojo fijo | Falla interna del equipo |
| Blanco 1 destello | Comando recibido y aplicado |

Buzzer 85 dB solo para alarmas críticas, con silenciado local de 30 min con el pulsador —
porque si no se puede silenciar localmente, el técnico lo desconecta y se pierde el
producto. El segundo relé (opción **+2R**) replica la alarma para sirena o baliza externa.

## 5. Pulsador único, tres gestos

| Gesto | Acción |
|---|---|
| Pulsación corta | Silenciar buzzer 30 min / confirmar alarma local |
| 3 s | Entrar en modo configuración (SoftAP + QR) |
| 10 s + confirmación con doble pulsación | Restablecer a fábrica |

Un solo control físico, sin menús: toda la configuración vive en la app. El dispositivo no
compite con el display del XR06, que sigue siendo la interfaz local de control — y que ahora
**muestra el mismo número que la app**, porque es la misma sonda.

## 6. Cable TTL al XR06 — ahora es una pieza de producto

Al eliminar el XJ485, este cable deja de ser un accesorio de tercero y pasa a ser una pieza
con especificación propia. Es la única conexión de la que depende todo el dato de proceso:

- Largo **1 m**, suficiente para dos equipos contiguos en el mismo riel; versión de 2 m para
  tableros profundos.
- **4 conductores + malla**, AWG 26, con malla aterrada **solo del lado del gateway**.
- Conector al XR06 **con retención mecánica y asimetría que impide invertirlo**. Un cable
  que se puede enchufar al revés se va a enchufar al revés.
- Serie de 100 Ω y clamp integrados **en la PCB del gateway**, no en el cable: el cable es
  pasivo, barato y reemplazable, y la protección del controlador nunca viaja en la pieza que
  se pierde.
- Rotulado en ambos extremos (`XR06 TTL` / `FC-E UART`) y color distintivo (blanco), para
  que nadie lo confunda con un cable de potencia en una canaleta gris.

## 7. Sonda T/HR — accesorio (SKU +HR)

Solo se instala donde la humedad importa (frescos, maduración, auditoría de HR). Cuando se
cotiza, la especificación es la de la rev. 1:

```
      ┌──────────────┐
 ═════╡ ░░░░░░░░░░░░ │  ← filtro PTFE sinterizado reemplazable
 cable└──────────────┘
 FEP   Ø 16 × 70 mm, IP67
```

- **Sensor** SHT45 sobre PCB recortada, conformal coating excepto sobre el sensor.
- **Cable** 3 m (opc. 10 m) FEP apto alimentos, flexible a −40 °C — el PVC se cristaliza y se
  quiebra a −25 °C. Apantallado, drenaje a masa solo en el gateway.
- **Calentador del SHT45** 10 s cada hora para recuperar la membrana de condensación.
- **Soporte** magnético de neodimio + oreja para tornillo. Montaje a **1.5 m de altura,
  lejos de la puerta y fuera del chorro directo del evaporador**.
- Conector **M12 de 4 p**, para cambiar la sonda sin bajar el equipo (recalibración anual
  como servicio opcional).

La misma entrada admite una **sonda de producto DS18B20 en vaina de acero inoxidable**, para
clientes cuya auditoría exige temperatura de producto y no de aire.

## 8. Etiquetado y lo que ve el instalador

- **Dentro de la tapa:** diagrama de conexionado completo, impreso. El técnico no va a
  buscar el manual, va a abrir la tapa.
- **Frente:** modelo, serie, MAC, **QR de apareamiento** (ID y clave de AP únicos), tensión
  de alimentación, marcados de seguridad, leyenda de código de LED.
- **Borneras rotuladas por función, no por número** (`DI1 PUERTA`, `RL1 ON/OFF`,
  `TTL XR06`), en el plástico y en la PCB.
- Kit en caja: gateway, **cable TTL armado**, whip de antena, juego de borneras, guía rápida
  plastificada de una hoja (lado A: conexionado; lado B: código de LED y 5 problemas
  frecuentes). La sonda T/HR, el clamp y los prensacables van en kits aparte.

## 9. Instalación objetivo: 15 minutos

1. Clipar en el riel (2 min).
2. Alimentación (ó 12-24 V del tablero) y verificación del LED de arranque (2 min).
3. **Enchufar el cable TTL al XR06** (1 min — era el paso de 8 min con XJ485).
4. Un hilo del relé 1 a la entrada digital del XR06 para el ON/OFF; el hilo de puerta solo
   si el fin de carrera no está ya cableado al controlador (4 min).
5. Escanear el QR, aparear Wi-Fi; el asistente prueba el enlace Modbus, lee la temperatura y
   la compara con el display, prueba el relé, dispara una alarma de prueba y confirma que
   llegó al teléfono (5 min).

El paso 5 es parte del producto: una instalación no se considera terminada hasta que **el
número de la app coincide con el del display y una alarma de prueba llegó al teléfono del
cliente**. El asistente lo exige y lo registra en el acta de puesta en marcha en PDF.

## 10. Lenguaje visual

Gris de tablero (RAL 7035) como base, para que el equipo se lea como instrumento industrial
y no como gadget de consumo; un único acento de color de marca en la etiqueta y en el aro del
LED; tipografía de palo seco condensada, alto contraste, cuerpo mínimo 6 pt para leer con
linterna; sin brillos, textura mate anti-huella. Con gabinete genérico, **la identidad la
carga la etiqueta y la app**, no el molde — y eso es exactamente lo que hay que optimizar
cuando el presupuesto manda: el cliente no paga por el plástico, paga por no perder la
mercadería.
