# POV Wheel Display — Resumen del proyecto

Documento de trabajo: resumen de la conversación de diseño, decisiones tomadas,
componentes seleccionados y pendientes.

> 📖 Para el montaje paso a paso, ver **`README.md`**.

---

## 1. Objetivo

Construir un **display POV (Persistencia de Visión)** en la **rueda delantera** de
una bicicleta, capaz de mostrar imágenes, texto y animaciones aprovechando el giro
de la rueda, controlado desde el teléfono.

---

## 2. Especificaciones de la bicicleta

| Dato | Valor |
|---|---|
| Tipo | **Fixie** |
| Rodado | **700c** (aro 622 mm) |
| Neumático | ~25 mm → diámetro total ≈ **672 mm**, **radio ≈ 336 mm** |
| Rueda objetivo | **Delantera**, de **radios** |
| Frenado | **Solo freno de zapata** sobre el aro metálico (sin disco ni cáliper) |
| Implicancia | El **buje/centro está libre** → buen lugar para electrónica y batería |
| Implicancia | Las zapatas muerden el aro a **radio ≈ 290–310 mm** → las tiras dejan margen |
| Material del aro | **Por confirmar** (test del imán) — se asume **aluminio** |

---

## 3. Proyecto de referencia

- **GitHub:** https://github.com/aleksei-severin/POV_Wheel_Display
- **Qué es:** display POV en rueda de MTB 29" con **528 LEDs** (6 brazos × 88 LEDs, **SK9822-A**, doble cara).
- **Hardware:** **ESP32-S3**, 16 MB flash, 8 MB PSRAM; **6 sensores Hall** + **1 imán en la horquilla**; **1S6P Li-Po 6000 mAh**; app Android por **BLE**.
- **Resolución:** 360 × 44 píxeles polares; paleta 256 colores.
- **Refresco:** el refresco efectivo = **nº de brazos × (rev/s)**.
- **Arquitectura:** **todo gira con la rueda**; el móvil hace la transformada polar.
- **Licencia:** **PolyForm Noncommercial 1.0.0** (uso personal gratis; comercial requiere licencia).

---

## 4. Física / fórmulas clave

Sea $R$ = radio al LED, $v$ = velocidad, $S$ = nº de brazos, $L$ = LEDs por tira, $N$ = LEDs totales.

- Velocidad de giro: $f_{rot} = \dfrac{v}{2\pi R}$  [rev/s]
- **Refresco (anti-parpadeo):** $f_{refresco} = S \cdot f_{rot}$
- **Columnas angulares por vuelta:** $C = \dfrac{f_{frame}}{f_{rot}}$
- **Resolución total ≈** $L \times C$ (radial × angular)
- Tiempo de un frame (bus): $t_{frame} \approx \dfrac{32 \cdot N}{f_{SCK}}$
- Inercia de una masa a radio $r$: $I = m\,r^2$

**Conclusión:** a baja velocidad el límite es el **refresco** ($S \cdot f_{rot}$),
**no** la resolución. Más **brazos** → baja la velocidad mínima; más **LEDs por tira**
→ más detalle radial, pero **no** baja la velocidad.

700c (velocidad → RPM):

| Velocidad | RPM |
|---:|---:|
| 10 km/h | 79 |
| 15 km/h | 118 |
| 20 km/h | 158 |
| 25 km/h | 197 |
| 30 km/h | 237 |

A 144 LED/m: tiras de **22 cm → ≈ 32 LEDs** por tira.

---

## 5. Decisiones técnicas alcanzadas

