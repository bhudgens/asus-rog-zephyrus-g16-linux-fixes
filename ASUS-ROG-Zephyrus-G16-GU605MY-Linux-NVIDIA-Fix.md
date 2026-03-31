# ASUS ROG Zephyrus G16 (GU605MY) — NVIDIA Driver on Linux

## System Details

| Component | Value |
|-----------|-------|
| Laptop | ASUS ROG Zephyrus G16 GU605MY |
| BIOS | GU605MY.329 (2025-06-06) |
| CPU | Intel Core Ultra (Meteor Lake-P) |
| iGPU | Intel Arc Graphics [8086:7d55] |
| dGPU | NVIDIA GeForce RTX 4090 Laptop GPU (GN21-X11) [10de:2757] |
| OS | Kubuntu 24.04.4 LTS |
| Kernel | 6.17.0-19-generic |
| Secure Boot | Enabled |
| Dual Boot | Windows (with Armoury Crate) |

## The Problem

The NVIDIA dGPU could not be initialized on Linux. The system would hang or loop endlessly during boot with the following error repeated hundreds of times in the kernel log:

```
nvidia 0000:01:00.0: Unable to change power state from D3cold to D0, device inaccessible
nvidia 0000:01:00.0: probe with driver nvidia failed with error -1
NVRM: The NVIDIA GPU 0000:01:00.0 (PCI ID: 10de:2757) has fallen off the bus
```

The NVIDIA HDA audio device on the same bus also failed:

```
snd_hda_intel 0000:01:00.1: Unable to change power state from D3cold to D0, device inaccessible
```

Without nvidia, the laptop ran on the Intel Arc iGPU only. The original goal was to get the NVIDIA driver working to also fix a separate sleep/wake issue where nouveau (the open-source driver) would crash on resume.

## Root Cause

**The ASUS firmware (controlled by Windows Armoury Crate) had disabled the dGPU entirely.**

The `asus-nb-wmi` kernel module exposes a sysfs attribute:

```
/sys/devices/platform/asus-nb-wmi/dgpu_disable
```

This was set to `1`, meaning the embedded controller had **physically powered off** the discrete GPU. Reading the GPU's PCI config space confirmed it — every byte returned `0xff`:

```
$ sudo lspci -s 01:00.0 -xxx
01:00.0 VGA compatible controller: NVIDIA Corporation GN21-X11 (rev a1)
00: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
10: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
...
```

The device was enumerated during early PCI bus scan but was completely inaccessible. No kernel parameter, modprobe option, or driver configuration can bring back a device that is hardware-disabled by the embedded controller.

This flag persists across reboots and is typically set when:
- Windows Armoury Crate is configured to "iGPU only" mode
- The BIOS GPU mode is set to integrated-only
- A previous Windows session set the GPU mode and it wasn't reset to "Auto"

## The Fix

### 1. Systemd Service to Re-enable the dGPU

Create `/etc/systemd/system/nvidia-gpu-setup.service`:

```ini
[Unit]
Description=Disable D3cold and load NVIDIA driver
Before=display-manager.service
After=systemd-udevd.service
DefaultDependencies=no

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/sbin/nvidia-gpu-setup.sh

[Install]
WantedBy=graphical.target
```

Create `/usr/local/sbin/nvidia-gpu-setup.sh`:

