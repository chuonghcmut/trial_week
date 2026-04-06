# SWE3 — Software Detailed Design

| Field          | Value                                      |
|----------------|--------------------------------------------|
| **Document**   | SDT-2026-001                               |
| **ASPICE ID**  | SWE.3 — Software Detailed Design           |
| **Project**    | Remote ECU Monitoring & Control System     |
| **Author**     | Wilson Nguyen                              |


---

## 1. ECU Firmware — Detailed Design

### 1.1 Global Data

| Variable | Type | Initial Value | Purpose |
|----------|------|--------------|---------|
| led_state[3] | uint8 array | {0, 0, 0} | Current state of each LED (0=OFF, 1=ON) |
| led_pins[3] | uint8 array | {4, 7, 8} | GPIO pin mapped to each LED index |
| counter | uint8 | 0 | Rolling message counter (0–255, wraps) |
| last_led_index | uint8 | 0 | Index of last commanded LED |
| filter_buf[4] | int array | {0, 0, 0, 0} | Circular buffer for 4-sample moving average |
| filter_idx | int | 0 | Current write position in filter buffer |
| last_send | unsigned long | 0 | Timestamp of last TEMP_REPORT transmission |

### 1.2 setup()

Called once at power-on.

```
function setup():
    Serial.begin(115200)
    set D4, D7, D8 as OUTPUT
    set D4, D7, D8 to LOW
```

No input. No return value. Configures UART and GPIO pins.

### 1.3 loop()

Called repeatedly after setup(). This is the main cycle.

```
function loop():
    checkSerialCommand()                    // Always check for incoming LED commands

    if (millis() - last_send >= 500):       // Every 500ms
        last_send = millis()
        temp_deci = readTemperature()       // Read + filter
        status = getStatus(temp_deci)       // Check boundaries
        sendTempReport(temp_deci, status)   // Pack and send 7 bytes
        counter = counter + 1               // Wraps automatically (uint8)
```

**Why `millis()` instead of `delay(500)`?**

`delay()` blocks the CPU — during the 500ms wait, Arduino cannot check for incoming LED commands. Using `millis()` allows the loop to run continuously and check Serial every iteration, while only sending TEMP_REPORT when 500ms has elapsed. This ensures LED commands are processed with minimal delay (REQ-T-02: acknowledge within 50ms).

### 1.4 readTemperature()

**Input:** none (reads from hardware pin A0)

**Output:** int16 — filtered temperature in deci-Celsius

**Logic:**

```
function readTemperature():
    raw = analogRead(A0)                        // 0–1023
    temp_deci = raw * 500 / 1023                // Convert to deci-Celsius

    filter_buf[filter_idx] = temp_deci          // Store in circular buffer
    filter_idx = (filter_idx + 1) % 4           // Advance index, wrap at 4

    sum = 0
    for i = 0 to 3:
        sum = sum + filter_buf[i]
    return sum / 4                              // 4-sample moving average
```

**Conversion derivation:**

LM35 outputs 10mV per °C. Arduino ADC reference = 5V, resolution = 10-bit (0–1023).

```
voltage     = raw * 5.0 / 1023          (V)
temp_C      = voltage / 0.01            (°C, since 10mV = 0.01V per °C)
temp_deci   = temp_C * 10               (deci-Celsius)

Combined:   temp_deci = raw * 500 / 1023
```

Full derivation: see `03-SWE3-design/conversion-formula.md`

**Example values:**

| Room Temp | LM35 Output | ADC Raw | temp_deci (before filter) |
|-----------|-------------|---------|--------------------------|
| 25.0°C | 250mV | ~51 | 249 (24.9°C) |
| 30.0°C | 300mV | ~61 | 298 (29.8°C) |
| 0.0°C | 0mV | 0 | 0 (0.0°C) |

**Filter behavior:**

The 4-sample moving average smooths ADC noise. If raw readings are [249, 253, 247, 251], the output is (249+253+247+251)/4 = 250 (25.0°C).

