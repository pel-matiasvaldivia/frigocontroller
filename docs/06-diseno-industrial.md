# 06 — Diseño industrial y mecánico (rev. 2)

El equipo vive en un lugar hostil y poco glamoroso: un tablero de refrigeración, instalado
por un técnico apurado, sobre una instalación existente. El diseño industrial se juzga por
tres cosas: **que entre, que se instale en 15 minutos y que el estado se entienda sin abrir
la app.**

> **Rev. 3.** El XR06CX no comunica, así que **la sonda propia vuelve y es obligatoria**: el
> producto son dos cuerpos (gateway en el tablero, sonda en la cámara). Se mantiene el
> gabinete **DIN 4M genérico con etiqueta impresa** de la rev. 2, que evita USD 8–15 k de
> herramienta de molde hasta tener volumen que la justifique. Desaparece el cable TTL (no hay
> puerto serie) y aparecen **tres hilos de detección de salidas**, que es el cableado nuevo
> del instalador.

## 1. Arquitectura de producto

```
  ┌────────────────────────────┐  3 hilos ──▶ salidas de relé del XR06 (en el mismo tablero)
  │  FRIGOCONTROLLER FC-A      │  1 hilo  ──▶ relé NC en serie con los 230 V del XR06
  │  riel DIN 4M (71×90×58)    │
  │                            │  cable 3 m FEP
  │  ◦ NTC1 cámara ────────────┼──────────────▶ [ SONDA DE CÁMARA — obligatoria ]
  │  ◦ NTC2 evaporador ────────┼─ ─ ─ ─ ─ ─ ─ ▶ [ opcional, SKU +P2 ]
  │  ◦ conector T/HR ──────────┼─ ─ ─ ─ ─ ─ ─ ▶ [ SONDA T/HR, SKU +HR ]
  └────────────────────────────┘
```

El gateway va en el tablero, junto al controlador que supervisa (el XR06 exige 0÷60 °C de
ambiente, así que nunca va dentro de la cámara). La medición necesita estar donde está el
producto: de ahí los dos cuerpos.

## 2. FC-E (riel DIN) — formato base

| | |
|---|---|
| Formato | Caja modular DIN **4M, 71 × 90 × 58 mm**, genérica de catálogo (sin molde propio) |
| Vecino de riel | Va al lado del XR06CX, que es de montaje en panel 71×29 mm con frontal IP65 (§9 y §13 de su manual): el controlador queda en la puerta del tablero, el gateway en el riel de atrás |
| Material | Policarbonato V-0, gris RAL 7035 (gris estándar de tablero) |
| Protección | IP20 (va dentro del tablero) |
| Borneras | A tornillo. Inferior: red, relé de corte y **3 entradas de detección de salidas** (todas a potencial de red), paso **5.08 mm**. Superior: señales (NTC, DI, sonda T/HR), paso **3.5 mm**. **Separación física y rotulado inequívoco entre los dos grupos**: una bornera de red y una de sonda a 10 mm de distancia es un accidente esperando |
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
| **Ámbar intermitente** | **Sonda propia en falla** (el equipo quedó ciego a la temperatura) o detección de salidas sin señal cuando debería haberla: es lo primero que mira el técnico |
| Rojo intermitente + buzzer | Alarma crítica activa (alta/baja temperatura, falla de sonda inferida, compresor sin corriente) |
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
compite con el display del XR06, que sigue siendo la interfaz local de control. Como son dos
sondas distintas, **el asistente de instalación obliga a comparar los dos números y a cargar
el offset `ot` del controlador** para que coincidan: es el paso que evita el llamado a
soporte "la app me marca distinto que el equipo".

## 6. Las tres tomas de detección — la pieza nueva de la instalación

En lugar de un cable de comunicación hay **tres hilos que se derivan de las salidas de relé
del XR06**. Es cableado de red de 230 V, y el diseño industrial tiene que hacerlo imposible
de equivocar:

- **Kit de derivación pre-armado:** 3 cables de 60 cm, 0,75 mm², con **punta pre-pelada y
  terminal tipo horquilla** para entrar en la misma bornera del XR06 donde ya está el cable
  del contactor, sin cortar ni empalmar nada.
- **Colores fijos y rotulados**, repetidos en la bornera del gateway, en la etiqueta de la
  tapa y en la guía rápida: **marrón = compresor · azul = deshielo · negro = ventiladores**.
- **Imposible invertir con daño:** las tres entradas son eléctricamente idénticas y
  bidireccionales (opto de CA), así que un cruce entre ellas no rompe nada — solo muestra el
  estado equivocado, y el asistente de instalación lo detecta pidiendo un deshielo manual con
  la tecla del XR06 y verificando que se encienda el canal correcto. **Un error de cableado
  que se detecta en la puesta en marcha, en lugar de uno que se descubre en la primera
  alarma falsa.**
- La protección (resistencias de 1/2 W, opto de 4 kV, creepage de 5,5 mm) vive **en la PCB**,
  no en el kit: el cable es pasivo, barato y reemplazable.
