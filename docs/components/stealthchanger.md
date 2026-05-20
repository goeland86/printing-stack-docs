# StealthChanger toolheads

The Voron 2.4 in this fleet runs the
[StealthChanger](https://github.com/CalKraken/StealthChanger) toolchange
mechanism with **5 toolheads** controlled by the
[viesturz/klipper-toolchanger](https://github.com/viesturz/klipper-toolchanger)
Klipper extra.

| | |
|---|---|
| **Platform** | Voron 2.4 |
| **Toolboard** | RP2040-based, one per tool |
| **Host** | Recore A7 (Linux) |
| **Toolchanger plugin** | [viesturz/klipper-toolchanger](https://github.com/viesturz/klipper-toolchanger) (upstream, unmodified) |
| **NFC reader** | One PN532 (USB-UART) shared across all 5 tools, scanned spools assigned via [multi-tool prompt](../workflows/nfc-spool.md) |

## Topology

```mermaid
flowchart TB
    Recore["Recore A7<br/>Linux host"]
    USB["USB hub<br/>(VIA Labs)"]
    Recore --- USB
    USB --- T0[T0<br/>RP2040]
    USB --- T1[T1<br/>RP2040]
    USB --- T2[T2<br/>RP2040]
    USB --- T3[T3<br/>RP2040]
    USB --- T4[T4<br/>RP2040]
    USB --- NFC[PN532 reader]

    Recore -->|Klipper MCU protocol| T0
    Recore -->|Klipper MCU protocol| T1
    Recore -->|Klipper MCU protocol| T2
    Recore -->|Klipper MCU protocol| T3
    Recore -->|Klipper MCU protocol| T4
```

## Why this stack

- **klipper-toolchanger** handles the dock/undock kinematics, per-tool
  offsets, and the `T0..T4` GCode contract.
- **NFC daemon's multi-tool mode** populates per-tool `save_variables`
  (`nfc_t0_*` … `nfc_t4_*`) on each spool scan; the print-start macro
  reads those to set temps, PA, and any other per-filament parameters.
- The combination means a tool can be swapped (mechanically or
  spool-wise) and the printer knows what to do without operator
  intervention beyond "tap the spool".

## Known issues / things worth knowing

!!! warning "Toolchange Y clearance"
    Tools currently ram into adjacent docks under some macro variants
    — the `close_y` parameter is too small. Pending fix; increase
    `close_y` per [viesturz/klipper-toolchanger
    docs](https://github.com/viesturz/klipper-toolchanger#configuration)
    when re-tuning the dock positions.

!!! warning "USB enumeration on power cycle"
    The VIA Labs hub has been observed to fail to re-enumerate one of
    the toolboards (`retool1`) after a power cycle. Root cause was
    firmware-version drift between toolboards. Mitigation:
    **flash every RP2040 to the same firmware version in one batch**
    before re-powering the printer. See
    [How-to: flash RP2040 toolheads](../how-to/flash-rp2040.md).

!!! warning "`make flash` on VIA Labs hub"
    The Klipper `make flash` command intermittently fails to write
    through the hub. Use the manual mount-and-copy approach (mount the
    RP2040 as USB mass storage, `cp build/out/klipper.uf2 /mnt/rp2040/`)
    rather than fighting `make flash`. Detailed in the flashing how-to.

## Pressure advance per filament

When a spool is scanned, the NFC daemon writes the spool's
`pressure_advance` from Spoolman into `nfc_t{N}_pressure_advance`. The
print-start macro applies it:

```ini
[gcode_macro PRINT_START]
gcode:
    {% set svv = printer.save_variables.variables %}
    {% for t in range(5) %}
        {% set pa = svv["nfc_t" + t|string + "_pressure_advance"]|default(0.04)|float %}
        SET_PRESSURE_ADVANCE EXTRUDER=extruder{% if t > 0 %}{{ t }}{% endif %} ADVANCE={pa}
    {% endfor %}
```

Spoolman's per-spool `extra.pressure_advance` field stores the value;
the daemon copies it over on every scan so a freshly tuned PA in
Spoolman propagates the moment that spool is loaded.

## Recore A7 specifics

The Recore is an A7 (Cortex-A7) Klipper host with onboard stepper
drivers. It runs Klipper natively, no Pi required. From the fleet's
perspective it's just "the Voron's host" — Mainsail, Moonraker, NFC
daemon, and Spoolman client all live on it.
