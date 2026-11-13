# SidewaysLevel

A precision measurement tool for the Seeed Studio **XIAO Round Display** (ESP32S3 + GC9A01, 240 px
round): a clock, a bubble level, and an alignment comparator, on one battery-powered puck.

Ported from the Dropbox project of the same name; the original git history is preserved and the
original long-form README lives in `docs/HARDWARE-NOTES.md` (it is the authority on this board's
quirks - read it before touching anything).

## Modes

| Mode | What it does | Sensor | Accuracy |
|---|---|---|---|
| Clock | Analog + digital clock, NTP-synced, BM8563 RTC, battery indicator | - | exact |
| Bubble | Bubble level, non-linear scale (1 deg = half travel, 8 deg = edge) | accelerometer | ~0.1-0.3 deg averaged |
| Align | Compares the *tilt vector* of a surface against a remembered reference | accelerometer | ~0.1-0.3 deg, drift-free |
| Align+ | In-plane rotation about the surface normal, with an uncertainty readout | gyroscope | window-limited (see below) |
| Off | Backlight off, IMU powered down | - | - |

## The measurement design, and why it is split this way

**Gravity is a free, absolute, drift-free reference.** An accelerometer reads its direction, so the
*tilt magnitude and tilt direction* of any surface can be compared against a remembered reference
with no accumulation of error - the sensor offset is common to both readings and cancels.

**A cheap MEMS gyro can only give relative rotation about one axis, and its error grows with time.**
Yaw error is drift rate x elapsed time. Measured bias residuals give:

- 0.05 deg/s -> 0.25 deg at 5 s, 1.5 deg at 30 s, 15 deg at 5 min
- 0.10 deg/s -> 0.5 deg at 5 s, 3 deg at 30 s, 30 deg at 5 min

So in-plane rotation is trustworthy for a quick tap-move-tap and useless for a long carry. The
display therefore reports it **with an uncertainty band** (+/- drift x elapsed) rather than as a
bare number, and colours it by the total, not the delta. A confident wrong "ALIGNED" is worse than
no reading.

The gyro is only needed for rotation *about the surface normal*; everything else is accelerometer
work. That is the central design decision in this repo.

## Roadmap

1. **Project the gyro onto the gravity axis** instead of integrating raw Z, so in-plane rotation is
   invariant to how the device is tilted while you turn it.
2. **Continuous bias estimation while still** (zero-velocity update): the 3-second calibration only
   has to get close, and drift then stops accumulating during a longer session.
3. **Uncertainty readout** in Align/Align+ (drift x elapsed + floor), colour-coded by the total.
4. **Contact repeatability**: a marked corner and three-point contact beats four flat feet; the
   residual error after the above is dominated by how repeatably the case seats on the surface.

## Hardware

Seeed Studio XIAO Round Display, ESP32S3/C3, GC9A01 240 px round panel, CHSC6X touch, BM8563 RTC,
**LSM6DSV16X 6-axis IMU** (I2C 0x6B, 120 Hz, gyro +/-250 dps, LP1 ULTRA_LIGHT), battery on D0 via a
2:1 divider, backlight on GPIO43.

Critical known traps, all documented in `docs/HARDWARE-NOTES.md`: GPIO9 is shared between the touch
reset and the display's SPI MOSI (initialise touch first), the CHSC6X touch controller bounces hard
(majority-vote filter with hysteresis), `configTime()` is asynchronous (poll for sync), and WiFi
operations corrupt the TFT sprite (recreate it afterwards).

## Layout

- `TFT_eSPI_Clock/` - the main sketch (all modes), fonts and driver
- `LSM6DSV16X_Test/`, `Touch_Test/`, `I2C_Scanner/`, `Backlight_Test/` - bring-up sketches that
  proved each peripheral, kept because they are how you debug this board
- `docs/HARDWARE-NOTES.md` - the long-form hardware and driver documentation
- `CLAUDE.md` - guidance for Claude Code in this repo

## License

Unlicense - public domain, as the original.
