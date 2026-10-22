# 16 — FC-A como esclavo ModBUS (emulación XR75)

Desarrollo de la **opción 2** del [doc 14](14-kit-de-conexion-rtu.md): el gateway FC-A mide el
XR06CH por fuera y se presenta en el bus RS-485 **como si fuera un XR75**, de modo que la RTU
SmartPath y la app funcionen sin cambios.

**Esquema visual del recorrido completo:** `docs/esquema-fc-a-esclavo-modbus.html`

## 1. La idea en una frase

> La RTU es un maestro ModBUS que sabe hablar con un XR75. El XR06CH no habla. **El FC-A no
> traduce el XR06CH: lo reemplaza como interlocutor** — mide la cámara por su cuenta y
> contesta en el idioma que la RTU ya sabe.

Para la RTU, para la nube y para la app, una cámara con XR06CH + FC-A es indistinguible de
una con XR75CX. **Cero cambios aguas arriba.**

## 2. Arquitectura

```
  XR06CH (intacto)                  FC-A                          RTU
 ┌────────────────┐        ┌────────────────────────┐       ┌──────────────┐
 │ relé compresor ├───────▶│ detección de tensión   │       │              │
 │ relé deshielo  ├───────▶│ (3 canales, opto CA)   │       │  maestro     │
 │ relé ventilad. ├───────▶│                        │       │  ModBUS      │
 │                │        │ sonda NTC propia       │◀─A/B─▶│  dir. 1      │
 │ 230 Vac  ◀─────┼────────┤ relé NC de corte       │       │  9600 8N1    │
 └────────────────┘        │                        │       └──────┬───────┘
   puerta ────────────────▶│ DI1 propia             │              │ LTE
                           │                        │              ▼
                           │ ► TABLA DE REGISTROS   │          nube → app
                           │   (mapa XR75)          │
                           └────────────────────────┘
```

El FC-A corre **dos cosas a la vez**: el muestreo físico (lo que ya estaba diseñado en el
[doc 02](02-hardware.md)) y un **servidor ModBUS RTU esclavo** que publica el resultado.

## 3. Traducción: de dónde sale cada dato

| Lo que la RTU espera de un XR75 | De dónde lo saca el FC-A |
|---|---|
| Temperatura sonda 1 (cámara) | **Sonda NTC propia**, con el offset `ot` del XR06 aplicado para que coincida con el display |
| Temperatura sonda 2 (evaporador) | Sonda NTC2 propia (SKU **+P2**). Sin ella, se responde "sonda no presente", que es un estado que el mapa ya contempla |
| Estado del relé de compresor | **Detección de tensión** en la salida de compresor del XR06 |
| Estado del relé de deshielo | Ídem, salida de deshielo |
| Estado del relé de ventiladores | Ídem, salida de ventiladores |
| Estado de la entrada digital (puerta) | **DI1 propia** del FC-A (la única entrada digital del XR06 no se toca) |
| Alarma de alta / baja temperatura | Calculada por el FC-A sobre su sonda, con los umbrales `AU`/`AL`/`Ad` **copiados del XR06 en la puesta en marcha** |
| Falla de sonda (P1/P2) | Dos orígenes: falla de la **sonda propia** (directa) y falla de la **sonda del XR06** (inferida por la firma de ciclado `Cy`/`Cn`, doc 10 §6) |
| Set point | El valor **leído del display en la puesta en marcha**. Es un dato de solo lectura: el XR06 no se puede escribir |
| Estado ON/OFF | Posición real del relé de corte, **confirmada observando que las tres salidas quedaron sin tensión** |
| Buzzer | El XR06CH **no tiene** zumbador. Se responde el estado del buzzer propio del FC-A, que es el que realmente suena |

**Las direcciones concretas de cada registro salen del mapa que la RTU ya usa**, no de un
mapa nuevo: es el activo interno del equipo del M3 ([doc 12](12-plataforma-smartpath-m3.md)).
El FC-A las toma de su perfil YAML, igual que hace hoy con los perfiles de lectura.

## 4. El camino inverso: qué pasa cuando la app manda un comando

1. El usuario toca **Apagar** en la app.
2. La nube lo manda a la RTU por la red celular.
3. La RTU escribe el registro de stand-by en el bus, como lo haría con un XR75.
4. **El FC-A recibe la escritura y energiza su relé NC**, que corta los 230 V del XR06CH.
5. El XR06CH se apaga: sus tres relés abren y la cámara deja de enfriar.
6. **El FC-A verifica el efecto**, no el comando: las tres detecciones de tensión tienen que
   quedar en cero. Recién entonces actualiza el registro de estado.
7. La RTU lee ese registro en el siguiente ciclo y la app muestra "apagado" — **porque se
   observó, no porque se ordenó**.

Si a los 60 s las salidas siguen con tensión, el FC-A deja el registro en "encendido" y
levanta la alarma de fallo de comando. **La app nunca muestra un estado que no se verificó.**

