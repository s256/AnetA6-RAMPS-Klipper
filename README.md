# Klipper for modded AnetA6
## Components
My Anet A6 is modded with
* RAMPS 1.4 Mainboard using ArduinoMega265
* BL Touch as Z-Probe
* Original Display from Anet A6 (RepRap)
* Dual Extruder with Single Nozzle
    * BMG Clone Dual Gear
    * BigTreeTech Hotend

## Working
* BL Touch as Z-Probe
* Homing
* XYZ Movement
* LCD
* LCD Kill Switch
* Heating (Hotend + Bed)
* Second Extruder
* Extruder Steps
* Input Shaping
* Pressure Advance

## Not Working
* TMC2209 Tuning (needs different connection to use UART)


## Installation 
Move the `ramps.cfg` to your Klipper Installation (I am using Mainsail OS) to `~/klipper.cfg`