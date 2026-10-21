# 06 — Diseño industrial y mecánico

El equipo vive en un lugar hostil y poco glamoroso: un tablero de refrigeración, en una
cámara de frío, instalado por un técnico apurado, sobre una instalación existente. El
diseño industrial se juzga por tres cosas: **que entre, que se instale en 20 minutos y que
el estado se entienda sin abrir la app.**

## 1. Arquitectura de producto: dos cuerpos, una sonda

```
  ┌───────────────────────────┐        cable 3 m FEP          ┌──────────┐
  │  CONTROLADOR (gateway)    │ ───────────────────────────── │  SONDA   │
  │  riel DIN 6M ó caja IP65  │   4 hilos, apantallado        │ T + HR   │
  └───────────────────────────┘                               │  IP67    │
```

Separar la sonda del cuerpo no es un detalle: la electrónica necesita estar donde hay
tablero y Wi-Fi, y la medición necesita estar donde está el producto. Un equipo con el
sensor a bordo mide la temperatura de su propia fuente de alimentación.

## 2. FC-100 Panel (riel DIN)

| | |
|---|---|
| Formato | Caja modular DIN 6M, **105 × 90 × 58 mm** (tipo Italtronic/Camdenboss serie 6M) |
| Material | Policarbonato V-0, gris RAL 7035 (gris estándar de tablero) |
| Protección | IP20 (va dentro del tablero) |
| Borneras | Inferior: red + relés, enchufables paso **5.08 mm**. Superior: señales (RS-485, DI, sonda, CT), paso **3.5 mm** |
| Frente | Ventana de LED RGB + logotipo + pulsador recesado + área de etiqueta con QR |
| Antena | SMA hembra en el lateral superior, dipolo rebatible 2 dBi |
| Servicio | USB-C detrás de una tapa ciega a presión |

**Por qué riel DIN:** es el lenguaje del tablerista. Un equipo DIN se instala en el espacio
que ya existe al lado del XR06, se cablea con la misma herramienta y no requiere perforar
nada. Reduce el tiempo de instalación y la objeción "no tengo dónde ponerlo".

## 3. FC-200 IP65 (a pared)

| | |
|---|---|
| Formato | Caja ABS/PC **130 × 80 × 70 mm**, tapa con 4 tornillos cautivos y junta de poliuretano |
| Protección | **IP65** (lavados con hidrolavadora en planta: requisito real en frigoríficos) |
| Pasacables | 4 × prensacables M16 IP68 + 1 × M12 para la sonda; posiciones ciegas pre-moldeadas para no perforar en obra |
| Frente | Policarbonato traslúcido con difusor para el LED, serigrafía por el reverso (no se borra con alcohol ni con detergente alcalino) |
| Montaje | 4 bridas externas (los tornillos **nunca** atraviesan la zona estanca) + opción fleje de riel DIN |
| Antena | SMA con antena tipo puck magnética sobre cable RG-174 de 2 m, para sacarla de la cámara metálica |

Detalle de campo importante: los prensacables van **hacia abajo** y la caja monta con las
borneras en la mitad inferior, para que el agua de condensación escurra y nunca entre por
el cable (efecto mecha). Se incluye un lazo de goteo en las instrucciones.

## 4. Comunicación de estado sin app: el LED y el buzzer

Un LED RGB difuso, con un lenguaje que el técnico aprende en una visita y que está impreso
en la tapa:

| Patrón | Significado |
|---|---|
| Verde fijo | Todo normal, conectado |
| Verde latido lento (1/2 s cada 3 s) | Normal, trabajando offline (registrando en memoria) |
| Celeste pulsante | En deshielo |
| Azul intermitente rápido | Modo configuración / SoftAP esperando aparear |
| Ámbar intermitente | Advertencia: sin comunicación con el controlador, o sonda propia en falla |
| Rojo intermitente + buzzer | Alarma crítica activa (alta/baja temperatura, sonda del controlador, compresor sin corriente) |
| Rojo fijo | Falla interna del equipo |
| Blanco 1 destello | Comando recibido y aplicado (feedback inmediato al apretar en la app estando al lado) |

Buzzer 85 dB solo para alarmas críticas, con silenciado local de 30 min con el pulsador —
porque si no se puede silenciar localmente, el técnico lo desconecta y pierde el producto.
El relé 2 replica la alarma para sirena o baliza externa.

