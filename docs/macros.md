# Custom G-Code Macros

This documents the custom macros defined in `ramps.cfg` that control extruder switching, print start/end sequences, and slicer integration.

---

## Extruder Switching: T0 and T1

Since this printer has a 2-in-1-out extruder (two motors, one nozzle), switching between filaments means activating one motor and deactivating the other. Klipper uses `SYNC_EXTRUDER_MOTION` to assign a stepper motor to an extruder's motion queue.

### T0 -- Activate Primary Extruder

```ini
[gcode_macro T0]
gcode:
    SYNC_EXTRUDER_MOTION EXTRUDER=belted_extruder MOTION_QUEUE=
    SYNC_EXTRUDER_MOTION EXTRUDER=extruder MOTION_QUEUE=extruder
    SET_PRESSURE_ADVANCE ADVANCE=0.0 EXTRUDER=extruder
```

1. **Deactivate the secondary motor** -- Setting `MOTION_QUEUE=` (empty) disconnects `belted_extruder` from any motion queue, so it stops moving.
2. **Activate the primary motor** -- Assigns `extruder` to its own motion queue, so it responds to extrusion commands.
3. **Reset pressure advance to 0** -- Pressure advance is explicitly disabled. See note below.

### T1 -- Activate Secondary Extruder

```ini
[gcode_macro T1]
gcode:
    SYNC_EXTRUDER_MOTION EXTRUDER=extruder MOTION_QUEUE=
    SYNC_EXTRUDER_MOTION EXTRUDER=belted_extruder MOTION_QUEUE=extruder
    SET_PRESSURE_ADVANCE ADVANCE=0.0 EXTRUDER=belted_extruder
```

Same logic in reverse: deactivate primary, activate secondary, reset pressure advance.

### Why Pressure Advance Is Set to 0.0

Both T0 and T1 explicitly set `pressure_advance` to `0.0`, effectively disabling it on every tool change. The extruder section has a commented-out value of `0.478`. This means **pressure advance is currently not being used**.

If you want to enable it, change the `SET_PRESSURE_ADVANCE` lines to use your calibrated value:
```
SET_PRESSURE_ADVANCE ADVANCE=0.478 EXTRUDER=extruder
```
You would need to calibrate the value separately for each extruder since they may have slightly different characteristics.

---

## ACTIVATE_EXTRUDER Override

```ini
[gcode_macro ACTIVATE_EXTRUDER]
description: Replaces built-in macro for a X-in, 1-out extruder configuration SuperSlicer fix
rename_existing: ACTIVATE_EXTRUDER_BASE
gcode:
    {% if 'EXTRUDER' in params %}
      {% set ext = params.EXTRUDER|default(EXTRUDER) %}
      {% if ext == "extruder"%}
        {action_respond_info("Switching to extruder.")}
        T0
      {% elif ext == "belted_extruder" %}
        {action_respond_info("Switching to belted_extruder.")}
        T1
      {% else %}
        {action_respond_info("EXTRUDER value being passed.")}
        ACTIVATE_EXTRUDER_BASE EXTRUDER={ext}
      {% endif %}
    {% endif %}
```

### What This Does

Some slicers (notably SuperSlicer/PrusaSlicer) use `ACTIVATE_EXTRUDER EXTRUDER=<name>` instead of `T0`/`T1` commands to switch tools. Klipper's built-in `ACTIVATE_EXTRUDER` doesn't work correctly with the `extruder_stepper` setup, so this macro **overrides** the built-in command.

### How It Works

1. `rename_existing: ACTIVATE_EXTRUDER_BASE` -- Saves the original Klipper command as `ACTIVATE_EXTRUDER_BASE` so it can still be called as a fallback.
2. When the slicer sends `ACTIVATE_EXTRUDER EXTRUDER=extruder`, it calls `T0`.
3. When the slicer sends `ACTIVATE_EXTRUDER EXTRUDER=belted_extruder`, it calls `T1`.
4. Any other extruder name falls through to the original Klipper command.

This ensures both `T0`/`T1` commands and slicer-generated `ACTIVATE_EXTRUDER` commands work correctly.

---

## START_PRINT

