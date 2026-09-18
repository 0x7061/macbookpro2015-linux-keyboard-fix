# MacBookPro12,1 — built-in keyboard & trackpad on Omarchy (applespi fix)

Reference for the 13" Early 2015 MacBook Pro (`MacBookPro12,1`). Omarchy 4.x, kernel `linux-omarchy` 7.x, Limine + LUKS, busybox initramfs. Compiled 18 Sept 2026.

---

## 1. Root cause

The topcase (keyboard + trackpad) can be driven over **USB or SPI**; ACPI methods select the interface:

| Method | Meaning |
|---|---|
| `UIST` | USB Interface Status (1 = USB enabled) |
| `UIEN n` | USB Interface Enable (0 = off, 1 = on) |
| `SIST` | SPI Interface Status |
| `SIEN n` | SPI Interface Enable (1 = on, disables USB) |

The kernel `applespi` driver checks `UIST` at probe; if USB is enabled it backs off (`applespi: USB interface already enabled`) and leaves the device to `usbhid`. On this unit the USB path is electrically dead (`usb 1-5: device descriptor read/64, error -71`, `Device not responding to setup address`), so no driver ever gets the keyboard. macOS never noticed because it uses SPI on this model.

Forcing SPI mode exposes a second, platform-wide bug: the Broadwell LPSS DMA engine (`00:15.0`, `dw_dmac_pci`) never completes SPI transfers, so `applespi` logs `SPI transfer timed out` / `Error reading from device: -110` forever.

**Fix = three parts**

1. Kernel parameter `initcall_blacklist=dw_pci_driver_init` → GSPI controller (`00:15.4`) falls back to PIO.
2. At every boot: `UIEN 0`, `SIEN 1` via `acpi_call`, then (re)load `applespi`.
3. Do part 2 in the initramfs too (LUKS passphrase prompt) and around suspend/resume.

Constants: ACPI path `\_SB_.PCI0.SPI1.SPIT` (from `/sys/bus/spi/devices/spi-APP000D:00/firmware_node/path`), ACPI device `APP000D`.

---

## 2. Diagnosis commands

```bash
cat /sys/class/dmi/id/product_name                          # MacBookPro12,1
sudo dmesg | grep -iE "applespi|PIO|dw_dmac|spi1"
sudo dmesg | grep -iE "usb 1-|hid|bcm5974"                   # usb 1-5 error -71 = dead USB path
lsusb | grep -i apple                                        # needs: sudo pacman -Sy usbutils
sudo libinput list-devices | grep -iA3 apple
cat /sys/bus/spi/devices/spi-APP000D:00/firmware_node/path   # ACPI path for the script/hook
lspci -nn | grep -E "15\.0|15\.4"                            # LPSS DMA + GSPI controllers
```

---

## 3. Part 1 — kernel parameters

```bash
sudo nano /etc/default/limine
```

Append (leave existing `KERNEL_CMDLINE[default]=` lines untouched):

```
KERNEL_CMDLINE[default]+=" initcall_blacklist=dw_pci_driver_init mem_sleep_default=s2idle"
```

```bash
sudo limine-update
sudo reboot
grep -oE "initcall_blacklist=[^ ]*|mem_sleep_default=[^ ]*" /proc/cmdline
sudo dmesg | grep -i PIO        # pxa2xx_spi_pci 0000:00:15.4: no DMA channels available, using PIO
```

---

## 4. Part 2 — acpi_call + switch script + systemd unit

### 4.1 acpi_call (DKMS)

```bash
sudo pacman -S --needed dkms base-devel linux-omarchy-headers   # linux-headers for plain -arch kernel
yay -S acpi_call-dkms
sudo modprobe acpi_call
```

### 4.2 Manual test

```bash
P=$(cat /sys/bus/spi/devices/spi-APP000D:00/firmware_node/path)
echo "$P.UIEN 0" | sudo tee /proc/acpi/call
echo "$P.SIEN 1" | sudo tee /proc/acpi/call
sudo modprobe -r applespi && sudo modprobe applespi
sudo dmesg | grep -i applespi | tail -3        # → modeswitch done.
```

### 4.3 Script — `/usr/local/bin/applespi-force`

