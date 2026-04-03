# Remote ECU Monitoring & Control System

| Field          | Value                                      |
|----------------|--------------------------------------------|
| **Project**    | Remote ECU Monitoring & Control System      |
| **Author**     | *(your name)*                              |
| **Date**       | *(creation date)*                          |

---

## 1. Project Overview

This project implements a remote monitoring and control system for an embedded ECU (Electronic Control Unit). An OEM engineer can open a web browser from a remote location, observe live temperature readings from the ECU, independently control 3 LEDs, and measure round-trip command latency.

The system consists of two nodes:

- **Target ECU** — Arduino Uno R3 with LM35 temperature sensor and 3 LEDs, running C firmware via Arduino IDE.
- **Host PC** — Python application serving a web dashboard (Flask + Socket.IO) and communicating with the ECU over UART (USB serial).

Remote access is provided via ngrok, allowing the OEM engineer to reach the dashboard through a public URL without VPN or special software.

---

## 2. System Architecture

```
┌─────────────────────────┐   UART (USB)   ┌─────────────────────────┐
│  HOST PC                │◄─────────────►│  TARGET ECU              │
│  (Supplier site)        │                │  (Arduino Uno R3)        │
│                         │                │                          │
│  • Python (Flask)       │                │  • LM35 → ADC (A0)      │
│  • Web dashboard        │                │  • 3 LEDs (GPIO)         │
│  • Socket.IO (realtime) │                │  • UART TX/RX            │
│  • pyserial (bus I/F)   │                │  • Command handler       │
└─────────────────────────┘                └──────────────────────────┘
         │
         │  ngrok tunnel
         ▼
  OEM Engineer (remote)
  Browser → live temperature, 3 LED controls, round-trip latency
```

---

## 3. Communication Protocol

- **Bus:** UART at 115200 baud via USB (REQ-C-01)
- **ECU → Host:** TEMP_REPORT — 6 bytes, header 0xAA, sent every 500ms
- **Host → ECU:** LED_COMMAND — 2 bytes, header 0xBB, sent on-demand

Full protocol specification: see `02-SWE2-architecture/frame-layout.md`

---

## 4. Hardware Requirements

| Component | Quantity | Purpose |
|-----------|----------|---------|
| Arduino Uno R3 (CH340) | 1 | Target ECU |
| LM35DZ Temperature Sensor | 1 | Analog temperature reading via ADC |
| LED (Red/Green/Yellow) | 3 | Independently controlled outputs (REQ-F-04) |
| 220Ω Resistor | 3 | Current limiting for LEDs |
| Breadboard | 1 | Solderless prototyping |
| Male-Male Jumper Wires | ~10 | Wiring connections |
| USB Type B Cable | 1 | Arduino to PC (included with board) |

Full BOM with justification: see `bom.md`

Pin assignments: see `02-SWE2-architecture/pin-assignment.md`

---

## 5. Development Environment

| Item | Choice | Why |
|------|--------|-----|
| **OS** | Windows | Familiar environment, easy to set up all tools, compatible with Arduino IDE and Python without additional configuration. |
| **Firmware IDE** | Arduino IDE | Simple and well-supported IDE for Arduino Uno. Built-in serial monitor useful for debugging UART communication during development. |
| **Firmware Language** | C (Arduino) | Native language for Arduino platform, direct access to hardware registers (ADC, GPIO, UART) with minimal abstraction. |
| **Host Language** | Python | Fast development for web dashboard and serial communication. Rich ecosystem of libraries (Flask, Socket.IO, pyserial). |
| **Web Framework** | Flask + Socket.IO | Flask is lightweight and easy to set up. Socket.IO enables real-time bidirectional communication between dashboard and server — required for live temperature updates every 500ms. |
| **Serial Library** | pyserial | Standard Python library for UART communication with Arduino over USB. |
| **Remote Access** | ngrok (free tier) | Generates public URL instantly, no public IP or router configuration needed. OEM only needs a browser. |
| **USB Driver** | CH340 | Required for Arduino Uno clone boards with CH340 USB-to-Serial chip. |

---

## 6. How to Build

### 6.1 Firmware (Arduino)

1. Open Arduino IDE.
2. Open `04-software/firmware/firmware.ino`.
3. Select board: **Tools → Board → Arduino Uno**.
4. Select port: **Tools → Port → COMx** (Windows) or **/dev/ttyUSBx** (Linux).
5. Click **Upload**.

