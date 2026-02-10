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

## 2. Purging Strategy

When switching between filaments through a shared nozzle, the old filament must be flushed out. You have two approaches: a wipe tower (separate purge object) or purging into infill/support (no extra waste object on the bed).

### Option A: Purge into Infill and Support (No Wipe Tower)

This deposits the purge filament into the infill or support material of your actual print instead of building a separate tower. No wasted bed space, less wasted filament.

**Setup in Print Settings > Multiple Extruders:**

1. **Enable Wipe Tower:** No (disable it)
2. On each object, right-click > **Print Settings** and enable:
   - **Wipe into this object's infill** -- purge filament becomes part of the infill
   - **Wipe into this object's support** -- purge filament becomes part of support material (if the object has support)
3. Set **Purging volumes** (the button is still accessible even without the wipe tower):
   - Start with **70 mm^3** and increase if you see color bleed
   - Light-to-dark needs less (~70), dark-to-light needs more (~140+)

**Requirements and limitations:**
- The object needs enough infill volume on tool-change layers to absorb the purge. Use **15%+ infill** for reliable results. With very sparse infill or small objects, there may not be enough volume.
- If infill + support can't absorb the full purge volume on a given layer, PrusaSlicer **silently falls back to a wipe tower** for that layer. Check the preview carefully.
- The infill will be a mix of both filament colors -- this is hidden inside the part and doesn't affect strength or appearance.
- Works best with grid, gyroid, or cubic infill patterns that have consistent volume per layer. Avoid adaptive cubic or lightning infill as their volume varies too much.

**Per-object setting:** The "wipe into infill/support" checkboxes are per-object. In a multi-object print, enable it on the largest objects (most infill volume) for best results.

### Option B: Wipe Tower

A dedicated purge block printed alongside your model. Uses extra bed space and filament but guarantees clean color transitions regardless of your model's infill.

**Setup in Print Settings > Multiple Extruders:**

1. **Enable Wipe Tower:** Yes
2. **Wipe tower position:** Pick a corner that doesn't interfere with your print. X=5 Y=5 works if your prints are centered.
3. **Wipe tower width:** 60mm (default, increase if you see color bleed)
4. **Wipe tower rotation:** 0
5. **Wipe tower extruder:** 0 (uses the first extruder for tower base layers)

### Purge Volumes (Applies to Both Options)

Controls how much filament is pushed through the nozzle to flush the old color. Set via **Purging volumes...** button in Print Settings > Multiple Extruders.

The matrix shows "from extruder X to extruder Y" volumes:

| Transition | Recommended Volume |
|---|---|
| Light to dark | ~70 mm^3 |
| Dark to light | ~140 mm^3 or more |
| Similar colors | ~50-70 mm^3 |

Start low and increase if you see color contamination in the first lines after a switch.

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
   - Tool changes happen where expected
   - If using purge-into-infill: verify no surprise wipe tower appeared (PrusaSlicer adds one if infill volume is insufficient on some layers)
   - If using wipe tower: verify it doesn't overlap with your model and fits on the bed
5. Look at the G-code preview for `T0` and `T1` commands at tool change points

---

## 6. Troubleshooting

### Color Bleeding After Tool Change
Increase purge volumes. Dark-to-light transitions need the most purging.

### Strings at Tool Changes
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

### Purge into Infill Not Working / Wipe Tower Keeps Appearing
PrusaSlicer falls back to a wipe tower when there isn't enough infill volume to absorb the purge on a layer. Increase infill percentage, use a denser infill pattern (grid/gyroid over lightning), or accept a small wipe tower for those layers.

---

## 7. Quick Reference

| Setting | Where | Value |
|---------|-------|-------|
| Extruder count | Printer Settings > General | 2 |
| Extruder offsets | Printer Settings > Extruder 1/2 | Both 0,0 |
| Start G-code | Printer Settings > Custom G-code | `START_PRINT BED_TEMP=... EXTRUDER_TEMP=... INITIAL_EXTRUDER=...` |
| Wipe into infill | Right-click object > Print Settings | Enabled per object |
| Wipe tower | Print Settings > Multiple Extruders | Disabled (unless purge-into-infill is insufficient) |
| Purge volumes | Print Settings > Multiple Extruders > Purging volumes | 70-140 mm^3 |
| Assign extruder | Right-click object > Change extruder | 1 or 2 |