- **Se deriva en paralelo, nunca en serie.** Está en la guía rápida en negrita: el gateway
  observa, no interrumpe. Si el gateway se quema, el contactor sigue funcionando.

## 7. Sondas

**NTC1 de cámara — obligatoria.** NTC 10 k B3435, el mismo tipo que usa el XR06 (−40÷110 °C,
§13 de su manual), en cápsula IP67 de Ø 16 × 70 mm, cable 3 m FEP. Dos indicaciones tomadas
del manual del controlador y que valen igual para la nuestra: **montar el bulbo hacia arriba**
para que una filtración de líquido no entre por el cable (§10.1), y **alejarla de los flujos
de aire** para medir la temperatura promedio de la cámara y no la del chorro del evaporador.

**NTC2 de evaporador — opcional (SKU +P2).** Misma sonda, cable de 1,5 m, sujeta al
evaporador. Habilita la alarma de deshielo inefectivo y la detección de `P2` en falla.

**Sonda T/HR — accesorio (SKU +HR).** Única forma de cubrir humedad, porque el XR06 no la
mide. Especificación:

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
- Cuando se cotiza **+HR**, la sonda SHT45 reemplaza a la NTC1 como fuente de temperatura de
  cámara (mide las dos cosas en una sola cápsula y un solo agujero): la NTC1 queda libre
  para la sonda de producto. Un cable menos y un soporte menos.

La misma entrada admite una **sonda de producto DS18B20 en vaina de acero inoxidable**, para
clientes cuya auditoría exige temperatura de producto y no de aire.

## 8. Etiquetado y lo que ve el instalador

- **Dentro de la tapa:** diagrama de conexionado completo, impreso. El técnico no va a
  buscar el manual, va a abrir la tapa.
- **Frente:** modelo, serie, MAC, **QR de apareamiento** (ID y clave de AP únicos), tensión
  de alimentación, marcados de seguridad, leyenda de código de LED.
- **Borneras rotuladas por función, no por número** (`DI1 PUERTA`, `RL1 ON/OFF`,
  `TTL XR06`), en el plástico y en la PCB.
- Kit en caja: gateway, **sonda de cámara con soporte**, **kit de 3 derivaciones rotuladas**,
  whip de antena, juego de borneras, guía rápida plastificada de una hoja (lado A:
  conexionado y colores; lado B: código de LED y 5 problemas frecuentes). La sonda de
  evaporador, la T/HR, el clamp y los prensacables van en kits aparte.

## 9. Instalación objetivo: 25 minutos

La rev. 3 agrega cableado de 230 V, así que la instalación es más larga que la de la rev. 2 —
pero sigue sin cortar el servicio ni tocar la configuración del controlador.

1. Clipar en el riel y alimentar; verificar el LED de arranque (4 min).
2. **Tres derivaciones** desde las borneras de salida del XR06 a las entradas de detección,
   siguiendo los colores del kit (6 min).
3. **Relé de corte:** intercalar el contacto NC en la alimentación del XR06 (4 min, **con el
   tablero sin tensión**).
4. Sonda de cámara: montar y pasar el cable (5 min). Fin de carrera de puerta a DI1 si existe.
5. Escanear el QR, aparear Wi-Fi, y el asistente (6 min):
   - pide `Pt`, `rL`, `SET`, `AU`, `AL`, `Ad`, `dA`, `Cy`, `Cn`, `td`, `Md`, `dt`, `Fd`, `ot`
     leídos del display, y los guarda en el acta;
   - **compara su temperatura con la del display** y aplica `ot`;
   - pide **un deshielo manual con la tecla del XR06** y verifica que se encienda el canal de
     deshielo (prueba de cableado de las tres derivaciones);
   - prueba el corte y la reposición, confirmando que las salidas quedan sin tensión;
   - **configura `od` ≥ 3 min en el XR06** (lo indica paso a paso, el técnico lo hace en el
     frente);
   - dispara una alarma de prueba y confirma que llegó al teléfono del cliente.

El paso 5 es parte del producto: una instalación no se considera terminada hasta que **el
número de la app coincide con el del display, las tres derivaciones responden al deshielo
manual y una alarma de prueba llegó al teléfono del cliente**. Queda registrado en el acta de
puesta en marcha en PDF, con los parámetros del controlador: ese acta es también el respaldo
de que el equipo del cliente se entregó configurado como estaba.

## 10. Lenguaje visual

Gris de tablero (RAL 7035) como base, para que el equipo se lea como instrumento industrial
y no como gadget de consumo; un único acento de color de marca en la etiqueta y en el aro del
LED; tipografía de palo seco condensada, alto contraste, cuerpo mínimo 6 pt para leer con
linterna; sin brillos, textura mate anti-huella. Con gabinete genérico, **la identidad la
carga la etiqueta y la app**, no el molde — y eso es exactamente lo que hay que optimizar
cuando el presupuesto manda: el cliente no paga por el plástico, paga por no perder la
mercadería.