```bash
#!/bin/bash
# ASUS ROG Zephyrus G16 GU605MY - NVIDIA GPU setup
# The dGPU is disabled by ASUS firmware (dgpu_disable=1) at boot.
# We must re-enable it, rescan PCI, then load the nvidia driver.

set -e

# Step 1: Enable the dGPU via ASUS WMI
DGPU_PATH="/sys/devices/platform/asus-nb-wmi/dgpu_disable"
if [ -f "$DGPU_PATH" ]; then
    CURRENT=$(cat "$DGPU_PATH")
    if [ "$CURRENT" = "1" ]; then
        echo "nvidia-gpu-setup: dGPU is disabled, enabling..." | systemd-cat -t nvidia-gpu-setup
        echo 0 > "$DGPU_PATH"
        sleep 1
    fi
fi

# Step 2: Remove stale GPU entries and rescan PCI bus
for dev in /sys/bus/pci/devices/0000:01:00.1 /sys/bus/pci/devices/0000:01:00.0; do
    if [ -f "$dev/remove" ]; then
        echo 1 > "$dev/remove" 2>/dev/null || true
    fi
done
sleep 1
echo 1 > /sys/bus/pci/rescan
sleep 2

# Step 3: Disable D3cold (GPU only supports D0/D3hot)
for dev in /sys/bus/pci/devices/0000:00:01.0 /sys/bus/pci/devices/0000:01:00.0 /sys/bus/pci/devices/0000:01:00.1; do
    if [ -f "$dev/d3cold_allowed" ]; then
        echo 0 > "$dev/d3cold_allowed" 2>/dev/null || true
    fi
done

# Step 4: Verify GPU is accessible
if [ -f /sys/bus/pci/devices/0000:01:00.0/vendor ]; then
    VENDOR=$(cat /sys/bus/pci/devices/0000:01:00.0/vendor)
    if [ "$VENDOR" = "0x10de" ]; then
        echo "nvidia-gpu-setup: GPU detected, loading driver" | systemd-cat -t nvidia-gpu-setup
        modprobe --ignore-install nvidia
        modprobe --ignore-install nvidia-drm
        echo "nvidia-gpu-setup: driver loaded" | systemd-cat -t nvidia-gpu-setup
    else
        echo "nvidia-gpu-setup: GPU not detected after rescan" | systemd-cat -t nvidia-gpu-setup
        exit 1
    fi
else
    echo "nvidia-gpu-setup: GPU sysfs not found after rescan" | systemd-cat -t nvidia-gpu-setup
    exit 1
fi
```

```bash
sudo chmod +x /usr/local/sbin/nvidia-gpu-setup.sh
sudo systemctl enable nvidia-gpu-setup.service
```

### 2. Blacklist NVIDIA from Auto-Loading

The nvidia module must NOT load automatically via PCI modalias detection — it will try to probe the GPU before the service has re-enabled it, and hang.

Create `/etc/modprobe.d/nvidia-blacklist-autoload.conf`:

```
# Don't auto-load nvidia on PCI device detection
# nvidia-gpu-setup.service loads it after disabling D3cold
blacklist nvidia
blacklist nvidia-drm
blacklist nvidia-modeset
blacklist nvidia-uvm
```

### 3. NVIDIA Module Options

Create `/etc/modprobe.d/nvidia.conf`:

```
# ASUS ROG Zephyrus G16 GU605MY - nvidia power management
options nvidia NVreg_DynamicPowerManagement=0x02
options nvidia NVreg_EnableGpuFirmware=0
options nvidia NVreg_PreserveVideoMemoryAllocations=1
options nvidia NVreg_TemporaryFilePath=/var
options nvidia-drm modeset=1 fbdev=1
```

### 4. Udev Rules for D3cold

Create `/etc/udev/rules.d/99-nvidia-d3cold.rules`:

```
# Disable D3cold for NVIDIA GPU and its PCIe root port
ACTION=="add", KERNEL=="0000:01:00.0", SUBSYSTEM=="pci", ATTR{d3cold_allowed}="0"
ACTION=="add", KERNEL=="0000:00:01.0", SUBSYSTEM=="pci", ATTR{d3cold_allowed}="0"
ACTION=="add", KERNEL=="0000:01:00.1", SUBSYSTEM=="pci", ATTR{d3cold_allowed}="0"
```

### 5. Safe Boot GRUB Entry

Add to `/etc/grub.d/40_custom` so you always have a way to boot if nvidia breaks:

