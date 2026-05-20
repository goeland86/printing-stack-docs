# Flash RP2040 toolheads

The Voron's 5 toolheads each run a Kalico-style firmware on an RP2040.
Flashing them reliably through a USB hub (this fleet uses a VIA Labs
hub) has specific pitfalls — this page documents what actually works.

## TL;DR

1. **Build firmware once**: `cd ~/klipper && make menuconfig && make`
2. **Flash every board in one batch** — don't power-cycle the printer
   between toolheads
3. **Use the manual mount-and-copy** path — `make flash` is unreliable
   through the VIA Labs hub
4. After all boards are flashed, **power cycle the whole printer**

## Why a batch

The VIA Labs hub will occasionally fail to re-enumerate a downstream
RP2040 after a power cycle. Diagnosed cause: **firmware-version drift
between toolboards**. If T0 is on commit A and T3 is on commit B, the
hub's port-state machine gets confused and one board comes up missing.

Solution: keep every toolboard on the same firmware build.

## Step-by-step

### 1. Build the firmware

```bash
cd ~/klipper
make menuconfig
# Architecture: rp2040
# Bootloader: 2 KiB (PicoBoot)
# Communication: USB
make clean
make
```

Result: `out/klipper.uf2`.

### 2. Put a toolboard into BOOTSEL mode

Power-cycle just the toolboard (don't touch the printer power) while
holding BOOT, or use the on-Pico BOOTSEL button if exposed.

The board enumerates as a USB mass storage device named `RPI-RP2`.

### 3. Mount and copy

```bash
# Find the mount point — usually under /media/$USER or /run/media/
lsblk | grep RPI-RP2
# Or wait for udisks2 to auto-mount it, then:
ls /media/$USER/RPI-RP2/

# Copy the firmware
cp ~/klipper/out/klipper.uf2 /media/$USER/RPI-RP2/

# Verify it disappeared (board reboots after the copy completes)
ls /media/$USER/RPI-RP2/ 2>&1  # should now error "No such file or directory"
```

The RP2040 jumps back into the firmware automatically. Klipper's USB
device should re-enumerate as `/dev/serial/by-id/usb-Klipper_rp2040_*`.

### 4. Repeat for every toolboard

Without restarting Klipper or the printer, walk through T0..T4 in
succession. The host doesn't care — it isn't running prints during
this.

### 5. Final power cycle

Once **all** toolboards have been flashed:

```bash
# Update Klipper config if any serial-by-id entries changed
sudo systemctl restart klipper
```

If you needed to power-cycle the printer for some other reason, that's
the moment to do it.

## Why `make flash` is unreliable here

`make flash` uses `picotool` / direct USB-mass-storage interaction. On
the VIA Labs hub it intermittently:

- Fails to find the device (timing issue with hub enumeration)
- Writes correctly but the host loses track of the device after reboot
- Picks the *wrong* RP2040 if multiple are in BOOTSEL mode at once

Manual `cp` to a mounted `RPI-RP2/` filesystem sidesteps all of these.

## Common gotchas

| Symptom | Cause | Fix |
|---------|-------|-----|
| Board doesn't enumerate after flash | Hub didn't re-enumerate cleanly | Unplug + replug *just that toolboard's USB*; if no joy, flash again |
| `klipper.uf2: Permission denied` on copy | Mount is owned by a different user | `sudo cp` or remount as your user |
| One toolboard missing in Klipper after power cycle | Firmware version drift between boards | Flash **all** boards to the same build in one batch |
| `make` produces a kernel-style error | `menuconfig` was edited but `.config` wasn't saved | Re-run `make menuconfig`, save, then `make clean && make` |

## When to also reflash the host MCU

The Recore A7 hosts Klipper natively — there's no "host MCU" to flash.
But if you're running a printer where the mainboard MCU is separate
(STM32, SAMD, etc.), reflash it from the same build, in the same
batch, for the same enumeration-stability reasons.
