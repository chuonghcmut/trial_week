# Dashboard Wireframe

| Field          | Value                                      |
|----------------|--------------------------------------------|
| **Document**   | SDT-2026-001                               |
| **Project**    | Remote ECU Monitoring & Control System     |
| **Author**     | Wilson Nguyen                              |

---

## 1. Overview

The dashboard is a single-page web application served by Flask on port 5000. It uses Socket.IO for realtime communication with the Python backend. The interface is designed to be simple and functional — all critical information visible at a glance without scrolling.

---

## 2. Layout Wireframe

```
┌──────────────────────────────────────────────────────────────┐
│                  ECU Remote Monitoring Dashboard              │
│                  ─────────────────────────────                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌────────────────────────────────────────────────────┐     │
│   │              TEMPERATURE SECTION                    │     │
│   │                                                    │     │
│   │          🌡  25.1 °C                               │     │
│   │                                                    │     │
│   │          Status: OK                                │     │
│   │          (changes to "⚠ OVER-TEMP" or              │     │
│   │           "⚠ UNDER-TEMP" on fault)                 │     │
│   └────────────────────────────────────────────────────┘     │
│                                                              │
│   ┌────────────────────────────────────────────────────┐     │
│   │              LED CONTROL SECTION                    │     │
│   │                                                    │     │
│   │   LED 0 (Red)     [ TOGGLE ]    State: ON          │     │
│   │                                 Latency: 23 ms     │     │
│   │                                                    │     │
│   │   LED 1 (Green)   [ TOGGLE ]    State: OFF         │     │
│   │                                 Latency: --        │     │
│   │                                                    │     │
│   │   LED 2 (Yellow)  [ TOGGLE ]    State: OFF         │     │
│   │                                 Latency: --        │     │
│   └────────────────────────────────────────────────────┘     │
│                                                              │
│   ┌────────────────────────────────────────────────────┐     │
│   │              COMMUNICATION STATUS                   │     │
│   │                                                    │     │
│   │   Message Counter: 142                             │     │
│   │   Counter Gaps: frame 87 → 90 (missed 2)          │     │
│   │                                                    │     │
│   │   ┌──────────────────────────────────────────┐     │     │
│   │   │         ⚠ COMM TIMEOUT                   │     │     │
│   │   │   No data received for 2 seconds         │     │     │
│   │   └──────────────────────────────────────────┘     │     │
│   │   (hidden when communication is normal)            │     │
│   └────────────────────────────────────────────────────┘     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Section Details

### 3.1 Temperature Section

| Element | Type | Data Source | Update Rate |
|---------|------|------------|-------------|
| Temperature value | Large text | TEMP_REPORT Byte 0–1 (deci-Celsius ÷ 10) | Every 500ms via Socket.IO |
| Status indicator | Text + color | TEMP_REPORT Byte 2 | Every 500ms via Socket.IO |

Status display rules:

| Byte 2 Value | Display | Color |
|-------------|---------|-------|
| 0x00 | "OK" | Green |
| 0x01 | "⚠ OVER-TEMP" | Red |
| 0x02 | "⚠ UNDER-TEMP" | Blue |

### 3.2 LED Control Section

Each LED has one row with 4 elements:

| Element | Type | Behavior |
|---------|------|----------|
| LED label | Static text | "LED 0 (Red)", "LED 1 (Green)", "LED 2 (Yellow)" |
| TOGGLE button | Clickable button | On click → sends LED_COMMAND with command=0x02 (TOGGLE) |
| State indicator | Dynamic text | Shows "ON" or "OFF" based on TEMP_REPORT Byte 4 bitmap |
| Latency display | Dynamic text | Shows round-trip latency in ms after button click, "--" when no command sent yet |

Latency measurement flow:
1. User clicks TOGGLE → Python records `t_send`
2. Python sends `0xBB [index] 0x02` to Arduino
3. Python monitors TEMP_REPORT until LED bitmap changes
4. `latency = t_ack - t_send` → pushed to browser

### 3.3 Communication Status Section

| Element | Type | Data Source |
|---------|------|------------|
| Message counter | Dynamic text | TEMP_REPORT Byte 3 (rolling 0–255) |
| Counter gap log | Dynamic list | Detected when received counter ≠ expected counter |
| COMM TIMEOUT banner | Conditional banner | Shown when no TEMP_REPORT received for 2 seconds (REQ-C-02) |

COMM TIMEOUT banner is **hidden by default** and only appears when communication is lost. It disappears automatically when communication resumes.

---

## 4. Interaction Flow

```
User opens browser → http://localhost:5000
    │
    ▼
Flask serves HTML/JS/CSS
    │
    ▼
Browser establishes Socket.IO connection
    │
    ▼
Every 500ms:
    Python pushes 'temp_update' event
    → Browser updates: temperature, status, LED states, counter
    │
User clicks TOGGLE button for LED 1:
    │
    ▼
Browser emits 'led_command' event {index: 1, action: 2}
    │
    ▼
Python sends 0xBB 0x01 0x02 to Arduino
    │
    ▼
Arduino toggles LED 1, echoes in next TEMP_REPORT
    │
    ▼
Python detects LED bitmap change, calculates latency
    │
    ▼
Browser updates: LED 1 state = ON, latency = 23ms
```

---

## 5. Requirement Mapping

| Dashboard Element | Requirement |
|------------------|-------------|
| Temperature display | REQ-F-05 |
| LED TOGGLE buttons | REQ-F-06 |
| LED state indicators | REQ-F-04 |
| Round-trip latency display | REQ-T-04 |
| COMM TIMEOUT banner | REQ-C-02 |
| Counter gap log | REQ-C-03 |
| Accessible via ngrok URL | REQ-F-07 |