### 6.2 Host Application (Python)

No build step required. Python scripts run directly.

---

## 7. How to Run

### 7.1 Hardware Setup

1. Wire the circuit according to `02-SWE2-architecture/pin-assignment.md`.
2. Connect Arduino to PC via USB cable.
3. Verify the serial port is recognized:
   - Windows: check Device Manager → Ports (COMx)
   - Linux: `ls /dev/ttyUSB*`

### 7.2 Run Dashboard (Local)

```bash
cd 04-software/host/
python app.py --port /dev/ttyUSB0
```

Open browser: `http://localhost:5000`

The dashboard should display:
- Live temperature reading (updated every 500ms)
- 3 LED toggle buttons (LED 0, LED 1, LED 2)
- Round-trip latency for each LED command
- COMM TIMEOUT warning if ECU disconnects

### 7.3 Run Simulation (No Hardware)

For SIL testing without Arduino hardware:

**Terminal 1 — Start simulated ECU:**
```bash
cd 04-software/simulation/
python sim_ecu.py
```

**Terminal 2 — Start dashboard against simulation:**
```bash
cd 04-software/host/
python app.py --sim
```

Open browser: `http://localhost:5000` — dashboard works identically to hardware mode.

### 7.4 Remote Access (OEM Verification)

1. Start the dashboard (Section 7.2).
2. In a separate terminal:

```bash
ngrok http 5000
```

3. ngrok outputs a public URL (e.g., `https://abc123.ngrok-free.app`).
4. Send this URL to the OEM engineer.
5. OEM opens the URL in their browser — they can see temperature and control LEDs remotely.

---

## 8. Test Execution

### MIL (Unit Tests — No Hardware)

```bash
cd 05-SWE4-unit-test/
pytest test_mil.py -v
```

### SIL (Integration Tests — Simulated Bus)

```bash
cd 06-SWE5-integration-test/
pytest test_sil.py -v
```

### HIL (Hardware Tests)

Follow test procedure in `07-SWE6-qualification-test/SWE6_QualificationTest_Spec.md`.
Results recorded in `07-SWE6-qualification-test/test_hil_report.md`.

---

## 9. Project Structure

```
/trial-project
├── README.md
├── bom.md
├── project-plan.md
│
├── 01-SWE1-requirements/
│   └── SWE1_SW_Requirements.md
│
├── 02-SWE2-architecture/
│   ├── SWE2_SW_Architecture.md
│   ├── system-diagram.png
│   ├── frame-layout.md
│   └── pin-assignment.md
│
├── 03-SWE3-design/
│   ├── SWE3_SW_DetailedDesign.md
│   ├── ntc-datasheet.pdf
│   ├── conversion-formula.md
│   └── dashboard-wireframe.md
│
├── 04-software/
│   ├── firmware/
│   ├── host/
│   └── simulation/
│
├── 05-SWE4-unit-test/
│   ├── SWE4_UnitTest_Spec.md
│   ├── test_mil.py
│   └── mil_results.txt
│
├── 06-SWE5-integration-test/
│   ├── SWE5_IntegrationTest_Spec.md
│   ├── test_sil.py
│   ├── sil_results.txt
│   └── sil_dashboard_screenshot.png
│
├── 07-SWE6-qualification-test/
│   ├── SWE6_QualificationTest_Spec.md
│   ├── test_hil_report.md
│   ├── evidence/
│   └── timing_measurements.csv
│
├── 08-traceability/
│   └── Traceability_Matrix.md
│
└── lessons-learned.md
```

---

## 10. Key Documents

| Document | Location | Purpose |
|----------|----------|---------|
| BOM | `bom.md` | Component list with justification |
| Project Plan | `project-plan.md` | 7-day development schedule |
| Requirements | `01-SWE1-requirements/` | Software requirements (ASPICE SWE.1) |
| Architecture | `02-SWE2-architecture/` | System design, frame layout, pin map (SWE.2) |
| Detailed Design | `03-SWE3-design/` | Conversion formula, wireframe (SWE.3) |
| Traceability Matrix | `08-traceability/` | REQ → Design → Code → Test → Result |
| Lessons Learned | `lessons-learned.md` | Deviations and retrospective |

