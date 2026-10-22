# 07 — BOM y costeo (rev. 2)

> Valores de referencia en **USD, octubre 2026**, para estimación de viabilidad; requieren
> cotización firme. El historial de recortes está en
> [09-optimizacion-de-costos.md](09-optimizacion-de-costos.md).

## 1. BOM del FC-E (equipo base: Wi-Fi, TTL directo al XR06, 1 relé, 2 DI)

| # | Componente | Ref. sugerida | Cant. | @100 u | @500 u | @2000 u |
|---|---|---|---|---|---|---|
| 1 | Módulo MCU + Wi-Fi | ESP32-C3-MINI-1U-N4 | 1 | 2.20 | 1.90 | 1.65 |
| 2 | Aislador digital + DC/DC aislado 1 W | ADuM1201 + B0505S-1W | 1 | 2.70 | 2.25 | 1.85 |
| 3 | Cable TTL armado al XR06 (1 m, conector) | — | 1 | 1.80 | 1.45 | 1.15 |
| 4 | Fuente AC/DC 3 W | HLK-PM05 | 1 | 2.40 | 2.00 | 1.70 |
| 5 | Buck 3V3 | MP2451 + L | 1 | 0.40 | 0.32 | 0.26 |
| 6 | Relé SPDT 16 A | Hongfa HF115F-I | 1 | 1.30 | 1.10 | 0.90 |
| 7 | Driver de relé + snubber RC + MOV | — | 1 | 0.45 | 0.38 | 0.30 |
| 8 | Entradas digitales opto | 2 × PC817 + R/C | 2 | 0.35 | 0.28 | 0.22 |
| 9 | RTC + respaldo | PCF85063A + supercap 0.47 F | 1 | 1.00 | 0.82 | 0.68 |
| 10 | Protecciones | MOV, fusible, TVS, PTC | — | 1.10 | 0.90 | 0.72 |
| 11 | Borneras a tornillo | 5.08 mm (6 p) + 3.5 mm (8 p) | — | 1.50 | 1.20 | 0.95 |
| 12 | Gabinete DIN 4M genérico + etiqueta impresa | — | 1 | 2.60 | 2.10 | 1.70 |
| 13 | Antena + pigtail | Whip 2 dBi + IPEX-SMA | 1 | 1.40 | 1.10 | 0.90 |
| 14 | LED RGB, buzzer, pulsador | — | — | 0.95 | 0.78 | 0.62 |
| 15 | PCB 2 capas, 75 × 55 mm | HASL | 1 | 1.30 | 0.85 | 0.55 |
| 16 | Pasivos, conector de sonda opcional, misc. | — | — | 1.60 | 1.30 | 1.05 |
| | **Subtotal materiales** | | | **23.05** | **18.73** | **15.20** |
| | Ensamblado + test funcional (EMS) | | | 8.00 | 5.50 | 3.80 |
| | Merma y scrap (3 %) | | | 0.69 | 0.56 | 0.46 |
| | **Costo total por equipo** | | | **≈ 31.74** | **≈ 24.79** | **≈ 19.46** |

**Sin módulo externo:** el costo instalado por punto es el costo del equipo. En la rev. 1
había que sumarle el XJ485 (USD 35-55).

## 2. Poblaciones y accesorios (misma PCB)

| SKU | Qué agrega | Δ BOM @500 | Cuándo se cotiza |
|---|---|---|---|
| **FC-E** | Base: TTL directo | — | Instalación estándar sobre XR06 |
| **FC-R** | SP3485 + bornera A/B en lugar del cable TTL (usa XJ485 o bus RS-485 existente) | +0.90 (y el cliente aporta/compra el XJ485) | Cliente que exige soporte oficial, bus ya existente, u otros combistatos |
| **+HR** | Sonda T/HR remota SHT45, cápsula IP67, filtro PTFE, 3 m FEP, soporte | +7.80 | Cámaras de frescos, maduración o con auditoría de humedad |
| **+CT** | Clamp 10 A + acondicionamiento | +2.60 | Clientes con mantenimiento predictivo de compresores |
| **+2R** | 2º relé (sirena/baliza/luz) | +1.48 | Cuando se pide aviso sonoro local externo |
| **IP65** | Caja IP65 130×80×70 + prensacables + antena puck | +9.50 | Montaje a la vista, zonas de lavado |
| **LTE** | SIM7670G + antena + SIM | +28.00 | Sitio sin Wi-Fi confiable |
| **DC-in** | Entrada 12-24 VDC en lugar de AC/DC | **−1.70** | Tablero que ya tiene fuente: **más barato que el base** |

## 3. Costo de la nube (sin cambios)

| Concepto | @100 equipos | @1000 equipos |
|---|---|---|
| Broker + ingesta | 60–90 /mes | 220–350 /mes |
| Series temporales + Postgres | 50–80 /mes | 180–280 /mes |
| Push, WhatsApp, email | 15–40 /mes | 90–250 /mes |
| Monitoreo y backups | 25 /mes | 70 /mes |
| **Por equipo** | **≈ 1.5–2.4 /mes** | **≈ 0.6–1.0 /mes** |

Eliminar los sensores propios **reduce** además el volumen de telemetría ~35 % (menos
campos por muestra), con efecto menor pero real en el costo de la base de series.

## 4. Modelo de precio (rev. 2)

| | Equipo (una vez) | Abono mensual por equipo |
|---|---|---|
| FC-E Wi-Fi (base) | **USD 95–130** | USD 6–8 |
| FC-R (con XJ485 del cliente) | USD 105–140 | USD 6–8 |
| + sonda T/HR | + USD 45–60 | — |
| + CT de compresor | + USD 25–35 | — |
| IP65 | + USD 30–40 | — |
| LTE | + USD 70–90 | USD 11–14 (incluye datos) |
| Instalación y puesta en marcha | USD 60–110 | — |

Margen bruto de hardware **70–78 %** a 500 u. (mejor que la rev. 1 pese al precio más bajo,
porque el costo cayó más que el precio); margen del abono > 85 %.

**El precio bajo es una decisión estratégica, no solo un ahorro.** A USD 110 el punto, el
equipo entra por decisión del encargado de mantenimiento; a USD 280 necesita aprobación de
gerencia y compite con un presupuesto anual. Un parque de 8 cámaras pasa de USD 2 240 a
USD 880: la diferencia entre "lo evaluamos el año que viene" y "pongamos las ocho".

Y el argumento de valor no cambia: una cámara de congelados llena vale USD 15 000–40 000 en
mercadería. Ahora **una sola pérdida evitada paga 120-400 equipos.**
