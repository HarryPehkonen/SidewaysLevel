# SidewaysLevel

A precision measurement puck for the Seeed Studio **XIAO Round Display** (ESP32S3 + GC9A01, 240 px
round): a clock, a bubble level, and an alignment comparator, on one battery-powered device.

The long-form hardware and driver documentation is in `docs/HARDWARE-NOTES.md` - it is the authority
on this board's quirks, so read it before touching anything.

## Modes

| Mode | What it measures | Sensor | Accuracy |
|---|---|---|---|
| Clock | Analog + digital clock, NTP-synced, BM8563 RTC, battery state | - | exact |
| Bubble | Bubble level, non-linear scale (1 deg = half travel, 8 deg = edge) | accelerometer | ~0.1-0.3 deg averaged |
| Align | Tilt-vector comparison of a surface against a remembered reference | accelerometer | ~0.1-0.3 deg, drift-free |
| Align (2nd readout) | In-plane rotation about the surface normal, with an uncertainty band | gyroscope | window-limited (below) |
| Off | Backlight off, IMU powered down | - | - |

## Why the measurement is split between two sensors

**Gravity is a free, absolute, drift-free reference.** An accelerometer measures its direction, so a
surface's *tilt magnitude and tilt direction* can be compared against a remembered reference with no
accumulated error - the sensor offset is common to both readings and cancels. This is the primary
Align readout, and it is the one to trust.

**A MEMS gyro can only give relative rotation, and its error grows with time.** Yaw error is drift
rate x elapsed time; measured bias residuals give:

- 0.05 deg/s -> 0.25 deg at 5 s, 1.5 deg at 30 s, 15 deg at 5 min
- 0.10 deg/s -> 0.5 deg at 5 s, 3 deg at 30 s, 30 deg at 5 min

So in-plane rotation is trustworthy for a quick tap-move-tap and useless for a long carry. It is
therefore reported **with an uncertainty band** (+/- drift rate x elapsed, plus a floor) and the
colour coding uses that total rather than the bare delta: a confident wrong "ALIGNED" is worse than
no reading at all.

The gyro exists in this design solely for rotation *about the surface normal*. Everything else is
accelerometer work.

## Roadmap

- [x] Project the gyro onto the gravity axis instead of integrating raw Z, so yaw is invariant to how
      the device is tilted while turning
- [x] Continuous bias estimation while still (zero-velocity update), so drift stops accumulating
- [x] Uncertainty readout, colour-coded by the total
- [ ] Contact repeatability: a marked corner and three-point contact beats four flat feet - the
      residual error after the above is dominated by how repeatably the case seats on a surface

## Hardware

GC9A01 240 px round panel, CHSC6X touch, BM8563 RTC, **LSM6DSV16X 6-axis IMU** (I2C 0x6B, 120 Hz,
gyro +/-250 dps), battery on D0 via a 2:1 divider, backlight on GPIO43 (KE switch must be ON).

Traps, all documented in `docs/HARDWARE-NOTES.md`: GPIO9 is shared between touch reset and display
SPI MOSI (initialise touch first), the CHSC6X bounces hard (majority-vote filter with hysteresis),
`configTime()` is asynchronous (poll), and WiFi corrupts the TFT sprite (recreate it).

## Layout

- `TFT_eSPI_Clock/` - the main sketch (all modes), font and driver
- `LSM6DSV16X_Test/`, `Touch_Test/`, `I2C_Scanner/`, `Backlight_Test/` - bring-up sketches, kept
  because they are how you debug this board
- `docs/HARDWARE-NOTES.md` - the long-form hardware documentation
- `CLAUDE.md` - guidance for Claude Code in this repository

## License

Unlicense - public domain.
