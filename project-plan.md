# Project Plan

| Field          | Value                                      |
|----------------|--------------------------------------------|
| **Document**   | SDT-2026-001                               |
| **Project**    | Remote ECU Monitoring & Control System      |
| **Author**     | *(your name)*                              |
| **Date**       | *(creation date)*                          |
| **Revision**   | A                                          |
| **Duration**   | 7 Calendar Days                            |

---

## 1. Timeline Overview

| Day | Focus | Key Deliverables | V-Model Phase |
|-----|-------|-----------------|---------------|
| 1 | Procurement & Project Planning | BOM, frame layout, pin assignment, project plan | — |
| 2 | Requirements Analysis & Architecture | SWE1, SWE2 documents, traceability started | Left side (design) |
| 3 | Software Development & Simulation | Firmware, dashboard, MIL tests, SIL tests | Bottom (implementation) |
| 4–5 | Hardware Integration & Verification | Wiring, bring-up, end-to-end local test, timing verification | Right side (verify) |
| 6 | Remote Access & OEM Verification | ngrok setup, OEM clicks LED from Germany | Right side (verify) |
| 7 | Documentation & Delivery | Test report, traceability matrix, lessons learned, final PR | — |

---

## 2. Detailed Daily Plan

### Day 1 — Procurement & Project Planning

| Task | Deliverable | Location |
|------|-------------|----------|
| Select and order hardware components | `bom.md` with vendor links, prices, justification | `/bom.md` |
| Screenshot of Shopee order confirmation | Embedded in `bom.md` Section 5 | `/bom.md` |
| Design communication protocol | `frame-layout.md` — TEMP_REPORT and LED_COMMAND byte layout | `/02-SWE2-architecture/` |
| Assign Arduino pins | `pin-assignment.md` — every pin with purpose and rationale | `/02-SWE2-architecture/` |
| Document LM35 circuit and ADC-to-temperature formula | `conversion-formula.md` | `/03-SWE3-design/` |
| Write project plan | `project-plan.md` | `/project-plan.md` |
| Document development environment | Included in `README.md` Section 5 | `/README.md` |

**Gate:** Frame layout and pin assignment documents must exist before any code is written.

---

### Day 2 — Requirements Analysis & Architecture

| Task | Deliverable | Location |
|------|-------------|----------|
| Expand requirements from Section 3 into supplier format | `SWE1_SW_Requirements.md` | `/01-SWE1-requirements/` |
| Create architecture document with block diagrams | `SWE2_SW_Architecture.md` | `/02-SWE2-architecture/` |
| Draw system diagram (system level + ECU software + host software) | `system-diagram.png` or ASCII | `/02-SWE2-architecture/` |
| Start traceability matrix: REQ-ID → architecture section | `Traceability_Matrix.md` (partial) | `/08-traceability/` |
| Document LM35 datasheet reference and conversion parameters | `conversion-formula.md` updated | `/03-SWE3-design/` |

---

### Day 3 — Software Development & Simulation

| Task | Deliverable | Location |
|------|-------------|----------|
| Write Arduino firmware (ADC read, frame packing, UART TX/RX, LED control) | `firmware.ino` | `/04-software/firmware/` |
| Write Python host application (Flask + Socket.IO dashboard, serial interface) | `app.py` + templates | `/04-software/host/` |
| Write simulated ECU script | `sim_ecu.py` | `/04-software/simulation/` |
| MIL unit tests: ADC-to-temperature, frame packing, filter, LED command parse | `test_mil.py` + `mil_results.txt` | `/05-SWE4-unit-test/` |
| SIL integration test: dashboard running against sim_ecu on virtual bus | `test_sil.py` + `sil_results.txt` | `/06-SWE5-integration-test/` |
| Screenshot of dashboard working against simulation | `sil_dashboard_screenshot.png` | `/06-SWE5-integration-test/` |

**Milestone:** Full system running in simulation before hardware arrives. All logic verified. Hardware day becomes integration day, not debug day.

