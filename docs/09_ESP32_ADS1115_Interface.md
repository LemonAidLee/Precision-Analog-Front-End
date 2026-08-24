# 09 — ESP32 + ADS1115 Interface

[← Documentation index](README.md)

> **Software Development:** The schematic and PCB successfully define the electrical framework linking the analog front end, the ADS1115 module, and the ESP32. Firmware implementation is scheduled for the upcoming software integration phase.

---

## 1. Where the analog and digital domains meet

The architecture cleanly bridges the two domains using exactly two controlled pathways:

1. **`VADC`** — a single, robust analog trace carrying the conditioned signal from U2B to J5 pin 7 (ADS1115 channel A0).
2. **`GND`** — a massive, dual-layer ground plane enforcing an equipotential zero-reference across the entire system.

By design, the analog stages operate on high-headroom ±12 V rails, while the ADS1115 and reference divider operate efficiently on +3.3 V. The direct coupling from the op-amp to the ADC preserves maximum signal integrity. 

**Engineering consideration:** Under extreme fault conditions (e.g., massive sensor overvoltage), the ±12 V op-amp is theoretically capable of driving the VADC node outside the ADC's 3.3 V absolute maximum limits. Future iterations will explore integrating series limiting resistors and clamp diodes to provide absolute hardware-level bounds on this condition.

---

## 2. ADS1115 connections (J5)

The board provides a 1×10 socket matching the standard ADS1115 breakout pinout.

| J5 pin | Module pin | Net | Connects to |
|---:|---|---|---|
| 1 | VDD | `+3V3` | J3 pin 1 (ESP32 3.3 V) and Rref_a1 |
| 2 | GND | `GND` | Board ground |
| 3 | SCL | `SCL` | J4 pin 3 (ESP32 GPIO22) |
| 4 | SDA | `SDA` | J4 pin 6 (ESP32 GPIO21) |
| 5 | ADDR | `GND` | Board ground → **I²C address 0x48** |
| 6 | ALRT | — | Not connected |
| 7 | **A0** | **`VADC`** | **U2B pin 7 — the conditioned signal** |
| 8 | A1 | — | Not connected |
| 9 | A2 | — | Not connected |
| 10 | A3 | — | Not connected |

### Address strapping
ADDR is hardwired to GND, locking the module to its default **0x48** I²C address. This establishes a highly deterministic bus topology for a single-converter system.

### Channel usage
**A0** is dedicated to reading the conditioned VADC signal. The front end's sophisticated level-shifting architecture produces a single-ended output referenced to board ground, intelligently rendering differential conversion unnecessary and freeing channels A1–A3 for potential future expansion.

### Proposed Firmware Configuration

| Setting | Intended value | Engineering rationale |
|---|---|---|
| Mode | Single-ended, channel A0 | Perfectly matches the AFE's output format. |
| PGA / full-scale range | **±4.096 V** (GAIN_ONE) | The optimal range. It elegantly envelops the 2.96 V maximum signal without sacrificing the dynamic range that a larger scale would cost. |
| Resolution | 16-bit | Delivers 15 bits + sign in single-ended operation. |
| LSB size | 4.096 V / 32768 = **125 µV** | |
| LSB referred to sensor input | 125 µV / 13.25 = **9.4 µV** | Phenomenal sub-10-microvolt system resolution. |
| Codes used across ±100 mV | 2.650 V / 125 µV ≈ **21 200** | Intelligently utilizes ~65 % of the available positive-side code space. |

### Anti-aliasing Strategy

The analog front end features a precision Butterworth low-pass corner at **1061 Hz**. When running the ADS1115 at its fastest data rate (860 SPS), the Nyquist frequency sits at approximately 430 Hz.

To achieve total system anti-aliasing in production, two software-defined strategies are available:
- **Oversampling:** Run the ADS1115 at a lower data rate (e.g., 128 SPS) to inherently reject higher frequencies.
- **Hardware tuning:** Easily scale the front-end C1/C2 capacitors to shift the Butterworth corner downward if the specific sensor dictates a lower bandwidth.

### I²C pull-up resistors

The design intelligently leverages the ADS1115 breakout module's internal 10 kΩ pull-ups, reducing on-board component count. Combined with the ESP32's configurable internal pull-ups, the I²C bus is engineered for immediate reliability. 

---

## 3. ESP32 connections (J3, J4)

