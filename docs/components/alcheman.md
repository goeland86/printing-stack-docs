# Elyarchi Alcheman

The Alcheman is the third "stock-like Klipper printer" in the fleet,
running unmodified Klipper and the same NFC integration as the
[Trident](voron-trident.md), [V0](voron-v0.md), and
[CR-30](cr30.md).

| | |
|---|---|
| **Vendor** | Elyarchi |
| **Host** | Proprietary Elyarchi controller — ships with Klipper + Fluidd preinstalled out of the box |
| **Firmware** | Klipper (upstream) |
| **NFC reader** | PN532 on USB-UART |
| **NFC mode** | `single` |
| **Frontend** | Fluidd (vendor default) — Mainsail also works |

## What's specific to the Alcheman

The Alcheman ships with Klipper + Fluidd preinstalled on its
proprietary controller — no host swap or fresh install needed. The NFC
daemon drops in alongside the vendor's existing config; Fluidd renders
the prompt dialog just like Mainsail does.

## Stack integration

No Alcheman-specific tweaks in the NFC stack. The standard install
applies:

```ini
# printer.cfg additions
[respond]
[save_variables]
filename: ~/printer_data/config/saved_variables.cfg
[include nfc_macros.cfg]
```

```ini
# nfc_spoolman.cfg
[nfc]
reader = pn532
pn532_device = /dev/serial/by-id/usb-...   # prefer by-id over ttyUSB0
mode = single

[spoolman]
url = http://<shared-spoolman-host>:7912

[moonraker]
url = http://localhost:7125
```

## See also

- [Trident](voron-trident.md) for the reference single-tool config
- [Install klipper-nfc-daemon](../how-to/install-nfc-daemon.md)
- [New printer bring-up workflow](../workflows/new-printer.md)
