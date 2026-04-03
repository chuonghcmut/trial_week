# Pin Assignment Document

| Field          | Value                                      |
|----------------|--------------------------------------------|
| **Document**   | SDT-2026-001                               |
| **Project**    | Remote ECU Monitoring & Control System     |
| **Author**     | *(your name)*                              |
| **Date**       | 03/04/2026                                 |
| **Board**      | Arduino Uno R3                             |

---

## 1. Pin Assignment Table

| Pin | Type | Connected To | Purpose | Requirement |
|-----|------|-------------|---------|-------------|
| A0 | Analog Input | LM35 Vout | Read temperature voltage. LM35 outputs 10mV/°C, Arduino ADC converts to 0–1023 (10-bit). | REQ-F-01 |
| D4 | Digital Output | LED 0 (Red) via 220Ω resistor | Independently controlled LED, index 0 | REQ-F-04 |
| D7 | Digital Output | LED 1 (Green) via 220Ω resistor | Independently controlled LED, index 1 | REQ-F-04 |
| D8 | Digital Output | LED 2 (Yellow) via 220Ω resistor | Independently controlled LED, index 2 | REQ-F-04 |
| D0 (RX) | UART RX | USB-to-Serial (CH340) | Receive LED_COMMAND from Host PC | REQ-F-03, REQ-C-01 |
| D1 (TX) | UART TX | USB-to-Serial (CH340) | Transmit TEMP_REPORT to Host PC | REQ-F-02, REQ-C-01 |
| 5V | Power | LM35 Vs (+), Breadboard power rail | 5V supply for LM35 sensor | — |
| GND | Power | LM35 GND, LED cathodes, Breadboard GND rail | Common ground for all components | — |

---

## 2. Pin Selection Rationale

### A0 — LM35 Temperature Sensor

A0 is the first analog input channel on Arduino Uno. Any analog pin (A0–A5) would work, but A0 is chosen by convention for single-sensor designs. LM35 output is connected directly — no voltage divider needed because LM35 outputs 0–1V for 0–100°C range, which is within the Arduino ADC reference voltage (5V).

### D4, D7, D8 — LEDs

These three pins are chosen because they avoid conflicts with special-function pins on Arduino Uno:

- **D0, D1** are reserved for UART (TX/RX) — cannot be used for LEDs.
- **D2, D3** are the only hardware interrupt pins — kept available for future use.
- **D13** has a built-in LED on the Arduino board — using it would cause confusion during testing (two LEDs responding to one command).
- **D4, D7, D8** are general-purpose digital pins with no special function, no boot mode conflict, and no shared peripherals.

The pins are intentionally non-consecutive to reduce the risk of accidental solder bridges or jumper wire shorts on the breadboard.

### D0 (RX), D1 (TX) — UART

These are the hardware UART pins on ATmega328P, directly connected to the CH340 USB-to-Serial chip on the Arduino board. No configuration needed — `Serial.begin(115200)` enables communication at the required baud rate (REQ-C-01).

**Note:** Because D0/D1 are shared between UART and USB, the serial monitor in Arduino IDE cannot be used while the dashboard is connected. During development, use the dashboard logs or add debug LEDs to verify behavior.

---

## 3. Wiring Diagram (ASCII)

```
                        Arduino Uno R3
                    ┌───────────────────┐
                    │                   │
   [LM35 Vout] ───►│ A0           D13  │  (built-in LED — not used)
                    │              D12  │
                    │              D11  │
                    │              D10  │
                    │              D9   │
                    │              D8   │◄─── [220Ω] ─── [LED 2 Yellow] ─── GND
                    │              D7   │◄─── [220Ω] ─── [LED 1 Green]  ─── GND
                    │              D6   │
                    │              D5   │
                    │              D4   │◄─── [220Ω] ─── [LED 0 Red]    ─── GND
                    │              D3   │
                    │              D2   │
   USB ◄───────────│ D1 (TX)      D1   │
   (to Host PC)    │ D0 (RX)      D0   │
                    │                   │
      [LM35 Vs] ──►│ 5V          GND   │◄─── [LM35 GND]
                    │              GND   │◄─── [LED cathodes via GND rail]
                    └───────────────────┘
```

### LM35 Wiring Detail

```
        ┌──────────┐
        │  LM35DZ  │
        │          │
        │ Vs  Vout GND
        └─┬───┬────┬─┘
          │   │    │
          │   │    └──── Arduino GND
          │   └───────── Arduino A0
          └───────────── Arduino 5V
```

**LM35 pin order (flat side facing you, left to right):** Vs (+5V), Vout (signal), GND.

---

## 4. LED Current Calculation

Each LED is connected in series with a 220Ω current-limiting resistor.

```
Arduino GPIO (HIGH = 5V) ──► [220Ω] ──► [LED anode (+)] ──► [LED cathode (–)] ──► GND
```

Calculation:
- GPIO output voltage: 5V
- Typical LED forward voltage: ~2V
- Voltage across resistor: 5V – 2V = 3V
- Current: 3V / 220Ω ≈ 13.6mA

This is within the safe operating range for Arduino GPIO pins (max 20mA per pin, max 100mA total for all pins). Three LEDs at 13.6mA = 40.8mA total — well within limits.

