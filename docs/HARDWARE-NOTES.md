# Multi-Mode Clock & Precision Measurement Tool

A feature-rich application for the Seeed Studio XIAO Round Display with ESP32S3, featuring an analog clock, precision bubble level, and gyroscope-based rotational alignment measurement.

## Features

### Four Display Modes

**1. Clock Mode**
- Analog clock with smooth sub-pixel hand movement
- Digital time display (HH:MM:SS)
- Battery voltage and percentage with color coding
- NTP time synchronization via WiFi
- Persistent timekeeping with BM8563 RTC

**2. Bubble Level Mode**
- High-precision bubble level visualization
- Non-linear sensitivity: 1° tilt = bubble halfway to edge, 8° tilt = at edge
- Crosshairs showing perfect level
- Clean, focused display for precision leveling work
- Based on LSM6DSV16X 6-axis IMU accelerometer

**3. Align Mode** - "A level, but sideways"
- Measures rotational alignment (yaw) between two surfaces
- Automatic gyroscope drift compensation
- Two-tap workflow:
  1. First tap: Calibrate drift (3-second hold-still calibration)
  2. Second tap: Set reference angle
- Real-time alignment feedback with color coding:
  - Green: Aligned (±2°)
  - Yellow: Close (±2-5°)
  - Red: Misaligned (>5°)
- Perfect for aligning shelves, picture frames, or any surfaces rotationally

**4. Off Mode**
- Sleep mode with backlight off
- IMU powered down for battery conservation
- Wake with long-press

### Touch Controls

- **Long press (1000ms):** Open mode menu
- **Tap left/right** in menu: Navigate modes
- **Tap center** in menu: Select mode
- **Tap** in Align mode: Calibrate/set reference
- **500ms debouncing** prevents accidental double-taps

### IMU-Based Measurements

**LSM6DSV16X 6-Axis IMU** (I2C address 0x6B):
- 3-axis accelerometer for tilt measurement (bubble level)
- 3-axis gyroscope for yaw/rotation measurement (align mode)
- 120Hz sampling rate
- Drift compensation: measures and corrects gyro bias on each use
- Exponential smoothing filter (alpha=0.1) for stable readings
- Powers down in sleep mode

### Power Management

- Battery voltage monitoring via D0 ADC pin
- 2:1 voltage divider (hardware on board)
- Color-coded battery display:
  - Green: Charging
  - Red: Low (<15%)
  - Gold: Normal
- IMU power control for battery conservation
- Backlight control via GPIO43

---

## Quick Start Guide

### Switching Modes

1. **Long press** anywhere on the screen
2. Mode menu appears showing: Clock, Bubble, Align, Off
3. **Tap left** or **tap right** to navigate
4. **Tap center** to select

### Using Align Mode

1. Enter Align mode from menu
2. **First tap:** Place device flat, tap anywhere, hold still for 3 seconds (calibrating drift)
3. Progress bar shows calibration countdown
4. **Second tap:** Place device on first surface (reference), tap anywhere
5. Move device to second surface
6. Display shows rotation difference with color feedback:
   - Green "ALIGNED" = surfaces are parallel (±2°)
   - Yellow = close (±2-5°)
   - Red = misaligned (>5°)

### Using Bubble Level

1. Enter Bubble mode from menu
2. Place device on surface
3. Bubble shows tilt direction (floats to high side)
4. Center bubble on crosshairs for perfect level
5. Very sensitive: 1° tilt moves bubble halfway to edge

---

## Hardware Quirks & Solutions

This section describes hardware-specific issues and workarounds for the Seeed Studio XIAO Round Display (GC9A01 driver) with ESP32S3/C3.

## Critical: GPIO9 Pin Conflict

**Problem:** GPIO9 is shared between two functions:
- Touch controller reset pin
- Display SPI MOSI pin

**Symptom:** If touch controller is initialized after display, the display freezes and stops updating.

**Solution:** Initialize touch controller BEFORE display in a specific order:

```cpp
void setup() {
  // 1. TOUCH RESET FIRST (before display claims GPIO9)
  pinMode(TOUCH_RST_PIN, OUTPUT);  // GPIO9
  digitalWrite(TOUCH_RST_PIN, LOW);
  delay(10);
  digitalWrite(TOUCH_RST_PIN, HIGH);
  delay(50);

  // 2. DISPLAY INIT (reconfigures GPIO9 as SPI MOSI)
  tft.init();

  // 3. I2C AND SOFTWARE INIT
  Wire.begin();
  rtc.begin();
  // Touch controller software init (no pin toggling!)
}
```

