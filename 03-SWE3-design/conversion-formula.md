# ADC to Temperature Conversion Formula

| Field          | Value                                      |
|----------------|--------------------------------------------|
| **Document**   | SDT-2026-001                               |
| **Project**    | Remote ECU Monitoring & Control System      |
| **Author**     | *(your name)*                              |
| **Date**       | *(creation date)*                          |
| **Revision**   | A                                          |
| **Sensor**     | LM35DZ                                     |

---

## 1. LM35 Sensor Characteristics

| Parameter | Value | Source |
|-----------|-------|--------|
| Output type | Analog voltage, linear | LM35 datasheet |
| Sensitivity | 10 mV/°C | LM35 datasheet |
| Output at 0°C | 0 mV | LM35 datasheet |
| Output at 25°C | 250 mV | LM35 datasheet |
| Output at 100°C | 1000 mV | LM35 datasheet |
| Accuracy | ±0.5°C (at 25°C) | LM35 datasheet |
| Operating range | -55°C to +150°C | LM35 datasheet |
| Supply voltage | 4V to 30V | LM35 datasheet |

Datasheet reference: Texas Instruments LM35 — https://www.ti.com/lit/ds/symlink/lm35.pdf

---

## 2. Arduino ADC Characteristics

| Parameter | Value |
|-----------|-------|
| Resolution | 10-bit (0–1023) |
| Reference voltage | 5.0V (default, from USB) |
| Input range | 0V to 5.0V |
| Voltage per step | 5.0V / 1023 ≈ 4.888 mV/step |

---

## 3. Conversion Chain

The conversion from ADC raw value to temperature goes through 3 steps:

```
ADC raw (0–1023)  →  Voltage (mV)  →  Temperature (°C)  →  Deci-Celsius (int16)
```

### Step 1: ADC raw → Voltage

```
voltage_mV = adc_raw × 5000 / 1023
```

The ADC maps 0–5V to 0–1023. Multiply by 5000 (mV) and divide by 1023 to get millivolts.

### Step 2: Voltage → Temperature

```
temp_C = voltage_mV / 10
```

LM35 outputs 10 mV per °C, so dividing millivolts by 10 gives degrees Celsius.

### Step 3: Temperature → Deci-Celsius

```
temp_deci = temp_C × 10
```

Multiply by 10 to get deci-Celsius (0.1°C resolution) as an integer. This avoids floating-point on the bus.

### Combined Formula

```
temp_deci = adc_raw × 5000 / 1023 / 10 × 10
          = adc_raw × 500 / 1023
```

The ×10 and ÷10 cancel out, giving a simple formula:

```
temp_deci = adc_raw × 500 / 1023
```

---

## 4. Worked Examples

| Actual Temp | LM35 Output | ADC Raw | Calculation | temp_deci | Decoded on PC |
|-------------|-------------|---------|-------------|-----------|---------------|
| 0.0°C | 0 mV | 0 | 0 × 500 / 1023 = 0 | 0 | 0.0°C |
| 20.0°C | 200 mV | 41 | 41 × 500 / 1023 = 20 | 20 | 2.0°C... |
| 25.0°C | 250 mV | 51 | 51 × 500 / 1023 = 24.9 → 24 | 24 | 2.4°C... |
| 25.0°C | 250 mV | 52 | 52 × 500 / 1023 = 25.4 → 25 | 25 | 2.5°C... |

**Note on precision:** Because integer division truncates, some values have a small error. For example, 25.0°C produces ADC ≈ 51.15, which rounds to 51. Then 51 × 500 / 1023 = 24.926 → truncates to 24 (in integer math) = 2.4°C. This ±0.1°C error is acceptable — the LM35 itself has ±0.5°C accuracy.

**Improved formula for Arduino (integer math with better rounding):**

```
temp_deci = (adc_raw * 500L + 511) / 1023
```