```sh
#!/bin/sh
P='\_SB_.PCI0.SPI1.SPIT'
modprobe acpi_call
echo "$P.UIEN 0" > /proc/acpi/call
echo "$P.SIEN 1" > /proc/acpi/call
modprobe -r applespi
modprobe applespi
```

```bash
sudo chmod 755 /usr/local/bin/applespi-force     # missing this → status=203/EXEC
```

### 4.4 Unit — `/etc/systemd/system/applespi-force.service`

```ini
[Unit]
Description=Force MacBookPro12,1 topcase to SPI
DefaultDependencies=no
After=systemd-modules-load.service
Before=sysinit.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/applespi-force

[Install]
WantedBy=sysinit.target
```

```bash
sudo systemctl enable --now applespi-force.service
systemctl status applespi-force.service --no-pager     # active (exited), status=0/SUCCESS
```

---

## 5. Part 3a — initramfs hook (keyboard at the LUKS prompt)

Omarchy's `/etc/mkinitcpio.conf.d/omarchy_hooks.conf` **replaces** `HOOKS` with a busybox set (`base udev plymouth … encrypt …`); the `systemd`/`sd-vconsole` line in `/etc/mkinitcpio.conf` is inert. Therefore: busybox-style `run_earlyhook`, and a drop-in named to sort *after* `omarchy_hooks.conf`.

### 5.1 `/etc/initcpio/install/applespi-force`

```bash
#!/bin/bash
build() {
    add_module acpi_call
    add_module intel_lpss_pci
    add_module spi_pxa2xx_pci
    add_module spi_pxa2xx_platform
    add_module applespi
    add_runscript
}
help() {
    echo "Switch MacBookPro12,1 topcase to SPI before the LUKS prompt."
}
```

### 5.2 `/etc/initcpio/hooks/applespi-force`

```sh
#!/usr/bin/ash
run_earlyhook() {
    P='\_SB_.PCI0.SPI1.SPIT'
    modprobe acpi_call
    printf '%s\n' "$P.UIEN 0" > /proc/acpi/call
    printf '%s\n' "$P.SIEN 1" > /proc/acpi/call
    modprobe intel_lpss_pci
    modprobe spi_pxa2xx_pci
    modprobe spi_pxa2xx_platform
    modprobe -r applespi 2>/dev/null
    modprobe applespi
}
```

`run_earlyhook` runs for every hook before any `run_hook`; the `encrypt` passphrase prompt is a `run_hook`, so ordering in HOOKS doesn't matter.

### 5.3 `/etc/mkinitcpio.conf.d/zz-applespi-force.conf`

```
HOOKS+=(applespi-force)
```

### 5.4 Rebuild

```bash
sudo mkinitcpio -P 2>&1 | grep -E "Running build hook|applespi|acpi_call|ERROR|WARNING"
#   must show: -> Running build hook: [applespi-force]
sudo limine-update
sudo reboot
sudo dmesg | grep -iE "acpi_call|modeswitch"     # both at ~1–3 s, not ~12 s
```

---

## 6. Part 3b — suspend / resume

Deep (S3) sleep wedges the SPI device until reboot; s2idle works.

### 6.1 `/etc/systemd/sleep.conf.d/mac-s2idle.conf`

```ini
[Sleep]
MemorySleepMode=s2idle
```

(`mem_sleep_default=s2idle` on the kernel cmdline is set in section 3.) Verify: `cat /sys/power/mem_sleep` → `[s2idle] deep`.

### 6.2 `/usr/lib/systemd/system-sleep/applespi`

Two things must be detached before sleep: `applespi` (wedges otherwise) and the Broadcom Wi-Fi driver `brcmfmac` (fails to enter D3 with `-5`, which aborts the suspend and makes systemd retry in a loop — the "lid logo blinks every few seconds" symptom).

```sh
#!/bin/sh
case "$1" in
    pre)
        modprobe -r applespi
        modprobe -r brcmfmac_wcc 2>/dev/null
        modprobe -r brcmfmac
        ;;
    post)
        modprobe brcmfmac
        /usr/local/bin/applespi-force
        ;;
esac
```

If `modprobe -r brcmfmac` ever reports "in use", add `nmcli radio wifi off` before the unload and `nmcli radio wifi on` after the reload.

