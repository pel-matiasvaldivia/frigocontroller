# 15 — Verificación de las borneras contra las fuentes oficiales

> ## ⚠️ No cablear con el esquema recibido
>
> El esquema de borneras que se nos pasó para los dos controladores **no coincide con la
> documentación oficial**. En el XR06 asigna el **compresor al borne 1, que en el manual
> oficial es la línea de alimentación de los relés**. Cablear 230 V siguiendo ese esquema
> pone tensión donde no corresponde: destruye el controlador en el mejor caso.
>
> Las fuentes primarias usadas acá son **el manual oficial del XR06CX** (`1594020160 XR06CX SP
> r1.2`, el PDF que subiste) y **el diagrama del XR75CH incluido en el manual de la RTU
> SmartPath**. Los dos están en [docs/manuales/](manuales/).

## 1. XR06CX / XR06CH — diagrama real

El manual oficial trae **dos variantes con numeración distinta**, y esto es lo más importante
de todo el documento:

### Variante "Comp. 20+8+5 A" (§14.1 del manual)

```
   Dig.Input   Evap.    Room          ← común compartido en el riel superior
      │  │      │ │      │ │
   ┌──┴──┴──────┴─┴──────┴─┴──┐
   │  9    10     11     12   │        BORNES DE SEÑAL
   └──────────────────────────┘
   ┌──────────────────────────────────────────┬─────────────┐
   │  1    2    3    4    5    6    7    8     │  Hot Key    │
   └──┬────┬────┬────┴────┴────┬────┬────┬─────┴─────────────┘
      │    │    │     └───┬────┘    │    │    └── N.C. (contacto NC del relé de deshielo)
      │    │    │         │         │    └─────── Def  (deshielo, 8(3)A 250V)
      │    │    │         │         └──────────── Line (línea del relé de deshielo)
      │    │    │         └────────────────────── Supply 230V~ 50-60Hz  (bornes 4 y 5)
      │    │    └──────────────────────────────── Compresor  (20(8)A, máx 20A, 16FLA/96LRA)
      │    └───────────────────────────────────── Fan  (5(2)A)
      └────────────────────────────────────────── Line (común de relés, máx 20A)
```

### Variante "Comp. 8+8+8 A" (§14.2) — **la numeración cambia**

```
   ┌──────────────────────────┐
   │  9    10     11     12   │        Dig.Input · Evap. · Room (igual que arriba)
   └──────────────────────────┘
   ┌───────────────────────────────────┬─────────────┐
   │  1    2    3    4    5    6    7   │  Hot Key    │
   └──┬────┬────┬────┬────┬────┴────┴───┴─────────────┘
      │    │    │    │    │    └───┬───── Supply 230V~ 50-60Hz  (bornes 6 y 7)
      │    │    │    │    └────────────── Comp
      │    │    │    └─────────────────── Fan
      │    │    └──────────────────────── N.C.
      │    └───────────────────────────── Def (deshielo, 8(3)A)
      └────────────────────────────────── Line (común de relés, máx 16A)
```

**El mismo modelo, con dos asignaciones de bornes completamente distintas.** El propio manual
lo advierte para la alimentación: *"modelos 120Vac. Utilizar terminales de alimentación 4 y
5"* en una variante y *"6 y 7"* en la otra. **Ninguna tabla genérica sirve: hay que leer la
etiqueta del equipo que se va a cablear.**

**Y el dato que define el proyecto:** el XR06 tiene **12 bornes + el conector Hot Key de 5
pines, y nada más. No hay bornes de RS485.** Confirma el doc 10: este controlador no habla
ModBUS por ningún medio.

## 2. XR75CH — diagrama real (del manual de la RTU)

```
        RS485            D.I.2  D.I.1  Pb3   Pb4   Evap.  Room
        ┌──┴──┐            │      │     │     │      │      │
   ┌────┴─────┴────────────┴──────┴─────┴─────┴──────┴──────┴───┐
   │  16(+)  17(−)   18    19    20    21    22    23    24     │   BORNES DE SEÑAL
   └────────────────────────────────────────────────────────────┘
   ┌────────────────────────────────────────────┬──────────────┐
   │  1    2    3    4    5    6    7    8      │ Hot Key      │
   └──┬────┬────┬────┬────┴────┴────┬────┬──────┴ X-Rep ───────┘
      │    │    │    │    └────┬────┘    │    └── N.C.
      │    │    │    │         │         └─────── Def (8(3)A 250V · UL 10A)
      │    │    │    │         └───────────────── Supply 230V~ 50-60Hz
      │    │    │    └─────────────────────────── Aux / Light (8(3)A)
      │    │    └──────────────────────────────── Comp (16(5)A)
      │    └───────────────────────────────────── Fan (5A 250V)
      └────────────────────────────────────────── Line (común de relés, máx 16A)
```

**`16 = RS485 +` y `17 = RS485 −`**, con el **Hot Key / X-Rep como conector aparte**. Cuatro
relés (Fan, Comp, Aux/Light, Def), dos entradas digitales (D.I.1, D.I.2) y cuatro sondas
(Pb3, Pb4, Evap., Room). **No existen bornes 9 a 15.**

