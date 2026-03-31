# ASUS ROG Zephyrus G16 (GU605MY) — Linux Fixes

Guides and fixes for running Linux on the ASUS ROG Zephyrus G16 (2024) with an Intel Core Ultra 9 185H, NVIDIA GeForce RTX 4090 Laptop GPU, and a 2560x1600 OLED display. Tested on Kubuntu 24.04 LTS with kernel 6.17.

This laptop has several issues out of the box on Linux — the dGPU won't initialize, brightness controls don't work, and sleep/wake crashes the system. This repo documents the root causes and working solutions found through extensive troubleshooting, including the many dead ends along the way.

## Hardware Specifications

### System

| Component | Details |
|-----------|---------|
| Manufacturer | ASUSTeK COMPUTER INC. |
| Model | ROG Zephyrus G16 GU605MY_GU605MY |
| Family | ROG Zephyrus G16 |
| BIOS | American Megatrends International, GU605MY.329 (2025-06-06) |
| BIOS Revision | 5.32 |
| Firmware Revision | 3.27 |
| Secure Boot | Enabled |

### Processor

| Component | Details |
|-----------|---------|
| CPU | Intel Core Ultra 9 185H (Meteor Lake-P) |
| Cores | 16 (6P + 8E + 2LPE) |
| Threads | 22 |
| Base / Max Freq | 400 MHz / 5100 MHz |
| L2 Cache | 18 MiB |
| L3 Cache | 24 MiB |

### Memory

| Component | Details |
|-----------|---------|
| Total RAM | 32 GB (8x 4GB modules) |
| Type | LPDDR5 |
| Speed | 7467 MT/s |
| Manufacturer | Samsung |
| Form Factor | Row Of Chips (soldered) |

### Graphics

| Component | PCI ID | Details |
|-----------|--------|---------|
| iGPU | `8086:7d55` | Intel Meteor Lake-P [Intel Arc Graphics] |
| dGPU | `10de:2757` | NVIDIA GeForce RTX 4090 Laptop GPU (GN21-X11, AD103M, 16GB GDDR6) |
| dGPU Audio | `10de:22bb` | NVIDIA HDA Audio (on dGPU bus 01:00.1) |

### Display

| Component | Details |
|-----------|---------|
| Panel | Samsung ATNA60DL01-0 |
| Type | OLED |
| Resolution | 2560x1600 |
| Connector | eDP |

### Storage

| Component | Details |
|-----------|---------|
| SSD | Micron 3400 (Hendrix) 2TB NVMe `[1344:5407]` |

### Networking

| Component | PCI ID | Details |
|-----------|--------|---------|
| WiFi | `8086:7e40` | Intel Meteor Lake PCH CNVi WiFi (Intel BE200) |
| Bluetooth | — | Intel AX211 Bluetooth (USB `8087:0033`) |

### Audio

| Component | PCI ID | Details |
|-----------|--------|---------|
| HDA Controller | `8086:7e28` | Intel Meteor Lake-P HD Audio Controller |
| Codec | — | Realtek ALC285 |
| Amplifiers | — | 2x Cirrus Logic CS35L56 Rev B0 (SPI, fw 3.4.4) |
| System Name | — | `10431C63-spkid0` (AMP1, AMP2) |

### Input Devices

| Component | Details |
|-----------|---------|
| Keyboard | Asus Keyboard (USB `0b05:19b6`), AT Raw Set 2 |
| Touchpad | ASUF1207:00 `2808:0219` (I2C HID) |
| Hotkeys | Asus WMI hotkeys (via asus-nb-wmi) |

### Other Hardware

| Component | PCI ID | Details |
|-----------|--------|---------|
| NPU | `8086:7d1d` | Intel Meteor Lake NPU |
| Thunderbolt | `8086:7ec4` | Meteor Lake-P Thunderbolt 4 |
| USB Controller | `8086:7ec0` | Meteor Lake-P Thunderbolt 4 USB Controller |
| USB Controller | `8086:7e7d` | Meteor Lake-P USB 3.2 Gen 2x1 xHCI |
| Card Reader | `10ec:525a` | Realtek RTS525A PCI Express Card Reader |
| Webcam | — | Shinetech USB2.0 FHD UVC WebCam (USB `3277:0051`) |

### Battery

| Component | Details |
|-----------|---------|
| Model | A32-K55 |
| Technology | Lithium-ion |
| Design Capacity | 90.0 Wh |
| Voltage | ~15V |

### Full PCI Device List

