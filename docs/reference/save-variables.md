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

### The canonical pattern

The fleet's actual approach factors the read into two macros:

1. **`_PA_DEFAULTS`** — a data-only macro that stores per-material
   fallback values. Empty gcode body; the variables are the payload.
2. **`NFC_APPLY_PA`** — a call macro that applies pressure advance with
   a three-tier priority:
   1. Spool-specific value from the last NFC scan (`nfc_pressure_advance > 0`)
   2. Per-material fallback from `_PA_DEFAULTS` if the spool didn't
      have a tuned PA
   3. Universal default if neither is set

`NFC_APPLY_PA` is called from `PRINT_START` after the spool data has
been loaded by the NFC daemon.

### The actual macros (Voron Trident 300)

```ini
# ─────────────────────────────────────────────────────────────────────────────
# PA DEFAULTS — per-material fallback values used when a spool has no
# tuned PA stored in Spoolman. Edit these to match your measured defaults.
# Values here should be conservative (slightly high is better than too low).
# ─────────────────────────────────────────────────────────────────────────────

[gcode_macro _PA_DEFAULTS]
description: Per-material Pressure Advance fallback values (data macro, not called directly)
variable_pla:     0.040
variable_petg:    0.065
variable_asa:     0.035
variable_abs:     0.035
variable_tpu:     0.000
variable_pc:      0.040
variable_pa_nylon: 0.040
variable_default: 0.040
gcode:
    # Intentionally empty — this macro is a data store only.
    # Call NFC_APPLY_PA to apply the appropriate value.


# ─────────────────────────────────────────────────────────────────────────────
# NFC_APPLY_PA — apply Pressure Advance from the last NFC scan.
#
# Priority order:
#   1. nfc_pressure_advance > 0 in saved variables  -> use spool-specific value
#   2. nfc_material is a known material              -> use _PA_DEFAULTS value
#   3. Neither                                       -> use _PA_DEFAULTS.default
# ─────────────────────────────────────────────────────────────────────────────

[gcode_macro NFC_APPLY_PA]
description: Apply Pressure Advance from NFC spool data, with per-material fallback
gcode:
    {% set svv = printer.save_variables.variables %}
    {% set pa_stored = svv.nfc_pressure_advance | default(-1.0) | float %}
    {% set material  = svv.nfc_material          | default("")  | string | upper %}
    {% set defaults  = printer["gcode_macro _PA_DEFAULTS"] %}

    {% if pa_stored > 0 %}
        SET_PRESSURE_ADVANCE ADVANCE={pa_stored}
        RESPOND TYPE=echo MSG="NFC PA: {pa_stored|round(4)} (spool-specific, material: {material})"

    {% else %}
        {% if   material == "PLA"                    %} {% set pa_fb = defaults.pla      %}
        {% elif material == "PETG"                   %} {% set pa_fb = defaults.petg     %}
        {% elif material in ["ASA", "ABS"]           %} {% set pa_fb = defaults.asa      %}
        {% elif material == "TPU"                    %} {% set pa_fb = defaults.tpu      %}
        {% elif material == "PC"                     %} {% set pa_fb = defaults.pc       %}
        {% elif material in ["PA", "NYLON", "PA-CF"] %} {% set pa_fb = defaults.pa_nylon %}
        {% else                                      %} {% set pa_fb = defaults.default  %}
        {% endif %}

        SET_PRESSURE_ADVANCE ADVANCE={pa_fb}

        {% if material == "" %}
            RESPOND TYPE=echo MSG="NFC PA: {pa_fb|round(4)} (no material info - using default fallback)"
        {% else %}
            RESPOND TYPE=echo MSG="NFC PA: {pa_fb|round(4)} (material fallback for {material} - no spool-specific value)"
        {% endif %}
    {% endif %}
```

Drop this verbatim into `printer.cfg` (or an included file). Edit the
`variable_*` defaults in `_PA_DEFAULTS` to match your tuned values per
material. Call `NFC_APPLY_PA` from your `PRINT_START`:

```ini
[gcode_macro PRINT_START]
gcode:
    # …homing, levelling, heat soak, etc…
    NFC_APPLY_PA
    # …prime line, first layer, etc…
```

### Multi-tool extension (Voron 2.4 / StealthChanger)

For a toolchanger the pattern extends to per-tool keys
(`nfc_t{N}_pressure_advance`, `nfc_t{N}_material`). Apply once per
tool — typically inside the toolchange `POST_TOOL_CHANGE` hook so the
new tool's PA is loaded immediately after a swap, and once per tool at
print-start to seed every tool's PA before the first extrusion.

The shape is the same `NFC_APPLY_PA` macro with `nfc_pressure_advance`
and `nfc_material` swapped for the tool-indexed variants and the
target extruder passed via `SET_PRESSURE_ADVANCE EXTRUDER=…`. See
[Toolchange workflow](../workflows/toolchange.md) for the multi-tool
shape.

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
