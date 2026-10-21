# 02 — Diseño de hardware

## 1. Diagrama de bloques

```
 100–240 VAC ──[F 1A T]──[MOV 275V]──[X-cap]──┬─▶ AC/DC 5 V 2 W (Mornsun LS03-13B05R3)
       ó 12–24 VDC ──────[TVS]────────────────┘            │
                                                           ├─▶ Buck/LDO 3V3 (TPS62743) ─▶ ESP32-S3
                                                           ├─▶ 5 V ─▶ bobinas relé, wetting DI 12 V (boost) 
                                                           └─▶ supercap 1 F ─▶ brown-out + RTC + flush de log
 ┌───────────────────────────── ESP32-S3-WROOM-1U-N8R2 ─────────────────────────────┐
 │ UART1 ─▶ RS-485 AISLADO (ADM2587E) ──▶ bornera A/B/GNDiso ──▶ XJ485 ──▶ XR06 TTL │
 │ I²C   ─▶ SHT45 (sonda remota, cable 3 m) + DS3231 (RTC) + EEPROM 24AA02E48 (MAC) │
 │ GPIO  ─▶ 4× DI opto-aisladas (NA/NC por software)                                │
 │ GPIO  ─▶ 2× relé SPDT 16 A (driver ULN2003 + snubber)                            │
 │ ADC   ─▶ 1× entrada de transformador de corriente (compresor)                    │
 │ GPIO  ─▶ LED RGB de estado · buzzer piezo 85 dB · pulsador (config/reset)        │
 │ USB-C ─▶ servicio: logs, flasheo de fábrica, diagnóstico en obra                 │
 └──────────────────────────────────────────────────────────────────────────────────┘
```

## 2. Selección de componentes principales

| Bloque | Elección | Alternativas | Justificación |
|---|---|---|---|
| MCU + Wi-Fi | **ESP32-S3-WROOM-1U-N8R2** (8 MB flash, 2 MB PSRAM, antena externa) | ESP32-WROOM-32E (más barato, sin USB nativo); ESP32-C6 (Wi-Fi 6 + Thread) | Módulo pre-certificado (FCC/CE y base para homologación ENACOM), 2 núcleos para separar Modbus de red, USB nativo para producción. Variante **U** = conector antena externa: imprescindible en tablero metálico |
| Transceptor RS-485 | **ADM2587E** (aislado + DC/DC integrado) | ISO1410 + DC/DC aparte (más barato, más BOM); MAX13487E (no aislado) | Aislación de 2.5 kV en un solo chip, sin fuente aislada externa. Elimina los lazos de masa del tablero |
| Sensor T/HR | **SHT45-AD1B** (±1.8 %HR, ±0.2 °C) | SHT41 (más barato, ±1.8 %HR); HDC3022; SHT85 (encapsulado industrial, caro) | Precisión apta para auditoría, I²C, excelente estabilidad a largo plazo a HR alta y condensación (tiene calentador integrado para recuperar la membrana) |
| Sonda de producto (opc.) | **DS18B20 en vaina inox** o PT1000 + MAX31865 | Termopar tipo T | Medir temperatura de producto, no de aire, es un diferencial fuerte para HACCP |
| Entradas digitales | Opto **TLP291-4** (4 canales), wetting 12 V, 2.2 kΩ + RC | PC817 discretos | Contactos de campo son secos y sucios: 12 V y ~5 mA dan margen de óxido. NA/NC configurable por software, no por jumper |
| Salidas | 2× relé **SPDT 16 A 250 VAC** (Hongfa HF115F-I) + snubber RC 100 Ω/100 nF + MOV | 2× SSR (sin partes móviles, más caro, fuga); relé 10 A | 16 A permite comandar luz de cámara o resistencia de marco de puerta directamente. Snubber contra arco inductivo |
| Medición de corriente | **CT 10 A/1 V tipo clamp** (SCT-013-010) en entrada ADC con buffer | ACS724 (requiere pasar el cable por la PCB — inviable en retrofit) | El clamp se instala sin cortar cables: clave para instalación en equipo en servicio |
| Fuente | **Mornsun LS03-13B05R3** (3 W, 100–240 VAC, aislada) | Hi-Link HLK-5M05 (más barato, menor calidad); entrada 12/24 VDC para FC-100 | Certificada, encapsulada, apta montaje en PCB con clearance; versión DC-in para tableros que ya tienen 24 V |
| RTC | **DS3231SN** + celda | Sin RTC (solo SNTP) | Los registros offline necesitan estampa de tiempo confiable; el DS3231 es compensado en temperatura (crítico en un producto que vive a temperaturas raras) |

