# 1. Introduction
This document defines the software requirements for the ECU temperature monitoring and LED control system.

# 2. Definitions
ECU: Electronic Control Unit  
ADC: Analog-to-Digital Converter  
HMI: Human-Machine Interface  

# 3.1 Functional Requirements
| ID       | Requirement                                                       | Type       | Verification  |
|----------|-------------------------------------------------------------------|------------|---------------|
| REQ-F-01 | The ECU shall read onboard temperature via ADC.                   | Functional | Test          |
| REQ-F-02 | The ECU shall transmit temperature on the communication bus.      | Functional | Test          |
| REQ-F-03 | The ECU shall receive LED control commands.                       | Functional | Test          |
| REQ-F-04 | The ECU shall control 3 LEDs independently by index (0–2).        | Functional | Test          |
| REQ-F-05 | The Host PC shall display current temperature on a web dashboard. | Functional | Demonstration |
| REQ-F-06 | The Host PC shall transmit LED commands on user interaction.      | Functional | Demonstration |
| REQ-F-07 | The web dashboard shall be accessible remotely.                   | Functional | Demonstration |
|----------|-------------------------------------------------------------------|------------|---------------|

# 3.2 Timing Requirements
| ID       | Requirement                                                | Type   | Verification |
|----------|------------------------------------------------------------|--------|--------------|
| REQ-T-01 | The ECU shall transmit temperature frames every 500ms ±5%. | Timing | Test         |
| REQ-T-02 | The ECU shall acknowledge LED commands within 50ms.        | Timing | Test         |
| REQ-T-03 | The dashboard shall display data no older than 1000ms.     | Timing | Test         |
| REQ-T-04 | The dashboard shall display round-trip latency.            | Timing | Inspection   |
|----------|------------------------------------------------------------|--------|--------------|