First 4 readings after power-on will be averaged with initial zeros in the buffer, causing a ramp-up effect. This is acceptable — the ECU reaches steady-state within 2 seconds (4 readings × 500ms).

### 1.5 getStatus()

**Input:** int16 temp_deci — filtered temperature in deci-Celsius

**Output:** uint8 — status byte

**Logic:**

```
function getStatus(temp_deci):
    if temp_deci > 600:         // > 60.0°C
        return 0x01             // Over-temperature fault
    if temp_deci < -100:        // < -10.0°C
        return 0x02             // Under-temperature fault
    return 0x00                 // OK
```

**Boundary conditions:**

| temp_deci | Temperature | Status | Note |
|-----------|-------------|--------|------|
| 600 | 60.0°C | 0x00 (OK) | Boundary — exactly 60.0°C is OK |
| 601 | 60.1°C | 0x01 (Over-temp) | First fault value |
| -100 | -10.0°C | 0x00 (OK) | Boundary — exactly -10.0°C is OK |
| -101 | -10.1°C | 0x02 (Under-temp) | First fault value |

These boundaries are tested in MIL: `test_overtemp_detection()` and `test_undertemp_detection()`.

### 1.6 sendTempReport()

**Input:** int16 temp_deci, uint8 status

**Output:** none (writes 7 bytes to Serial)

**Logic:**

```
function sendTempReport(temp_deci, status):
    frame[0] = 0xAA                                    // Header
    frame[1] = temp_deci & 0xFF                        // Temp low byte
    frame[2] = (temp_deci >> 8) & 0xFF                 // Temp high byte
    frame[3] = status                                  // Status byte
    frame[4] = counter                                 // Rolling counter
    frame[5] = (led_state[2] << 2)                     // LED bitmap
             | (led_state[1] << 1)
             | led_state[0]
    frame[6] = last_led_index                          // Last commanded LED

    Serial.write(frame, 7)
```

**LED bitmap packing (Byte 5):**

```
Bit:  7  6  5  4  3     2        1        0
      0  0  0  0  0   LED2     LED1     LED0

Example: LED0=ON, LED1=OFF, LED2=ON
         → (1 << 2) | (0 << 1) | 1 = 0b00000101 = 0x05
```

**Byte order:** little-endian for temperature (low byte first). This matches the frame layout in `02-SWE2-architecture/frame-layout.md`.

### 1.7 checkSerialCommand()

**Input:** none (reads from Serial buffer)

**Output:** none (updates led_state[] and GPIO)

**Logic:**

```
function checkSerialCommand():
    while Serial.available() >= 3:              // Need at least 3 bytes
        header = Serial.read()

        if header != 0xBB:                      // Not a LED command
            continue                            // Discard, scan next byte

        led_index = Serial.read()               // Byte 0: LED index
        command = Serial.read()                 // Byte 1: command

        if led_index > 2:  continue             // Invalid index, discard
        if command > 2:    continue             // Invalid command, discard

        // Execute command
        if command == 0x00:                     // OFF
            led_state[led_index] = 0
        else if command == 0x01:                // ON
            led_state[led_index] = 1
        else if command == 0x02:                // TOGGLE
            led_state[led_index] = NOT led_state[led_index]

        digitalWrite(led_pins[led_index], led_state[led_index])
        last_led_index = led_index
```

**State machine:**

```
    ┌──────────────┐    byte != 0xBB    ┌──────────────┐
    │              │◄───────────────────│              │
    │ WAIT_HEADER  │                    │ WAIT_HEADER  │
    │              │────────────────────►│              │
    └──────┬───────┘    byte == 0xBB    └──────────────┘
           │
           ▼
    ┌──────────────┐
    │ READ_INDEX   │── read 1 byte → led_index
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │ READ_COMMAND │── read 1 byte → command
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │ VALIDATE     │── if valid → execute, else discard
    └──────┬───────┘
           │
           └──► back to WAIT_HEADER
```