```
menuentry 'Kubuntu (safe - no nvidia)' --class kubuntu --class gnu-linux --class gnu --class os {
    load_video
    insmod gzio
    insmod part_gpt
    insmod ext2
    search --no-floppy --fs-uuid --set=root YOUR-UUID-HERE
    linux /boot/vmlinuz-KERNEL-VERSION root=UUID=YOUR-UUID-HERE ro quiet splash nomodeset module_blacklist=nvidia,nvidia_drm,nvidia_modeset,nvidia_uvm i915.enable_dpcd_backlight=1
    initrd /boot/initrd.img-KERNEL-VERSION
}
```

Also set GRUB to show the menu:

```
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=5
```

### 6. Driver Installation

Use the pre-built Canonical-signed modules — NOT DKMS — so Secure Boot works without MOK enrollment:

```bash
sudo apt install nvidia-driver-580-open linux-modules-nvidia-580-open-generic-hwe-24.04
```

The key package is `linux-modules-nvidia-580-open-6.17.0-19-generic` which contains modules signed by `Canonical Ltd. Kernel Module Signing`. These are trusted by Secure Boot out of the box.

**Do NOT install `nvidia-dkms-580-open` directly** — DKMS builds unsigned modules that Secure Boot will reject.

### 7. Rebuild Initramfs and GRUB

```bash
sudo update-initramfs -u -k $(uname -r)
sudo update-grub
```

**Important: Do NOT add nvidia modules to `/etc/initramfs-tools/modules` or `/etc/modules-load.d/`.** The modules must not load during early boot — they must only load after the systemd service has re-enabled the dGPU.

### 8. Dual-Boot Windows Fix

If you dual-boot Windows, open Armoury Crate and set the GPU mode to **"Auto"** (not "iGPU only"). The "iGPU only" setting writes `dgpu_disable=1` to the embedded controller, which persists across reboots into Linux.

## Brightness Fix

Brightness keys adjust the slider but the screen doesn't actually change. This is a known issue on this laptop where the NVIDIA driver intercepts backlight control instead of letting Intel i915 handle it.

Add to GRUB kernel parameters (`/etc/default/grub`):

```
i915.enable_dpcd_backlight=1
```

If the NVIDIA driver is loaded, also set these in `/etc/modprobe.d/nvidia.conf`:

```
options nvidia NVreg_EnableBacklightHandler=0
options nvidia NVreg_RegistryDwords=EnableBrightnessControl=0
```

