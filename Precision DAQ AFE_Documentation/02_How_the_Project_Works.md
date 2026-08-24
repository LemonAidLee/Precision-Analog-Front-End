# 02 — How the Project Works

## Signal Path (Overview)

```text
Real-World Sensor
       ↓
Noisy Analog Signal
       ↓
Signal Conditioning
       ↓
Amplification
       ↓
Filtering
       ↓
Level Shifting
       ↓
ADC
       ↓
Digital Data
       ↓
Microcontroller
       ↓
Computer
```

## Power Path (Overview)

```text
USB 5V
  ↓
DC-DC Converter
  ↓
Raw ±14V
  ↓
Linear Regulators
  ↓
Clean ±12V
  ↓
Analog Circuit
```

These two paths run in parallel: the power path keeps the analog electronics fed with clean, stable voltage, while the signal path carries the actual measurement from sensor to computer.

## 1. Power Supply

- **What enters the system:** ordinary 5V USB power — convenient, but not clean or high-voltage enough for precision analog circuits.
- **Why the analog circuit needs its own suitable supply:** op-amps and precision analog parts typically need higher voltage headroom (e.g., ±12V) than USB provides, and they're sensitive to noise on their supply rails.
- **Why switching conversion is useful:** a DC-DC switching converter can efficiently boost 5V up to a higher raw voltage (**Planned:** roughly ±14V) without wasting much energy as heat.
- **Why switching converters can introduce noise:** switching regulators work by rapidly turning a switch on and off, which creates high-frequency electrical noise (switching ripple) on the output.
- **Why linear post-regulation is useful:** a linear regulator smooths and stabilizes the raw switching output into clean rails (**Planned:** ±12V) suitable for precision analog circuitry. Note: parts like the LM7812/LM7912 are **linear regulators**, not LDOs (low-dropout regulators) — that's a different, more specific category.

## 2. Signal Input

- **What a sensor signal is:** an electrical voltage or current that changes in response to a physical quantity (temperature, pressure, vibration, etc.).
- **Why it can be weak:** many sensors produce only millivolts of signal for a meaningful physical change.
- **Why it can contain noise:** wiring, nearby electronics, and the environment can all inject unwanted electrical interference onto the signal path.

## 3. Amplification / Buffering

> The amplifier makes the useful signal larger so the ADC can measure it more effectively.

**Buffering**, in simple terms, means placing a circuit between the sensor and the rest of the system so the sensor doesn't get "loaded down" or disturbed by what comes after it — like a middleman that passes the signal along faithfully without letting downstream circuitry pull on it.

**Technical term: Op-amp buffer / instrumentation amplifier** — circuits commonly used for this purpose.

## 4. Filtering

> A filter is like a sieve for electrical signals. It lets some frequencies through while reducing others.

The filter allows the wanted part of the signal (e.g., a slow-changing temperature reading) to pass through, while reducing unwanted high-frequency noise.

**Technical term: Low-pass filter** — a filter that passes low frequencies and attenuates high frequencies. (**Planned:** the exact cutoff frequency will be determined through simulation and measurement, not assumed in advance.)

## 5. Level Shifting

A sensor might output a signal like:

```text
Sensor:
-1V to +1V
```

But a typical ADC only accepts:

```text
ADC:
0V to 3.3V
```

Feeding a negative voltage directly into most ADCs can damage them or simply fail to register. **Level shifting** adds a voltage offset to move the signal into the ADC's usable range, so the full swing of the sensor signal maps onto the ADC's input window.

## 6. ADC (Analog-to-Digital Converter)

> **The ADC turns a continuously changing voltage into a number that a computer can understand.**

**Planned:** an ADS1115 (16-bit ADC) is being considered. Note: a 16-bit ADC does not automatically deliver 16 bits of *usable, noise-free* precision — the real effective resolution depends on noise and will need to be measured.

## 7. Microcontroller

The microcontroller (**Planned:** ESP32 or STM32) reads digital values from the ADC and passes the measurement data along to a computer, typically over USB or serial.

## 8. Computer

Once the data reaches a computer, it can be used to:

- Display the waveform
- Record measurements
- Plot data over time
- Analyze noise
- Evaluate overall system performance

## Summary

```text
Sensor
  ↓
Clean
  ↓
Amplify
  ↓
Filter
  ↓
Shift
  ↓
Digitize
  ↓
Analyze
```
