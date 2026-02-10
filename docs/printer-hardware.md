# Printer Hardware Configuration (`ramps.cfg`)

This documents the hardware configuration of the modded Anet A6 running Klipper on a RAMPS 1.4 board.

**Config file:** `ramps.cfg`
**Board:** RAMPS 1.4 on Arduino Mega 2560
**Firmware:** Klipper (compiled for AVR atmega2560)

---

## MCU (Microcontroller)

```ini
[mcu]
serial: /dev/serial/by-id/usb-Arduino__www.arduino.cc__0042_956323132343516172E0-if00
```

The MCU connects via USB serial. The path uses `/dev/serial/by-id/...` which is a **stable identifier** -- it won't change if you plug it into a different USB port (unlike `/dev/ttyUSB0` which can shift). This specific ID belongs to an Arduino Mega 2560 (the `0042` is Arduino's Mega USB product ID).

If you ever reflash the Arduino bootloader or replace the board, this serial path will change and needs updating.

---

## Stepper Motors

### How Stepper Config Works

Each stepper has:
- **step_pin / dir_pin / enable_pin**: Physical pins on the RAMPS board that control the stepper driver. The `!` prefix means inverted logic.
- **microsteps**: Subdivisions per full step. 16 is standard for A4988/TMC2209 in legacy mode.
- **rotation_distance**: How far (in mm) the axis moves per full rotation of the motor. This replaces the old `steps_per_mm` calculation. The formula is: `steps_per_mm = (full_steps_per_rotation * microsteps) / rotation_distance`.
- **endstop_pin**: The pin for the homing switch. `^` means internal pullup, `!` means inverted.

### X Axis

```ini
[stepper_x]
step_pin: PF0
dir_pin: !PF1
enable_pin: !PD7
microsteps: 16
rotation_distance: 20.15
endstop_pin: ^!PE5
position_endstop: 0
position_min: 0
position_max: 214
homing_speed: 50
```

- **Pins PF0/PF1/PD7** = standard RAMPS X driver slot
- **rotation_distance: 20.15** -- This is a calibrated value. The theoretical value for a GT2 belt with a 20-tooth pulley is exactly 20mm (`20 teeth * 2mm pitch`). The 20.15 means it was measured and slightly adjusted for accuracy (0.75% correction).
- **Travel: 0 to 214mm** -- The printable width
- **Endstop at position 0** (left side, min endstop)

### Y Axis

```ini
[stepper_y]
step_pin: PF6
dir_pin: !PF7
enable_pin: !PF2
microsteps: 16
rotation_distance: 20.3
endstop_pin: ^!PJ1
position_endstop: -14
position_min: -14
position_max: 217
homing_speed: 50
```

- **Pins PF6/PF7/PF2** = standard RAMPS Y driver slot
- **rotation_distance: 20.3** -- Also calibrated (1.5% correction from theoretical 20mm). Y axis belt may have slightly different tension or pulley wear.
- **position_endstop: -14** -- The endstop triggers at Y=-14, meaning the nozzle is 14mm *before* the front edge of the bed when homed. This is used for the purge line in START_PRINT (which moves to Y=-3).
- **Travel: -14 to 217mm**

### Z Axis

```ini
[stepper_z]
step_pin: PL3
dir_pin: PL1
enable_pin: !PK0
microsteps: 16
rotation_distance: 4.05
endstop_pin: probe:z_virtual_endstop
position_max: 250
```

- **Pins PL3/PL1/PK0** = standard RAMPS Z driver slot
- **rotation_distance: 4.05** -- This is for a leadscrew. A standard T8 leadscrew with 2mm pitch and 2 starts has 4mm per revolution, so 4.05 is a slight calibration. If your leadscrew has a different pitch/start count, this would be different.
- **No physical endstop** -- The line `#endstop_pin: ^PD3` is commented out. Instead, `probe:z_virtual_endstop` uses the BLTouch as the Z endstop. This means Z homes by probing the bed surface.
- **position_max: 250mm** -- Maximum Z height
- **No position_min set** -- Defaults to 0. See [Configuration Review](configuration-review.md) for why you might want to change this.

---

## Safe Z Home

```ini
[safe_z_home]
home_xy_position: 105, 110
speed: 100
z_hop: 10
z_hop_speed: 5
```

