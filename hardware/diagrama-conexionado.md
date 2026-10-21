# Diagrama de conexionado de campo

## 1. Vista general

```
                           ╔════════════════════════════════════════════════╗
  220 VAC ──[F 1A]─────────╢ L   N   ⏚                 FRIGOCONTROLLER      ║
                           ║                                                ║
  XR06 ── TTL ── XJ485 ────╢ A   B   GNDiso   [jumper 120Ω]                 ║
                           ║                                                ║
  Fin de carrera puerta ───╢ DI1+ DI1-        (NA: cierra al abrir)         ║
  Presostato alta (NC) ────╢ DI2+ DI2-                                      ║
  Monitor de fase (NC) ────╢ DI3+ DI3-                                      ║
  Contacto alarma XR06 ────╢ DI4+ DI4-                                      ║
                           ║                                                ║
  Sonda T/HR (4 hilos) ────╢ +3V3 SDA SCL GND  (malla a ⏚ solo aquí)        ║
  CT clamp compresor ──────╢ CT+  CT-                                       ║
                           ║                                                ║
  → entrada digital XR06 ──╢ RL1-C  RL1-NA  RL1-NC   (ON/OFF remoto)        ║
  → sirena / baliza ───────╢ RL2-C  RL2-NA  RL2-NC                          ║
                           ╚════════════════════════════════════════════════╝
```

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

## 3. CT del compresor

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
2. Apantallado del RS-485 y de la sonda: malla aterrada **en un solo extremo** (el gateway).
   Aterrarla en los dos extremos crea un lazo de masa y es la causa típica de errores de CRC.
3. Terminación de 120 Ω con el jumper puesto solo si el gateway es el extremo físico del bus.
4. Cable de sonda con aislación apta baja temperatura (FEP/silicona). El PVC se quiebra.
5. Lazo de goteo en todo cable que entre a la caja IP65, y prensacables apretados al par
   correcto con el elastómero adecuado al diámetro del cable (un prensacables flojo
   convierte una IP65 en una IP20).
6. Dejar 30 cm de reserva de cable en el tablero: el próximo técnico lo va a agradecer y el
   equipo se podrá bajar para servicio sin descablear.
