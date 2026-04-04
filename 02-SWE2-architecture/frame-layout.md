# Communication Protocol & Frame Layout

| Field          | Value                                      |
|----------------|--------------------------------------------|
| **Document**   | SDT-2026-001                               |
| **Project**    | Remote ECU Monitoring & Control System     |
| **Author**     | Wilson Nguyen                              |

---

## 1. Bus Configuration

| Parameter | Value | Requirement |
|-----------|-------|-------------|
| Physical Layer | UART over USB (CH340) | REQ-C-01 |
| Baud Rate | 115200 | REQ-C-01 |
| Data Bits | 8 | — |
| Parity | None | — |
| Stop Bits | 1 | — |
| Flow Control | None | — |

UART is selected instead of CAN because the developer is already familiar with UART and the 7-day timeline requires fast development with minimal hardware risk. Arduino Uno has built-in UART over USB — no additional transceiver needed. This is permitted by REQ-C-01.

---

## 2. Message Overview

| Message | Direction | ID (Header) | DLC | Period | Purpose |
|---------|-----------|-------------|-----|--------|---------|
| TEMP_REPORT | ECU → Host | 0xAA | 7 bytes (header + 6 data) | 500ms ±5% | Periodic temperature and status report |
| LED_COMMAND | Host → ECU | 0xBB | 3 bytes (header + 2 data) | On-demand | Operator LED control command |

---

## 3. TEMP_REPORT (ECU → Host)

Arduino sends this message every 500ms (REQ-T-01).

### Frame Structure

| Byte | Name | Type | Range | Description |
|------|------|------|-------|-------------|
| Header | HEADER | uint8 | 0xAA | Fixed header byte — Host uses this to identify start of TEMP_REPORT frame |
| 0 | TEMP_LOW | uint8 | 0x00–0xFF | Temperature low byte (little-endian) |
| 1 | TEMP_HIGH | uint8 | 0x00–0xFF | Temperature high byte (little-endian) |
| 2 | STATUS | uint8 | 0x00–0x02 | Sensor status flag |
| 3 | COUNTER | uint8 | 0x00–0xFF | Rolling message counter, wraps 255 → 0 |
| 4 | LED_STATE | uint8 | 0x00–0x07 | LED state bitmap |
| 5 | LAST_LED_IDX | uint8 | 0x00–0x02 | Index of last commanded LED |

**Total: 7 bytes on wire (1 header + 6 data)**

### Temperature Encoding (Byte 0–1)

Temperature is encoded as **int16, little-endian, in deci-Celsius** (0.1°C resolution).

| Raw Value (int16) | Temperature | Byte 0 | Byte 1 |
|-------------------|-------------|--------|--------|
| 251 | 25.1°C | 0xFB | 0x00 |
| 300 | 30.0°C | 0x2C | 0x01 |
| 0 | 0.0°C | 0x00 | 0x00 |
| -100 | -10.0°C | 0x9C | 0xFF |
| 601 | 60.1°C (over-temp) | 0x59 | 0x02 |

Conversion on Host side: `temperature_celsius = (byte1 << 8 | byte0) / 10.0` (treat as signed int16).

### Status Byte (Byte 2)

| Value | Meaning | Condition |
|-------|---------|-----------|
| 0x00 | OK | -10.0°C ≤ temperature ≤ 60.0°C |
| 0x01 | Over-temperature fault | temperature > 60.0°C |
| 0x02 | Under-temperature fault | temperature < -10.0°C |

### Rolling Counter (Byte 3)

Counter increments by 1 for each TEMP_REPORT sent. Wraps from 255 → 0. Host shall detect and log counter discontinuities as missed frames (REQ-C-03).

Example: if Host receives counter 10, then 13, it logs a gap of 2 missed frames.

### LED State Bitmap (Byte 4)

| Bit | LED | Meaning |
|-----|-----|---------|
| bit 0 | LED 0 (Red, D4) | 1 = ON, 0 = OFF |
| bit 1 | LED 1 (Green, D7) | 1 = ON, 0 = OFF |
| bit 2 | LED 2 (Yellow, D8) | 1 = ON, 0 = OFF |
| bit 3–7 | Reserved | Always 0 |

Example: LED 0 ON, LED 1 OFF, LED 2 ON → bitmap = 0b00000101 = 0x05.