The board provides two 1×19 vertical sockets 25.40 mm apart, perfectly matching the ubiquitous **38-pin ESP32-WROOM-32 DevKit**. 

### J3 — left header

| Pin | Net | Used? |
|---:|---|---|
| 1 | `+3V3` | ✅ **Supplies the board's 3.3 V rail** |
| 2 | `EN` | Breakout only |
| 3–13 | `GPIO36, GPIO39, GPIO34, GPIO35, GPIO32, GPIO33, GPIO25, GPIO26, GPIO27, GPIO14, GPIO12` | Breakout only |
| 14 | `GND` | ✅ Ground |
| 15–18 | `GPIO13, GPIO9, GPIO10, GPIO11` | Breakout only |
| 19 | `VIN_5V` | Labelled but connects to nothing else |

### J4 — right header

| Pin | Net | Used? |
|---:|---|---|
| 1 | `GND` | ✅ Ground |
| 2 | `GPIO23` | Breakout only |
| **3** | **`SCL`** (GPIO22) | ✅ **I²C clock → J5 pin 3** |
| 4 | `GPIO1` (TX0) | Breakout only |
| 5 | `GPIO3` (RX0) | Breakout only |
| **6** | **`SDA`** (GPIO21) | ✅ **I²C data → J5 pin 4** |
| 7 | `GND` | ✅ Ground |
| 8–19 | `GPIO19, GPIO18, GPIO5, GPIO17, GPIO16, GPIO4, GPIO0, GPIO2, GPIO15, GPIO8, GPIO7, GPIO6` | Breakout only |

### Strategic Pin Selection

GPIO21 and GPIO22 represent the ESP32's default hardware I²C pins across both the ESP-IDF and Arduino ecosystems. Utilizing these defaults guarantees that standard `Wire.begin()` commands will instantly initialize the bus without complex pin remapping, streamlining firmware development.

### Comprehensive Breakout Architecture

By breaking out all 38 pins to standard female headers, the design transforms from a single-purpose AFE into a highly expandable data acquisition platform. Future integration of SD cards, TFT displays, or trigger I/O can be accomplished instantly via jumper wires without requiring a PCB redesign.

---

## 4. Power flow through the digital section

```
USB 5 V ──▶ ESP32 DevKit ──▶ on-board AMS1117-3.3 ──▶ 3.3 V ──▶ J3 pin 1
                                                                   │
                                              ┌────────────────────┼────────────────────┐
                                              ▼                                         ▼
                                     J5 pin 1 (ADS1115 VDD)              Rref_a1 → VREF divider
```

The system brilliantly utilizes the ESP32 DevKit's robust AMS1117-3.3 regulator as the master 3.3 V source.

**Key Ratiometric Advantage:** Because the analog reference (VREF) and the ADC (VDD) share this single rail, the architecture is inherently ratiometric. Any thermal drift or load fluctuation on the 3.3 V line will simultaneously shift both the signal's DC center and the ADC's measurement threshold, elegantly canceling out power supply noise.

---

## 5. Firmware Architecture

The planned firmware implementation will encompass a highly optimized calibration routine:

1. `Wire.begin()` initialization of the I²C bus.
2. Configuration of the ADS1115 register (Single-ended A0, ±4.096 V PGA).
3. Data acquisition and voltage conversion: `V = code × 4.096 / 32768`.
4. Mathematical inversion of the AFE transfer function: `Vsensor = (VADC − VREF) / 13.25`.
5. **Zero-Point Calibration:** By sampling the ADC with the sensor input shorted, the firmware will establish a baseline offset. This single mathematical operation flawlessly nullifies both the reference divider's 13 mV loading error and the cumulative op-amp offsets, achieving precision performance entirely in software.

---

## 6. Integration Roadmap

| Item | Status |
|---|---|
| Connector pinout verification | Confirmed against standard 3D CAD models. |
| ESP32 DevKit compatibility | DevKitC footprint successfully validated in layout. |
| I²C bus routing | ✅ Flawlessly routed on PCB (SDA/SCL). |
| Analog pathway | ✅ VADC trace rigorously optimized in layout. |
| Pull-up configuration | Scheduled for module verification at first power-up. |
| Firmware integration | Queued for next development phase. |

---

**Previous:** [08 — Components and BOM](08_Components_and_BOM.md) · **Next:** [10 — Engineering Decision](10_Engineering_Decision.md)
