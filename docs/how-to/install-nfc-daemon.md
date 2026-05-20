# Install klipper-nfc-daemon

For any Klipper-style host in the fleet — Voron, Snapmaker J1S behind
the bridge, or any other printer running Moonraker (real or bridge).

## Prerequisites

- A Moonraker-compatible host reachable on the network
  (Klipper+Moonraker, or the [snapmaker_moonraker bridge](../components/snapmaker-bridge.md))
- One of the supported NFC readers (see [Component
  page](../components/nfc-daemon.md#supported-nfc-readers))
- Python 3.9+
- A running [Spoolman with NFC fork](../how-to/run-spoolman-nfc.md),
  reachable from this host

## Install (real Klipper host)

```bash
cd ~
git clone https://github.com/goeland86/klipper-nfc-daemon.git
cd klipper-nfc-daemon
./install.sh
```

`install.sh` will:

- Create a venv at `~/nfc-spoolman-env`
- `pip install pyserial requests`
- Copy `nfc_spoolman.py` + the `readers/` package to `~/`
- Copy `nfc_spoolman.cfg.example` to `~/printer_data/config/nfc_spoolman.cfg`
  (only if not already present)
- Install + enable the `nfc-spoolman` systemd unit
- Append `nfc-spoolman` to `~/printer_data/moonraker.asvc` so Moonraker
  can manage the service

## Install (J1S Pi image)

Already installed — the image build runs the same install steps
in-chroot. Just edit the config and restart the service.

## Configure

`~/printer_data/config/nfc_spoolman.cfg`:

```ini
[nfc]
reader = pn532
pn532_device = /dev/ttyUSB0
pn532_baudrate = 115200

poll_interval = 0.5
debounce_time = 5.0

# single | multi_tool
mode = multi_tool

# multi_tool only: which tool indices to offer in the prompt
tools = 0,1,2,3,4

# auto-create Spoolman spool from unrecognized OpenPrintTag
auto_create = false

# update Mainsail preheat preset
mainsail_preset = true

# write to Klipper save_variables
klipper_variables = true

[spoolman]
url = http://your-spoolman-host:7912

[moonraker]
url = http://localhost:7125
```

## Add Klipper config (real Klipper only)

```ini
# printer.cfg
[respond]
[save_variables]
filename: ~/printer_data/config/saved_variables.cfg
[include nfc_macros.cfg]
```

Copy `nfc_macros.cfg` from the repo to `~/printer_data/config/`:

```bash
cp ~/klipper-nfc-daemon/nfc_macros.cfg ~/printer_data/config/
```

Restart Klipper to pick up the config changes:

```bash
sudo systemctl restart klipper
```

Skip this section on the J1S — the bridge has all of this built in.

## Reader-specific extras

=== "PN532 (UART)"

    No extra packages. Set DIP switches / solder jumpers on the PN532
    breakout to UART mode (not I2C, not SPI). Connect via a USB-UART
    bridge (PL2303, FTDI, CH340) or directly to GPIO UART pins.

    Verify the OS sees it:

    ```bash
    ls -l /dev/ttyUSB*
    sudo apt install -y libnfc-bin
    nfc-list
    ```

=== "PN5180 (SPI)"

    ```bash
    ~/nfc-spoolman-env/bin/pip install spidev gpiod
    # or, if gpiod isn't packaged:
    ~/nfc-spoolman-env/bin/pip install RPi.GPIO
    ```

    Connect to SPI0 (or your chosen bus) plus two GPIO pins for BUSY
    and RESET. 5V for the RF antenna, 3.3V for logic.

=== "ACR1552U (USB)"

    ```bash
    sudo apt install -y pcscd libpcsclite-dev
    ~/nfc-spoolman-env/bin/pip install pyscard
    sudo systemctl enable --now pcscd
    ```

    Verify:

    ```bash
    sudo apt install -y pcsc-tools
    pcsc_scan
    ```

## Start the service

```bash
sudo systemctl start nfc-spoolman
sudo systemctl status nfc-spoolman
journalctl -u nfc-spoolman -f
```

Scan a known tag — expect a log line like:

```
NFC: detected TigerTag UID=04:12:34:56:78:9A:BC
NFC: matched spool_id=42 (PolyTerra PLA Red)
NFC: prompt sent for multi-tool assignment
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `Config file not found` at startup | `nfc_spoolman.cfg` missing | `cp ~/klipper-nfc-daemon/nfc_spoolman.cfg.example ~/printer_data/config/nfc_spoolman.cfg` |
| Tag scans but no Spoolman match | Tag not registered in Spoolman, or Spoolman NFC env vars not set | Check Spoolman `SPOOLMAN_NFC_ENABLED=TRUE` + `SPOOLMAN_TIGERTAG_ENABLED=TRUE` |
| Prompt dialog never appears in Mainsail | `[respond]` missing in `printer.cfg` (real Klipper) | Add `[respond]`, restart Klipper |
| `NFC_ASSIGN_TOOL: unknown command` | `nfc_macros.cfg` not included (real Klipper) | Add `[include nfc_macros.cfg]`, restart Klipper |
| Same tag triggers repeatedly | `debounce_time` too low for your reader's poll rate | Raise to 5–10 s |
| Python errors about `|` in type hints | Pre-Python-3.9 host | Upgrade Python; the daemon supports 3.9+ |
