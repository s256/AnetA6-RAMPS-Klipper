# PrusaSlicer Setup for Dual Extrusion (2-in-1-out)

Guide for setting up PrusaSlicer to use both extruders with your single-nozzle, dual-motor setup.

**Key constraint:** Both filaments share one nozzle and one heater. The nozzle is always at one temperature, so both filaments must be compatible at that temperature. Two PLA colors or two PETG colors works fine. PLA + ABS does not -- the nozzle can't be 210C and 250C at the same time.

---

## 1. Printer Profile

In PrusaSlicer go to **Printer Settings**:

### General
- **Extruders:** 2

### Custom G-code

**Start G-code:**
```
START_PRINT BED_TEMP={first_layer_bed_temperature[0]} EXTRUDER_TEMP={first_layer_temperature[0]} INITIAL_EXTRUDER={current_extruder}
```

- `INITIAL_EXTRUDER={current_extruder}` -- PrusaSlicer resolves `{current_extruder}` to `0` or `1` depending on which extruder it uses first. The macro calls `T0` or `T1` accordingly to activate the correct feeder motor.
- Temperature is always the same regardless of which feeder is active -- there's only one nozzle and one heater. `{first_layer_temperature[0]}` is fine since both extruder profiles should have the same temperature set anyway.
- Works for single-extruder prints too: if you assign everything to T1, PrusaSlicer sets `current_extruder` to `1` and the macro activates the belted extruder.

**End G-code:**
```
END_PRINT
```

**Tool change G-code (Before tool change):**

Leave this empty. PrusaSlicer inserts `T0`/`T1` commands automatically, and your Klipper macros handle the motor switching. Adding extra G-code here risks conflicts.

**Tool change G-code (After tool change):**

Leave empty as well unless you find you need additional purging beyond what the wipe tower does.

### Extruder 1 (T0)
- **Nozzle diameter:** 0.4
- **Extruder offset X/Y:** 0, 0 (primary extruder, no offset)

### Extruder 2 (T1)
- **Nozzle diameter:** 0.4
- **Extruder offset X/Y:** 0, 0 (same nozzle, so no offset)

Both extruders must have **identical offsets (0,0)** because they share the same physical nozzle. If PrusaSlicer thinks there's an offset, it will shift the print for the second extruder and everything will be misaligned.

---

## 2. Wipe Tower (Critical)

Go to **Print Settings > Multiple Extruders**:

- **Enable Wipe Tower:** Yes (mandatory for single-nozzle setups)
- **Wipe tower position:** Pick a corner that doesn't interfere with your print. X=5 Y=5 works if your prints are centered.
- **Wipe tower width:** 60mm (default, increase if you see color bleed)
- **Wipe tower rotation:** 0

### Purge Volumes

This is the most important setting. It controls how much filament is pushed through the nozzle during a tool change to flush out the old color.

Go to **Purging volumes...** (button next to the wipe tower toggle):

The matrix shows "from extruder X to extruder Y" volumes. For example:
- **T0 -> T1:** 70-140 mm^3
- **T1 -> T0:** 70-140 mm^3

Start with **70 mm^3** and increase if you see color contamination (the first few lines after a switch show the old color). Light-to-dark transitions need less purge than dark-to-light:
- White to Black: ~70 mm^3
- Black to White: ~140 mm^3 or more

The wipe tower wastes filament, but that's inherent to single-nozzle multi-material printing. There's no way around it.

### Wipe Tower Extruder

Set the "Wipe tower extruder" to 0 (so it uses the first extruder for the base layers of the tower).

---

## 3. Filament Profiles

Create two filament profiles (or use the same profile twice if both filaments are the same type/brand).

**Important:** Both filament profiles should have the **same nozzle temperature** set in PrusaSlicer. There's only one heater -- if you set different temperatures, PrusaSlicer will insert M104 temperature changes at every tool switch, causing the nozzle to heat up and cool down repeatedly. This wastes time and produces inconsistent results.

Pick the temperature that works for both filaments and set it identically in both profiles.

### Example: Two PLA Colors
- Both filament profiles: 210C nozzle, 60C bed
- Works perfectly -- same material, same temperature

