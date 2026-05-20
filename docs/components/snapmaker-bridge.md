# snapmaker_moonraker — the J1S bridge

A Go application that pretends to be Moonraker on one side and talks
SACP to a Snapmaker J1S on the other.

| | |
|---|---|
| **Repo** | [`goeland86/snapmaker_moonraker`](https://github.com/goeland86/snapmaker_moonraker) |
| **Language** | Go 1.22+ |
| **Listen** | `:7125` (Moonraker default) — HTTP + WebSocket JSON-RPC |
| **Talks to printer** | SACP TCP `:8888`, Snapmaker HTTP API `:8080` |
| **License** | MIT |
| **Latest** | [![latest tag](https://img.shields.io/github/v/tag/goeland86/snapmaker_moonraker?sort=semver)](https://github.com/goeland86/snapmaker_moonraker/releases) |

## What it does

```
[Mainsail/Fluidd] ←─ HTTP/WebSocket ─→ [Bridge :7125] ←─ SACP/HTTP ─→ [J1S]
```

From the UI's perspective the J1S looks like a normal Klipper printer.
The bridge exposes the same Moonraker JSON-RPC namespace, the same
websocket notifications, and the same printer-object tree.

From the printer's perspective the bridge is just another SACP client
issuing temperature/move/upload commands over TCP.

## Why a bridge (and not patches to either end)

- **Snapmaker firmware is closed.** No way to put Klipper on the J1S
  without losing the touchscreen and the closed-source slicer
  integration.
- **Moonraker is tied to Klipper.** Its serial protocol assumes
  Klipper's MCU protocol on the other end. Adapting it to SACP would be
  a larger surgery than just reimplementing the Moonraker surface.

So the bridge owns the Moonraker contract on one side and the SACP
contract on the other.

## What it implements

| Surface | Notes |
|---------|-------|
| `printer.info`, `printer.objects.list`, `printer.objects.query`, `printer.objects.subscribe` | Standard Klipper objects synthesised from J1S state. |
| `printer.gcode.script`, `printer.gcode.help` | GCode script execution. Some commands are intercepted natively; the rest are translated to SACP. |
| `printer.print.start`, `.pause`, `.resume`, `.cancel` | Print control. |
| `printer.emergency_stop` | Mapped to J1S emergency-stop SACP packet. |
| `server.files.*` (`list`, `upload`, `delete`, `metadata`, `move`, `directory`) | File management. Upload uses streaming `MultipartReader` (no RAM blow-up — see [v1.6.0](https://github.com/goeland86/snapmaker_moonraker/blob/main/Release_Notes.md)). |
| `server.spoolman.*` | Proxied to a configured Spoolman instance. |
| `server.database.*` | Persistent key-value namespace (used by `[save_variables]` emulation). |
| `notify_*` websocket notifications | `notify_status_update`, `notify_gcode_response`, `notify_klippy_ready`, etc. |
| `[respond] action:prompt_*` emulation | Bridge re-broadcasts intercepted `RESPOND TYPE=command MSG="action:..."` calls as `notify_gcode_response`, which Mainsail renders as a prompt dialog. |
| `printer.save_variables` object | Backed by the bridge's persistent database namespace. Read by Mainsail / macros / `NFC_STATUS`. |

See the [Bridge intercepts reference](../reference/bridge-intercepts.md)
for the exact list of intercepted gcode commands.

## What it does **not** implement (yet)

- `printer.bed_mesh.*` — J1S handles bed levelling internally.
- `update_manager` — no autoupdate; the bridge is self-contained and
  versioned independently.
- Anything Klipper-MCU-specific (`gcode_macro` saving, `pid_calibrate`
  output, etc.).

If you hit a missing endpoint that Mainsail expects, file an issue with
the request/response from a real Moonraker.

## Configuration

```yaml
server:
  host: "0.0.0.0"
  port: 7125

printer:
  ip: "192.168.1.100"        # Your J1S IP
  token: ""                  # Confirmed at the printer HMI on first connect
  model: "Snapmaker J1S"
  poll_interval: 2           # Status polling cadence (s)

files:
  gcode_dir: "gcodes"        # Local gcode staging directory

spoolman:
  url: "http://localhost:7912"   # Optional — only if running NFC daemon
```

## How it runs on the J1S Pi image

The CI pipeline builds a turnkey Raspberry Pi 3 SD image:

```
[Browser] → [nginx :80] → [Mainsail static files]
                        → proxy_pass → [snapmaker_moonraker :7125]
                                            ↓
                                  [J1S via SACP/HTTP]
```

- Raspberry Pi OS Lite (Bookworm, 32-bit)
- nginx serving the latest upstream Mainsail
- `snapmaker_moonraker` as a systemd unit (`snapmaker-moonraker.service`)
- Optionally `klipper-nfc-daemon` as `nfc-spoolman.service`
- SSH enabled, default user `pi` / password `temppwd`

See [How-to: build the J1S Pi image](../how-to/build-j1s-image.md).

## Memory & systemd hardening (v1.6.0)

The systemd unit includes:

```ini
[Service]
SyslogIdentifier=snapmaker-moonraker
Environment=GOMEMLIMIT=600MiB
```

- `SyslogIdentifier` ensures `journalctl -u snapmaker-moonraker` actually
  finds the bridge's stdout/stderr (default identifier would be the
  truncated process name `snapmaker_moonr`).
- `GOMEMLIMIT=600MiB` is a defence-in-depth Go-runtime soft cap on the
  Pi's 921 MB RAM. Combined with the streaming upload pipeline this
  prevents the OOM-kills that historically plagued the Pi 3 build on
  large prints.

## Provenance

The SACP implementation in `sacp/` is adapted from
[sm2uploader](https://github.com/macdylan/sm2uploader)
(by [@macdylan](https://github.com/macdylan), MIT) and its ancestor
[snapmaker-sm2uploader](https://github.com/kanocz/snapmaker-sm2uploader)
(by [@kanocz](https://github.com/kanocz)). The code is vendored rather
than imported because sm2uploader is a standalone program
(`package main`).