```bash
sudo chmod 755 /usr/lib/systemd/system-sleep/applespi
systemctl suspend      # test, then after wake:
sudo journalctl -b -o short-monotonic | grep -iE "PM: |brcmfmac|Failed to put" | tail -15
#   good: one "suspend entry (s2idle)" → "suspend exit", no "returns -5", Wi-Fi re-registers
nmcli device status    # wifi connected again
```

Diagnosing sleep problems: `cat /proc/acpi/wakeup` (ACPI wake devices), `sudo cat /sys/kernel/debug/wakeup_sources` (event counts), and the journal grep above. If the journal shows `Some devices failed to suspend` / `Failed to put system to sleep`, it is a device refusing to suspend, not a wake source.

Trade-off: s2idle drains ~10 %/day closed. Shut down for long stretches, or set up hibernation (needs a disk-backed swapfile with non-negative priority; zram alone won't hibernate — see the matthiasjg gist for suspend-then-hibernate).

---

## 7. Files touched (summary)

| File | Purpose |
|---|---|
| `/etc/default/limine` | `initcall_blacklist=dw_pci_driver_init mem_sleep_default=s2idle` |
| `/usr/local/bin/applespi-force` | UIEN 0 / SIEN 1 / reload applespi |
| `/etc/systemd/system/applespi-force.service` | runs the script at boot (main system) |
| `/etc/initcpio/install/applespi-force` | adds modules + runscript to initramfs |
| `/etc/initcpio/hooks/applespi-force` | early hook: switch to SPI before LUKS prompt |
| `/etc/mkinitcpio.conf.d/zz-applespi-force.conf` | `HOOKS+=(applespi-force)` |
| `/etc/systemd/sleep.conf.d/mac-s2idle.conf` | force s2idle |
| `/usr/lib/systemd/system-sleep/applespi` | detach/reattach `applespi` + `brcmfmac` around suspend |

---

## 8. Maintenance & quirks

- `acpi_call` is DKMS: rebuilds on kernel updates as long as `linux-omarchy-headers` stays installed. Check `sudo dkms status`, `ls /lib/modules/$(uname -r)/updates/dkms/`.
- `mkinitcpio -P` runs on kernel updates and picks the hook up automatically.
- If the ACPI path ever changes, re-read `firmware_node/path` and update the script (4.3) and hook (5.2).
- Harmless dmesg: `Unknown touchpad model 3 – falling back to MB8 touchpad` (no geometry table for the 12,1); one `crc mismatch` during the mode switch.
- Trackpad tuning: Super + Space → Setup → Input, or `omarchy-trackpad-plus`.
- `journalctl` for system units needs `sudo`; `/boot` is root-only; the UKI is an `.efi`, readable with `lsinitcpio`.

---

## 9. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `applespi: USB interface already enabled` | firmware set USB mode | run/enable `applespi-force.service` |
| `usb 1-5: … error -71` | dead USB path on topcase | expected; use SPI mode |
| `applespi: SPI transfer timed out` / `-110` | LPSS DMA bug | `initcall_blacklist=dw_pci_driver_init` |
| unit `status=203/EXEC` "Permission denied" | script not executable | `chmod 755 /usr/local/bin/applespi-force` |
| `acpi_call` first loads at ~12 s, keyboard dead at LUKS | hook not in initramfs (drop-in clobbered) | `zz-` drop-in name, rebuild, check for build-hook line |
| keyboard dead after wake | S3 sleep or driver not reattached | s2idle + sleep hook; `sudo systemctl restart applespi-force` |
| lid closed → logo lights up every few seconds | `brcmfmac` fails D3 (`-5`), suspend aborts, systemd retries | unload/reload `brcmfmac` in the sleep hook (6.2) |

---

## 10. Sources

- Omarchy manual, Mac support — `omarchy.org/manual/mac-support/`
- Omarchy issue #1954 (MacBook8,1 applespi timeouts, DMA root cause) · PR #9735 (PIO switch for 8,1)
- `openwebcraft.com/archive/2026/omarchy-4-on-12-macbook8-1` · `gist.github.com/matthiasjg/78aaf7802146f0b89be3da9e4feb111f`
- Kernel `drivers/input/keyboard/applespi.c` (UIEN/UIST/SIEN/SIST)

Worth reporting in issue #1954: the 12,1 needs the PIO fix **and** a forced `SIEN` (dead USB path), so it can be added to the installer's automatic Mac fixes.
