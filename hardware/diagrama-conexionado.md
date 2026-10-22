# Diagrama de conexionado de campo (rev. 2)

## 1. Vista general — FC-A (envolvente del XR06CX)

```
                            ╔══════════════════════════════════════════════════╗
  230 VAC ──[F 500mA]───────╢ L   N   ⏚            FRIGOCONTROLLER FC-A        ║
                            ║                                                  ║
  ── CORTE DE ALIMENTACIÓN DEL XR06 (ON/OFF remoto) ─────────────────────────   ║
  L de red ─────────────────╢ RL1-C                                            ║
  → L del XR06 ─────────────╢ RL1-NC    (NC: sin energía, el XR06 queda ON)    ║
                            ║                                                  ║
  ── DETECCIÓN DE SALIDAS (derivación EN PARALELO, 230 V) ───────────────────   ║
  XR06 salida compresor ────╢ AC1   (marrón)                                   ║
  XR06 salida deshielo ─────╢ AC2   (azul)                                     ║
  XR06 salida ventiladores ─╢ AC3   (negro)                                    ║
  Neutro común ─────────────╢ ACN                                              ║
                            ║                                                  ║
  ── SEÑALES ────────────────────────────────────────────────────────────────   ║
  Sonda NTC1 cámara ────────╢ NTC1+ NTC1-   (obligatoria)                      ║
  Sonda NTC2 evaporador ────╢ NTC2+ NTC2-   (opcional, +P2)                    ║
  Fin de carrera puerta ────╢ DI1+  DI1-    (contacto limpio, NA/NC por soft)  ║
  Térmico / presostato ─────╢ DI2+  DI2-                                       ║
                            ║                                                  ║
  OPCIONALES:                                                                  ║
  Sonda T/HR (M12) ─────────╢ +3V3 SDA SCL GND   (+HR)                         ║
  CT clamp compresor ───────╢ CT+  CT-           (+CT)                         ║
  Sirena / baliza ──────────╢ RL2-C RL2-NC RL2-NA (+2R)                        ║
  [FC-R: RS-485 ────────────╢ A  B  GNDiso  [120Ω] — camino B ]                ║
                            ╚══════════════════════════════════════════════════╝
```

**El XR06CX no se modifica ni se reconfigura**, con una sola excepción: se ajusta `od` ≥ 3 min
para proteger el compresor cuando el gateway restituye la alimentación.

## 1.1 Las tres derivaciones de detección

```
  XR06 bornera salida compresor ──┬──────────▶ bobina del contactor   (cableado existente)
                                  │
                                  └──▶ AC1 del gateway   (hilo nuevo, 0,75 mm², marrón)
```

- **En paralelo, nunca en serie.** El gateway observa; no interrumpe ni carga el circuito de
  mando. Si el gateway se quema, el contactor sigue funcionando.
- Terminal tipo horquilla en la **misma bornera** donde ya está el cable del contactor: no hay
  que cortar ni empalmar nada.
- **Colores fijos:** marrón = compresor · azul = deshielo · negro = ventiladores. Repetidos en
  la bornera del gateway, en la etiqueta de la tapa y en la guía rápida.
- El canal de deshielo va al **común del relé SPDT de deshielo**, para que funcione tanto con
  `td = EL` (resistencia eléctrica) como con `td = in` (gas caliente).
- Verificación en obra: **un deshielo manual con la tecla del XR06** debe encender el canal
  AC2 en la app. Es la prueba de cableado que exige el asistente de instalación.

## 2. ON/OFF remoto: el detalle que define la seguridad

```
   Red                                             XR06CX
    L ──────┬── RL1-C                           ┌──────────┐
            │                                   │          │
            └── RL1-NC ────────────────────────▶│ L        │  3,5 VA
                                                │          │
    N ─────────────────────────────────────────▶│ N        │
                ▲                               └──────────┘
                │ contacto NC: relé SIN energía = XR06 alimentado = cámara ENFRIANDO
         ┌──────┴────────┐
         │FRIGOCONTROLLER│  + watchdog de hardware: si el firmware se cuelga con el
         └───────────────┘    relé energizado, se libera solo
```

Con el contacto **NC**, un gateway sin alimentación, colgado o retirado deja la cámara
funcionando. Para apagar, el gateway debe energizar activamente el relé. **Nunca al revés.**

**Y nunca el relé del gateway en serie con la bobina del contactor del compresor:** el XR06 no
se enteraría del corte, y al reponer cerraría su relé sin esperar el retardo anticiclos `AC`.
Es la forma más fácil de romper un compresor.

**Dos parámetros del XR06 que hay que dejar configurados** al usar este camino:
`od` ≥ 3 min (retarda las salidas al reponer tensión: protege el compresor) y verificar `dP`
(si está en `y`, el controlador deshiela cada vez que vuelve la tensión).

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
2. Apantallado de las sondas (y del RS-485 en FC-R): malla aterrada **en un solo extremo** (el
   gateway). Aterrarla en los dos extremos crea un lazo de masa y mete ruido en la medición.
3. **Las tres derivaciones son cableado de red:** van por la canaleta de potencia, con cable
   de 0,75 mm² resistente al calor (el manual del XR06 lo exige para sus propias conexiones),
   y **nunca** por la canaleta de señales junto a las sondas.
4. Terminación de 120 Ω con el jumper puesto solo si el gateway es el extremo físico del bus
   (solo aplica a FC-R, camino B).
5. Cable de sonda con aislación apta baja temperatura (FEP/silicona). El PVC se quiebra. Y la
   sonda se monta **con el bulbo hacia arriba** y lejos de los flujos de aire, como indica el
   manual del XR06 para las suyas (§10.1).
6. Lazo de goteo en todo cable que entre a la caja IP65, y prensacables apretados al par
   correcto con el elastómero adecuado al diámetro del cable (un prensacables flojo
   convierte una IP65 en una IP20).
7. Dejar 30 cm de reserva de cable en el tablero: el próximo técnico lo va a agradecer y el
   equipo se podrá bajar para servicio sin descablear.