```ini
[gcode_macro START_PRINT]
gcode:
    {% set BED_TEMP = params.BED_TEMP|default(50)|float %}
    {% set EXTRUDER_TEMP = params.EXTRUDER_TEMP|default(210)|float %}
    M140 S{BED_TEMP}
    G90
    SET_GCODE_OFFSET Z=0.0
    M190 S{BED_TEMP}
    M109 S{EXTRUDER_TEMP}
    G28
    G0 X105 Y-3 Z 2.3 F 3000
    G0 X115 Y-3 Z 2.3 F 3000
    G92 E0
    G1 E5 F240
    G0 X115 Y-3 Z 0.4 F 3000
    G0 X115 Y-3 Z 20 F 120
    G92 E0
    G0 X120 Y-3 Z0.3 F2400
    G0 X125 Y-3 Z0.2  F240
```

### Slicer Setup

Your slicer should call this macro in its start G-code with temperature parameters:
```
START_PRINT BED_TEMP={bed_temperature} EXTRUDER_TEMP={first_layer_temperature}
```

If no temperatures are passed, defaults are BED_TEMP=50 and EXTRUDER_TEMP=210 (typical PLA settings).

### Step-by-Step Sequence

| Step | G-code | What It Does |
|------|--------|-------------|
| 1 | `M140 S{BED_TEMP}` | Start heating the bed (non-blocking, doesn't wait) |
| 2 | `G90` | Switch to absolute coordinates |
| 3 | `SET_GCODE_OFFSET Z=0.0` | Reset any active Z offset to 0 (clean slate) |
| 4 | `M190 S{BED_TEMP}` | Wait for bed to reach target temperature |
| 5 | `M109 S{EXTRUDER_TEMP}` | Wait for nozzle to reach target temperature |
| 6 | `G28` | Home all axes (X, Y, then Z via BLTouch) |
| 7-8 | `G0 X105/X115 Y-3 Z2.3` | Move to front-left of bed, off the print surface (Y=-3 is in front of the bed edge) |
| 9 | `G92 E0` | Reset extruder position counter to 0 |
| 10 | `G1 E5 F240` | Extrude 5mm of filament at 4mm/s -- this is a "purge blob" to prime the nozzle |
| 11-12 | Move Z down then up | Move close to bed then lift -- this wipes/detaches the purge blob |
| 13 | `G92 E0` | Reset extruder counter again |
| 14-15 | `G0 X120-125 Y-3 Z0.2-0.3` | Move to start position at first-layer height, ready for printing |

### Why Heat Before Homing?

The bed is heated first, then the nozzle, then homing happens. This means the BLTouch probes the bed while it's at printing temperature, which accounts for thermal expansion of the bed. This is correct practice.

### The Purge Sequence

The purge happens at Y=-3 (in front of the bed's printable area, since Y min is -14). The idea is:
1. Extrude a small blob to clear any ooze or old filament
2. Move down close to the bed to smear it
3. Lift up to break the string
4. Move to the side and lower to first-layer height

This leaves a small waste blob off the print area rather than on your print surface.

### Commented-Out Section (Lines 206-220)

The block of commented-out G-code at the bottom is the **old start sequence** from before the macro was converted to Klipper format. It used direct Marlin-style G-code with slicer variables like `[first_layer_temperature_[current_extruder]]`. It's kept as a reference and can be safely deleted.

---

## END_PRINT

```ini
[gcode_macro END_PRINT]
gcode:
    M140 S0          # Turn off bed heater
    M104 S0          # Turn off extruder heater
    M106 S0          # Turn off part cooling fan
    G91              # Switch to relative coordinates
    G1 Z10 F3000     # Raise nozzle 10mm
    G90              # Switch back to absolute coordinates
    G28 X0 Y0        # Home X and Y (moves head to corner)
    G1 Y210 F5000    # Move bed forward to present the print
    M84              # Disable all stepper motors
```

### Slicer Setup

Your slicer's end G-code should simply call:
```
END_PRINT
```

### Step-by-Step Sequence

| Step | What It Does |
|------|-------------|
| Turn off heaters | Bed and nozzle heaters off immediately to start cooling |
| Turn off fan | Part cooling fan off |
| Raise Z by 10mm | Lifts nozzle away from the print to prevent heat damage |
| Home X/Y | Moves the toolhead to the home corner (out of the way) |
| Move Y to 210 | Slides the bed forward so you can easily grab the print |
| Disable motors | Releases all steppers so you can move axes by hand |

### Notes

- The Z lift uses relative movement (`G91`) then switches back to absolute (`G90`) before homing. This prevents crashing into the print.
- `G28 X0 Y0` only homes X and Y, not Z -- so the nozzle stays at the raised position.
- Stepper disable (`M84`) means the printer won't hold position after the print. The bed and toolhead can be moved by hand.
