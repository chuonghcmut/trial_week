# SWE2 — Software Architecture Document

| Field          | Value                                      |
|----------------|--------------------------------------------|
| **ASPICE ID**  | SWE.2 — Software Architectural Design      |
| **Project**    | Remote ECU Monitoring & Control System     |
| **Author**     | Wilson Nguyen                              |

---

## 1. System Overview

The system enables an OEM engineer to remotely monitor temperature and control 3 LEDs on an ECU from a web browser. It consists of three layers:

- **Target ECU** — Arduino Uno R3 reading temperature (LM35) and controlling 3 LEDs.
- **Host PC** — Python application serving a web dashboard and communicating with ECU over UART.
- **Remote Access** — ngrok tunnel exposing the dashboard to the internet for OEM access.

---

## 2. System Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                          INTERNET                                    │
│                                                                      │
│    ┌──────────────┐         ngrok tunnel         ┌────────────────┐  │
│    │ OEM Engineer  │◄──────────────────────────► │ ngrok server    │  │
│    │ (Germany)     │   https://xxx.ngrok-free.app │ (cloud)        │  │
│    │ Browser       │                              └───────┬────────┘  │
│    └──────────────┘                                       │          │
└───────────────────────────────────────────────────────────┼──────────┘
                                                            │
                                                            │ tunnel
                                                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        HOST PC (Supplier site)                       │
│                                                                      │
│    ┌──────────────────────────────────────────────────────────────┐  │
│    │                  Python Application                          │  │
│    │                                                              │  │
│    │  ┌──────────────┐    ┌──────────────┐    ┌───────────────┐  │  │
│    │  │ Flask Web     │◄──►│ Socket.IO    │◄──►│ Serial        │  │  │
│    │  │ Server        │    │ Handler      │    │ Interface     │  │  │
│    │  │               │    │              │    │ (pyserial)    │  │  │
│    │  │ Serves HTML/  │    │ Realtime     │    │               │  │  │
│    │  │ JS/CSS to     │    │ push temp    │    │ UART TX/RX    │  │  │
│    │  │ browser       │    │ data to      │    │ to Arduino    │  │  │
│    │  │               │    │ browser      │    │               │  │  │
│    │  │ Port 5000     │    │              │    │ 115200 baud   │  │  │
│    │  └──────────────┘    └──────────────┘    └──────┬────────┘  │  │
│    │                                                  │          │  │
│    └──────────────────────────────────────────────────┼──────────┘  │
│                                                       │             │
└───────────────────────────────────────────────────────┼─────────────┘
                                                        │
                                                   USB Cable
                                                   (UART 115200 8N1)
                                                        │
┌───────────────────────────────────────────────────────┼─────────────┐
│                     TARGET ECU (Arduino Uno R3)       │             │
│                                                       │             │
│    ┌─────────────────────────────────────────────────────────────┐  │
│    │                  Arduino Firmware (C)                        │  │
│    │                                                             │  │
│    │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐ │  │
│    │  │ ADC Module    │  │ LED Module   │  │ UART Module       │ │  │
│    │  │               │  │              │  │                   │ │  │
│    │  │ Read LM35     │  │ Control 3    │  │ TX: TEMP_REPORT   │ │  │
│    │  │ on A0         │  │ LEDs on      │  │   (7 bytes/500ms) │ │  │
│    │  │               │  │ D4, D7, D8   │  │                   │ │  │
│    │  │ Convert to    │  │              │  │ RX: LED_COMMAND   │ │  │
│    │  │ deci-Celsius  │  │ ON/OFF/      │  │   (3 bytes/       │ │  │
│    │  │               │  │ TOGGLE       │  │    on-demand)     │ │  │
│    │  │ 4-sample      │  │              │  │                   │ │  │
│    │  │ moving avg    │  │              │  │                   │ │  │
│    │  └──────┬───────┘  └──────┬───────┘  └────────┬──────────┘ │  │
│    │         │                 │                    │            │  │
│    │         └─────────────────┴────────────────────┘            │  │
│    │                    Main Loop (500ms cycle)                   │  │
│    └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│    Hardware:                                                        │
│    ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│    │ LM35    │  │ LED 0   │  │ LED 1   │  │ LED 2   │             │
│    │ (A0)    │  │ Red(D4) │  │ Grn(D7) │  │ Yel(D8) │             │
│    └─────────┘  └─────────┘  └─────────┘  └─────────┘             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. ECU Software Architecture

### 3.1 Module Description

