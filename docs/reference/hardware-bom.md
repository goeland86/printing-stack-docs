# Hardware bill of materials

The hardware actually deployed in this fleet. Not a buying guide — a
truthful inventory so other operators can match it (or substitute
deliberately).

## Voron 2.4 (StealthChanger)

| Component | Make / Model | Notes |
|-----------|--------------|-------|
| **Printer frame** | Voron 2.4 | Standard Voron BOM applies — see [vorondesign.com](https://vorondesign.com) |
| **Host** | [Recore A7](../components/recore.md) | Linux SBC with onboard stepper drivers; see [iAgent wiki](https://www.iagent.no) |
| **Toolchanger plates / docks** | StealthChanger | Open-source design — see [CalKraken/StealthChanger](https://github.com/CalKraken/StealthChanger) |
| **Toolheads** | 5 × StealthChanger toolheads | Each carries its own hotend, extruder, fans |
| **Toolboards** | 5 × RP2040-based | Same firmware build across all 5 (avoids hub enumeration drift — see [Flashing guide](../how-to/flash-rp2040.md)) |
| **USB hub** | VIA Labs | Has the enumeration quirk noted in the flashing how-to |
| **NFC reader** | PN532 on USB-UART | Shared across all 5 tools — operator scans spools at a single station |

## Voron Trident 300

| Component | Make / Model | Notes |
|-----------|--------------|-------|
| **Printer frame** | Voron Trident 300 | Standard Voron BOM |
| **Host** | [Recore A8](../components/recore.md) | Linux SBC with onboard stepper drivers; see [iAgent wiki](https://www.iagent.no) |
| **NFC reader** | PN532 on USB-UART | Single-tool, single reader |

## Voron V0

| Component | Make / Model | Notes |
|-----------|--------------|-------|
| **Printer frame** | Voron V0 | Standard Voron BOM |
| **Host** | [Recore A6](../components/recore.md) | Linux SBC with onboard stepper drivers; see [iAgent wiki](https://www.iagent.no) |
| **NFC reader** | PN532 on USB-UART | External reader (limited internal volume on V0) |

## CR-30 (Klipper modded)

| Component | Make / Model | Notes |
|-----------|--------------|-------|
| **Base printer** | Creality CR-30 ("PrintMill") | Belt printer, converted to Klipper |
| **Klipper controller** | Operator's choice of MCU | Any Klipper-supported board |
| **NFC reader** | PN532 on USB-UART | Single reader |

## Elyarchi Alcheman

| Component | Make / Model | Notes |
|-----------|--------------|-------|
| **Printer** | Elyarchi Alcheman | Stock chassis, running upstream Klipper |
| **Klipper controller** | Per Elyarchi spec | Operator-managed |
| **NFC reader** | PN532 on USB-UART | Single reader |

## Snapmaker J1S

| Component | Make / Model | Notes |
|-----------|--------------|-------|
| **Printer** | Snapmaker J1S | Stock, unmodified firmware |
| **Host SBC** | Raspberry Pi 3 | 921 MB RAM — Pi 4 also works |
| **SD card** | ≥ 8 GB Class 10 | Image build is ~3 GB |
| **NFC reader** | PN532 on USB-UART | Pre-installed on the [J1S Pi image](../how-to/build-j1s-image.md) |

## Shared infrastructure

| Component | Notes |
|-----------|-------|
| **Spoolman host** | Docker / Linux box on the LAN; runs the [goeland86/Spoolman NFC fork](../components/spoolman-nfc.md) on `:7912` |
| **Postgres** | Backing DB for Spoolman (SQLite works for single-printer setups) |

## NFC readers — sourcing notes

PN532 modules are the cheapest reliable option. Recommendations:

- **Avoid Aliexpress no-name PN532s** that ship pre-set to I2C with
  the SEL0/SEL1 traces cut wrong. Buy from a vendor that lets you
  select UART mode in stock.
- **USB-UART bridges**: CH340 and FTDI both work. PL2303 also works,
  some older Linux kernels have quirky behavior — prefer CH340/FTDI
  if you have a choice.
- **HSU wakeup**: Once the daemon was upgraded to emit the 16-byte
  `0x55` preamble before every command, no ESP32-based bridge MCU is
  needed. Direct PN532-to-USB-UART works fine.

## NFC tags — sourcing notes

- **TigerTag**: bought with spools that carry them pre-printed (some
  vendors). Standalone NTAG213 stickers can be written by the
  manufacturer but typically you receive them pre-encoded.
- **OpenPrintTag**: NFC-V (ICODE SLIX2) stickers from a generic NFC
  supplier. Format is community-defined, so you write your own.

## Wiring summaries

### PN532 to USB-UART

```
PN532  ── UART  ──  USB-UART bridge  ──  USB  ──  Pi / Recore
TX                  RX
RX                  TX
GND                 GND
VCC (3.3V / 5V)     3.3V (most boards) or 5V (check breakout's regulator)
```

DIP switches on the PN532: set to **UART** (typically `00`). Some
boards have solder jumpers instead — same idea.

### PN5180 (SPI)

Wiring varies per controller (Pi GPIO header, Klipper MCU SPI, etc.)
and per PN5180 breakout. Refer to your **controller's documentation**
and your **PN5180 reader's documentation** for the correct pin
assignments, then plug those GPIO numbers into `pn5180_busy_pin` and
`pn5180_reset_pin` in `nfc_spoolman.cfg`.

### ACR1552U

USB cable. That's it.
