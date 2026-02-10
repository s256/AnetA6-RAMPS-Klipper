# Configuration Review

An audit of the current Klipper configuration for issues, missing features, and recommendations.

---

## Pressure Advance Needs Recalibration

The T0 and T1 macros both set `SET_PRESSURE_ADVANCE ADVANCE=0.0`, and the extruder section has an old value commented out (`# pressure_advance: 0.478`). This means pressure advance is currently not active.

Pressure advance compensates for the lag between extruder motor movement and actual filament flow -- without it, corners tend to bulge and there's more stringing/oozing.

The old value (0.478) was calibrated previously but needs to be redone. To recalibrate:

1. Make sure only one extruder is active (run `T0`)
2. Heat the nozzle to your usual printing temperature
3. Run the Klipper pressure advance tuning test: print the [tuning tower model](https://www.klipper3d.org/Pressure_Advance.html) while commanding:
   ```
   SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY=1 ACCEL=500
   TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START=0 FACTOR=.005
   ```
4. Measure which band looks best, calculate the value: `pressure_advance = start + measured_height * factor`
5. Update the T0 macro: `SET_PRESSURE_ADVANCE ADVANCE=<value> EXTRUDER=extruder`
6. Repeat for T1 with the belted extruder

Each extruder may need a different value since bowden path length, friction, and gear wear differ.

---

## Important: No `position_min` on Z Axis

```ini
[stepper_z]
# ...
endstop_pin: probe:z_virtual_endstop
position_max: 250
# no position_min → defaults to 0
```

### The Problem

When Klipper homes Z using the BLTouch, it defines the probe trigger point as Z=0 (adjusted by `z_offset`). But `z_offset` is a calibration value that changes -- when you run `PROBE_CALIBRATE`, Klipper uses `TESTZ` to let you manually lower the nozzle in tiny increments until it just grips a piece of paper. If the true contact point is below the currently stored Z=0, the nozzle needs to move into negative Z territory.

With the default `position_min: 0`, Klipper **refuses to move below Z=0**. So during calibration:
- `TESTZ Z=-0.1` would fail if you're already at Z=0
- You'd get: `"Must home axis first"` or `"Move out of range"`
- You can't actually find the correct offset

### The Fix

```ini
position_min: -2
```

This allows the nozzle to travel 2mm below the currently defined Z=0. That's enough room to calibrate properly without risk of crashing (the nozzle would need to go much further than -2mm to actually damage the bed).

### When This Matters

- Every time you run `PROBE_CALIBRATE` or `Z_ENDSTOP_CALIBRATE`
- After replacing the nozzle (different nozzle length changes the offset)
- After adjusting the BLTouch mount height
- After any maintenance that changes the nozzle-to-probe distance

Once `z_offset` is correctly calibrated and saved, the printer never actually commands negative Z during normal printing -- it only matters during the calibration process itself.

---

## Important: No Input Shaper Type Specified

```ini
[input_shaper]
shaper_freq_x: 28.52
shaper_freq_y: 32.96
# no shaper_type_x or shaper_type_y → both default to 'mzv'
```

### What Input Shaping Does

Every printer frame has resonant frequencies -- if the toolhead accelerates/decelerates and hits one of these frequencies, the frame vibrates and you see "ringing" or "ghosting" (ripples near sharp corners in the print). Input shaping applies a counter-signal to the motion commands that cancels out these vibrations.

### The Shaper Types

Klipper offers several input shaper algorithms, each with different trade-offs:

| Shaper | Vibration Reduction | Max Recommended Accel | Smoothing | Best For |
|--------|--------------------|-----------------------|-----------|----------|
| `zv` | Lowest | Highest | Least | Very stiff frames, high-speed printing |
| `mzv` | Good | High | Low | **Default.** Good all-rounder |
| `ei` | Good | Medium-High | Medium | Slightly better vibration reduction than mzv |
| `2hump_ei` | Very good | Medium | More | Printers with problematic resonances |
| `3hump_ei` | Best | Lowest | Most | Last resort for very resonant frames |

**Higher smoothing** means fine details (text, small features) get slightly rounded. **Lower max accel** means the shaper limits how fast the printer can accelerate before the vibration compensation breaks down.

### Why the Type Matters

When you ran `SHAPER_CALIBRATE` with the accelerometer, Klipper analyzed the resonance data and recommended both a frequency *and* a shaper type for each axis. The type determines the shape of the compensation filter. Using `mzv` (the default) is safe but may not be optimal:

- If the calibration recommended `ei` for X, you could run at higher acceleration with the same vibration reduction
- If it recommended `2hump_ei`, your printer has a secondary resonance that `mzv` doesn't handle well

### How to Check / Recalibrate

If you still have the accelerometer (ADXL345):

1. Run `SHAPER_CALIBRATE` (this probes both axes automatically)
2. Klipper prints output like:
   ```
   Recommended shaper_type_x = mzv, shaper_freq_x = 28.5 Hz
   Recommended shaper_type_y = ei, shaper_freq_y = 33.0 Hz
   ```
3. Apply with `SAVE_CONFIG` or manually set:
   ```ini
   [input_shaper]
   shaper_type_x: mzv
   shaper_freq_x: 28.52
   shaper_type_y: ei
   shaper_freq_y: 32.96
   ```

If you don't have the accelerometer anymore, leaving it at the default `mzv` is fine. The frequencies (28.52 and 32.96 Hz) are the more important part and those are already set.

### Relationship to `max_accel`

Your `max_accel` is 1200 mm/s^2. The recommended maximum acceleration for each shaper type at your frequencies:

- `mzv` at ~30 Hz: allows roughly **4,500 mm/s^2** before smoothing becomes excessive
- `ei` at ~30 Hz: allows roughly **6,600 mm/s^2**

So at 1200 mm/s^2, you're well within safe limits for any shaper type. The current acceleration is conservative enough that the shaper type choice mostly affects vibration reduction quality, not speed limits.

---

## Applied: Object Cancellation Support

**This has been added to the config files.**

Object cancellation lets you cancel individual objects in a multi-object print if one fails (spaghetti, detached from bed, etc.) without aborting the entire print. The surviving objects continue printing normally.

Requirements:
- `[exclude_object]` in `ramps.cfg` -- tells Klipper to track object boundaries
- `enable_object_processing: True` in `moonraker.cfg` -- tells Moonraker to parse uploaded G-code and identify objects
- Your slicer must label objects in the G-code (PrusaSlicer and SuperSlicer do this by default with "Label objects" enabled in Print Settings > Output options)

In Mainsail, during a print, you'll see an "Exclude Object" button that shows a visual map of objects on the bed. Click any object to cancel just that one.

---

## Recommended: Add `max_extrude_only_distance`

The default `max_extrude_only_distance` is 50mm. With a geared extruder (5:1 ratio), filament loading/unloading commands may need to push more than 50mm. If you ever get a "Move exceeds maximum extrude only distance" error when loading filament, this is why.

**Add to `[extruder]`:**

```ini
max_extrude_only_distance: 200
```

---

## Minor: Commented-Out Code in START_PRINT

Lines 206-220 in `ramps.cfg` contain a block of old Marlin-style start G-code that's entirely commented out. It serves no purpose and could be removed for clarity. It appears to be the original start sequence before converting to Klipper macros.

---

## Minor: No `[idle_timeout]` Section

Klipper defaults to 600 seconds (10 minutes) idle timeout. After this, heaters turn off and steppers are disabled. This is fine for most usage, but if you frequently pause prints for longer than 10 minutes (e.g., filament changes), you may want to increase it:

```ini
[idle_timeout]
timeout: 1200   # 20 minutes
```

The Mainsail pause macro does have an option to extend idle timeout during pause (via `_CLIENT_VARIABLE`), but it's commented out.

---

## Minor: No `[firmware_retraction]`

The Mainsail macros support firmware retraction (G10/G11 commands handled by Klipper instead of the slicer), but the `_CLIENT_VARIABLE` has `use_fw_retract` commented out and there's no `[firmware_retraction]` section. This is fine if your slicer handles retraction, which it likely does. Only add this if you want Klipper to manage retraction centrally.

---

## Note: TMC2209 Drivers

The config currently uses no TMC-specific sections, meaning the TMC2209 drivers are running in standalone (legacy/step-dir) mode. They work fine this way but you miss out on:
- StealthChop (silent operation)
- Sensorless homing
- Current tuning via software
- Stall detection

To enable UART mode, each driver needs a single-wire UART connection to the MCU. On RAMPS 1.4, this requires additional wiring since the board doesn't have dedicated UART pins for the stepper drivers.

---

## Summary

| Priority | Issue | Status |
|----------|-------|--------|
| Action needed | Pressure advance disabled | Needs recalibration, then update T0/T1 macros |
| Important | No Z `position_min` | Add `position_min: -2` to `[stepper_z]` |
| Important | No input shaper type | Re-run `SHAPER_CALIBRATE` or leave as default `mzv` |
| Done | No object cancellation | Added `[exclude_object]` and enabled Moonraker processing |
| Recommended | No `max_extrude_only_distance` | Add to `[extruder]` if you get extrude errors |
| Minor | Dead commented code | Cleanup when convenient |
| Minor | No idle timeout config | Extend if pauses are longer than 10 min |
