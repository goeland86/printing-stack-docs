# Recore (A6 / A7 / A8)

The [Recore](https://www.iagent.no) is the Klipper host of choice for
every Voron in this fleet. It's a Linux SBC with onboard stepper
drivers — no separate Pi + control board needed.

| | |
|---|---|
| **Vendor** | iAgent |
| **Docs / wiki** | <https://www.iagent.no> |
| **Variants used in this fleet** | A6 (Voron V0), A7 (Voron 2.4 / StealthChanger), A8 (Voron Trident 300, CR-30) |

## What's important here

**The Recore class matters more than the exact revision.** From the
stack's perspective the A6, A7, and A8 are interchangeable — same
firmware family, same Klipper config conventions, same way of running
Moonraker and the NFC daemon. Pick the revision that fits the
printer's size, motor count, and accelerometer needs; everything in
this site applies regardless.

For installation, firmware updates, MCU config, accelerometer setup,
and anything board-specific, refer to the **[iAgent
wiki](https://www.iagent.no)** — they own that documentation.

## Why this fleet uses Recore

- One board, one OS, one Klipper instance — no separate host
- Onboard drivers tuned for the supported motor footprints
- Integrated accelerometer support
- Active community in the iAgent ecosystem

## Variant → printer mapping in this fleet

| Recore | Printer | Why this variant |
|--------|---------|------------------|
| A6 | Voron V0 | Smallest footprint, matches V0's compact build |
| A7 | Voron 2.4 (StealthChanger) | Adequate driver count + headroom for 5 toolheads + host load |
| A8 | Voron Trident 300, CR-30 (Klipper mod) | Larger motor / current envelope |

If you're picking a board for a new printer, the iAgent wiki has the
current selection guide — that's the canonical source.

## Not running a Recore?

Nothing in this docs site is Recore-specific. The NFC daemon, the
bridge, and every macro pattern documented here run unchanged on a
Raspberry Pi, BTT CB1 / Pi 4-equivalent carrier, or any other Klipper
host. The Recore is just the reference fleet's choice — substitute
freely.