## 3. Reglas de layout (las que realmente evitan devoluciones)

1. **Tres zonas separadas en la PCB:** (a) red/relés, (b) lógica 3V3, (c) lado aislado
   RS-485. Barrera física sin cobre debajo, con ranuras (slots) en zonas de aislación.
2. **Creepage/clearance ≥ 3.0 mm** entre red y lógica para 250 VAC grado de polución 2
   (IEC 60664-1); ≥ 5.5 mm bajo los relés y la fuente; **slot** en la barrera del ADM2587E.
3. **Antena:** módulo variante U con pigtail IPEX→SMA al panel. **Nunca** antena de PCB
   dentro de un gabinete metálico o detrás de una pared de cámara: es la causa típica de
   "el equipo anda en el banco y no en obra".
4. **Plano de masa continuo** bajo la lógica; masa aislada (GNDiso) solo en su isla.
5. **TVS en todas las borneras de campo:** SM712 en A/B del RS-485, PESD en las DI.
   Fusible resetable (PTC 100 mA) en la línea de wetting.
6. **Térmica:** la fuente AC/DC y los relés lejos del SHT45 (el SHT45 va en sonda remota,
   no en la PCB, justamente por esto) y lejos del módulo Wi-Fi.
7. **Puntos de prueba** y testpads para cama de agujas: 3V3, 5V, UART, A/B, bobinas.
8. Terminación RS-485 de 120 Ω **conmutable por jumper** (el gateway casi siempre es el
   extremo del bus, pero no siempre).

## 4. Asignación de E/S por defecto (configurable desde la app)

| Canal | Función por defecto | Tipo | Notas |
|---|---|---|---|
| DI1 | Fin de carrera de puerta | NA (cierra al abrir) | Antirrebote 2 s; alarma a los 90 s |
| DI2 | Presostato de alta / térmico de compresor | NC | Alarma inmediata, crítica |
| DI3 | Falla de fase / monitor de tensión | NC | Alarma inmediata |
| DI4 | Libre (contacto de alarma del XR06, bomba de condensado, etc.) | Config. | Permite capturar la alarma del XR06 aun sin Modbus |
| RELÉ 1 | ON/OFF remoto → entrada digital del XR06 | NA, fail-safe en reposo = cámara encendida | Nunca en serie con el compresor |
| RELÉ 2 | Sirena / baliza local o luz de cámara | NA | Dispara también con la red caída |
| CT1 | Corriente del compresor | 0–10 A RMS | Confirma marcha real, ciclado corto, duty cycle |

**Fail-safe de los relés:** el cableado se define de modo que **relé desenergizado = cámara
refrigerando**. Un gateway sin alimentación nunca apaga una cámara.

## 5. Especificación eléctrica

| Parámetro | Valor |
|---|---|
| Alimentación | 100–264 VAC 50/60 Hz (ó 12–24 VDC, variante) |
| Consumo | < 1.5 W reposo, < 3 W con 2 relés y Wi-Fi TX |
| Temperatura de operación | −20 a +60 °C (electrónica); sonda −40 a +85 °C |
| Humedad | 0–95 %HR sin condensación en electrónica; sonda con filtro PTFE sinterizado apta condensación |
| Aislación campo ↔ lógica | 2.5 kV RMS (RS-485), 4 kV (relés/red) |
| Retención de datos sin red | ≥ 30 días a 1 muestra/min (≈ 2.6 M registros, ring buffer en flash) |
| Respaldo de reloj | ≥ 5 años (celda CR2032) |
