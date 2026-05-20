# Voron V0

The smallest printer in the fleet. Single-tool, [Recore A6](recore.md)
host, PN532 reader on USB-UART. Same NFC stack as every other Klipper
printer here — just a smaller envelope.

| | |
|---|---|
| **Host** | [Recore A6](recore.md) |
| **Firmware** | Klipper (upstream) |
| **NFC reader** | PN532 on USB-UART |
| **NFC mode** | `single` |
| **Frontend** | Mainsail (vanilla) |

## What's specific to V0

The V0 has limited space inside the chassis, which makes the
USB-UART-bridged PN532 the obvious choice — the reader sits outside
the printer, the cable is short, no GPIO juggling needed.

Apart from board sizing, the configuration is identical to the
[Trident](voron-trident.md) and any other single-tool Klipper printer.

## Configuration shape

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
pn532_device = /dev/ttyUSB0
mode = single

[spoolman]
url = http://<shared-spoolman-host>:7912

[moonraker]
url = http://localhost:7125
```

## See also

- [Trident](voron-trident.md) for the closest sibling configuration
- [Install klipper-nfc-daemon](../how-to/install-nfc-daemon.md) for the
  step-by-step