Sources: [Arch Forums](https://bbs.archlinux.org/viewtopic.php?id=305237), [Ubuntu Bug #2112165](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2112165)

---

## Things That Were Tried But Didn't Fix It

These are documented because they address real issues that show the same symptoms — if `dgpu_disable` is not your problem, one of these might be.

### Wrong Driver Version (nvidia-driver-590-open)

The first attempt installed `nvidia-driver-590-open` which is a Blackwell/RTX 50-series driver. `ubuntu-drivers devices` recommended `nvidia-driver-580-open` for the RTX 4090 Laptop. The 590 driver may not properly support Ada Lovelace GPUs.

**Symptom**: Same D3cold error, plus boot hang.

**Lesson**: Always check `ubuntu-drivers devices` for the `recommended` tag.

### Secure Boot Rejection (DKMS-built modules)

Installing via DKMS (`nvidia-dkms-580-open`) builds modules from source. These are signed with a local MOK key (`/var/lib/shim-signed/mok/MOK.der`) that must be enrolled in firmware via `mokutil --import`. If the MOK key isn't enrolled, Secure Boot rejects the modules:

```
Loading of module with unavailable key is rejected
modprobe: ERROR: could not insert 'nvidia': Key was rejected by service
```

**Symptom**: System boots but nvidia-smi fails. `dmesg` shows key rejection.

**Lesson**: On Ubuntu with Secure Boot, use `linux-modules-nvidia-*-generic-hwe-*` packages (Canonical-signed) instead of DKMS.

### Loading NVIDIA Modules in Initramfs

Adding nvidia to `/etc/initramfs-tools/modules` causes the driver to probe the GPU extremely early in boot — before any opportunity to fix the power state. If the GPU is in D3cold or disabled, this guarantees a boot hang.

**Symptom**: Boot hangs immediately, even recovery mode is slow to start.

**Lesson**: Keep nvidia out of initramfs on hybrid GPU laptops with power management issues.

### Kernel Parameters That Don't Prevent D3cold

These parameters were tried and did not prevent the GPU from being inaccessible:

| Parameter | What It Does | Why It Didn't Help |
|-----------|-------------|-------------------|
| `pcie_port_pm=off` | Disables PCIe port power management | Doesn't affect embedded controller dGPU disable |
| `pcie_aspm=off` | Disables Active State Power Management | ASPM was already unsupported per FADT |
| `nvidia.NVreg_DynamicPowerManagement=0x02` | Fine-grained D3 power management | Only applies after driver loads; GPU was off before that |
| `nvidia.NVreg_EnableGpuFirmware=0` | Disables GSP firmware | Same — driver never gets to apply this |
| `nvidia-drm.modeset=1` | Kernel modesetting | Irrelevant if GPU is powered off |

These parameters may be useful for other D3cold issues (e.g., runtime suspend problems after the driver loads), but they cannot override a firmware-level dGPU disable.

### PCI Config Space as a Diagnostic

The definitive way to tell if the GPU is truly powered off vs. having a driver issue:

```bash
sudo lspci -s 01:00.0 -xxx | head -5
```

If every byte is `ff`:
```
00: ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff
```

The device is **hardware-disabled**. No driver fix will help. Check `dgpu_disable`, BIOS settings, or Armoury Crate GPU mode.

If you see real data (vendor/device IDs):
```
00: de 10 57 27 00 00 10 00 a1 00 00 03 00 00 80 00
```

The device is powered on and your issue is elsewhere (driver signing, module options, etc.).

## How to Check dgpu_disable

```bash
cat /sys/devices/platform/asus-nb-wmi/dgpu_disable
```

- `0` = dGPU enabled (normal)
- `1` = dGPU disabled by firmware (GPU is powered off)

To re-enable manually:

```bash
echo 0 | sudo tee /sys/devices/platform/asus-nb-wmi/dgpu_disable
# Then remove stale PCI entries and rescan
echo 1 | sudo tee /sys/bus/pci/devices/0000:01:00.1/remove
echo 1 | sudo tee /sys/bus/pci/devices/0000:01:00.0/remove
sleep 1
echo 1 | sudo tee /sys/bus/pci/rescan
```

## Working Configuration Summary

```
Driver:     nvidia-driver-580-open (580.126.09)
Modules:    Canonical-signed (linux-modules-nvidia-580-open-6.17.0-19-generic)
Secure Boot: Enabled (no MOK enrollment needed)
dGPU:       Re-enabled at boot via systemd service
D3cold:     Disabled via udev rules + systemd service
Module load: Deferred (blacklisted, loaded by service after dGPU enable)
GRUB:       pcie_port_pm=off pcie_aspm=off i915.enable_dpcd_backlight=1
```

## References

- [linux-on-zephyrus (GitHub)](https://github.com/lihe07/linux-on-zephyrus) — Linux on ROG Zephyrus G16 Air (2024)
- [GU605MI Linux Guide](https://www.ehmiiz.se/blog/linux_asus_g16_2024/) — Sound, Keyboard, Brightness & asusctl
- [asus-linux.org](https://asus-linux.org/faq/) — Linux for ROG Notebooks
- [NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/ubuntu-24-04-unable-to-change-power-state-from-d3cold-to-d0-device-inaccessible/304459) — D3cold power state issue
- [Arch Wiki — NVIDIA/Troubleshooting](https://wiki.archlinux.org/title/NVIDIA/Troubleshooting)
- [Arch Forums — Asus G16 brightness](https://bbs.archlinux.org/viewtopic.php?id=305237)
- [Ubuntu Bug #2112165](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2112165) — Brightness control on ASUS ROG Zephyrus