### Example: PLA + PETG (Not Recommended)
- PLA wants 210C, PETG wants 235C
- No single temperature satisfies both: 210C is too cold for PETG, 235C will overheat PLA
- Stick to filaments of the same type

---

## 4. Assigning Extruders to Parts

### Option A: Color per Object
1. Import your model
2. Right-click the object in the object list (right panel)
3. Select **Change extruder** > **Extruder 2** (or 1)
4. Each object prints entirely with one extruder

### Option B: Color Change at Layer Height
1. In the preview, use the layer slider on the right side
2. Click the **+** icon at the desired layer height
3. Select **Change extruder** and pick the extruder number
4. Everything above that layer uses the new extruder

### Option C: Multi-Material Painting (Per-Region)
1. Select your object
2. Click the paint bucket icon in the left toolbar (**MMU Painting**)
3. Paint regions of the model with Extruder 1 or Extruder 2
4. PrusaSlicer generates tool changes at the boundaries

### Option D: Modifier Volumes
1. Right-click object > **Add modifier** > **Box/Cylinder/etc.**
2. Position the modifier volume over the region you want
3. Right-click the modifier > **Change extruder**
4. Everything inside the modifier uses that extruder

---

## 5. Slicing and Verification

Before printing:
1. **Slice** the model
2. Switch to **Preview** tab
3. Use the **Color Print** view (dropdown at top) -- this shows which extruder is used in each section
4. Check that:
   - The wipe tower is present and doesn't overlap with your model
   - Tool changes happen where expected
   - The wipe tower fits within your bed (X: 0-214, Y: -14 to 217)
5. Look at the G-code preview for `T0` and `T1` commands at tool change points

### What the Generated G-code Does

At each tool change, PrusaSlicer generates something like:
```gcode
; toolchange T1          ; comment marking the switch
T1                        ; your Klipper macro activates belted_extruder
M104 S215                 ; set new temperature (if different)
; wipe tower moves...     ; prints the purge block to flush old filament
G1 X... Y... E... F...    ; purge lines on wipe tower
; resume model...         ; continues printing the actual model
```

---

## 6. Troubleshooting

### Color Bleeding After Tool Change
Increase purge volumes in the wipe tower settings. Dark-to-light transitions need the most purging.

### Strings Between Wipe Tower and Print
Add retraction before/after the tool change. In **Printer Settings > Extruder > Retraction**:
- Retraction length: 1-2mm (for direct-drive BMG)
- Retraction speed: 30-40 mm/s
- Set the same for both extruders

### "Unknown command ACTIVATE_EXTRUDER" or Wrong Extruder
PrusaSlicer should send `T0`/`T1` commands by default. If it sends `ACTIVATE_EXTRUDER`, your Klipper macro override handles this. If you still have issues, make sure the extruder names in PrusaSlicer's printer profile don't conflict with Klipper's names.

### First Layer of Second Extruder Doesn't Stick
Both extruders share the same Z offset and nozzle. If one filament sticks and the other doesn't, it's a temperature or filament issue, not a calibration one. Check that the second filament's temperature is high enough.

### PrusaSlicer Shows "Extruder X is not used" Warning
This is fine -- it just means some extruders you defined aren't assigned to any object. You can ignore it for single-extruder prints.

### Wipe Tower Takes Too Much Space
Reduce the tower width (minimum ~40mm). You can also rotate it with the angle setting. The tower height depends on how many tool changes happen -- more switches = taller tower.

---

## 7. Quick Reference

| Setting | Where | Value |
|---------|-------|-------|
| Extruder count | Printer Settings > General | 2 |
| Extruder offsets | Printer Settings > Extruder 1/2 | Both 0,0 |
| Start G-code | Printer Settings > Custom G-code | `START_PRINT BED_TEMP=... EXTRUDER_TEMP=...` |
| Wipe tower | Print Settings > Multiple Extruders | Enabled |
| Purge volumes | Print Settings > Multiple Extruders > Purging volumes | 70-140 mm^3 |
| Assign extruder | Right-click object > Change extruder | 1 or 2 |