> El diagrama es del **XR75CH**. El equipo del cliente es **XR75CX-5N7I3**: misma familia y
> casi con seguridad la misma disposición, pero **hay que confirmarlo con la etiqueta de ese
> equipo** antes de cablear.

## 3. Comparación, error por error

| Afirmación recibida | Documentación oficial | Riesgo |
|---|---|---|
| XR06: "Pin 1 = Salida Relé Compresor (NO)" | **Borne 1 = Line**, común de alimentación de los relés | **Crítico.** Se cablea una carga al vivo común |
| XR06: "Pin 2 = Relé Deshielo" | **Borne 2 = Fan** (variante 20+8+5) ó Def (variante 8+8+8) | Alto: carga al relé equivocado |
| XR06: "Pin 3 = Relé Ventilador" | **Borne 3 = Compresor** (variante 20+8+5) | Alto: **el ventilador queda en el relé de 20 A y el compresor en el de 5 A** |
| XR06: "Pin 8 = común sondas; 9 = P1; 10 = P2; 11 = DI" | **8 = N.C.** (contacto de red del relé de deshielo). Las señales están en **9-12** | **Crítico.** Pone una sonda en un borne a 230 V |
| XR06: "12 bornes en fila corrida" | **8 bornes de potencia + 4 de señal, en dos borneras separadas**, más Hot Key | Confunde la lectura completa |
| XR06: "IP65 frontal / IP20 posterior" | Correcto (el manual dice IP20, frontal IP65) | — |
| XR06: capacidades 20(8)A / 8(3)A / 5(2)A | Correcto | — |
| XR75: "Sondas en pines 11-15, DI2 en 10, RS485 en pin 16" | Sondas en **21-24**, DI en **19-20**, **RS485 en 16(+) y 17(−)**. **No hay bornes 9-15** | **Crítico** |
| XR75: "Pin 16: DI2 **o** RS485 **o** Hot Key" | Son **tres cosas distintas**: 16/17 RS485, 19/20 entradas digitales, Hot Key conector aparte | Alto |
| XR75: 4 relés, hasta 4 sondas NTC, 2 DI, formato 32×74 | Correcto | — |

El patrón es claro: **las capacidades y las características generales están bien; la
numeración de bornes está inventada.** Es exactamente el tipo de dato que se ve plausible y
que hay que verificar siempre contra el manual del fabricante o la etiqueta del equipo.

## 4. Los enlaces oficiales son el camino correcto, pero no los puedo descargar

Los dos enlaces que pasaste apuntan al sitio oficial de Copeland y son la fuente que
corresponde:

- `webapps.copeland.com/.../E-CLASS/XR06CHNG/XR06CH-NG-ES.pdf`
- `webapps.copeland.com/.../XR-CX/XR75CX/XR75CX-GB.pdf`

**El contenedor de esta sesión tiene el acceso a ese dominio bloqueado** (el proxy devuelve
403), así que no pude abrirlos. **Si los descargás y los subís al chat, los proceso igual que
hice con el manual del XR06CX**, y con eso cierro:

- la numeración exacta de bornes del **XR06CH-NG**, que es variante *NG* y puede diferir del
  XR06CX que ya analizamos;
- si el **XR75CX** trae la bornera RS-485 (Kit A) o solo el puerto TTL (Kit B, + XJ485);
- la lista real de parámetros de cada uno, con la que se cargan los umbrales de alarma;
- si el XR75CX tiene parámetro `Adr` y el formato serie, para la conexión a la RTU.

## 5. Mientras tanto, la verificación que no depende de ningún PDF

**En el equipo mismo, con el tablero sin tensión y una foto:**

1. **Foto de la etiqueta lateral/trasera** con el código completo y la serigrafía de bornes.
   El propio controlador trae impreso su esquema: es la fuente más confiable, porque
   corresponde exactamente a la variante que se tiene en la mano.
2. **Contar los bornes.** XR06: 8 + 4 + Hot Key. XR75: 8 + 9 + Hot Key/X-Rep.
3. **Buscar los bornes 16 y 17 rotulados `RS485 + −`** en el XR75CX → define Kit A o Kit B.
4. **Entrar al menú y buscar `Adr`** → confirma si es esclavo ModBUS.

## 6. Qué no cambia y qué sí

**No cambia** el diseño: el XR06 sigue sin ModBUS (ahora con el diagrama oficial a la vista,
que muestra 12 bornes y ningún RS485), y el XR75 sigue siendo el candidato de nivel 2.

**Sí cambia** una cosa en el procedimiento de instalación, y hay que meterla en la guía
rápida y en el asistente de la app: **la numeración de bornes varía entre sub-variantes del
mismo modelo.** El instalador nunca debe cablear de memoria ni desde una tabla impresa: tiene
que leer la serigrafía del equipo que tiene adelante. Eso vale tanto para las tres
derivaciones de detección del FC-A como para el bus RS-485 de la RTU.