**Why:** Once the TFT library initializes, GPIO9 becomes SPI MOSI. Any subsequent attempt to toggle it as a GPIO will corrupt SPI communication and freeze the display.

---

## Touch Controller (CHSC6X) Extreme Bouncing

**Problem:** The CHSC6X capacitive touch controller at I2C address 0x2E exhibits severe contact bouncing - it rapidly toggles between touched/not-touched states even when a finger is steadily held down.

**Symptoms:**
- Single tap registers as multiple rapid taps (16ms, 68ms, 94ms apart)
- Long press impossible - breaks into many short presses
- Touch state changes 10+ times per 100ms

**Root Cause:** Hardware exhibits **false negatives** (misses touches) not false positives. The controller intermittently loses contact detection while finger remains down.

**Solution:** Majority vote filter with aggressive hysteresis:

```cpp
// Keep last 8 touch readings in bit history
static uint8_t touchHistory = 0;
touchHistory = (touchHistory << 1) | (rawTouch ? 1 : 0);

// Count bits
uint8_t touchCount = __builtin_popcount(touchHistory);

// Hysteresis thresholds
constexpr uint8_t TOUCH_ON_THRESHOLD = 6;   // Need 6/8 to START touch
constexpr uint8_t TOUCH_OFF_THRESHOLD = 0;  // Need 0/8 to END touch

// Update filtered state with hysteresis
if (!filteredTouch && touchCount >= TOUCH_ON_THRESHOLD) {
  filteredTouch = true;   // Strong signal to start
} else if (filteredTouch && touchCount <= TOUCH_OFF_THRESHOLD) {
  filteredTouch = false;  // Must be completely clear to end
}
// Otherwise maintain current state (sticky zone)
```

**Key insight:** Once touch is detected (6/8 samples), it remains "touched" even if only 1/8 subsequent samples detect touch. Only when 0/8 samples (completely clear) does it register as released.

**Debouncing:** Additional 500ms minimum between touch actions prevents accidental double-triggers.

---

## NTP Time Synchronization

**Problem:** `configTime()` is asynchronous - calling `getLocalTime()` immediately after may fail or return stale time.

**Solution:** Poll with timeout until NTP sync completes:

```cpp
configTime(0, 0, ntpServer);
setenv("TZ", "PST8PDT,M3.2.0,M11.1.0", 1);
tzset();

// Wait up to 10 seconds for NTP sync
struct tm timeInfo;
bool gotTime = false;
for (int i = 0; i < 20; i++) {
  if (getLocalTime(&timeInfo)) {
    gotTime = true;
    break;
  }
  delay(500);
}

if (gotTime) {
  // Set RTC from NTP time
  rtc.setTime(...);
}
```

---

## Display Sprite Corruption During WiFi

**Problem:** WiFi operations can corrupt the TFT sprite buffer, causing subsequent renders to fail.

**Solution:** Delete and recreate sprite after WiFi/NTP initialization:

```cpp
// After WiFi.begin() and configTime()...

// Recreate sprite for clean state
face.deleteSprite();
face.createSprite(FACE_W, FACE_H);
face.loadFont(NotoSansBold15);

// Now safe to render
renderFace(time_secs);
```

---

## Backlight Control

**Pin:** GPIO43 (D6 on XIAO)
**Logic:** Normal (HIGH = on, LOW = off)
**Hardware switch:** KE switch on board must be ON for software control

**CRITICAL:** The "KE switch" on the board must be set to ON. When OFF, the backlight stays on regardless of GPIO state.

**Implementation:**
```cpp
const int TFT_BL_PIN = 43;  // GPIO43 = D6 on XIAO

void setup() {
  pinMode(TFT_BL_PIN, OUTPUT);
  digitalWrite(TFT_BL_PIN, HIGH);  // Backlight ON
}

// Sleep mode - turn off backlight
void enterSleep() {
  digitalWrite(TFT_BL_PIN, LOW);   // Backlight OFF
}

// Wake mode - turn on backlight
void wakeFromSleep() {
  digitalWrite(TFT_BL_PIN, HIGH);  // Backlight ON
}
```

**PWM brightness control** (if supported by hardware):
```cpp
analogWrite(TFT_BL_PIN, 255);  // Full brightness
analogWrite(TFT_BL_PIN, 128);  // 50% brightness
analogWrite(TFT_BL_PIN, 0);    // Off
```

**Note:** On XIAO ESP32S3, D6 maps to GPIO43 (NOT GPIO6). Do not use GPIO19/GPIO20 as they are reserved for USB.

---

## Battery Voltage Reading

**ADC Pin:** D0
**Voltage Divider:** 2:1 (hardware divider on board)

