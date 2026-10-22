# Diagrama de conexionado de campo (rev. 2)

## 1. Vista general — FC-E (equipo base)

```
                           ╔════════════════════════════════════════════════╗
  220 VAC ──[F 500mA]──────╢ L   N   ⏚              FRIGOCONTROLLER FC-E    ║
   (ó 12-24 VDC) ──────────╢ V+  V-                                         ║
                           ║                                                ║
  XR06 puerto TTL ─────────╢ ▭ conector TTL (cable armado de 1 m, blanco)   ║
   [FC-R: XJ485 ───────────╢ A   B   GNDiso   [jumper 120Ω] ]               ║
                           ║                                                ║
  Fin de carrera puerta ───╢ DI1+ DI1-   (solo si NO está cableado al XR06) ║
  Alarma XR06 / térmico ───╢ DI2+ DI2-   (NA/NC por software)               ║
                           ║                                                ║
  → entrada digital XR06 ──╢ RL1-C  RL1-NC  RL1-NA   (ON/OFF remoto)        ║
                           ║                                                ║
  OPCIONALES (sin poblar en el equipo base):                                ║
  Sonda T/HR accesorio ────╢ +3V3 SDA SCL GND  (M12, malla a ⏚ solo aquí)   ║
  CT clamp compresor ──────╢ CT+  CT-                                       ║
  Sirena / baliza ─────────╢ RL2-C  RL2-NC  RL2-NA                          ║
                           ╚════════════════════════════════════════════════╝
```

**Dos hilos menos que la rev. 1, y una caja menos.** La temperatura, el estado del
compresor, el deshielo, las alarmas y —si el fin de carrera está cableado al controlador—
las aperturas de puerta viajan todas por el cable TTL. En la instalación típica solo hay que
conectar: alimentación, cable TTL y un hilo del relé 1.

## 2. ON/OFF remoto: el detalle que define la seguridad

```
   XR06  i1 (entrada digital configurada i1F = oFF)
         ┌───────────────┐
         │ i1    GND     │
         └──┬────────┬───┘
            │        │
       RL1-NC     RL1-C        ← se usa el contacto NC: relé SIN energía = circuito
            │        │            cerrado = cámara ENCENDIDA
         ┌──┴────────┴───┐
         │ FRIGOCONTROLLER│
         └───────────────┘
```

Con el contacto **NC**, un gateway sin alimentación, colgado o retirado deja la cámara
funcionando. Para apagar, el gateway debe energizar activamente el relé. **Nunca al
revés.** Y nunca el relé del gateway en serie con la bobina del contactor del compresor:
se perderían los tiempos mínimos de protección que gestiona el XR06.

## 3. CT del compresor (accesorio, SKU +CT)

**El estado de marcha del compresor ya llega por Modbus** (bit del relé), que es lo que pide
el requerimiento. El clamp agrega *confirmación física*, para clientes con mantenimiento
predictivo.

El clamp se cierra alrededor de **una** fase de alimentación del compresor, del lado de la
carga del contactor. No requiere cortar ni desconectar nada: se instala con el equipo en
servicio. Sirve para tres diagnósticos que el Modbus solo no da:

- el XR06 pidió compresor y no hay corriente → térmico, fase o contactor (**alarma crítica**);
- hay corriente sin pedido → contactor pegado (**alarma crítica, riesgo de congelar producto
  fresco y de dañar el compresor**);
- arranques por hora y consumo medio → ciclado corto, pérdida de carga de gas, evaporador
  sucio (**mantenimiento predictivo**).

## 4. Buenas prácticas de cableado

1. Señales (RS-485, DI, sonda) **separadas ≥ 10 cm** de los cables de potencia y cruzándolos
   a 90°, nunca en paralelo dentro de la misma canaleta.
2. Apantallado del cable TTL (y del RS-485 y la sonda): malla aterrada **en un solo extremo**
   (el gateway). Aterrarla en los dos extremos crea un lazo de masa y es la causa típica de
   errores de CRC.
3. **El cable TTL no se prolonga ni se empalma.** Es TTL asimétrico de 5 V: tolera 1-2 m
   dentro del tablero, no 10 m por una canaleta. Si hace falta más distancia, se pasa a la
   población FC-R con RS-485, que para eso existe.
4. Terminación de 120 Ω con el jumper puesto solo si el gateway es el extremo físico del bus
   (solo aplica a FC-R).
5. Cable de sonda con aislación apta baja temperatura (FEP/silicona). El PVC se quiebra.
6. Lazo de goteo en todo cable que entre a la caja IP65, y prensacables apretados al par
   correcto con el elastómero adecuado al diámetro del cable (un prensacables flojo
   convierte una IP65 en una IP20).
7. Dejar 30 cm de reserva de cable en el tablero: el próximo técnico lo va a agradecer y el
   equipo se podrá bajar para servicio sin descablear.
