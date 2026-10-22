# 07 — BOM y costeo (rev. 3)

> Valores de referencia en **USD, octubre 2026**. Rev. 3 = camino A (envolvente no invasiva
> del XR06CX), según [doc 10](10-hallazgos-manual-xr06cx.md).

## 1. BOM del FC-A (equipo base: Wi-Fi, 3 detecciones de CA, 2 NTC, 1 relé de corte, 2 DI)

| # | Componente | Ref. sugerida | Cant. | @100 u | @500 u | @2000 u |
|---|---|---|---|---|---|---|
| 1 | Módulo MCU + Wi-Fi | ESP32-C3-MINI-1U-N4 | 1 | 2.20 | 1.90 | 1.65 |
| 2 | **Detección de salidas del XR06** | 3 × H11AA1 + 6 × 100 kΩ ½ W + RC | 3 | 1.35 | 1.10 | 0.88 |
| 3 | **ADC de precisión** | ADS1115 + ref. 0,1 % | 1 | 1.05 | 0.85 | 0.68 |
| 4 | **Sonda de cámara armada** | NTC 10 k B3435, cápsula IP67, 3 m FEP, soporte | 1 | 3.90 | 3.20 | 2.60 |
| 5 | Fuente AC/DC 3 W | HLK-PM05 | 1 | 2.40 | 2.00 | 1.70 |
| 6 | Buck 3V3 | MP2451 + L | 1 | 0.40 | 0.32 | 0.26 |
| 7 | Relé de corte SPDT 8 A | Hongfa HF32F / HF115F | 1 | 1.30 | 1.10 | 0.90 |
| 8 | Driver de relé + snubber RC + MOV | — | 1 | 0.45 | 0.38 | 0.30 |
| 9 | Entradas digitales opto | 2 × PC817 + R/C | 2 | 0.35 | 0.28 | 0.22 |
| 10 | RTC + respaldo | PCF85063A + supercap 0.47 F | 1 | 1.00 | 0.82 | 0.68 |
| 11 | Protecciones | MOV, fusible, TVS, PTC | — | 1.10 | 0.90 | 0.72 |
| 12 | Borneras a tornillo | 5.08 mm (8 p) + 3.5 mm (10 p) | — | 1.50 | 1.20 | 0.95 |
| 13 | Gabinete DIN 4M genérico + etiqueta impresa | — | 1 | 2.60 | 2.10 | 1.70 |
| 14 | Antena + pigtail | Whip 2 dBi + IPEX-SMA | 1 | 1.40 | 1.10 | 0.90 |
| 15 | LED RGB, buzzer, pulsador | — | — | 0.95 | 0.78 | 0.62 |
| 16 | PCB 2 capas, 80 × 60 mm | HASL | 1 | 1.40 | 0.90 | 0.58 |
| 17 | Pasivos, conectores, misc. | — | — | 1.60 | 1.30 | 1.05 |
| | **Subtotal materiales** | | | **24.95** | **20.23** | **16.39** |
| | Ensamblado + test funcional (EMS) | | | 8.00 | 5.50 | 3.80 |
| | Merma y scrap (3 %) | | | 0.75 | 0.61 | 0.49 |
| | **Costo total por equipo** | | | **≈ 33.70** | **≈ 26.34** | **≈ 20.68** |

**Sin módulos externos.** El costo instalado por punto es el costo del equipo: no hay XJ485
(USD 35-55) ni ningún accesorio obligatorio de comunicación, porque no hay comunicación.

### Comparación de las tres revisiones

| | Rev. 1 (XJ485) | Rev. 2 (TTL directo) | **Rev. 3 (envolvente)** |
|---|---|---|---|
| Equipo @500 u. | 63.88 | 24.79 | **26.34** |
| Interfaz obligatoria | +45.00 | 0 | 0 |
| **Costo instalado** | **108.88** | **24.79** | **26.34** |
| **¿Funciona sobre un XR06CX?** | **No** | **No** | **Sí** |

La rev. 3 cuesta USD 1.55 más que la rev. 2 y es la única de las tres que se puede instalar.
Lo que se agregó (sonda, ADC y detección de salidas: +5.43) casi se compensó con lo que se
eliminó al no haber bus serie (aislador, DC/DC aislado y cable TTL: −3.70).

## 2. Poblaciones y accesorios (misma PCB)