| Tema | Decisión |
|---|---|
| **Montaje** | **Todo gira con la rueda** (ESP32 + LEDs + batería). **Nada de slip ring.** Solo el **imán** va fijo en la horquilla. |
| **Sincronización** | **Sensores Hall** (sin contacto) detectan el imán → fase, RPM y sentido. |
| **Nº de tiras** | **7 brazos × 2 caras = 14 tiras** de **22 cm**. |
| **Total de LEDs** | **≈ 448** (32 por tira × 14). |
| **Resolución** | **≈ 360 × 32** píxeles polares (angular × radial). |
| **LED** | **HD107S** (compatible **APA102/SK9822**), **144 LED/m**, **IP30**, **PCB negro**, 5 V, reloj+datos. |
| **Microcontrolador** | **ESP32-S3-MINI N16R8** (16 MB flash + 8 MB PSRAM), placa con USB; admite **antena externa IPEX**. |
| **Nivel lógico** | **Buffer 74AHCT244N** (DIP-20) 3,3 V → 5 V en reloj y datos. **No sirve HC244.** |
| **Corte por baja velocidad** | **1 MOSFET P de lado alto** corta el rail de 5 V de los LEDs cuando la rueda va lenta/parada. |
| **Procesamiento** | La **transformada polar y el color se calculan en el teléfono**. El ESP32 solo **reproduce frames**. |
| **Control remoto** | **BLE** para el día a día; **WiFi AP + web** para subir contenido (bici parada). |
| **Almacenamiento** | **Flash interna (16 MB)**; microSD **descartada** por ahora. |
| **Batería** | **Pack 2S2P** (4 × 18650 Samsung 30Q) = **7,4 V · 6000 mAh · 44 Wh**. |
| **Regulación** | **Sin boost.** Dos **bucks desde 7,4 V**: principal **→ 5 V (LEDs)** y auxiliar **→ 5 V (ESP32, línea limpia)**. |
| **BMS** | **2S con balanceo, puerto común, 10 A**. |
| **Carga** | **Cargador 2S USB-C** (sube 5 V→8,4 V) **+ BMS 2S** de protección. |
| **Protección polaridad** | Conector **XT30** (polarizado); protección extra **opcional** (repuesto SUP75P03-07). |
| **Protección ambiental** | **Barniz conformal de silicona** propio sobre tiras (IP30) y electrónica. |
| **Sensor Hall (montaje)** | **7 sensores** (uno por brazo) a **27 cm** de radio. Aro de **aluminio** → sin apantallado. |
| **Formato de frame** | Paleta de 256 colores (768 B) + índices (1 byte/píxel) ≈ 360 × L. |
| **Topología de datos** | **Paralelo, 8 pines** (CLK + 7 DATA; las 2 caras de cada brazo en serie por línea). |
| **Mecánica / encapsulado** | **Fuera de alcance por ahora**. |

---

## 6. Componentes — código, artículo, tipo de conexión y enlace

**Estado:** 🟢 elegido/en carrito · 🟡 verificar variante · 🔴 descartar · ⬜ por elegir

> Carrito de AliExpress **revisado el 2026-10-16**. Todos los 🟢 están en el carrito **excepto C1** (falta). Ver **§6.1**.