Adding 511 (half of 1023) before dividing rounds to the nearest integer instead of truncating. This reduces the systematic -0.1°C bias.

| Actual Temp | ADC Raw | Without rounding | With rounding |
|-------------|---------|-----------------|---------------|
| 25.0°C | 51 | 24 (24.9 truncated) | 25 (24.9 rounded) |
| 30.0°C | 61 | 29 (29.8 truncated) | 30 (29.8 rounded) |
| 50.0°C | 102 | 49 (49.8 truncated) | 50 (49.9 rounded) |

---

## 5. Moving Average Filter

### Why filtering is needed

Arduino ADC has noise of ±1–2 LSB. At the LM35 sensitivity, 1 LSB ≈ 0.49°C. Without filtering, the temperature display would jump between values like 24.5°C and 25.5°C even when the actual temperature is stable.

### Filter design

4-sample moving average — the simplest filter that provides adequate noise reduction.

```
buffer[4] = {0, 0, 0, 0}
write_index = 0

function filter(new_sample):
    buffer[write_index] = new_sample
    write_index = (write_index + 1) % 4

    sum = buffer[0] + buffer[1] + buffer[2] + buffer[3]
    return sum / 4
```

### Filter characteristics

| Property | Value |
|----------|-------|
| Window size | 4 samples |
| Sample period | 500 ms |
| Settling time | 4 × 500ms = 2 seconds |
| Noise reduction | ±1 LSB → ±0.25 LSB (noise reduced by half) |
| Latency added | 1.5 samples average = 750 ms |

### Startup behavior

At power-on, the buffer contains all zeros. The first 4 readings will be averaged with zeros, producing lower-than-actual values:

```
Reading 1: buffer = [25, 0, 0, 0] → output = 6   (too low)
Reading 2: buffer = [25, 25, 0, 0] → output = 12  (too low)
Reading 3: buffer = [25, 25, 25, 0] → output = 18  (too low)
Reading 4: buffer = [25, 25, 25, 25] → output = 25  (correct)
```

This ramp-up takes 2 seconds. This is acceptable — the dashboard will show a brief ramp after power-on. No special handling needed.

### Example with noisy input

```
Actual temperature: 25.0°C (temp_deci = 250)
ADC readings (with noise): 249, 253, 247, 251

Filter output:
  Step 1: buffer = [249, 0, 0, 0]       → 249/4 = 62   (startup)
  Step 2: buffer = [249, 253, 0, 0]     → 502/4 = 125  (startup)
  Step 3: buffer = [249, 253, 247, 0]   → 749/4 = 187  (startup)
  Step 4: buffer = [249, 253, 247, 251] → 1000/4 = 250 ✓ (stable)
  Step 5: buffer = [249, 253, 247, 251] → 1000/4 = 250 ✓
```

After startup, the filter smooths out ±2 unit noise to a stable output.

---

## 6. Status Byte Logic

After computing filtered temp_deci, the firmware determines the status byte:

```
function getStatus(temp_deci):
    if temp_deci > 600:    return 0x01    // Over-temp  (> 60.0°C)
    if temp_deci < -100:   return 0x02    // Under-temp (< -10.0°C)
    return 0x00                            // OK
```

| Condition | temp_deci | Status | Meaning |
|-----------|-----------|--------|---------|
| Normal | -100 to 600 | 0x00 | OK |
| Over-temperature | > 600 | 0x01 | Fault — sensor reads above 60.0°C |
| Under-temperature | < -100 | 0x02 | Fault — sensor reads below -10.0°C |

Note: boundaries are inclusive. 60.0°C (600) = OK. 60.1°C (601) = Over-temp.

---

## 7. Requirement Mapping

| Requirement | Addressed in this document |
|-------------|---------------------------|
| REQ-F-01 | §3 Conversion chain — ADC reads LM35 via A0 |
| REQ-T-01 | §5 Filter — 4-sample moving average at 500ms intervals |