| SKU | Qué agrega | Δ BOM @500 | Cuándo se cotiza |
|---|---|---|---|
| **FC-A** | Base: envolvente del XR06CX | — | Instalación estándar, sin tocar el controlador |
| **FC-R** | SP3485 + bornera A/B **+ puerto TTL con driver de ≥ 50 mA** | +1.30 | **Camino B:** controlador con ModBUS. Las dos salidas, porque la mayoría de los candidatos usa TTL + adaptador (doc 11 §1) |
| **+P2** | Segunda sonda NTC de evaporador | +3.20 | Habilita alarma de deshielo inefectivo y detección de `P2` en falla |
| **+HR** | Sonda T/HR SHT45 (cápsula IP67, filtro PTFE, 3 m FEP) | +7.80 | **Única forma de cubrir humedad.** Frescos, maduración, auditoría de HR |
| **+CT** | Clamp 10 A + acondicionamiento | +2.60 | Confirmación física de marcha y consumo del compresor |
| **+2R** | 2º relé (sirena/baliza) | +1.48 | Aviso sonoro externo |
| **IP65** | Caja IP65 + prensacables + antena puck | +9.50 | Zonas de lavado, montaje a la vista |
| **LTE** | SIM7670G + antena + SIM | +28.00 | Sitio sin Wi-Fi confiable |

## 3. Camino B: costo de reemplazar el controlador

Para el cliente que exija **setup de temperatura remoto**, que el XR06CX no puede dar:

| Concepto | USD |
|---|---|
| Gateway FC-R @500 u. | 24.60 |
| Controlador con ModBUS RTU documentado (Danfoss ERC 213, Carel IR33+ o equivalente vigente — doc 11) | 60–120 |
| Adaptador TTL→RS-485, **solo si el controlador elegido no trae RS-485 a bordo** | 0 ó 35–55 |
| Mano de obra de reemplazo y re-cableado (1–2 h, con corte de servicio) | 50–90 |
| **Total instalado por cámara** | **135–235** (con RS-485 a bordo) · **170–290** (si requiere adaptador) |

**El controlador se elige por el mapa de registros, no por el precio** (doc 11): un
controlador USD 20 más barato cuyo fabricante no publica los registros cuesta semanas de
ingeniería inversa. Y con RS-485 a bordo se ahorran USD 35-55 por cámara de adaptador.

Se cotiza **por cámara**, no por sitio: lo habitual es que el cliente quiera consigna remota
en dos o tres cámaras críticas y monitoreo en el resto. El producto soporta las dos cosas en
el mismo tablero y en la misma app.

## 4. Costo de la nube (sin cambios)

| Concepto | @100 equipos | @1000 equipos |
|---|---|---|
| Broker + ingesta | 60–90 /mes | 220–350 /mes |
| Series temporales + Postgres | 50–80 /mes | 180–280 /mes |
| Push, WhatsApp, email | 15–40 /mes | 90–250 /mes |
| Monitoreo y backups | 25 /mes | 70 /mes |
| **Por equipo** | **≈ 1.5–2.4 /mes** | **≈ 0.6–1.0 /mes** |

## 5. Modelo de precio (rev. 3)

| | Equipo (una vez) | Abono mensual |
|---|---|---|
| **FC-A** Wi-Fi (monitoreo + ON/OFF remoto) | **USD 105–140** | USD 6–8 |
| + sonda de evaporador | + USD 20–28 | — |
| + sonda T/HR | + USD 45–60 | — |
| + CT de compresor | + USD 25–35 | — |
| IP65 | + USD 30–40 | — |
| LTE | + USD 70–90 | USD 11–14 |
| **Upgrade "control total"** (FC-R + controlador con ModBUS + instalación) | **USD 290–390**, ó USD 340–450 si el controlador requiere adaptador TTL | USD 8–10 |
| Instalación y puesta en marcha FC-A | USD 60–110 | — |

Margen bruto de hardware **72–78 %** a 500 u.; margen del abono > 85 %.

**El catálogo de dos niveles es el mejor resultado comercial de este replanteo.** En lugar de
prometer una función que el controlador del cliente no puede dar, hay:

- **Nivel 1 (FC-A, USD 105-140):** monitoreo completo, alarmas, deshielo, compresor, puerta y
  ON/OFF remoto, **sin tocar el controlador ni cortar el servicio**. Entra por decisión del
  encargado de mantenimiento, se instala en 15 minutos.
- **Nivel 2 (upgrade, USD 290-390):** agrega consigna remota reemplazando el controlador. Se
  vende solo donde se necesita, y la objeción de precio se resuelve sola porque el cliente ya
  tiene el nivel 1 funcionando y pide el upgrade donde le hace falta.

Y el argumento de valor no cambia: una cámara de congelados llena vale USD 15 000–40 000 en
mercadería. **Una sola pérdida evitada paga más de 100 equipos del nivel 1.**