This ensures Z homing is safe:
- Before probing Z, the toolhead moves to **X=105 Y=110** (roughly center of the bed)
- **z_hop: 10** -- Lifts the nozzle 10mm before moving to the homing position, preventing the nozzle from dragging across the bed or crashing into clips
- **z_hop_speed: 5** -- Slow Z lift speed (5 mm/s) to be gentle

This section is required when using `probe:z_virtual_endstop` because the BLTouch must probe the bed (not air) for Z homing.

---

## BLTouch (Z Probe)

```ini
[bltouch]
sensor_pin: ^PD3
control_pin: PB5
pin_up_touch_mode_reports_triggered: False
pin_up_reports_not_triggered: True
stow_on_each_sample: True
probe_with_touch_mode: False
x_offset: 35
y_offset: 26
z_offset: 0.6
```

The BLTouch is an auto bed leveling probe that uses a deployable pin to detect the bed surface.

### Pin Configuration
- **sensor_pin: ^PD3** -- Reads the probe trigger signal (shared with the Z endstop pin on RAMPS, since the physical Z endstop is not used)
- **control_pin: PB5** -- Servo signal pin to deploy/retract the BLTouch pin (RAMPS servo header)

### Probe Behavior
- **stow_on_each_sample: True** -- Retracts the pin between each probe point. Slower but more reliable.
- **probe_with_touch_mode: False** -- Uses the standard probe mode (not the newer "touch" mode). Touch mode can be more reliable on some clones.
- **pin_up_touch_mode_reports_triggered: False** and **pin_up_reports_not_triggered: True** -- These handle BLTouch clone quirks where the sensor reports unexpected states when the pin is retracted.

### Offsets
- **x_offset: 35, y_offset: 26** -- The BLTouch is mounted 35mm to the right and 26mm behind the nozzle. Klipper uses these to compensate when probing -- the reported probe position is adjusted to reflect where the nozzle actually is.
- **z_offset: 0.6** -- The distance between the BLTouch trigger point and the actual nozzle tip touching the bed. A value of 0.6mm means when the BLTouch triggers, the nozzle is still 0.6mm above the bed. This is the most critical calibration value for first layer quality. Tune it using `PROBE_CALIBRATE`.

### Commented-Out Options
The commented lines (`#speed`, `#samples`, etc.) are sampling options that were experimented with but left at defaults. Multiple samples improve accuracy at the cost of probing speed.

---

## Extruders

This printer has a **dual extruder, single nozzle** (2-in-1-out) setup using a BigTreeTech hotend. Two separate extruder motors feed filament into one shared nozzle.

### Primary Extruder

```ini
[extruder]
step_pin: PA4
dir_pin: !PA6
enable_pin: !PA2
microsteps: 16
rotation_distance: 19
gear_ratio: 5:1
nozzle_diameter: 0.400
filament_diameter: 1.750
heater_pin: PB4
sensor_type: EPCOS 100K B57560G104F
sensor_pin: PK5
control: pid
pid_Kp: 21.252
pid_Ki: 1.133
pid_Kd: 99.618
min_temp: 0
max_temp: 260
```

- **Pins PA4/PA6/PA2** = RAMPS E0 driver slot
- **rotation_distance: 19** with **gear_ratio: 5:1** -- This is a BMG clone dual-gear extruder. The 5:1 gear ratio means the motor turns 5 times for 1 turn of the drive gear. Effective distance per motor rotation = 19/5 = 3.8mm of filament. BMG extruders typically have ~7.5mm rotation distance at the drive gear, so the full calculation works out with the motor steps.
- **EPCOS 100K B57560G104F** -- A common NTC thermistor used in many printer hotends and beds
- **PID values** -- These were auto-tuned using `PID_CALIBRATE HEATER=extruder TARGET=<temp>`. Never copy PID values from another printer; they depend on your specific heater, thermistor, and cooling.
- **max_temp: 260** -- Safe maximum for the hotend. Most PTFE-lined hotends should stay below 240-250C. All-metal hotends can go higher.
- **pressure_advance: 0.478** (commented out) -- See [Macros Documentation](macros.md) for how pressure advance is handled in the T0/T1 macros.

### Secondary Extruder (Belted)

```ini
[extruder_stepper belted_extruder]
extruder = extruder
step_pin: PC1
dir_pin: !PC3
enable_pin: !PC7
microsteps: 16
rotation_distance: 19
gear_ratio: 5:1
```