---

### Day 4–5 — Hardware Integration & Verification

| Task | Deliverable | Location |
|------|-------------|----------|
| Wire circuit according to pin-assignment.md | Photo of wiring matching document | `/07-SWE6-qualification-test/evidence/` |
| ECU bring-up: LED blinks on power, ADC reads plausible room temp, UART TX visible on PC | Bring-up log | `/07-SWE6-qualification-test/` |
| End-to-end local test: dashboard displays real temperature, LED toggles on command | Video/photo evidence | `/07-SWE6-qualification-test/evidence/` |
| Timing verification — REQ-T-01: measure 20 consecutive TX intervals (min/max/avg) | `timing_measurements.csv` | `/07-SWE6-qualification-test/` |
| Timing verification — REQ-T-02: measure 10 LED command round-trips | `timing_measurements.csv` | `/07-SWE6-qualification-test/` |
| Timing verification — REQ-T-03: verify dashboard staleness < 1000ms | Recorded in test report | `/07-SWE6-qualification-test/` |
| Timing verification — REQ-T-04: confirm latency displayed on dashboard | Screenshot | `/07-SWE6-qualification-test/evidence/` |
| COMM TIMEOUT test — REQ-C-02: disconnect USB, verify timeout within 2.5s | Screenshot | `/07-SWE6-qualification-test/evidence/` |
| Counter gap test — REQ-C-03: reconnect, verify gap logged | Screenshot | `/07-SWE6-qualification-test/evidence/` |

---

### Day 6 — Remote Access & OEM Verification

| Task | Deliverable | Location |
|------|-------------|----------|
| Configure ngrok for remote access | Document setup steps in README Section 7.4 | `/README.md` |
| Send public URL to OEM engineer | URL shared via agreed channel | — |
| OEM verification: OEM opens dashboard remotely | — | — |
| OEM reads live temperature value | Evidence screenshot | `/07-SWE6-qualification-test/evidence/` |
| OEM clicks LED button → LED toggles on ECU in Vietnam | Evidence photo/video | `/07-SWE6-qualification-test/evidence/` |
| OEM observes round-trip latency on dashboard | Latency value recorded | `/07-SWE6-qualification-test/` |

**Acceptance gate:** OEM clicks a button in Germany. LED toggles in Vietnam. Latency is displayed.

---

### Day 7 — Documentation & Delivery

| Task | Deliverable | Location |
|------|-------------|----------|
| Complete test report with all measured values | `test_hil_report.md` | `/07-SWE6-qualification-test/` |
| Complete traceability matrix — every REQ linked to design, code, test, result | `Traceability_Matrix.md` | `/08-traceability/` |
| Write lessons learned document | `lessons-learned.md` | `/lessons-learned.md` |
| Verify folder structure matches Section 8 | All directories and files present | `/trial-project/` |
| Final review of all documents | — | — |
| Submit pull request | PR with organized deliverables | GitHub |

---

## 3. Dependencies & Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Hardware delivery delayed beyond Day 3 | Cannot start HIL testing on Day 4 | Use Day 1–3 for complete MIL/SIL verification so all logic is proven before hardware arrives |
| LM35 reads noisy ADC values | Temperature display unstable | Implement 4-sample moving average filter, tested at MIL level |
| ngrok free tier URL changes on restart | OEM loses access | Keep ngrok session running, have backup URL ready, document restart procedure |
| CH340 driver not recognized on PC | Cannot communicate with Arduino | Download and install CH340 driver before hardware arrives (Day 1) |
| UART frame sync lost | Dashboard shows corrupted data | Implement header byte (0xAA/0xBB) validation and frame length check in parser |

---

## 4. Daily Progress Tracking

Progress will be reported daily via GitHub issue comments using this template:

```
Date: ___
Completed today: ___
Blocked on: ___
Plan for tomorrow: ___
Evidence: [screenshot / photo / link]
```