This bitmap is used by the Host to verify LED state after sending a command (REQ-T-02 acknowledgment).

### Last Commanded LED Index (Byte 5)

Echoes the index of the most recently commanded LED (0, 1, or 2). Used together with LED_STATE bitmap to confirm which LED was last changed.

---

## 4. LED_COMMAND (Host → ECU)

Host sends this message when the operator clicks a button on the dashboard.

### Frame Structure

| Byte | Name | Type | Range | Description |
|------|------|------|-------|-------------|
| Header | HEADER | uint8 | 0xBB | Fixed header byte — ECU uses this to identify start of LED_COMMAND frame |
| 0 | LED_INDEX | uint8 | 0x00–0x02 | Target LED |
| 1 | COMMAND | uint8 | 0x00–0x02 | Action to perform |

**Total: 3 bytes on wire (1 header + 2 data)**

### LED Index (Byte 0)

| Value | LED | Arduino Pin |
|-------|-----|-------------|
| 0x00 | LED 0 (Red) | D4 |
| 0x01 | LED 1 (Green) | D7 |
| 0x02 | LED 2 (Yellow) | D8 |

ECU shall ignore commands with LED_INDEX > 0x02.

### Command (Byte 1)

| Value | Action | Description |
|-------|--------|-------------|
| 0x00 | OFF | Set LED to OFF |
| 0x01 | ON | Set LED to ON |
| 0x02 | TOGGLE | Flip current LED state |

ECU shall ignore commands with COMMAND > 0x02.

---

## 5. Frame Synchronization

Because UART is a byte stream (no built-in message boundaries), the Host and ECU use **header bytes** to find the start of each frame:

- Host scans incoming bytes until it finds **0xAA**, then reads the next **6 bytes** as TEMP_REPORT data.
- ECU scans incoming bytes until it finds **0xBB**, then reads the next **2 bytes** as LED_COMMAND data.

If a frame is corrupted or incomplete (e.g., fewer bytes received than expected), the receiver discards the partial frame and waits for the next header byte.

---

## 6. Timing & Latency Measurement

### TX Period (REQ-T-01)

ECU sends TEMP_REPORT every 500ms ±5% (475–525ms). Host can verify by measuring the time delta between consecutive 0xAA headers.

### Command Acknowledgment (REQ-T-02)

ECU acknowledges LED commands by reflecting the new LED state in the LED_STATE bitmap (Byte 4) of the next TEMP_REPORT. The acknowledgment shall appear within 50ms of command reception.

### Round-Trip Latency Measurement (REQ-T-04)

The dashboard measures round-trip latency as follows:

1. User clicks LED button → Host records `t_send = now()`.
2. Host sends LED_COMMAND over UART.
3. Host monitors incoming TEMP_REPORT frames.
4. When LED_STATE bitmap reflects the new state → Host records `t_ack = now()`.
5. Round-trip latency = `t_ack - t_send`.

This value is displayed on the dashboard for each LED command.

### COMM TIMEOUT (REQ-C-02)

If Host receives no TEMP_REPORT for **2 consecutive seconds**, the dashboard displays "COMM TIMEOUT". Timer resets on each valid TEMP_REPORT received.

---

## 7. Example Communication Sequence

```
Time (ms)   Direction       Raw Bytes (hex)              Meaning
────────────────────────────────────────────────────────────────────
0           ECU → Host      AA FB 00 00 00 00 00         Temp=25.1°C, status=OK, counter=0, LEDs=all OFF
500         ECU → Host      AA FB 00 00 01 00 00         Temp=25.1°C, counter=1, LEDs=all OFF
620         Host → ECU      BB 01 01                     LED 1 ON
650         ECU → Host      (internal: set D7 HIGH)
1000        ECU → Host      AA FB 00 00 02 02 01         Temp=25.1°C, counter=2, LED1=ON (bitmap=0x02), last_idx=1
1500        ECU → Host      AA FB 00 00 03 02 01         Temp=25.1°C, counter=3, LED1 still ON
2100        Host → ECU      BB 02 02                     LED 2 TOGGLE
2500        ECU → Host      AA FC 00 00 04 06 02         Temp=25.2°C, counter=4, LED1+LED2=ON (bitmap=0x06), last_idx=2
```

In this example, round-trip latency for LED 1 ON command = 1000ms - 620ms = 380ms (latency includes waiting for next TEMP_REPORT cycle).

