# Mainsail Client Macros (`mainsail.cfg`)

This documents the Mainsail client macros that handle pause/resume/cancel functionality and the web UI integration.

**Config file:** `mainsail.cfg`
**Source:** Standard Mainsail OS client macros (by Alex Zellner), with some customizations
**Included by:** `ramps.cfg` via `[include mainsail.cfg]`

> This file is mostly boilerplate from the Mainsail project. You generally don't need to edit it directly. Customization is done through the `_CLIENT_VARIABLE` macro at the top.

---

## Client Variables

```ini
[gcode_macro _CLIENT_VARIABLE]
variable_use_custom_pos   : True
variable_park_at_cancel   : True
variable_park_at_cancel_x : 0
variable_park_at_cancel_y : 200
```

This macro stores configuration variables that all other Mainsail macros read. It's the central place to customize pause/cancel behavior without editing the macros themselves.

### Active Settings

| Variable | Value | Meaning |
|----------|-------|---------|
| `use_custom_pos` | `True` | Use custom X/Y park coordinates instead of defaults |
| `park_at_cancel` | `True` | Move the toolhead to a park position when canceling a print |
| `park_at_cancel_x` | `0` | Park at X=0 when canceling |
| `park_at_cancel_y` | `200` | Park at Y=200 when canceling (near the back of the bed) |

### Commented-Out Options (Available but not active)

| Variable | Default | What It Controls |
|----------|---------|-----------------|
| `custom_park_x` | 0.0 | Custom X park position during PAUSE |
| `custom_park_y` | 0.0 | Custom Y park position during PAUSE |
| `custom_park_dz` | 2.0 | How many mm to lift the nozzle when parking |
| `retract` | 1.0 | mm to retract filament on PAUSE |
| `cancel_retract` | 5.0 | mm to retract filament on CANCEL |
| `speed_retract` | 35.0 | Retract speed in mm/s |
| `unretract` | 1.0 | mm to push filament back on RESUME |
| `speed_unretract` | 35.0 | Unretract speed in mm/s |
| `speed_hop` | 15.0 | Z lift speed in mm/s |
| `speed_move` | 100.0 | XY move speed in mm/s |
| `use_fw_retract` | False | Use firmware retraction (`[firmware_retraction]`) instead of manual G1 E commands |
| `idle_timeout` | 0 | Custom idle timeout during pause (0 = don't change) |
| `runout_sensor` | "" | Filament runout sensor name (empty = none configured) |

Since `use_custom_pos` is `True` but `custom_park_x`/`custom_park_y` are commented out, the PAUSE park position falls through to defaults: the toolhead parks at the maximum X/Y position minus 5mm (i.e., top-right corner).

---

## Virtual SD Card

```ini
[virtual_sdcard]
path: ~/printer_data/gcodes
on_error_gcode: CANCEL_PRINT
```

- **path** -- Where Mainsail stores uploaded G-code files. This is the standard Mainsail OS path.
- **on_error_gcode** -- If Klipper encounters an error while reading from the virtual SD card, it automatically runs `CANCEL_PRINT` to safely abort.

---

## Supporting Sections

```ini
[pause_resume]     # Enables PAUSE/RESUME functionality in Klipper
[display_status]   # Enables M73 progress display on LCD
[respond]          # Enables RESPOND command for sending messages to the UI
```

These are required Klipper modules that the macros depend on. They don't need configuration beyond being present.

---

## CANCEL_PRINT

What happens when you hit "Cancel" in Mainsail:

1. Restores any modified idle timeout
2. Parks the toolhead at X=0 Y=200 (as configured in `_CLIENT_VARIABLE`)
3. Retracts filament (default 5mm)
4. Turns off all heaters
5. Turns off the part cooling fan
6. Clears any "pause at layer" settings
7. Calls Klipper's base cancel function

---

## PAUSE

What happens when a print is paused (button in Mainsail, M600, or pause-at-layer):

1. Saves the current extruder temperature (for restore on resume)
2. Optionally extends the idle timeout (so heaters don't turn off while you work)
3. Calls Klipper's base pause (saves position)
4. Parks the toolhead (lifts Z, moves to park position, retracts filament)

---

## RESUME

What happens when you hit "Resume" after a pause:

1. Checks if the printer went to idle timeout while paused
   - If so, reheats the extruder to the saved temperature and waits
2. Checks if the extruder is hot enough to extrude
   - If not, shows an error telling you to heat up first
3. Checks the filament runout sensor (if configured)
   - If no filament detected, shows an error
4. If all checks pass: restores idle timeout, unretracks filament, and resumes printing from where it left off

---

## Pause-at-Layer Macros

### SET_PAUSE_NEXT_LAYER

```
SET_PAUSE_NEXT_LAYER [ENABLE=1] [MACRO=PAUSE]
```

Pauses the print when the next layer change is detected. Useful for inserting magnets, changing filament at a specific point, etc.

### SET_PAUSE_AT_LAYER

```
SET_PAUSE_AT_LAYER [ENABLE=1] [LAYER=<number>] [MACRO=PAUSE]
```

Pauses when a specific layer number is reached. The slicer must be configured to send `SET_PRINT_STATS_INFO` with layer information (most modern slicers do this automatically).

### SET_PRINT_STATS_INFO (Override)

This overrides Klipper's built-in `SET_PRINT_STATS_INFO` to intercept layer change notifications. When the slicer reports a new layer, this macro checks if a pause was requested for that layer.

---

## Internal Helper Macros

These are called by the main macros and shouldn't be called directly:

### _TOOLHEAD_PARK_PAUSE_CANCEL

Handles the actual toolhead parking logic:
1. Reads park configuration from `_CLIENT_VARIABLE`
2. Retracts filament
3. If the printer is homed: lifts Z, moves to park X/Y position
4. If not homed: prints a warning message

The park position logic (simplified):
- If custom position is enabled and set: use custom coordinates
- For cartesian printers (like yours): park at `(max_x - 5, max_y - 5)` by default

### _CLIENT_EXTRUDE

Pushes filament forward (unretract) before resuming. Checks that the extruder is hot enough first. Supports both manual extrusion (G1 E) and firmware retraction (G10/G11).

### _CLIENT_RETRACT

Pulls filament back (retract) when pausing or canceling. Delegates to `_CLIENT_EXTRUDE` with a negative length.