| Module | Responsibility | Input | Output |
|--------|---------------|-------|--------|
| ADC Module | Read LM35 analog voltage on A0, convert to deci-Celsius, apply 4-sample moving average filter | Raw ADC value (0–1023) | Filtered temperature (int16, deci-Celsius) |
| LED Module | Control 3 LEDs independently based on received commands | LED index (0–2) + command (ON/OFF/TOGGLE) | GPIO state on D4, D7, D8 |
| UART Module | Transmit TEMP_REPORT every 500ms, receive and parse LED_COMMAND | Temperature + LED state (TX), raw bytes from Host (RX) | Serial bytes out (TX), parsed command (RX) |
| Main Loop | Orchestrate all modules in a 500ms cycle | — | — |

### 3.2 Main Loop Flow

```
setup():
    Initialize Serial at 115200 baud
    Set D4, D7, D8 as OUTPUT
    Initialize LED states to OFF
    Initialize counter to 0

loop():
    ┌─────────────────────────────────┐
    │ 1. Read ADC (A0)                │
    │    → raw value 0–1023           │
    │                                 │
    │ 2. Convert to deci-Celsius      │
    │    → temp = raw * 500 / 1023    │
    │                                 │
    │ 3. Apply 4-sample moving avg    │
    │    → filtered temperature       │
    │                                 │
    │ 4. Determine status byte        │
    │    → OK / Over-temp / Under-temp│
    │                                 │
    │ 5. Check Serial for LED_COMMAND │
    │    → if 0xBB received:          │
    │      parse LED index + command  │
    │      update LED GPIO            │
    │                                 │
    │ 6. Pack TEMP_REPORT (7 bytes)   │
    │    → header + temp + status     │
    │      + counter + LED bitmap     │
    │      + last LED index           │
    │                                 │
    │ 7. Send TEMP_REPORT via Serial  │
    │                                 │
    │ 8. Increment counter (wrap 255) │
    │                                 │
    │ 9. Delay to maintain 500ms      │
    └─────────────────────────────────┘
```

### 3.3 ADC to Temperature Conversion

LM35 outputs 10mV/°C. Arduino ADC reference is 5V with 10-bit resolution (0–1023).

```
voltage    = adc_raw * 5.0 / 1023
temp_C     = voltage / 0.01            (since 10mV = 0.01V per °C)
temp_deci  = temp_C * 10               (convert to deci-Celsius)

Simplified: temp_deci = adc_raw * 500 / 1023
```

Example: room temperature 25.0°C → LM35 outputs 250mV → ADC reads ~51 → 51 * 500 / 1023 ≈ 249 → 24.9°C.

Full conversion documentation: see `03-SWE3-design/conversion-formula.md`

### 3.4 Temperature Filter

4-sample moving average to reduce ADC noise:

```
buffer[4] = {0, 0, 0, 0}
index = 0

filter(new_value):
    buffer[index] = new_value
    index = (index + 1) % 4
    return sum(buffer) / 4
```

---

## 4. Host Software Architecture

### 4.1 Module Description

| Module | Responsibility | Input | Output |
|--------|---------------|-------|--------|
| Serial Interface | Read/write UART via pyserial, parse TEMP_REPORT, send LED_COMMAND | Raw bytes from COM port | Parsed temperature, LED state, counter |
| Flask Web Server | Serve dashboard HTML/JS/CSS to browser on port 5000 | HTTP request from browser | HTML page |
| Socket.IO Handler | Push realtime data to browser, receive LED button clicks | Parsed data from Serial Interface (push), browser events (receive) | WebSocket messages to browser, LED_COMMAND bytes to Serial Interface |
| Latency Tracker | Measure round-trip time for each LED command | Timestamp at send, timestamp at acknowledgment | Latency value (ms) |
| COMM Timeout Detector | Monitor last TEMP_REPORT timestamp, trigger warning if > 2s | Timestamps of received frames | COMM TIMEOUT event to browser |
| Counter Gap Detector | Compare consecutive rolling counters, log discontinuities | Counter values from consecutive frames | Gap warning log |

### 4.2 Data Flow

```
Serial Interface (background thread)
    │
    │ Receives 7 bytes from Arduino every 500ms
    │ Parses: temperature, status, counter, LED state
    │
    ├──► COMM Timeout Detector
    │       → if no frame for 2s → emit 'comm_timeout' to browser
    │
    ├──► Counter Gap Detector
    │       → if counter gap detected → log warning
    │
    └──► Socket.IO Handler
            │
            │ emit('temp_update', {temp, status, led_state, counter})
            │
            ▼
         Browser (dashboard)
            │
            │ User clicks "LED 1 ON"
            │
            │ emit('led_command', {index: 1, action: 1})
            │
            ▼
         Socket.IO Handler
            │
            │ Records t_send = now()
            │
            ├──► Serial Interface
            │       → sends 0xBB 0x01 0x01 to Arduino
            │
            │ Waits for TEMP_REPORT with updated LED bitmap
            │
            │ Records t_ack = now()
            │ latency = t_ack - t_send
            │
            └──► emit('latency_update', {led: 1, latency_ms: xxx})
                    │
                    ▼
                 Browser displays latency
```

