# CamLab Motor Control System - RPM Implementation Guide

## Overview

The CamLab motor control system has been updated to work with RPM (Revolutions Per Minute) units instead of mm/s. This document explains how the motor control works when you press the plus (+) or minus (-) buttons.

## Motor Control Architecture

### 1. User Interface Layer (`JogGroupBox.py`)
- **Speed Input Field**: Now displays "Speed (RPM)" with current RPM value
- **Plus Button (+)**: When pressed, starts motor rotation in positive direction
- **Minus Button (-)**: When pressed, starts motor rotation in negative direction
- **Button Release**: Stops motor rotation

### 2. Signal Flow
```
UI Button Press → Qt Signal → Device Method → Hardware Command

Plus Button:
Press   → positiveJogEnabled  → jog_positive_on_C1()  → PWM + Direction
Release → positiveJogDisabled → jog_positive_off_C1() → Stop PWM

Minus Button:
Press   → negativeJogEnabled  → jog_negative_on_C1()  → PWM + Direction  
Release → negativeJogDisabled → jog_negative_off_C1() → Stop PWM

Speed Change:
Input → secondarySetPointChanged → set_speed_C1() → Set PWM Frequency
```

### 3. RPM to Frequency Conversion
The motor control system converts RPM to PWM frequency using:

```python
# Convert RPM to frequency for hardware
frequency_hz = (speed_rpm * CPR) / 60

# Where:
# CPR = 6400 (Counts Per Revolution for the encoder)
# 60 = seconds per minute

# Example: 100 RPM = (100 * 6400) / 60 = 10,667 Hz
```

### 4. Hardware Interface (`device.py`)
- **LabJack T7**: Controls stepper motor drivers via PWM
- **PWM Frequency**: Determines motor speed (higher frequency = higher RPM)
- **Direction Control**: Digital pins EIO1 (C1) and EIO3 (C2)
- **Motor Enable**: Digital pins EIO0 (C1) and EIO2 (C2)

## How to Use Motor Control in RPM

### Step 1: Set Desired Speed
1. Look for the "Speed (RPM)" field in the motor control interface
2. Enter desired RPM value (e.g., 100, 500, 1000)
3. Valid range: 0.1 to 4000 RPM
4. Press Enter or click away to apply the speed setting

### Step 2: Control Motor Direction
1. **Forward Rotation**: Press and hold the **Plus (+)** button
   - Motor starts spinning at the set RPM in positive direction
   - Release button to stop

2. **Reverse Rotation**: Press and hold the **Minus (-)** button
   - Motor starts spinning at the set RPM in negative direction
   - Release button to stop

### Step 3: Safety Features
- **Speed Limits**: Maximum 4000 RPM (hardware limit)
- **Position Limits**: Software limits prevent over-travel
- **Hard Limits**: Physical limit switches stop motor at boundaries
- **Emergency Stop**: Stop button immediately halts all motion

## Examples of RPM Values

| RPM   | Application           | Frequency (Hz) |
|-------|-----------------------|----------------|
| 50    | Slow positioning      | 5,333          |
| 100   | Normal operation      | 10,667         |
| 500   | Fast positioning      | 53,333         |
| 1000  | High speed operation  | 106,667        |
| 4000  | Maximum speed         | 426,667        |

## Code Changes Made

### 1. Default Configuration (`manager.py`)
```python
# Changed from:
"secondarySetPoint": 3.000,
"secondaryUnit": "mm/s",

# To:
"secondarySetPoint": 100.000,
"secondaryUnit": "RPM",
```

### 2. Speed Conversion (`device.py`)
```python
def set_speed_C1(self, speed=0.0):
    """Set speed on control channel C1."""
    # Convert RPM to frequency: frequency = (RPM * CPR) / 60
    target_frequency = int((speed * self.CPR) / 60)
    self.freqC1, self.rollC1, self.width_C1 = self.set_clock(1, target_frequency)
    # Convert back to RPM for storage: RPM = (frequency * 60) / CPR
    self.speed_C1 = (self.freqC1 * 60) / self.CPR
    log.info("Speed on control channel C1 set to {speed} RPM.".format(speed=speed))
```

### 3. Speed Limits (`device.py`)
```python
def set_speed_limit(self):
    """Set speed limit for motors."""
    # Since we're now working directly in RPM, speed limit is simply max_rpm
    self.speed_limit = self.max_rpm  # 4000 RPM
```

## Troubleshooting

### Motor Not Responding
1. Check LabJack T7 connection
2. Verify motor enable switch is ON
3. Check speed value is > 0
4. Ensure not at position limits

### Incorrect Speed
1. Verify RPM value in speed field
2. Check encoder CPR setting (should be 6400)
3. Confirm PWM frequency calculation

### Direction Issues
1. Check motor wiring
2. Verify direction pin states (EIO1/EIO3)
3. Test with opposite button

## Hardware Requirements

- **LabJack T7**: Data acquisition and control device
- **Stepper Motor**: With 6400 count/revolution encoder
- **Motor Driver**: Compatible with PWM + direction control
- **Limit Switches**: Optional but recommended for safety

## Safety Notes

- Always start with low RPM values (50-100) for testing
- Use physical limit switches to prevent mechanical damage
- Monitor motor temperature during high-speed operation
- Have emergency stop readily accessible
- Test direction and limits before high-speed operation

## Support

For technical support or questions about the motor control system:
1. Check this documentation first
2. Review log files for error messages
3. Test with minimal RPM values
4. Contact system administrator if hardware issues persist