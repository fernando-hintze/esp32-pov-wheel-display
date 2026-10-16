# POV Wheel Display — Guía de armado

Display **POV (Persistencia de Visión)** en la rueda delantera de una bici fixie 700c.

> 📋 **Componentes, decisiones y enlaces:** ver **`chat.md`**.
> Esta guía asume que ya tienes todo el material comprado.

---

## 0. Resumen del sistema

- **14 tiras LED** (7 brazos × 2 caras) de **22 cm** con **HD107S** (APA102/SK9822), 144 LED/m.
- **ESP32-S3** que solo **reproduce frames**; el móvil hace la transformada polar.
- **7 sensores Hall** (uno por brazo) + **1 imán** en la horquilla → fase, RPM y sentido.
- **Todo gira con la rueda**; solo el imán va fijo en la horquilla. **Sin slip ring.**
- **Batería 2S2P** (4×18650) con **BMS 2S**, cargador **USB-C** y **2 bucks** (5 V LEDs + 5 V ESP32).

### Diagrama de bloques

```
                       ┌─────────────────────────────────────────────┐
                       │            RUEDA (gira)                     │
                       │                                             │
  Móvil ──BLE/WiFi──►  │  ESP32-S3 ──► 74AHCT244 ──► 14 tiras HD107S  │
  (contenido)          │     ▲  3,3V                    (7×CLK+DATA)  │
                       │     │                                       │
                       │  7× Hall A3144 ◄── imán (horquilla, fijo)   │
                       │     ▲                                       │
                       │  [Buck 5V/1A]   [Buck 5V/5A] ──► LEDs       │
                       │        ▲              ▲                     │
                       │        └──── Pack 2S2P ┴─ BMS 2S            │
                       │                              ▲              │
                       └──────────────────────────────┼──────────────┘
                                     USB-C (cargador 2S) ──┘
```

---

## 1. Herramientas necesarias

- **Cautín** con temperatura ajustable (o el **soldador USB** del carrito) + **estaño con fundente** (0,6–0,8 mm).
- **Multímetro** ⭐ (imprescindible para calibrar los bucks).
- Cortacables / pelacables, pinzas, **tercera mano**.
- Alcohol isopropílico, termorretráctil, bridas.
- Cinta aislante / kapton.
- Destornilladores; tornillería **NO ferrosa** (nailon/latón).
- (Opcional) Dremel/taladro para soportes.

---

## 2. Orden de trabajo (importante)

**No montes todo de una.** Sigue este orden y **prueba cada bloque por separado**:

1. ESP32 solo (banco)
2. Nivel lógico + 1 tira LED (banco)
3. Sistema de alimentación (batería, BMS, bucks)
4. Sensores Hall
5. Rotor completo (14 tiras)
6. Ensamblaje en la rueda + balanceo + barnizado
7. Firmware + app

---

## 3. ESP32 (prueba en banco)

1. Instala **Arduino IDE** o **PlatformIO** con soporte **ESP32** (core ≥ 3.x).
2. Conecta por USB-C y **verifica la PSRAM**: imprime el tamaño (`ESP.getPsramSize()` / `esp_psram_get_size()`).
   - **Debe reportar 8 MB**. Si sale 0 o 2 MB → la placa está mal etiquetada.
3. Verifica **WiFi y BLE** con un sketch mínimo.

> **Variante con antena:** si tu placa tiene conector IPEX, conecta la antena antes de probar radio.

---

## 4. Nivel lógico: 74AHCT244 (DIP-20)

Convierte las señales **3,3 V del ESP32 → 5 V** para las tiras.

| Pin 74AHCT244 | Conexión |
|---|---|
| VCC (pin 20) | **+5 V** (del buck principal) |
| GND (pin 10) | GND común |
| 1OE, 2OE (pines 1 y 19) | **GND** (habilitar, activo por nivel bajo) |
| Entradas (A) | Señales del ESP32 (3,3 V): CLK + 7× DATA |
| Salidas (Y) | CLK + 7× DATA hacia las tiras (5 V) |

- **Capacitor de desacoplo** 0,1 µF entre VCC y GND, lo más cerca posible del chip.
- Pon la **resistencia serie 33–100 Ω** en cada línea DATA, junto al primer LED.

**Prueba:** con **una tira** y un sketch que la ponga en rojo/verde/azul.

---

## 5. Sistema de alimentación

### 5.1 Pack 2S2P (4×18650)
1. Coloca las **4 celdas** (mismo modelo y misma carga) en los portapilas.
2. Cablea **2 en serie** (para 7,4 V) y esas **2 ramas en paralelo** (capacidad ×2).

### 5.2 BMS 2S (puerto común)
| Terminal BMS | Va a |
|---|---|
| B+ | Positivo del pack (7,4 V) |
| B− | Negativo del pack |
| BM / B1 | **Punto medio** entre las dos celdas en serie |
| P+ | **Positivo de salida** → fusible → bucks |
| P− | **Negativo de salida** (GND del sistema) |

### 5.3 Bucks — **CALIBRAR ANTES DE CONECTAR NADA**
- **Buck principal (Mini560 PRO):** salida **fija 5 V** (no ajustable) → listo.
- **Buck auxiliar (MP1584EN):** es **ajustable**.
  - Aliméntalo **solo** (sin carga), mide la salida y **gíralo hasta los 5,00 V exactos**.
  - ⚠️ Si está por encima de 5 V **puedes freír los LEDs y el ESP32**.