- **Pins PC1/PC3/PC7** = RAMPS E1 driver slot
- Uses `[extruder_stepper]` instead of `[extruder1]` because both extruders share the same nozzle and heater. This type syncs the motor to the primary extruder's motion queue rather than creating an independent extruder with its own heater.
- **Same rotation_distance and gear_ratio** as the primary -- both are the same BMG clone model
- The `extruder = extruder` line assigns it to the primary extruder's motion queue at startup

The T0/T1 macros (see [Macros Documentation](macros.md)) handle switching between the two motors.

---

## Heated Bed

```ini
[heater_bed]
heater_pin: PH5
sensor_type: EPCOS 100K B57560G104F
sensor_pin: PK6
control: pid
min_temp: 0
max_temp: 110
pid_Kp: 71.693
pid_Ki: 0.928
pid_Kd: 1384.564
```

- **PH5** = standard RAMPS heated bed MOSFET output
- **Same thermistor type** as the hotend (EPCOS 100K)
- **max_temp: 110** -- Suitable for PLA (50-60C), PETG (70-80C), and ABS (90-110C)
- **PID tuned** -- Auto-tuned via `PID_CALIBRATE HEATER=heater_bed TARGET=<temp>`. Bed PID values tend to be stable once set.

---

## Part Cooling Fan

```ini
[fan]
pin: PH6
```

- **PH6** = RAMPS fan output (active when printing)
- This is the part cooling fan, controlled by `M106`/`M107` in G-code
- No speed limits or custom cycle time set -- uses defaults (full PWM range 0-255)

---

## Display

```ini
[display]
lcd_type: st7920
cs_pin: PH1
sclk_pin: PA1
sid_pin: PH0
encoder_pins: ^PC6, ^PC4
click_pin: ^!PC2
kill_pin: ^!PG0

[output_pin beeper]
pin: PC0
```

- **RepRapDiscount 128x64 Full Graphic Smart Controller** -- The original Anet A6 display, which is a RepRap-compatible 128x64 pixel LCD
- **ST7920** -- The LCD controller chip, driven via SPI (cs/sclk/sid pins)
- **Encoder** -- The rotary knob for menu navigation (PC6/PC4)
- **Click** -- The encoder push-button (PC2, active low)
- **Kill pin** -- Emergency stop button on the display board (PG0, active low). Triggers an emergency shutdown when pressed.
- **Beeper** -- Piezo buzzer on PC0, can be activated via G-code for notifications

---

## Input Shaper

```ini
[input_shaper]
shaper_freq_x: 28.52
shaper_freq_y: 32.96
```

Input shaping compensates for mechanical resonance (ringing/ghosting) in the printer frame. Each axis vibrates at a natural frequency, and the input shaper cancels those vibrations.

- **X resonant frequency: 28.52 Hz** -- Measured using an accelerometer (typically an ADXL345)
- **Y resonant frequency: 32.96 Hz**
- **No shaper_type specified** -- Defaults to `mzv` (Modified Zero Vibration) for both axes. This is a reasonable default that offers good ringing reduction with minimal impact on print speed.

These values were obtained by running `SHAPER_CALIBRATE` or by analyzing accelerometer data. They are specific to your printer's frame stiffness, belt tension, and mass distribution. If you change belts, tighten screws, or modify the frame, you should re-run the calibration.

---

## Printer Kinematics

```ini
[printer]
kinematics: cartesian
max_velocity: 300
max_accel: 1200
max_z_velocity: 20
max_z_accel: 100
```

- **Cartesian** -- Standard XYZ motion system (bed moves Y, toolhead moves X, Z moves the gantry or bed vertically)
- **max_velocity: 300 mm/s** -- The absolute speed limit. The printer won't exceed this regardless of what the slicer requests. For an Anet A6 with RAMPS, 300 is reasonable but you'll rarely hit it in practice.
- **max_accel: 1200 mm/s^2** -- How fast the printer can speed up or slow down. This value is conservative/safe. Higher values (2000-3000) are possible if input shaping handles the ringing, but the Anet A6 frame is not very rigid.
- **max_z_velocity: 20 mm/s** -- Z is limited because leadscrew-driven Z axes are slow by nature
- **max_z_accel: 100 mm/s^2** -- Very conservative Z acceleration, appropriate for a leadscrew

---

## Delayed G-code

```ini
[delayed_gcode activate_default_extruder]
initial_duration: 1
gcode:
    ACTIVATE_EXTRUDER EXTRUDER=extruder
```

This runs 1 second after Klipper starts and activates the primary extruder as the default. This ensures T0 (the primary extruder motor) is synced and ready after every restart.