```cpp
const int BAT_PIN = D0;
constexpr float BATTERY_VOLTAGE_DIVIDER = 2.0f;

float voltage = (analogReadMilliVolts(BAT_PIN) * BATTERY_VOLTAGE_DIVIDER) / 1000.0f;
```

**18650 Battery Voltage Range:**
- Fully charged: 4.2V
- Nominal: 3.7V
- Low cutoff: 3.0V
- Observed minimum: 2.65V (protection circuit)

**Non-linear discharge curve:** Use cubic polynomial or lookup table for accurate percentage calculation.

---

## Touch and Long Press Detection

**Gestures supported:**
- **Tap:** Quick touch and release
- **Long press:** Hold for 1000ms

**Debouncing:** 500ms minimum between actions
**Long press threshold:** 1000ms hold time

**Implementation notes:**
- Majority vote filter runs every loop iteration (~100ms rate)
- Long press detection checks filtered touch state, not raw
- Once long press triggers, tap is suppressed on release

---

## Hardware Configuration Summary

| Component | Pin/Address | Notes |
|-----------|-------------|-------|
| Display (GC9A01) | SPI | 240x240 round, MOSI on GPIO9 |
| Touch (CHSC6X) | I2C 0x2E | Reset on GPIO9 (shared!) |
| RTC (BM8563) | I2C 0x51 | Persistent timekeeping |
| IMU (LSM6DSV16X) | I2C 0x6B | 6-axis accelerometer + gyroscope |
| Battery ADC | D0 | 2:1 voltage divider |
| Backlight | GPIO43 (D6) | HIGH=on, LOW=off. KE switch must be ON |

**IMU Wiring:**
- SDA → D4 (GPIO6)
- SCL → D5 (GPIO7)
- VCC → 3.3V
- GND → GND

---

## Initialization Order (Critical!)

1. Battery pin setup
2. **Backlight pin setup** (GPIO43/D6 as output, set HIGH for on)
3. **Touch controller hardware reset** (GPIO9 as output)
4. **Display initialization** (GPIO9 becomes SPI MOSI)
5. Sprite creation
6. Font loading
7. Initial render (optional)
8. WiFi connection
9. NTP sync with polling
10. RTC initialization
11. **Sprite recreation** (post-WiFi cleanup)
12. Touch controller software init (no pin manipulation)
13. **IMU initialization** (I2C, accelerometer + gyroscope)
14. Battery reading
15. Final render with correct time

**Violating this order will cause display freeze or touch malfunction.**

---

## Lessons Learned

- **GPIO conflicts are real:** Shared pins require careful initialization sequencing
- **Capacitive touch is noisy:** Software filtering essential for reliable operation
- **Hysteresis matters:** Different thresholds for on/off transitions handle bouncing
- **WiFi corrupts memory:** Reinitialize graphics buffers after network operations
- **NTP is async:** Always poll for completion, never assume immediate availability
- **Gyroscope drift is real:** MEMS gyros drift ~0.1°/sec, varies with temperature
- **Drift compensation works:** Measure bias at startup, subtract from all readings
- **Non-linear scaling rocks:** Power functions (x^0.336) make small tilts more visible
- **Accelerometers measure gravity:** When stationary, they only see 1g downward
- **Units matter:** LSM6DSV16X returns millig (mg) and millidegrees/sec (mdps)
- **Document hardware quirks:** Future you (and others) will thank you

---

## References

**Hardware:**
- [Seeed Studio XIAO Round Display Getting Started](https://wiki.seeedstudio.com/get_started_round_display_xiao/)
- [Seeed Studio Round Display Usage Guide](https://wiki.seeedstudio.com/seeedstudio_round_display_usage/)
- [Hardware Schematics (SCH & PCB)](https://files.seeedstudio.com/wiki/round_display_for_xiao/SeeedStudio_Round_Display_for_XIAO_v1.0_SCH&PCB_230308.zip)
- [GC9A01 Display Driver Datasheet](https://www.waveshare.com/w/upload/5/5e/GC9A01A.pdf)
- [CHSC6X Touch Controller](https://github.com/Xinyuan-LilyGO/T-Display-S3-Long/blob/main/doc/CST816S_DataSheet_EN.pdf)
- [LSM6DSV16X IMU Datasheet](https://www.st.com/resource/en/datasheet/lsm6dsv16x.pdf)

**Libraries:**
- [TFT_eSPI Library](https://github.com/Bodmer/TFT_eSPI)
- [SparkFun LSM6DSV16X Library](https://github.com/sparkfun/SparkFun_LSM6DSV16X_Arduino_Library)
- [I2C_BM8563 RTC Library](https://github.com/m5stack/M5Unified/tree/master/src/utility)
