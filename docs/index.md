---
title: Home
hide:
  - navigation
---

# 3D Printing Stack

A small fleet, a lot of moving parts, and the open-source patches that
make them behave as one coherent printing platform.

This site documents how the various tools — some upstream, some forks,
some written from scratch — fit together. If you've landed here trying
to figure out **"do I need this repo, that branch, and that env var?"**,
start with [Architecture → Fleet overview](architecture/overview.md).

## The fleet

| Printer | Host | Firmware | Notes |
|---------|------|----------|-------|
| **Voron 2.4 (StealthChanger)** | [Recore A7](components/recore.md) | Klipper + [`klipper-toolchanger`](https://github.com/viesturz/klipper-toolchanger) | 5 toolheads on RP2040 toolboards |
| **Voron Trident 300** | [Recore A8](components/recore.md) | Klipper | Single-tool, NFC-equipped |
| **Voron V0** | [Recore A6](components/recore.md) | Klipper | Small single-tool, NFC-equipped |
| **CR-30 (Klipper modded)** | (operator's host of choice) | Klipper | Belt printer, NFC-equipped |
| **Elyarchi Alcheman** | (operator's host of choice) | Klipper | NFC-equipped |
| **Snapmaker J1S** | RPi 3 (custom image) | Snapmaker stock + [`snapmaker_moonraker`](https://github.com/goeland86/snapmaker_moonraker) bridge | Closed firmware, bridged to Mainsail |

Every printer in the fleet runs a [PN532 NFC reader on a USB-UART
cable](how-to/install-nfc-daemon.md) and the [klipper-nfc-daemon](components/nfc-daemon.md),
sharing a single [Spoolman](components/spoolman-nfc.md) instance.

## What's documented here

<div class="grid cards" markdown>

-   :material-sitemap: **Architecture**

    Block diagrams of every printer, where the data lives, and which
    process talks to which.

    [:octicons-arrow-right-24: Fleet overview](architecture/overview.md)

-   :material-puzzle: **Components**

    One page per repo. What each project does, what's vanilla vs forked,
    and why.

    [:octicons-arrow-right-24: snapmaker_moonraker](components/snapmaker-bridge.md)

-   :material-source-branch: **Workflows**

    End-to-end sequence diagrams: NFC spool selection, the J1S upload
    pipeline, a Voron toolchange, bringing up a new printer.

    [:octicons-arrow-right-24: NFC spool selection](workflows/nfc-spool.md)

-   :material-tools: **How-to guides**

    Concrete recipes — flash an RP2040, build the Pi image, install the
    NFC daemon, add a toolhead.

    [:octicons-arrow-right-24: Build the J1S Pi image](how-to/build-j1s-image.md)

-   :material-book-open-variant: **Reference**

    The unglamorous bits: tag formats, the `save_variables` keys the
    daemon writes, intercept tables, bill of materials.

    [:octicons-arrow-right-24: Bridge intercepts](reference/bridge-intercepts.md)

</div>

## Why this exists

Each individual project — the Voron, the J1S, NFC spool management,
Spoolman, the bridge — is documented (well or badly) elsewhere. What's
**not** documented anywhere is how the patches and small custom tools
combine into a single workflow where:

- You scan an NFC-tagged spool on any printer in the fleet
- The right tool on the right printer learns which filament is loaded
- Print-start macros read the right temps, the right pressure advance,
  the right offsets
- Mainsail shows the right preheat preset
- Spoolman tracks usage per printer, per tool, per spool

That's the integration story this site tells.

## Status

!!! info "Work in progress"
    This site went live on **2026-05-20** and is filling out as the
    underlying stack stabilizes. Pages marked :material-progress-clock:
    are stubs awaiting content.

## License

Documentation MIT. Each linked project carries its own license; see the
[repo map](architecture/repo-map.md) for the breakdown.