| Código | Nombre | Artículo | Tipo de conexión | Estado | Enlace |
|---|---|---|---|---|---|
| A1 | Tira LED | HD107S, 144 LED/m, IP30, PCB negro | Tira (pads a soldar) | 🟢 | https://es.aliexpress.com/item/1005009281726424.html |
| A2 | Condensador electrolítico | 1000 µF / 16 V | Through-hole (radial) | 🟢 | https://es.aliexpress.com/item/1005003012636265.html |
| A3 | Kit condensadores cerámicos | 10 pF–10 µF ×480 | Through-hole (P = 5,08 mm) | 🟢 | https://es.aliexpress.com/item/1005005952696538.html |
| A4 | Kit conectores JST-XH | incl. 4 pines ×10 | Through-hole (a soldar) | 🟢 | https://es.aliexpress.com/item/1005010363547574.html |
| A5 | Kit resistencias | 1/4 W, 10 Ω–1 MΩ, 30 valores, 1 % | Through-hole | 🟢 | https://es.aliexpress.com/item/1005011772534173.html |
| B1 | Microcontrolador | ESP32-S3-MINI N16R8 (placa) | Módulo (pines + USB-C) | 🟢 | https://es.aliexpress.com/item/1005013303536525.html |
| B2 | Buffer de nivel | **SN74AHCT244N** (DIP-20) | Through-hole DIP-20 | 🟢 | https://es.aliexpress.com/item/1005012866652310.html |
| B3 | *pull-ups Hall* | **cubierto por A5** (10 kΩ) | — | ➡️ A5 | no comprar (era un kit de potenciómetros) |
| B4 | MOSFET de corte | **SUP75P03-07** (P-channel, TO-220) | Through-hole TO-220 | 🟢 | https://es.aliexpress.com/item/1005013342964977.html |
| B5 | Driver MOSFET | Kit transistores (2N2222) | Through-hole (TO-92) | 🟢 | https://es.aliexpress.com/item/1005006043959455.html |
| B6 | Resistencias MOSFET | cubierto por A5 | — | ⛔ | (no comprar) |
| B7 | Pulsador | Tact switch 6×6 mm | Through-hole | 🟢 | https://es.aliexpress.com/item/1005004863393624.html |
| C1 | Sensor Hall | **A3144 TO-92** | Through-hole | ⬜ | https://es.aliexpress.com/item/1005008779557248.html |
| C2 | Imán | Neodimio 10×2 mm ×50 | — | 🟢 | https://es.aliexpress.com/item/1005012390865565.html |
| D1 | Batería | Samsung INR18650-30Q ×4 | Celda | 🟢* | https://www.mercadolibre.cl/baterias-18650-samsung-30q-3000-mah-originales-2-unidades/up/MLCU29698268 |
| D2 | Cargador 2S USB-C | Sube 5 V→8,4 V, 15 W (~1,8 A), con balanceo. **Sin protección descarga** | Módulo | 🟢 | https://es.aliexpress.com/item/1005012404336341.html |
| D3 | Buck principal | **Mini560 PRO, fijo 5 V, 5 A** (entrada 6–30 V, sin potenciómetro) | Módulo | 🟢 | https://es.aliexpress.com/item/1005012462207466.html |
| D4 | Buck auxiliar | MP1584EN — **versión 5 V** | Módulo | 🟢 | https://es.aliexpress.com/item/1005007136976280.html |
| D5 | BMS 2S de protección | 2S 7,4 V, puerto común, balanceo pasivo, corte por sobredescarga — variante **10 A, sin nickel** | Módulo | 🟢 | https://es.aliexpress.com/item/1005006349102886.html |
| D6 | Portapilas 18650 | 1 ranura ×10 (usar 4) | Through-hole (cables) | 🟢 | https://es.aliexpress.com/item/1005012030990064.html |
| D7 | Fusible rearmable | PPTC **JK30, 8 A** (variante "8A 800"), 30 V | Through-hole | 🟢 | https://es.aliexpress.com/item/1005013229953103.html |
| D8 | Protección polaridad | **Opcional**: mejor un **P-MOSFET SUP75P03-07 de repuesto** (pack B4) que el módulo de 4 A | Through-hole | 🟡 | (usar repuesto de B4) |
| D9 | Conector batería | **XT30** macho/hembra (con cable 16 AWG) | Crimp/soldadura | 🟢 | https://es.aliexpress.com/item/1005009052283719.html |
| E1 | Cable potencia | **Kit silicona 6 colores** → elegir **18 AWG**, 16.4 ft | Estañar | 🟢 | https://es.aliexpress.com/item/1005011692969059.html |
| E2 | Cable reparto de brazos | mismo kit → **22 AWG**, 5 ft | — | 🟢 | https://es.aliexpress.com/item/1005011692969059.html |
| E3 | Cable datos LED | mismo kit → **28 AWG**, 5 ft | — | 🟢 | https://es.aliexpress.com/item/1005011692969059.html |
| E4 | Cable Hall | mismo kit → **28 AWG** (igual que E3) | — | 🟢 | https://es.aliexpress.com/item/1005011692969059.html |
| E5 | Cable buck aux → ESP32 | mismo kit → **26 AWG**, 5 ft | — | 🟢 | https://es.aliexpress.com/item/1005011692969059.html |
| E6 | Interconexión celdas 2S | mismo kit → **18 AWG** (igual que E1) | — | 🟢 | https://es.aliexpress.com/item/1005011692969059.html |
| Z1 | Perfboard | Placas perforadas (2×8…10×15 cm) | Through-hole (a soldar) | 🟢 | https://es.aliexpress.com/item/1005012158303665.html |
| Z2 | Protoboard | Sin soldadura (400 puntos) | — | 🟢 | https://es.aliexpress.com/item/1005012587776046.html |

\* D1 se compra en **MercadoLibre** (local), no en AliExpress.

**Consumibles (barniz conformal, estaño + fundente, termorretráctil, epóxico):** se consiguen **localmente**.

### 6.1 Checklist del carrito de AliExpress (revisado 2026-10-16)

- ➕ **FALTA: C1** — Sensor Hall **A3144 TO-92** → https://es.aliexpress.com/item/1005008779557248.html
- ➖ **SOBRA: IRF9310 SOP-8** (`1005012273844982`) → quitar (sustituido por **B4 SUP75P03-07**)

**Verificar la variante al comprar:**

| Ítem | Variante correcta |
|---|---|
| A1 Tira LED | **144 LED/m · IP30 · PCB negro** (⚠️ revisar el **precio**) |
| B1 ESP32 | **con antena** |
| C2 Imán | **10 mm** |
| D2 Cargador 2S | **2S** |
| D3 Buck | **Mini560 PRO · 5 V** |
| D4 Buck | **MP1584EN · 5 V** |
| D5 BMS | **10 A · sin nickel** |
| D7 Fusible | **8 A** ("8A 800") |
| B4 MOSFET | **P-channel (SUP75P03-07)** |
| Cables E | cubrir **18 / 22 / 26 / 28 AWG** |

**Descartados:**
- LiPo 3,7 V 400 mAh de dron → https://es.aliexpress.com/item/1005010597258476.html
- LiPo 102050 1000 mAh con PCB → https://es.aliexpress.com/item/1005012952293433.html
- **Booster 5 V** (sustituido por 2 bucks con el pack 2S)