**Edge cases:**
- Partial frame (only header received, then no more bytes): `Serial.available() >= 3` prevents reading incomplete frames.
- Garbage bytes before header: `continue` skips non-0xBB bytes.
- Invalid LED index (> 2): command is discarded silently.
- Invalid command value (> 2): command is discarded silently.

---

## 2. Host Python Application — Detailed Design

### 2.1 Module: Serial Interface

**Purpose:** Read TEMP_REPORT from Arduino, send LED_COMMAND to Arduino.

**Runs in:** Background thread (separate from Flask web server).

#### parse_temp_report()

**Input:** 7 raw bytes from Serial

**Output:** dictionary {temp, status, counter, led_state, last_led_idx}

```
function parse_temp_report(data):
    if data[0] != 0xAA:
        return None                             // Not a TEMP_REPORT

    temp_raw = data[1] | (data[2] << 8)        // Little-endian int16
    if temp_raw > 32767:                        // Handle signed int16
        temp_raw = temp_raw - 65536
    temp_celsius = temp_raw / 10.0

    status = data[3]                            // 0x00, 0x01, or 0x02
    counter = data[4]                           // 0–255
    led_bitmap = data[5]                        // 3 bits
    last_led_idx = data[6]                      // 0–2

    led_state = [
        (led_bitmap >> 0) & 1,                  // LED 0
        (led_bitmap >> 1) & 1,                  // LED 1
        (led_bitmap >> 2) & 1                   // LED 2
    ]

    return {
        temp: temp_celsius,
        status: status,
        counter: counter,
        led_state: led_state,
        last_led_idx: last_led_idx
    }
```

**Signed int16 handling:** Arduino sends temp_deci as int16. Values above 32767 in unsigned representation are negative temperatures. Example: -100 (deci-Celsius) is sent as 0xFF9C → unsigned 65436 → subtract 65536 → -100 → ÷10 = -10.0°C.

#### send_led_command()

**Input:** int led_index (0–2), int command (0=OFF, 1=ON, 2=TOGGLE)

**Output:** none (writes 3 bytes to Serial)

```
function send_led_command(led_index, command):
    frame = bytes([0xBB, led_index, command])
    serial_port.write(frame)
```

#### serial_read_loop()

**Runs continuously in background thread.**

```
function serial_read_loop():
    while running:
        byte = serial_port.read(1)

        if byte == 0xAA:
            data = 0xAA + serial_port.read(6)       // Read remaining 6 bytes
            result = parse_temp_report(data)
            if result != None:
                update_latest_data(result)
                check_counter_gap(result.counter)
                reset_comm_timeout_timer()
                socketio.emit('temp_update', result) // Push to browser
```

### 2.2 Module: COMM Timeout Detector

**Purpose:** Detect communication loss with ECU (REQ-C-02).

```
last_frame_time = now()

function reset_comm_timeout_timer():
    last_frame_time = now()

function check_comm_timeout():                  // Called every 100ms
    if now() - last_frame_time > 2000:          // 2 seconds
        socketio.emit('comm_timeout')
```

**Behavior:**
- Timer resets every time a valid TEMP_REPORT is received.
- If 2 seconds pass with no frame → emit 'comm_timeout' to browser.
- When communication resumes → timer resets → browser clears warning.

### 2.3 Module: Counter Gap Detector

**Purpose:** Detect missed frames using rolling counter (REQ-C-03).

```
expected_counter = None

function check_counter_gap(received_counter):
    if expected_counter == None:                // First frame
        expected_counter = (received_counter + 1) % 256
        return

    if received_counter != expected_counter:
        missed = (received_counter - expected_counter) % 256
        log("Counter gap: expected {expected_counter}, got {received_counter}, missed {missed} frames")
        socketio.emit('counter_gap', {
            expected: expected_counter,
            received: received_counter,
            missed: missed
        })

    expected_counter = (received_counter + 1) % 256
```

