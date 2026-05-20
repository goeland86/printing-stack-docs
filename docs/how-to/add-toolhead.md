# Add a toolhead to StealthChanger

Going from N toolheads to N+1 — physical install, firmware, Klipper
config, NFC integration.

## Prerequisites

- The N+1th StealthChanger toolhead, assembled and mounted to its dock
- A spare RP2040 toolboard, flashed to the same firmware build as the
  existing ones (see [Flash RP2040 toolheads](flash-rp2040.md))
- The dock physically installed and roughly aligned on the gantry beam

## Steps

### 1. Connect the new toolboard

Wire it into the same VIA Labs USB hub the other toolboards use. Power
it from the same 5V rail.

Verify enumeration:

```bash
ls /dev/serial/by-id/usb-Klipper_rp2040_*
# Expect N+1 entries now
```

If the new one doesn't show up, see the
[VIA Labs gotchas in the flashing guide](flash-rp2040.md#why-make-flash-is-unreliable-here)
— this is usually firmware drift, not a wiring problem.

### 2. Add the MCU to `printer.cfg`

```ini
[mcu T5]
serial: /dev/serial/by-id/usb-Klipper_rp2040_<new-serial-here>
```

Capture the serial from the `ls` output above.

### 3. Define the tool's extruder, fan, and toolhead

Mirror the existing tool definitions (T0..T4). The pattern looks like:

```ini
[extruder5]
step_pin: T5:gpio2
dir_pin: T5:gpio3
enable_pin: !T5:gpio4
# …rotation_distance, microsteps, heater_pin, sensor_pin, etc.

[fan_generic T5_partfan]
pin: T5:gpio5

[heater_fan T5_hotend_fan]
pin: T5:gpio6
heater: extruder5
```

Then the toolchanger tool block:

```ini
[tool T5]
tool_number: 5
extruder: extruder5
fan: T5_partfan
gcode_x_offset: 0
gcode_y_offset: 0
gcode_z_offset: 0
params_park_x: <dock X>
params_park_y: <dock Y>
params_park_z: <dock Z>
pickup_gcode:
    # standard StealthChanger pickup
dropoff_gcode:
    # standard StealthChanger dropoff
```

The `<dock X/Y/Z>` values are measured per-dock; consult
[viesturz/klipper-toolchanger docs](https://github.com/viesturz/klipper-toolchanger#configuration).

### 4. Update the NFC daemon's tool list

Edit `~/printer_data/config/nfc_spoolman.cfg`:

```ini
[nfc]
mode = multi_tool
tools = 0,1,2,3,4,5
```

Restart the daemon:

```bash
sudo systemctl restart nfc-spoolman
```

The next NFC scan will offer T5 as a prompt button.

### 5. Update macros that hardcode tool counts

Anything iterating tools — `PRINT_START`, `POST_TOOL_CHANGE`,
`NFC_STATUS` — usually has a literal range like:

```ini
{% for t in range(5) %}
```

Bump to 6:

```ini
{% for t in range(6) %}
```

Or, better, derive from `[toolchanger].tool_numbers` once and store in
a save_variable.

### 6. Restart Klipper

```bash
sudo systemctl restart klipper
```

Watch the Klipper log for any config errors:

```bash
journalctl -u klipper -n 200
```

### 7. Calibrate the new dock

Standard StealthChanger calibration:

- Probe each axis at the dock position
- Tune the `close_y`/`open_y` clearance values until pickup/dropoff is
  smooth and clears adjacent docks
- Use the printer's tool-offset probing (if equipped) to align T5 to
  the same `gcode_*_offset` reference as T0

!!! warning "Y clearance is currently undersized in this fleet"
    See [Toolchange workflow](../workflows/toolchange.md#known-issue-y-clearance-on-close-approach).
    Use a generous `close_y` for the new dock until the fleet-wide
    re-tune happens.

### 8. Scan a spool onto T5

Tap an NFC-tagged spool, click **T5** in the prompt dialog. Confirm:

```bash
curl -s http://localhost:7125/printer/objects/query?save_variables \
    | jq '.result.status.save_variables.variables | with_entries(select(.key | startswith("nfc_t5_")))'
```

Expect `nfc_t5_material`, `nfc_t5_extruder_temp`, etc. to be populated.

## Decommissioning a toolhead

The reverse — remove the dock physically, then:

1. Drop the `[mcu T5]` / `[extruder5]` / `[tool T5]` blocks from
   `printer.cfg`
2. Drop `5` from `tools = …` in `nfc_spoolman.cfg`
3. Restart Klipper + daemon
4. Optionally clear `nfc_t5_*` from `save_variables`:
   ```bash
   curl -X POST http://localhost:7125/printer/gcode/script \
       -d 'script=SAVE_VARIABLE VARIABLE=nfc_t5_spool_id VALUE=0'
   ```
   (repeat for each `nfc_t5_*` key)