**Nota:** A5, B2 y B4 son **through-hole** ✅ → todo montable en la perfboard Z1.

---

## 7. Pendientes

1. ➕ **Añadir C1** (Sensor Hall A3144) al carrito.
2. ➖ **Quitar IRF9310 SOP-8** del carrito.
3. 🔍 **Verificar variantes** (ver §6.1).
4. 📏 Confirmar **material del aro** (test del imán) — se asume aluminio.
5. 🧰 **Local:** baterías 30Q, barniz, estaño, termorretráctil, epóxico.
6. 📦 Definir el **encapsulado** (fuera de alcance por ahora).

---

## 8. Cables necesarios

Todos de **silicona**, **multifilar** (nunca rígido): aguanta vibración y fatiga.

| Uso | Corriente | Calibre | Nota |
|---|---|---|---|
| Pack → BMS/fusible → buck principal | ~3,5–5 A | **18–20 AWG** | Potencia principal (E1, E6) |
| Buck 5 V → bus de LEDs | ~4,5 A | **18–20 AWG** | Reparto grueso |
| Reparto 5 V por brazo | ~0,65 A | **22–24 AWG** | Hacia cada tira (E2) |
| Retorno GND (bus) | suma de corrientes | **18–20 AWG** | Tierra en estrella |
| Datos LED (CLK + 7 DATA) | mA | **28–30 AWG** | Par trenzado con GND (E3) |
| VCC/GND de los Hall | ~35 mA total | **28–30 AWG** | Bus de alimentación (E4) |
| Señal Hall (×7) | ~5 mA | **28–30 AWG** | Una por sensor (E4) |
| Buck auxiliar → ESP32 | ~0,5–1 A | **24–26 AWG** | Línea limpia (E5) |

---

## 9. Notas y advertencias

- **No alimentar los LEDs a través de la placa ESP32.** Los LEDs van desde su **buck de 5 V**; la placa solo manda **señales** y comparte **GND**.
- **Tierra en estrella**: que el retorno de corriente de los LEDs no comparta tramo con el del ESP32.
- **GND común** entre batería, bucks, ESP32 y tiras.
- **Calibrar los bucks a 5,0 V** (con multímetro) **antes** de conectar nada.
- **Balanceo**: repartir el peso de batería/placa de forma **simétrica**. La inercia crece con $r^2$.
- **Tornillería NO ferrosa** cerca del sensor Hall.
- **Carga (2S) — 2 piezas:** **cargador 2S USB-C** (D2, sube 5 V→8,4 V con balanceo) **+ BMS 2S de protección** (D5). Se carga con un cable USB-C.
  - ⚠️ D2 **solo carga** (no protege en descarga) → el **BMS (D5) es imprescindible**.
- **B3 (potenciómetros):** ❌ no se usa. Se necesitan **resistencias fijas de 10 kΩ** (pull-up de cada Hall) → ya vienen en **A5**.
- **Soldadura**: cautín con temperatura ajustable + estaño con fundente + **multímetro**. En tiras IP30 se suelda fácil; aplicar barniz al final.
- **Envío de baterías a Chile**: revisar restricciones; las 18650 se consiguen localmente.

---

## 10. Arquitectura de alimentación

```
Pack 2S2P (7,4 V, 4×18650) ──┬─► BMS 2S ──► Fusible ──► [Buck 7,4→5 V, 5 A] ──► LEDs (5 V)
                             │                                   └─ caps 1000 µF
                             │                                   └─ MOSFET de corte (bajo
                             │                                      umbral de velocidad)
                             └─► [Buck 7,4→5 V, 1 A] ──► pin 5V ESP32 (línea limpia) ──► 3,3 V → Hall
                                          [Carga EXTERNA: USB-C → 8,4 V vía D2 + BMS]
```

---

## 11. Estado general

- **Fase:** **compra** (carrito de AliExpress prácticamente cerrado).
- **Cerrado:** MCU, LED, nº de tiras (14 × 22 cm), total de LEDs (≈448), resolución (≈360×32),
  topología de datos (8 pines), 7 sensores Hall a 27 cm, batería **2S2P** + **2 bucks**, BMS 2S,
  carga USB-C a 8,4 V; todos los semiconductores **through-hole** (montaje en perfboard).
- **Pendiente:** ➕ añadir **C1 (Hall)**, ➖ quitar **IRF9310**, verificar **variantes** (§6.1),
  confirmar **material del aro**, **encapsulado**.
- **Siguiente:** **montaje** → ver **`README.md`**.
