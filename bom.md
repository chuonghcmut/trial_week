# Bill of Materials (BOM)

| Field          | Value                                      |
|----------------|--------------------------------------------|
| **Document**   | SDT-2026-001                               |
| **Project**    | Remote ECU Monitoring & Control System     |
| **Author**     | *(your name)*                              |
| **Date**       | *(creation date)*                          |
| **Revision**   | A                                          |
| **Budget**     | < 30 USD (~750,000 VND)                    |

---

## 1. Purchased Components

| # | Part Name | Qty | Unit Price (VND) | Vendor | Link | Justification |
|---|-----------|-----|------------------|--------|------|---------------|
| 1 | Arduino Uno R3 (SMD/CH340) | 1 | 164,000 | Shopee | *(paste order link)* | General-purpose MCU board with built-in UART-over-USB (CH340), sufficient GPIO for 3 LEDs + 1 ADC channel for LM35. Large community and documentation base — enables rapid prototyping within 7-day timeline. USB type B cable included. |
| 2 | LM35DZ Temperature Sensor | 1 | 16,000 | Shopee | *(paste order link)* | Analog temperature sensor with linear output (10mV/°C), read directly via ADC — satisfies REQ-F-01. Advantages over NTC: linear output eliminates the need for voltage divider circuit and Steinhart-Hart conversion formula, ±0.5°C accuracy out of the box. Advantages over DHT11: ADC-based interface matches requirement specification, faster read speed (no 1–2s minimum interval between readings). |

**Total purchased: 180,000 VND (~7.2 USD)**

---

## 2. Available Components (Already Owned)

| # | Part Name | Qty | Notes |
|---|-----------|-----|-------|
| 3 | LED (Red/Green/Yellow) | 3 | Three independent LED channels (REQ-F-04). Different colors for easy visual identification during testing. |
| 4 | 220Ω Resistor | 3 | Current limiting for LEDs. Calculation: (5V - 2V) / 220Ω ≈ 13.6mA — safe for Arduino GPIO (max 20mA per pin). |
| 5 | Breadboard | 1 | Solderless prototyping board. |
| 6 | Male-Male Jumper Wires | ~10 | Connections between Arduino, LEDs, and LM35 on breadboard. |

---

## 3. Software & Tools (Free)

| # | Tool | Version | Purpose | Cost |
|---|------|---------|---------|------|
| 7 | Arduino IDE | 2.x | Firmware development and upload for Arduino Uno | Free |
| 8 | Python | 3.10+ | Host application: web dashboard + serial bus interface | Free |
| 9 | ngrok | Free tier | Remote access — generates public URL for OEM to access dashboard remotely. Selected because: setup takes <2 minutes, no public IP required, no router configuration needed, OEM only needs a browser. | Free |
| 10 | pyserial | latest | Python library for UART communication with Arduino over USB | Free |
| 11 | CH340 Driver | latest | USB-to-Serial driver for Arduino Uno clone (CH340 chip) | Free |

---

## 4. Design Decisions Summary

### Why Arduino Uno over ESP32 or STM32?

Arduino Uno is easy to debug, easy to develop with, and well-suited for a 7-day development project. It meets all technical requirements (ADC for LM35, GPIO for 3 LEDs, UART over USB) without unnecessary complexity. ESP32's built-in WiFi is not needed since the Host PC handles web serving and remote access. STM32 requires an additional debugger (ST-Link) and a more complex toolchain. Arduino Uno lets the team focus on process and documentation rather than toolchain issues.

### Why UART instead of CAN?

UART is chosen because the developer is already familiar with UART communication, and the 7-day timeline requires fast development with minimal hardware risk. Arduino Uno has built-in UART over USB — no additional transceiver module needed. CAN would require an external MCP2515 module, adding cost and wiring complexity. UART at 115200 baud provides sufficient bandwidth for the two message types (TEMP_REPORT 6 bytes @ 500ms + LED_COMMAND 2 bytes on-demand). This choice is permitted by REQ-C-01.

### Why ngrok instead of SSH tunnel or VPN?

ngrok does not require a public IP address (most Vietnamese ISPs use CGNAT, making public IP unavailable). It does not require OEM to install any software — only a browser is needed. Setup takes under 2 minutes. SSH tunnel requires a public IP or port forwarding on the router. VPN solutions (Tailscale/ZeroTier) require OEM to install a client application, which may be blocked by corporate IT policy. Given the 7-day timeline, ngrok presents the lowest risk for Day 6 remote access delivery.

---

## 5. Order Confirmation

*(Paste Shopee order screenshot here, including order date and estimated delivery date)*

**Order date:** *(fill in)*

**Expected delivery:** *(fill in)*

