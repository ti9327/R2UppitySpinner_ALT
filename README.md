# R2 Uppity Spinner ALT

**A community fork of [reeltwo/R2UppitySpinnerV3](https://github.com/reeltwo/R2UppitySpinnerV3) — the ESP32 firmware for the Uppity Spinner periscope lifter PCB.**

[![Uppity Spinner Demo](https://i.vimeocdn.com/video/1153816619-d_640x360)](https://vimeo.com/558277516)

---

## What Is This?

The [Uppity Spinner](https://astromech.net/forums/) is a community-designed PCB for R2-D2 builders that controls the periscope — the motorized pop-up that raises the dome's center section, rotates it, and lowers it back down. It runs on an ESP32 microcontroller, drives two TB9051FTG motor controllers (one for the lifter, one for the rotary), and accepts commands over serial (Marcduino protocol) or via a built-in WiFi web interface.

This fork started as a "hey, wouldn't it be cool if..." project while working through the real-world challenges of getting a periscope mechanism working reliably — tight tolerances, metal dome EMI, high-friction lead screws, and the general chaos of droid builds. After throwing the original codebase at Claude Code and asking a lot of questions, it turned into something worth sharing.

**This is a drop-in replacement for the stock firmware.** All original Marcduino serial commands are fully preserved. If you're already running the stock firmware, you can flash this and your existing sequences and wiring will work exactly as before.

---

## What's Different

### Redesigned Web Interface

The web UI has been completely rebuilt as a responsive phone/tablet/desktop interface. Instead of the original static pages, you get a canvas joystick for lifter control, a rotary dial for rotation, live position feedback, and a dedicated Rescue page for manual recovery — all without needing a USB connection.

The status bar has been expanded to six columns (Height / Rotation / Safety / Motors / Last Command / Firmware Version) so you can see at a glance what's going on — motor state, limit switches, last commands received, everything. Rotary position displays in degrees (0–359) rather than raw encoder ticks.

### Rotary Homes Before Descent

The `:PH` command now raises the periscope first, homes the rotary, *then* lowers — in that order. On the original firmware, it was possible to command a descent with the rotary in an arbitrary position, which can cause the periscope head to catch on tight dome panel openings on the way down. This sequence enforces safe geometry every time.

### Three-Pass Rotary Homing

Getting the rotary to land in exactly the same position repeatably is harder than it sounds. This fork uses a three-pass approach: an encoder-based move to get close, a slow creep until the limit switch triggers, then a back-off and re-approach at minimum speed for a clean, repeatable final position. The result is a rotary that homes accurately every single time rather than landing "somewhere near zero."

### Pulsed Drive for Soft Stops

All vertical motion uses a 3ms on / 1ms off pulse pattern throughout the move. Because a lead screw self-locks when power is cut, each pulse moves roughly one encoder tick and the mechanism brakes mechanically in the off period. This prevents the violent impacts at the limit switches that continuous-drive firmware causes when momentum builds up over a long travel — the mechanism slows itself down by physics, not just by a speed ramp.

### Safe Random Mode

Sending a bare `:PM` fires a randomized auto-sequence that cycles through 6 action types — raises, rotations, pauses, and combinations — all enforcing safe rotary height and homing as needed between moves. It's designed to run at events without operator intervention and without putting the mechanism in a state it can't recover from. One command, genuine-looking random behavior, safe.

### E-STOP That Actually Stops

In the original firmware, the web E-STOP (`/api/estop`) set an abort flag — but the rotary homing functions didn't check it, so the rotary would keep spinning through its full homing sequence even after you hit stop. This fork adds abort checks inside all movement loops so active seeks and rotations truly halt immediately. The abort flag is also cleared properly afterward so the next command isn't silently dropped.

### Comprehensive Diagnostics

A 60-second heartbeat logs the current state of limit switches, encoder positions, motor fault lines, and WiFi status — output to both USB serial and the web log page. When you're debugging a periscope that's misbehaving, having a running log of what the hardware thinks is happening is invaluable. Combined with the expanded web status bar, you can usually figure out what's wrong without pulling the dome.

---

## Additional Reliability Improvements

Beyond the headline features, several under-the-hood changes help in real-world builds:

**Stall detection that works with slower motors.** The original stall detection requires a minimum encoder tick rate that slower motors at moderate power can fail to meet, causing false stall trips that corrupt position tracking over time. This fork replaces the tick-rate check with a position-change timeout — any movement resets the timer.

**Breakaway boost for high-friction mechanisms.** Lead-screw lifters with tight belts or 3D-printed mounts can have significant static friction. This fork adds a short continuous-power burst at startup to break static friction before transitioning to the pulsed drive pattern. Short seeks skip it entirely.

**EMI-resilient calibration.** Metal domes cause interference on encoder wires. During calibration, measured travel distance is validated against the stored value and implausible readings are rejected. Run calibration once with the dome off and the stored baseline is protected from future EMI corruption.

**Encoder drift correction.** After a seek completes, the firmware periodically checks for drift and re-seeks if needed, giving up after three consecutive failures to avoid infinite retry loops.

**Firmware version visible everywhere.** Shown in the serial boot banner, web status bar, `/api/status` JSON, and `#PCONFIG` output.

---

## Hardware

Same hardware as stock — no changes required from the standard Uppity Spinner PCB setup.

- **Microcontroller:** ESP32-WROOM-32 (dual-core: WiFi/web on Core 0, motor control on Core 1)
- **Motor drivers:** Two TB9051FTG (lifter + rotary)
- **Optional:** PCF8574 I2C GPIO expander (remaps limit switches to free ESP32 pins for SD card use)

### Tested Lifter Mechanisms

| Mechanism | Min Power | Seek-to-Bottom Power | Full Travel |
|---|---|---|---|
| Greg Hulette | 65% | 40% | ~450 ticks |
| IA-Parts | 30% | 30% | ~845 ticks |

Both are auto-detected during calibration.

### Recommended Motors (Pololu)

| Voltage | Lifter | Rotary |
|---|---|---|
| 12V | [#4841](https://www.pololu.com/product/4841) | [#4847](https://www.pololu.com/product/4847) |
| 6V | [#4801](https://www.pololu.com/product/4801) | [#4807](https://www.pololu.com/product/4807) |

6V motors give a snappier, faster periscope. 12V motors are slower and more deliberate.

---

## Libraries Required

**Important:** This fork requires a patched version of Reeltwo for ESP32 Arduino core 3.x / ESP-IDF v5.5 compatibility. All three required libraries are included in the `libraries/` folder of this repo — copy them to your Arduino `libraries/` folder (or symlink them).

* **Reeltwo (patched)** — Core droid framework (WiFi, web server, OTA, SMQ remote)
* **Adafruit NeoPixel** — RGB status LED
* **PCF8574** — I2C GPIO expander (optional)

---

## Getting Started

1. Wire the ESP32, motor drivers, encoders, and limit switches per the Uppity Spinner PCB documentation.
2. Copy the three libraries from this repo's `libraries/` folder into your Arduino `libraries/` folder.
3. Flash the firmware via USB or OTA.
4. Connect to the WiFi access point: **SSID: `R2Uppity` / Password: `Astromech`**
5. Open `http://192.168.4.1/` in a browser.
6. Run calibration from the Setup page (or send `#PSC` over serial).

Default serial baud rate: **9600**. Default I2C address: **0x20**.

---

## Serial Command Reference

All commands use the Marcduino protocol. Multiple commands can be chained with colons:

```
:PP100:W2:P0    ← raise, wait 2 seconds, lower
```

### Lifter / Rotary Commands

| Command | Description |
|---|---|
| `:PP<0-100>[,speed]` | Move lifter to position % (0 = down, 100 = up) |
| `:PPR[,speed]` | Random position |
| `:PH[speed]` | Home (raise, rotate to 0°, lower) |
| `:PA<degrees>[,speed][,maxspeed]` | Rotate to absolute degrees |
| `:PAR[,speed][,maxspeed]` | Random absolute rotation |
| `:PD<degrees>[,speed][,maxspeed]` | Rotate relative degrees (+ = CCW, − = CW) |
| `:PDR[,speed][,maxspeed]` | Random relative rotation |
| `:PR<speed>` | Continuous spin (+ = CCW, − = CW; 0 = stop/home) |
| `:PM[,liftSpd,rotSpd,minInt,maxInt]` | Safe random animation mode |
| `:PW[R]<seconds>` | Wait (R = randomize 1..N seconds) |
| `:PL<0-7>` | Light kit mode |
| `:PS<0-100>` | Play stored sequence |

### Configuration Commands

| Command | Description |
|---|---|
| `#PSC` | Run calibration |
| `#PCONFIG` | Display full configuration (includes firmware version) |
| `#PSTATUS` | Show WiFi/remote status |
| `#PFACTORY` | Factory reset (clear all preferences and sequences) |
| `#PS<n>:<seq>` | Store sequence (e.g. `#PS1:H`) |
| `#PL` | List stored sequences |
| `#PD<n>` | Delete sequence n |
| `#PDEBUG[0|1]` | Enable/disable verbose debug output |
| `#PRESTART` | Reboot |

### Example Stored Sequences

```
#PS1:H                              ← home
#PS2:P100                           ← periscope up
#PS3:P100:L5:R30                    ← searchlight, spin CCW
#PS4:P100,100:L7:M,80,80,2,4       ← random fast with sparkle
#PS5:P100:L7:M,50,40,5,5           ← random slow with sparkle
#PS6:A0                             ← face forward
#PS7:P100:L5:R-30                   ← searchlight, spin CW
#PS8:H:P50:W2:P85,35:A90,25:W2:A270,20,100:W2:P100,100:L5:R50:W4:H  ← sneaky periscope
```

---

## Troubleshooting

**Periscope hesitates or stalls on startup** — If your lead screw has high static friction, check the serial output for `BREAKAWAY` messages. You should see Phase 1 (burst) and Phase 2 (ramp) completing. If it consistently times out, increase minimum power on the Parameters web page.

**Periscope won't lower** — The safety system blocks descent if the rotary isn't homed. Use the Rescue page's 3-second long-press safety override to bypass for 60 seconds.

**Erratic behavior with a metal dome** — Twist encoder and limit switch wires tightly and route them away from dome surfaces. The software mitigations (distance rejection, drift correction, position clamping) provide a second layer of protection, but good wiring is the foundation.

**Calibration distance seems wrong** — If the measured distance differs more than 20% from stored, it's automatically rejected. Run calibration with the dome removed to establish a clean baseline.

---

## PCB Assembly Videos

For builders starting from scratch, the original assembly walkthroughs are still the best reference:

- [Part 1](https://www.youtube.com/watch?v=x4_3irdV4C8)
- [Part 2](https://www.youtube.com/watch?v=MdSRJsYx1T8)

---

## Changelog

### v3.4.0 — Multi-motor support + first-run wizard
- **19:1 lifter motor support.** Added user-selectable motor type (Pololu 4757 6.3:1 / Pololu 4751 19:1 / IA-Parts). Each motor ships with its own tuned profile: throttle ceiling, stall timeout, pulse timing, calibration sweep cap, soft-approach ramp. The 19:1 profile is locked at 0.75 throttle with a 700ms stall timeout and 15% top-of-travel ramp to prevent the frame damage Greg Hulette hit during early 19:1 bring-up.
- **First-run setup wizard** (`/wizard`). Motion is gated until the builder walks through motor select → sensor check → low-power creep test → calibration → acceptance seeks. Existing calibrated installs auto-promote on boot so firmware upgrades don't force a re-calibration.
- **`#PMOTOR<n>` Marcduino command** to query or switch motor type. `#PCONFIG` now reports active motor profile and wizard state.
- Stall timeouts in calibration and seek-to-bottom paths now honor the active motor profile instead of hard-coded 2000ms.

### v3.3.5 — Code-review fixes
- Fixed duplicate `rotateHome()` call causing the rotary home-spam bug.
- `volatile`-qualified `sRotaryCircleEncoderCount` to protect the cross-core read.
- Widened stall-detect tick-rate math to int64 to prevent integer overflow on long-duration seeks.
- ESTOP checks added inside rotary creep loops so `/api/estop` halts active homing sequences.
- Snapshot `sRescueOverrideExpiry` before comparing to avoid torn cross-core reads.

### v3.3.4 — Initial fork
Baseline fork of reeltwo/R2UppitySpinnerV3. See "What's Different" above for the feature set.

---

## Credits

Original firmware by [reeltwo / Skelmir](https://github.com/reeltwo/R2UppitySpinnerV3). This fork is maintained by [highfalutintodd](https://github.com/highfalutintodd) with contributions from the R2 Builders Club community.

---

## License

LGPL-2.1 — see [LICENSE](LICENSE) for details.
