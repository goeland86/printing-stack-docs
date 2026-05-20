# Klipper, Moonraker, Mainsail

The standard upstream stack. Documented here only for the parts that
matter to the rest of the system — for general usage refer to each
project's own docs.

| Project | Version we run | Notes |
|---------|----------------|-------|
| [**Klipper**](https://github.com/Klipper3d/klipper) | upstream `master` | Voron 2.4 only. J1S has its own firmware. |
| [**Moonraker**](https://github.com/Arksine/moonraker) | upstream `master` | Voron 2.4 only. J1S uses the [bridge](snapmaker-bridge.md). |
| [**Mainsail**](https://github.com/mainsail-crew/mainsail) | upstream latest release | Both printers. |
| [**Fluidd**](https://github.com/fluidd-core/fluidd) | upstream latest release | Optional alternative UI; works against both hosts. |
| [**KlipperScreen**](https://github.com/KlipperScreen/KlipperScreen) | upstream `master` | Voron 2.4 only (optional touchscreen). |

## What our stack relies on

### Klipper `[respond]`

Required on the Voron host for the NFC daemon to push prompt dialogs:

```ini
[respond]
```

Without `[respond]`, `RESPOND TYPE=command MSG="action:prompt_*"` has
no effect. The bridge implements this natively for the J1S — no config
needed.

### Klipper `[save_variables]`

Required on the Voron host to persist NFC metadata across reboots:

```ini
[save_variables]
filename: ~/printer_data/config/saved_variables.cfg
```

Again, the bridge backs this with its own database namespace — no
config needed on the J1S.

### Klipper `[gcode_macro NFC_ASSIGN_TOOL]` & `[gcode_macro NFC_CANCEL]`

Defined in `nfc_macros.cfg` shipped with the NFC daemon. Include from
`printer.cfg`:

```ini
[include nfc_macros.cfg]
```

The bridge intercepts both names natively — no include needed on the
J1S.

### Moonraker `[spoolman]`

Required for the daemon's `POST /server/spoolman/spool_id` to work:

```ini
[spoolman]
server: http://your-spoolman-host:7912
sync_rate: 5
```

The bridge implements the same `server.spoolman.*` JSON-RPC namespace
and reads its Spoolman URL from `config.yaml`.

## Versioning policy

- **Klipper**: track `master`, pin per known-good commit on the Voron's
  Klipper checkout. Updates land deliberately, not via cron.
- **Moonraker**: same — track `master`, advance deliberately.
- **Mainsail / Fluidd**: latest release is fine. The J1S Pi image build
  fetches `mainsail-crew/mainsail` latest at image-build time, so
  rebuild the image to pick up UI updates.

## Why we don't use Kalico / KAMP / other downstream Klipper distros

Kalico (formerly Danger-Klipper) and friends are interesting but each
add a maintenance burden. Vanilla Klipper plus a handful of upstream
plugins (`klipper-toolchanger`, optionally Klippain's bits) covers
everything the fleet needs without picking sides in the fork ecosystem.

If a specific feature shows up only downstream and matters enough,
we'll revisit. Until then: vanilla.
