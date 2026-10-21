# 07 — BOM y costeo

> Valores de referencia en **USD, octubre 2026**, para estimación de viabilidad. Requieren
> cotización firme: los precios de módulos y relés se mueven mucho y el costo final depende
> del canal de importación.

## 1. BOM del FC-100 (Wi-Fi, riel DIN)

| # | Componente | Ref. sugerida | Cant. | @100 u | @500 u | @2000 u |
|---|---|---|---|---|---|---|
| 1 | Módulo MCU + Wi-Fi | ESP32-S3-WROOM-1U-N8R2 | 1 | 3.90 | 3.40 | 3.00 |
| 2 | Transceptor RS-485 aislado | ADM2587EBRWZ | 1 | 6.80 | 5.90 | 5.10 |
| 3 | Sensor T/HR | SHT45-AD1B | 1 | 3.60 | 3.00 | 2.50 |
| 4 | Fuente AC/DC 3 W | Mornsun LS03-13B05R3 | 1 | 5.20 | 4.40 | 3.80 |
| 5 | Buck 3V3 | TPS62743 + L | 1 | 0.90 | 0.75 | 0.60 |
| 6 | Relés SPDT 16 A | Hongfa HF115F-I | 2 | 2.60 | 2.20 | 1.80 |
| 7 | Opto 4 canales | TLP291-4 | 1 | 0.55 | 0.45 | 0.38 |
| 8 | RTC + celda | DS3231SN + CR2032 | 1 | 2.10 | 1.75 | 1.45 |
| 9 | Protecciones | MOV, TVS SM712, PESD, PTC, fusible | — | 1.60 | 1.30 | 1.05 |
| 10 | Borneras enchufables | 5.08 mm (6 p) + 3.5 mm (12 p) | — | 3.40 | 2.80 | 2.30 |
| 11 | Gabinete DIN 6M | Italtronic 6M + frente serigrafiado | 1 | 4.80 | 3.90 | 3.10 |
| 12 | Antena + pigtail | Dipolo 2 dBi + IPEX-SMA | 1 | 1.70 | 1.35 | 1.10 |
| 13 | LED RGB, buzzer, pulsador | — | — | 1.20 | 0.95 | 0.75 |
| 14 | PCB 4 capas | 100 × 80 mm, ENIG | 1 | 4.20 | 2.60 | 1.70 |
| 15 | Pasivos, conectores, misceláneos | — | — | 3.50 | 2.90 | 2.40 |
| 16 | **Sonda armada** (PCB, cápsula, filtro PTFE, 3 m FEP, soporte) | — | 1 | 9.50 | 7.60 | 6.20 |
| 17 | CT clamp 10 A | SCT-013-010 | 1 | 3.20 | 2.60 | 2.10 |
| 18 | Embalaje + guía impresa | — | 1 | 2.40 | 1.80 | 1.40 |
| | **Subtotal materiales** | | | **66.65** | **54.25** | **45.33** |
| | Ensamblado + test funcional (EMS) | | | 12.00 | 8.00 | 5.50 |
| | Merma y scrap (3 %) | | | 2.00 | 1.63 | 1.36 |
| | **Costo total por equipo** | | | **≈ 80.65** | **≈ 63.88** | **≈ 52.19** |

**Módulo XJ485 (interfaz oficial al XR06):** USD 35–55 por punto, **no incluido**. Es el
mayor componente de costo unitario y la razón del objetivo de interfaz TTL propia de la v2
(docs/03 §1 opción B): eliminarlo baja el costo instalado ~40 %.

**Variante FC-200 IP65:** +USD 9–13 (caja, prensacables, junta, antena puck).
**Variante FC-300 LTE:** +USD 28–38 (SIM7670G + SIM + antena) + plan de datos
(~USD 1.5–3/mes/equipo con un plan IoT de bajo volumen).

## 2. Costo de la nube

| Concepto | @100 equipos | @1000 equipos |
|---|---|---|
| Broker gestionado + ingesta | 60–90 /mes | 220–350 /mes |
| Base de series temporales + Postgres | 50–80 /mes | 180–280 /mes |
| Push, WhatsApp (≈ 0.05/msj), email | 15–40 /mes | 90–250 /mes |
| Monitoreo, backups, CDN | 25 /mes | 70 /mes |
| **Total** | **150–235 /mes → ≈ 1.5–2.4 /equipo** | **560–950 /mes → ≈ 0.6–1.0 /equipo** |

Costo marginal real por equipo: **USD 0.6–2.4/mes**. Deja mucho margen en un abono.

## 3. Modelo de precio sugerido

| | Equipo (una vez) | Abono mensual por equipo |
|---|---|---|
| FC-100 Wi-Fi | USD 210–260 | USD 7–9 |
| FC-200 IP65 | USD 240–290 | USD 7–9 |
| FC-300 LTE | USD 290–350 | USD 12–15 (incluye datos) |
| XJ485 (si hace falta) | + USD 60–80 | — |
| Instalación y puesta en marcha | USD 80–140 | — |

Margen bruto de hardware ≈ 60–65 % a 500 u.; margen del abono > 85 %.

**Argumento de venta, para que el precio no se discuta contra un datalogger de USD 60:**
una cámara de congelados de 20 m³ llena vale típicamente USD 15 000–40 000 en mercadería.
Una sola pérdida evitada paga 50–150 equipos. El competidor del producto no es el
datalogger: es la factura del siniestro, el rechazo de una auditoría y la rotura de un
compresor que avisó durante tres semanas con ciclado corto.

**Incluir en el abono** (lo que sostiene la renovación): alertas ilimitadas con escalado,
reportes HACCP automáticos, 2 años de histórico, OTA, soporte y reposición del equipo ante
falla. El abono se cae si el cliente no ve valor todos los meses: el reporte semanal
automático por email es el recordatorio barato de que el servicio existe.