## 5. Pulsador único, tres gestos

| Gesto | Acción |
|---|---|
| Pulsación corta | Silenciar buzzer 30 min / confirmar alarma local |
| 3 s | Entrar en modo configuración (SoftAP + QR) |
| 10 s + confirmación con doble pulsación | Restablecer a fábrica |

Un solo control físico, sin menús: toda la configuración vive en la app. El dispositivo no
compite con el display del XR06, que sigue siendo la interfaz local de control.

## 6. Diseño de la sonda T/HR

```
      ┌──────────────┐
 ═════╡ ░░░░░░░░░░░░ │  ← filtro PTFE sinterizado (deja pasar vapor, no agua ni polvo)
 cable└──────────────┘
 FEP   Ø 16 × 70 mm, inox 304 o PC blanco, IP67
```

- **Sensor:** SHT45 sobre PCB-FR4 recortada, con corte de aislación térmica y conformal
  coating excepto sobre el sensor.
- **Cable:** 3 m (opc. 10 m) FEP apto alimentos, flexible a −40 °C — un PVC común se
  cristaliza y se quiebra a −25 °C. Apantallado, drenaje a masa **solo en el gateway**.
- **Filtro:** PTFE sinterizado reemplazable; con calentador del SHT45 activado 10 s cada
  hora para recuperar la membrana de condensación. En cámara de congelados se escarcha:
  el calentador y el filtro son lo que mantiene la HR creíble.
- **Soporte:** base magnética de neodimio + oreja para tornillo. Montaje a **1.5 m de
  altura, lejos de la puerta y fuera del chorro directo del evaporador** (instructivo
  ilustrado en la tapa y en la app).
- Se calibra en fábrica y se entrega con certificado de verificación de 2 puntos; para
  clientes con auditoría, opción de certificado trazable y servicio de recalibración anual
  (sonda intercambiable por conector M12, para no bajar el equipo).

## 7. Serigrafía, etiquetado y lo que ve el instalador

- **Dentro de la tapa:** diagrama de conexionado completo, impreso. El técnico no va a
  buscar el manual, va a abrir la tapa.
- **Frente:** modelo, número de serie, MAC, **QR de apareamiento** (apunta a la app con el
  ID y la clave de AP únicos), tensión de alimentación, marcados de seguridad.
- **Borneras numeradas y rotuladas** con la función, no solo con el número (`DI1 PUERTA`,
  `RL1 ON/OFF`, `A/B RS485`), en el plástico y en la PCB.
- **Código de colores** del cableado sugerido, repetido en el diagrama y en el kit de
  instalación.
- Kit en caja: gateway, sonda con soporte, dipolo, 2 prensacables de repuesto, juego de
  borneras enchufables, guía rápida de una hoja plastificada (lado A: conexionado; lado B:
  código de LED y 5 problemas frecuentes).

## 8. Instalación objetivo: 20 minutos

1. Clipar en el riel / amurar la caja (3 min).
2. Alimentación y verificación del LED de arranque (2 min).
3. Dos hilos de RS-485 al XJ485, un hilo al fin de carrera de puerta, un hilo del relé 1 a
   la entrada digital del XR06, clamp del CT sobre la fase del compresor (8 min).
4. Montar la sonda y pasar el cable (4 min).
5. Escanear el QR, apareamiento Wi-Fi, el asistente prueba E/S y comunicación con el XR06,
   dispara una alarma de prueba y se confirma que llegó al teléfono (3 min).

El paso 5 es parte del producto: una instalación no se considera terminada hasta que **una
alarma de prueba llegó al teléfono del cliente**. El asistente lo exige y lo registra en el
acta de puesta en marcha que se genera en PDF.

## 9. Lenguaje visual

Gris de tablero (RAL 7035) como base, para que el equipo se lea como instrumento
industrial y no como gadget de consumo; un único acento de color de marca en el frente y en
el aro del LED; tipografía de palo seco condensada en la serigrafía, alto contraste, cuerpo
mínimo 6 pt para que se lea con linterna; sin brillos, con textura mate anti-huella, radios
de 2 mm. El producto debe parecer lo que es: **un instrumento que vigila mercadería valiosa.**