### Escrituras que no se pueden cumplir

| La RTU escribe | Respuesta del FC-A |
|---|---|
| Stand-by (ON/OFF) | **Se ejecuta** con el relé de corte |
| Set point | **Excepción ModBUS** (`0x04`, fallo del dispositivo esclavo). El XR06 no tiene canal de escritura, y mentir sería peor. A definir con el equipo de la RTU cómo lo muestra la app: lo ideal es un control deshabilitado con una línea de explicación |
| Forzar deshielo | **Se puede**, con la opción **+2R**: el segundo relé del FC-A se cablea a la entrada digital del XR06 configurada como `iF = dF` (inicio de deshielo). Es la única función de escritura real que el XR06 acepta |
| Parámetros de configuración | Excepción. El FC-A no reescribe la configuración del controlador |

## 5. Comportamiento en el bus

| | |
|---|---|
| Rol | Esclavo ModBUS RTU |
| Dirección | Configurable. **1** por defecto, que es lo que la RTU espera |
| Formato | 9600 8N1 (el de la RTU) |
| Funciones soportadas | 0x03/0x04 lectura, 0x06/0x10 escritura, 0x01/0x05 coils — **las que use la RTU**, a confirmar |
| Tiempo de respuesta | < 50 ms, con el silencio de 3,5 caracteres entre tramas que exige el protocolo |
| Si el FC-A se cuelga | Deja de responder, y **la RTU lo reporta como "pérdida de comunicación con el BUS"**: el mismo aviso que daría un controlador caído. No hay un modo silencioso de fallar |
| Convivencia | Si la RTU soporta varias direcciones, en un mismo bus pueden estar un XR75CX real en la 1 y un FC-A en la 2. **Es la pregunta que más baja el costo por cámara** |

## 6. Lo que esta opción **no** resuelve (y es del controlador, no del adaptador)

- **Sin consigna remota.** El XR06CH no se puede escribir. Es la diferencia real entre el
  nivel 1 y el nivel 2.
- **Falla de sonda del controlador, con retardo.** Se infiere por la firma de ciclado: unas
  2 h con los valores por defecto, contra el aviso inmediato de un XR75.
- **Parámetros cargados a mano.** Si alguien cambia un umbral en el frente del XR06, el FC-A
  no se entera. La app muestra la antigüedad de la última verificación y recuerda revalidar
  cada 6 meses.
- **Sin humedad**, salvo que se cotice la sonda +HR.

## 7. Qué hay que construir

| Componente | Esfuerzo | Nota |
|---|---|---|
| Pila de **esclavo** ModBUS RTU en el FC-A | Bajo | `esp-modbus` ya trae el rol de esclavo; no hay que escribir el protocolo |
| Tabla de registros desde el perfil YAML | Bajo | Reusa el mecanismo de perfiles que ya tiene el firmware |
| Mapeo de las magnitudes físicas al mapa XR75 | **Medio** | Es el trabajo real: escalas, signos, bits de estado y códigos de falla. **Depende de tener el mapa de la RTU** |
| Manejo de escrituras y excepciones | Bajo | Según §4 |
| Driver RS-485 en la PCB | **Ya previsto** | Es la población FC-R (doc 02), con bornera A/B y driver de ≥ 50 mA |

**Prerequisito único y bloqueante: el mapa de registros del XR75 que usa la RTU.** Sin eso no
se puede empezar; con eso, el resto es trabajo acotado.

## 8. Validación: prueba de equivalencia

La forma correcta de probarlo es **comparar contra un XR75CX real**, que el cliente ya tiene:

1. Banco con **dos cámaras simuladas**: una con XR75CX real, otra con XR06CH + FC-A.
2. Las dos colgadas de la misma RTU (o de dos RTU idénticas) y visibles en la misma app.
3. Se ejecuta el mismo guion en las dos: enfriar, deshielo, abrir puerta, alarma de alta,
   desconectar una sonda, apagar y encender.
4. **Criterio de aceptación: la app muestra lo mismo en las dos**, con las diferencias
   esperadas y documentadas (sin consigna editable, falla de sonda con retardo).

Es una prueba que el cliente puede presenciar, y es la que convierte "funciona" en "se ve
igual que el equipo que ya conocés".

## 9. Preguntas abiertas al equipo de la RTU

1. **El mapa de registros del XR75.** Es el prerequisito.
2. **¿Qué funciones ModBUS usa la RTU** y con qué cadencia?
3. **¿Soporta varias direcciones de esclavo?** Define si es una RTU por cámara o por tablero.
4. **¿Cómo quiere que se comporte la escritura de consigna** que no se puede cumplir:
   excepción ModBUS, o algún valor convenido que la app interprete como "no editable"?
5. **¿La RTU verifica algún registro de identificación** (modelo, versión de firmware)? Si lo
   hace, hay que decidir qué contesta el FC-A: lo honesto es identificarse como FC-A y que la
   app lo muestre, en lugar de hacerse pasar por un XR75 también en eso.