```
00:00.0 Host bridge [0600]: Intel Corporation Device [8086:7d01] (rev 04)
00:01.0 PCI bridge [0604]: Intel Corporation Device [8086:7ecc] (rev 10)
00:02.0 VGA compatible controller [0300]: Intel Corporation Meteor Lake-P [Intel Arc Graphics] [8086:7d55] (rev 08)
00:04.0 Signal processing controller [1180]: Intel Corporation Device [8086:7d03] (rev 04)
00:06.0 PCI bridge [0604]: Intel Corporation Device [8086:7e4d] (rev 20)
00:07.0 PCI bridge [0604]: Intel Corporation Meteor Lake-P Thunderbolt 4 PCI Express Root Port #0 [8086:7ec4] (rev 10)
00:08.0 System peripheral [0880]: Intel Corporation Device [8086:7e4c] (rev 20)
00:0a.0 Signal processing controller [1180]: Intel Corporation Device [8086:7d0d] (rev 01)
00:0b.0 Processing accelerators [1200]: Intel Corporation Meteor Lake NPU [8086:7d1d] (rev 04)
00:0d.0 USB controller [0c03]: Intel Corporation Meteor Lake-P Thunderbolt 4 USB Controller [8086:7ec0] (rev 10)
00:0d.2 USB controller [0c03]: Intel Corporation Meteor Lake-P Thunderbolt 4 NHI #0 [8086:7ec2] (rev 10)
00:14.0 USB controller [0c03]: Intel Corporation Meteor Lake-P USB 3.2 Gen 2x1 xHCI Host Controller [8086:7e7d] (rev 20)
00:14.2 RAM memory [0500]: Intel Corporation Device [8086:7e7f] (rev 20)
00:14.3 Network controller [0280]: Intel Corporation Meteor Lake PCH CNVi WiFi [8086:7e40] (rev 20)
00:15.0 Serial bus controller [0c80]: Intel Corporation Meteor Lake-P Serial IO I2C Controller #0 [8086:7e78] (rev 20)
00:15.3 Serial bus controller [0c80]: Intel Corporation Meteor Lake-P Serial IO I2C Controller #3 [8086:7e7b] (rev 20)
00:16.0 Communication controller [0780]: Intel Corporation Device [8086:7e70] (rev 20)
00:1c.0 PCI bridge [0604]: Intel Corporation Device [8086:7e3b] (rev 20)
00:1e.0 Communication controller [0780]: Intel Corporation Meteor Lake-P Serial IO UART Controller #0 [8086:7e25] (rev 20)
00:1e.2 Serial bus controller [0c80]: Intel Corporation Meteor Lake-P Serial IO SPI Controller #0 [8086:7e27] (rev 20)
00:1f.0 ISA bridge [0601]: Intel Corporation Device [8086:7e02] (rev 20)
00:1f.3 Audio device [0403]: Intel Corporation Meteor Lake-P HD Audio Controller [8086:7e28] (rev 20)
00:1f.4 SMBus [0c05]: Intel Corporation Meteor Lake-P SMBus Controller [8086:7e22] (rev 20)
00:1f.5 Serial bus controller [0c80]: Intel Corporation Meteor Lake-P SPI Controller [8086:7e23] (rev 20)
00:1f.7 Non-Essential Instrumentation [1300]: Intel Corporation Meteor Lake-P Trace Hub [8086:7e24] (rev 20)
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GN21-X11 [10de:2757] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:22bb] (rev a1)
02:00.0 Non-Volatile memory controller [0108]: Micron Technology Inc 3400 NVMe SSD [Hendrix] [1344:5407]
2d:00.0 Unassigned class [ff00]: Realtek Semiconductor Co., Ltd. RTS525A PCI Express Card Reader [10ec:525a] (rev 01)
```

### Software Environment

| Component | Details |
|-----------|---------|
| OS | Kubuntu 24.04.4 LTS (Noble Numbat) |
| Kernel | 6.17.0-19-generic |
| NVIDIA Driver | nvidia-driver-580-open (580.126.09, Canonical-signed) |
| CUDA | 13.0 |

## Guides

| File | Description |
|------|-------------|
| [ASUS-ROG-Zephyrus-G16-GU605MY-Linux-NVIDIA-Fix.md](ASUS-ROG-Zephyrus-G16-GU605MY-Linux-NVIDIA-Fix.md) | Getting the NVIDIA RTX 4090 Laptop GPU working on Linux. The dGPU is hardware-disabled by ASUS firmware (`dgpu_disable=1`), causing "Unable to change power state from D3cold to D0" and "GPU has fallen off the bus" errors. Includes the systemd service fix, Secure Boot considerations, brightness control, and a detailed log of approaches that didn't work but address similar symptoms. |

## Keywords

ASUS ROG Zephyrus G16, GU605MY, GU605MZ, GU605MI, NVIDIA GeForce RTX 4090 Laptop GPU, GN21-X11, 10de:2757, AD103M, Intel Core Ultra 9 185H, Meteor Lake-P, Intel Arc Graphics, D3cold, dgpu_disable, asus-nb-wmi, "Unable to change power state from D3cold to D0", "GPU has fallen off the bus", "device inaccessible", Linux, Ubuntu, Kubuntu 24.04, Secure Boot, Armoury Crate, OLED brightness, sleep, suspend, Cirrus Logic CS35L56, ATNA60DL01
