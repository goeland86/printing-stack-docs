# Voron Trident 300

Single-tool corexy production printer on a [Recore A8](recore.md) host.
NFC-equipped via a PN532 on USB-UART; runs the standard
[klipper-nfc-daemon](nfc-daemon.md) in single-tool mode.

| | |
|---|---|
| **Hostname** | `biggyprint` |
| **Host** | [Recore A8](recore.md) |
| **Firmware** | Klipper (upstream) |
| **NFC reader** | PN532 on USB-UART |
| **NFC mode** | `single` |
| **Frontend** | Mainsail (vanilla) |

## Why this is in the docs

The Trident is the **simplest Klipper representative** in the fleet —
single-tool, no toolchanger, no bridge. If you're trying to figure out
the minimum needed to integrate a new Klipper printer with the NFC
stack, mirror this one's setup.

## Configuration shape

```ini
# printer.cfg additions for the NFC stack
[respond]
[save_variables]
filename: ~/printer_data/config/saved_variables.cfg
[include nfc_macros.cfg]
```

```ini
# nfc_spoolman.cfg
[nfc]
reader = pn532
pn532_device = /dev/ttyUSB0
mode = single
mainsail_preset = true
klipper_variables = true

[spoolman]
url = http://<shared-spoolman-host>:7912

[moonraker]
url = http://localhost:7125
```

Identical pattern works for any single-tool Klipper printer (see
[CR-30](cr30.md), [Alcheman](alcheman.md), [V0](voron-v0.md)).

## What gets written on a tag scan

The single-tool [`save_variables` keys](../reference/save-variables.md#single-tool-mode):
`nfc_spool_id`, `nfc_material`, `nfc_extruder_temp`, `nfc_bed_temp`,
`nfc_pressure_advance`, etc.

The `PRINT_START` macro reads them — see the
[reference snippet](../reference/save-variables.md#reading-from-macros).