### 4.3 Dashboard Interface

The web dashboard displays the following elements:

| Element | Description | Update Method |
|---------|-------------|--------------|
| Temperature display | Current temperature in °C | Socket.IO push every 500ms |
| Status indicator | OK / Over-temp / Under-temp | Socket.IO push every 500ms |
| LED 0 button (Red) | Toggle LED 0 ON/OFF | User click → Socket.IO emit |
| LED 1 button (Green) | Toggle LED 1 ON/OFF | User click → Socket.IO emit |
| LED 2 button (Yellow) | Toggle LED 2 ON/OFF | User click → Socket.IO emit |
| LED state indicators | Show current ON/OFF state of each LED | Socket.IO push from TEMP_REPORT bitmap |
| Round-trip latency | Latency in ms for last LED command | Calculated after acknowledgment |
| COMM TIMEOUT warning | Displayed when no frame received for 2s | COMM Timeout Detector |
| Message counter | Current rolling counter value | Socket.IO push every 500ms |
| Counter gap log | Log of detected counter discontinuities | Counter Gap Detector |

Wireframe: see `03-SWE3-design/dashboard-wireframe.md`

---

## 5. Communication Architecture

### 5.1 Protocol Summary

| Message | Direction | Header | Data | Period | Ref |
|---------|-----------|--------|------|--------|-----|
| TEMP_REPORT | ECU → Host | 0xAA | 6 bytes | 500ms ±5% | REQ-T-01 |
| LED_COMMAND | Host → ECU | 0xBB | 2 bytes | On-demand | REQ-F-06 |

Full frame layout: see `02-SWE2-architecture/frame-layout.md`

### 5.2 COMM TIMEOUT Logic

```
last_frame_time = now()

on_frame_received():
    last_frame_time = now()
    clear_timeout_warning()

check_timeout():                     (runs every 100ms)
    if now() - last_frame_time > 2000ms:
        display "COMM TIMEOUT"       (REQ-C-02)
```

### 5.3 Counter Gap Detection

```
expected_counter = 0

on_frame_received(counter):
    if counter != expected_counter:
        gap = counter - expected_counter     (handle wrap-around)
        log("Counter gap: expected {expected}, got {counter}, missed {gap-1} frames")
    expected_counter = (counter + 1) % 256   (REQ-C-03)
```

---

## 6. Remote Access Architecture

```
┌──────────────┐     HTTPS     ┌──────────────┐    tunnel    ┌──────────────┐
│ OEM Browser  │◄────────────►│ ngrok server  │◄───────────►│ Host PC       │
│ (Germany)    │               │ (cloud)       │              │ port 5000     │
└──────────────┘               └──────────────┘              └──────────────┘
```

| Component | Role |
|-----------|------|
| ngrok client | Runs on Host PC, creates tunnel from ngrok cloud to localhost:5000 |
| ngrok server | Public endpoint, forwards HTTPS requests to Host PC through tunnel |
| OEM browser | Accesses dashboard via ngrok public URL, same interface as local access |

The dashboard behaves identically for local and remote users. No code changes needed for remote access — ngrok handles all network routing transparently.

---

## 7. Requirement Traceability

| Requirement | Architecture Section | Implementation |
|-------------|---------------------|----------------|
| REQ-F-01 | §3.1 ADC Module | ADC read + conversion on A0 |
| REQ-F-02 | §3.1 UART Module, §5.1 | TEMP_REPORT TX every 500ms |
| REQ-F-03 | §3.1 UART Module, §5.1 | LED_COMMAND RX parsing |
| REQ-F-04 | §3.1 LED Module | 3 independent LEDs on D4, D7, D8 |
| REQ-F-05 | §4.1 Flask + Socket.IO | Live temperature on dashboard |
| REQ-F-06 | §4.2 Socket.IO Handler | LED button click → LED_COMMAND |
| REQ-F-07 | §6 Remote Access | ngrok tunnel to port 5000 |
| REQ-T-01 | §3.2 Main Loop | 500ms delay in loop |
| REQ-T-02 | §4.2 Latency Tracker | LED bitmap echo in TEMP_REPORT |
| REQ-T-03 | §4.2 Socket.IO push | Dashboard staleness < 1000ms |
| REQ-T-04 | §4.2 Latency Tracker | Round-trip latency displayed |
| REQ-C-01 | §5.1 Protocol | UART 115200 baud |
| REQ-C-02 | §5.2 COMM Timeout | 2s timeout detection |
| REQ-C-03 | §5.3 Counter Gap | Rolling counter monitoring |
| REQ-D-01 | §5.1 | frame-layout.md |
| REQ-D-02 | §3.1 | pin-assignment.md |