**Wrap-around handling:** Counter wraps 255 → 0. If expected=254 and received=1, missed = (1 - 254) % 256 = 3 frames missed.

### 2.4 Module: Latency Tracker

**Purpose:** Measure round-trip latency for LED commands (REQ-T-04).

```
pending_command = None

function on_led_button_click(led_index, command):
    pending_command = {
        led_index: led_index,
        t_send: now()
    }
    send_led_command(led_index, command)

function on_temp_report_received(result):
    if pending_command != None:
        expected_led = pending_command.led_index
        current_state = result.led_state[expected_led]

        // Check if LED state has changed since command was sent
        if state_changed(expected_led, current_state):
            latency = now() - pending_command.t_send
            socketio.emit('latency_update', {
                led: expected_led,
                latency_ms: latency
            })
            pending_command = None
```

**Note:** Latency includes time waiting for the next TEMP_REPORT cycle. Worst case: command sent just after a TEMP_REPORT → wait ~500ms for next one. Best case: command sent just before next TEMP_REPORT → latency ≈ UART transit time only.

### 2.5 Module: Flask Web Server

**Purpose:** Serve dashboard page and handle Socket.IO events.

```
app = Flask()
socketio = SocketIO(app)

@app.route('/')
function serve_dashboard():
    return render_template('dashboard.html')

@socketio.on('led_command')
function handle_led_command(data):
    led_index = data.index            // 0, 1, or 2
    command = data.action             // 0=OFF, 1=ON, 2=TOGGLE
    on_led_button_click(led_index, command)

socketio.run(app, host='0.0.0.0', port=5000)
```

**`host='0.0.0.0'`** makes the server accessible from any network interface — required for ngrok to forward traffic from the internet to localhost.

### 2.6 Module: Browser JavaScript (dashboard.html)

**Purpose:** Display data and capture user input.

```
socket = io.connect()

socket.on('temp_update', function(data):
    update temperature display with data.temp
    update status display with data.status
    update LED state indicators with data.led_state
    update counter display with data.counter
)

socket.on('latency_update', function(data):
    update latency display for data.led with data.latency_ms
)

socket.on('comm_timeout', function():
    show "COMM TIMEOUT" warning banner
)

socket.on('counter_gap', function(data):
    append to gap log: "frame {data.expected} → {data.received}"
)

function onLedButtonClick(led_index):
    socket.emit('led_command', {index: led_index, action: 2})    // TOGGLE
)
```

---

## 3. Requirement Traceability

| Requirement | Design Section | Function/Module |
|-------------|---------------|-----------------|
| REQ-F-01 | §1.4 readTemperature() | analogRead(A0) + conversion |
| REQ-F-02 | §1.6 sendTempReport() | Serial.write 7 bytes every 500ms |
| REQ-F-03 | §1.7 checkSerialCommand() | Parse 0xBB + 2 bytes from Serial |
| REQ-F-04 | §1.7 checkSerialCommand() | led_state[3], led_pins[3], digitalWrite |
| REQ-F-05 | §2.5, §2.6 | Flask serves dashboard, Socket.IO pushes temp |
| REQ-F-06 | §2.5, §2.4 | Socket.IO receives click, sends LED_COMMAND |
| REQ-F-07 | — | ngrok tunnel (no code change needed) |
| REQ-T-01 | §1.3 loop() | millis() check for 500ms interval |
| REQ-T-02 | §1.6 sendTempReport() Byte 5 | LED bitmap echoed in TEMP_REPORT |
| REQ-T-03 | §2.1 serial_read_loop() | Socket.IO push within 500ms of reception |
| REQ-T-04 | §2.4 Latency Tracker | t_ack - t_send displayed on dashboard |
| REQ-C-01 | §1.2 setup() | Serial.begin(115200) |
| REQ-C-02 | §2.2 COMM Timeout | 2s timeout → 'comm_timeout' event |
| REQ-C-03 | §2.3 Counter Gap | expected vs received counter check |

