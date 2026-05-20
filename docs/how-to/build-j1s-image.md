# Build the J1S Pi image

The [snapmaker_moonraker](https://github.com/goeland86/snapmaker_moonraker)
repo ships a CI pipeline that builds a turnkey Raspberry Pi 3 SD card
image with Mainsail, the bridge, and (optionally)
[klipper-nfc-daemon](../components/nfc-daemon.md) pre-installed.

## What's on the image

| Software | Role |
|----------|------|
| Raspberry Pi OS Lite (Bookworm, 32-bit) | Base OS |
| nginx | Serves Mainsail on port 80, proxies `/server` and `/websocket` to the bridge |
| Latest upstream Mainsail release | Web UI |
| `snapmaker_moonraker` | Bridge to the J1S, systemd unit `snapmaker-moonraker.service` |
| `klipper-nfc-daemon` | Optional NFC daemon, `nfc-spoolman.service` |
| SSH enabled | `pi` / `temppwd` — **change before exposing to a network** |

## Build with the Jenkins agent

The CI pipeline uses a Docker image with all build deps. Build it
locally with:

```bash
cd snapmaker_moonraker
docker build --network=host -t snapmaker-jenkins-agent -f image/Dockerfile.jenkins-agent .
```

Inside the agent, the pipeline:

1. Cross-compiles the bridge for armv7
2. Mounts a base Pi OS Lite image in QEMU
3. Runs `image/chroot-install.sh` to install Mainsail, the bridge, the
   NFC daemon, and the nginx config
4. Compresses the result to `snapmaker-moonraker-rpi3-YYYYMMDD.img.xz`

## Build locally (without Jenkins)

```bash
# Cross-compile
GOOS=linux GOARCH=arm GOARM=7 CGO_ENABLED=0 \
    go build -ldflags="-s -w" -o snapmaker_moonraker-armv7 .

# Build the image (requires sudo for loop mount / chroot)
sudo image/build-image.sh snapmaker_moonraker-armv7
```

Output: `snapmaker-moonraker-rpi3-YYYYMMDD.img.xz` in the working
directory.

## Flash and boot

```bash
# Decompress
xz -d snapmaker-moonraker-rpi3-YYYYMMDD.img.xz

# Flash (replace /dev/sdX with your SD card)
sudo dd if=snapmaker-moonraker-rpi3-YYYYMMDD.img of=/dev/sdX bs=4M status=progress
sync
```

Insert into the Pi, boot, find it on the network at hostname
`snapmaker` (mDNS) or via your router.

## First-boot configuration

SSH in:

```bash
ssh pi@snapmaker.local    # password: temppwd
```

1. **Change the password**: `passwd`
2. **Point at your J1S**:
   ```bash
   sudo nano /home/pi/printer_data/config/snapmaker-moonraker.yaml
   ```
   Set `printer.ip` and `printer.token` (the J1S HMI prompts on first
   connect).
3. **Restart the bridge**: `sudo systemctl restart snapmaker-moonraker`
4. **Point at Spoolman** (if running):
   ```bash
   sudo nano /home/pi/printer_data/config/nfc_spoolman.cfg
   ```
   Set `[spoolman] url`.
5. **Restart the NFC daemon**: `sudo systemctl restart nfc-spoolman`

## Verify

```bash
# Bridge talking to printer
curl -s http://localhost:7125/printer/info | jq .

# Mainsail loading
curl -sI http://localhost/ | head -1

# NFC daemon healthy
journalctl -u nfc-spoolman -n 50
```

Open a browser to `http://<pi-ip>/`. Mainsail should connect and start
showing live status from the J1S.

## Logs and recovery

| What | Where |
|------|-------|
| Bridge stdout/stderr | `journalctl -u snapmaker-moonraker` (note `SyslogIdentifier`) |
| NFC daemon | `journalctl -u nfc-spoolman` + `~/printer_data/logs/nfc_spoolman.log` |
| nginx access/error | `/var/log/nginx/access.log`, `/var/log/nginx/error.log` |
| Mainsail console | Browser devtools |

## Common gotchas

- **Token mismatch** → bridge logs `auth failed`. Re-enter the token in
  `config.yaml` and restart.
- **OOM kills on uploads** → fixed in v1.6.0 (streaming pipeline). If
  you're on an older build, upgrade — see
  [J1S upload workflow](../workflows/j1s-upload.md).
- **Mainsail says "Disconnected"** → check `systemctl status
  snapmaker-moonraker`; the bridge probably can't reach the J1S
  (network change, J1S off).
