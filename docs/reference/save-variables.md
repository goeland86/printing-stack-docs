# `save_variables` keys

Every key the [NFC daemon](../components/nfc-daemon.md) writes via
`SAVE_VARIABLE`, with type, source, and consumer.

Backed on real Klipper by `[save_variables]` (file on disk), on the
bridge by the persistent DB's `save_variables` namespace.

## Single-tool mode

| Key | Type | Source | Consumed by |
|-----|------|--------|-------------|
| `nfc_spool_id` | int | Spoolman `/api/v1/nfc/lookup` | `PRINT_START`, debug |
| `nfc_filament_id` | int | Spoolman spool detail | reference |
| `nfc_filament_name` | string | Spoolman `filament.name` | UI, debug |
| `nfc_material` | string | Spoolman `filament.material` | conditional logic in `PRINT_START` |
| `nfc_vendor` | string | Spoolman `filament.vendor.name` | UI, debug |
| `nfc_color_hex` | string | Spoolman `filament.color_hex` | LED/UI |
| `nfc_diameter` | float | Spoolman `filament.diameter` | extruder rotation_distance sanity check |
| `nfc_extruder_temp` | int | Spoolman `filament.settings_extruder_temp` | `M109` in `PRINT_START` |
| `nfc_bed_temp` | int | Spoolman `filament.settings_bed_temp` | `M190` in `PRINT_START` |
| `nfc_pressure_advance` | float | Spoolman `extra.nozzle_<size>_pressure_advance` | `SET_PRESSURE_ADVANCE` in `PRINT_START` |

## Multi-tool mode

Same keys, with `nfc_t{N}_` prefix per tool index. Example for T2:

| Key | Type | Notes |
|-----|------|-------|
| `nfc_t2_spool_id` | int | |
| `nfc_t2_filament_id` | int | |
| `nfc_t2_filament_name` | string | |
| `nfc_t2_material` | string | |
| `nfc_t2_vendor` | string | |
| `nfc_t2_color_hex` | string | |
| `nfc_t2_diameter` | float | |
| `nfc_t2_extruder_temp` | int | |
| `nfc_t2_bed_temp` | int | |
| `nfc_t2_pressure_advance` | float | per-nozzle from Spoolman extras |

Up to whatever tool count your printer has (`tools = …` in
`nfc_spoolman.cfg`).

## Reading from macros

```ini
[gcode_macro PRINT_START]
gcode:
    {% set svv = printer.save_variables.variables %}

    # Single-tool
    {% set t = svv.nfc_extruder_temp|default(200)|int %}
    M109 S{t}

    # Multi-tool, dynamic
    {% for tool in range(5) %}
        {% set key = "nfc_t" + tool|string + "_extruder_temp" %}
        {% set tt = svv[key]|default(0)|int %}
        {% if tt > 0 %}
            SET_HEATER_TEMPERATURE HEATER=extruder{% if tool > 0 %}{{ tool }}{% endif %} TARGET={tt}
        {% endif %}
    {% endfor %}
```

## Inspecting from CLI

```bash
# Real Klipper
cat ~/printer_data/config/saved_variables.cfg

# Either real Klipper or bridge
curl -s http://localhost:7125/printer/objects/query?save_variables \
    | jq '.result.status.save_variables.variables'
```

## Clearing

A scanned spool overwrites the same keys, so unloading a spool doesn't
auto-clear. To clear T2:

```bash
for k in spool_id filament_id filament_name material vendor color_hex \
         diameter extruder_temp bed_temp pressure_advance; do
    curl -X POST http://localhost:7125/printer/gcode/script \
        -d "script=SAVE_VARIABLE VARIABLE=nfc_t2_${k} VALUE=0"
done
```

(Or use `NFC_CLEAR` if you've added a macro for it — not shipped by
default.)
