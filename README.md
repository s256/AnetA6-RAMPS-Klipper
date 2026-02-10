# Klipper for modded Anet A6

## Components

My Anet A6 is modded with:
* RAMPS 1.4 Mainboard using Arduino Mega 2560
* TMC2209 stepper drivers (standalone mode, no UART)
* BLTouch as Z-Probe
* Original Display from Anet A6 (RepRapDiscount 128x64)
* Dual Extruder with Single Nozzle (2-in-1-out)
    * BMG Clone Dual Gear extruders
    * BigTreeTech Hotend
* Home Assistant integration for PSU power control

## Working
* BLTouch as Z-Probe
* Homing (XY endstops + BLTouch virtual Z endstop)
* XYZ Movement
* LCD + Encoder + Kill Switch
* Heating (Hotend + Bed, PID tuned)
* Dual Extruder switching (T0/T1)
* Extruder Steps (calibrated rotation_distance with 5:1 gear ratio)
* Input Shaping (calibrated resonance frequencies)
* Pressure Advance (calibrated but currently disabled -- see [config review](docs/configuration-review.md))

## Not Working
* TMC2209 UART mode (needs additional wiring for UART on RAMPS 1.4)

## Installation

Move `ramps.cfg` to your Klipper installation (Mainsail OS) as `~/printer_data/config/printer.cfg`.
Place `mainsail.cfg` in the same directory.
Place `moonraker.cfg` in `~/printer_data/config/moonraker.conf`.

## Config Files

| File | Purpose |
|------|---------|
| `ramps.cfg` | Main printer config (hardware, steppers, extruders, macros) |
| `mainsail.cfg` | Mainsail web UI macros (pause, resume, cancel) |
| `moonraker.cfg` | Moonraker API server config (file manager, Home Assistant PSU) |

## Documentation

* [Printer Hardware](docs/printer-hardware.md) -- MCU, steppers, extruders, bed, display, BLTouch, input shaper
* [Custom Macros](docs/macros.md) -- T0/T1 switching, START_PRINT, END_PRINT
* [Mainsail Macros](docs/mainsail-macros.md) -- Pause, resume, cancel, layer-based pausing
* [Moonraker Config](docs/moonraker.md) -- File manager, job queue, Home Assistant PSU integration
* [Configuration Review](docs/configuration-review.md) -- Issues found, missing features, recommendations