### 5.4 Protección y conector
- **Fusible PPTC 8 A** entre el BMS (P+) y los bucks.
- **(Opcional) protección de polaridad**: usa un **SUP75P03-07** de repuesto (o el módulo anti-inversa D8).
- **Conector XT30** entre el pack y la electrónica (polarizado).

### 5.5 Primeras medidas
Con el pack conectado, comprueba con el multímetro:
- Salida del buck principal = **5,0 V**
- Salida del buck auxiliar = **5,0 V**
- **GND común** entre batería, bucks, ESP32 y tiras.

---

## 6. Sensores Hall (×7)

- **Montaje:** a **27 cm** de radio (máximo sin molestar al freno de zapata).
- **Alimentación en PARALELO** (bus VCC = 5 V / GND), **nunca en serie**.
- **Pull-up de 10 kΩ a 3,3 V** en la salida de cada sensor (A3144 = colector abierto).
- **Desacoplo 0,1 µF** entre VCC y GND junto a **cada** sensor.
- Aro de **aluminio** → sin problemas. Si fuera de acero, soporte **no ferromagnético**.

**Prueba:** sketch que encienda un LED interno al pasar el imán. Ajusta la separación (2–5 mm).

---

## 7. Rotor: las 14 tiras

1. **Corta 14 tiras de 22 cm** (≈ 32 LEDs cada una). Verifica continuidad de cada una.
2. **Prueba cada tira** antes de montarla.
3. Suelda **cables de datos y alimentación** en cada extremo. Pon **alivio de tensión** (termorretráctil + epóxico).
4. Fija las tiras a los **7 brazos** (2 por brazo, una por cara).
5. **Cableado central**: concentra las conexiones junto al eje (no dejes cables en la periferia).
6. Conexión de datos: **8 pines** = 1 reloj compartido + 7 datos (una línea por brazo; las 2 caras del brazo en serie).

**Prueba antes de cerrar:** enciende las 14 tiras y comprueba que todas responden.

---

## 8. Ensamblaje en la rueda

1. **Reparte el peso simétricamente**: batería, BMS, bucks y placa repartidos alrededor del eje (la inercia va con $r^2$).
2. Fija todo con **bridas** + soportes; nada que pueda soltarse a velocidad.
3. **Tornillería NO ferrosa** (nailon/latón) cerca de los sensores Hall.
4. **Barniz conformal de silicona** sobre placas y soldaduras.
   - **Deja libres**: conectores, pulsador y la **cara del sensor Hall**.
5. **Balanceo estático**: con la rueda en la horquilla, debe quedar sin "punto pesado". Añade contrapeso si hace falta.
6. Deja el **USB-C** del cargador accesible.

---

## 9. Firmware y app

1. Carga el **firmware** del ESP32.
2. Empareja desde el móvil:
   - **BLE** para control del día a día.
   - **WiFi AP + web** para subir contenido (con la bici parada).
3. Sube una imagen y verifica la orientación/velocidad.

---

## 10. Primer encendido y calibración

- **Umbral de velocidad:** define a partir de qué RPM se enciende el display.
- **Calibración de Hall:** corrige el offset angular de cada sensor (para que la imagen no "tiemble").
- **Brillo:** limita la corriente media (≈ 8–10 mA/LED) para no exceder el buck de 5 A.
- Verifica que la **imagen se mantiene estable** al acelerar y frenar.

---

## 11. Pinout sugerido (ajústalo a tu placa)

> ⚠️ Evita: **GPIO 33–37** (PSRAM octal), **26–32** (flash), strapping (**0, 3, 45, 46**) y USB (**19, 20**).

| Función | GPIO sugerido |
|---|---|
| CLK (SPI) | 18 |
| DATA brazo 1–7 | 4, 5, 6, 7, 15, 16, 17 |
| Hall 1–7 | 1, 2, 8, 9, 10, 11, 12 |
| Control MOSFET (corte 5 V) | 21 |
| Pulsador | 14 |

*(Ajusta según los pines que exponga tu placa.)*

---

## 12. Seguridad ⚠️

- **Li-ion:** respeta la polaridad, no cortocircuites, usa el BMS siempre. Carga en superficie no inflamable.
- **Calibra los bucks a 5,0 V ANTES** de conectar LEDs o ESP32.
- **Soldadura:** ventila, no toques la punta, usa gafas.
- **Rueda delantera:** un fallo mecánico puede hacerte caer. Sujeta **todo** con redundancia.
- **Brillo de noche:** limita el máximo para no deslumbrar.

---

## 13. Checklist final

- [ ] Bucks calibrados a **5,0 V**
- [ ] GND común en todo
- [ ] 74AHCT244 alimentado a 5 V y OE a GND
- [ ] Resistencias serie en las líneas DATA
- [ ] 7 sensores Hall con pull-up 10 k y desacoplo
- [ ] 14 tiras probadas una a una
- [ ] Peso balanceado
- [ ] Barniz conformal aplicado (dejando libres conectores/botón/Hall)
- [ ] Fusible y (opcional) protección de polaridad
- [ ] USB-C accesible para carga
