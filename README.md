# ASUS ROG Zephyrus G16 (GU605MY) — Linux Fixes

Guides and fixes for running Linux on the ASUS ROG Zephyrus G16 (2024) with Intel Meteor Lake-P, NVIDIA GeForce RTX 4090 Laptop GPU, and an OLED display. Tested on Kubuntu 24.04 with kernel 6.17.

This laptop has several issues out of the box on Linux — the dGPU won't initialize, brightness controls don't work, and sleep/wake crashes the system. This repo documents the root causes and working solutions found through extensive troubleshooting, including the many dead ends along the way.

## Hardware

| Component | Details |
|-----------|---------|
| Model | ASUS ROG Zephyrus G16 GU605MY |
| CPU | Intel Core Ultra (Meteor Lake-P) |
| iGPU | Intel Arc Graphics |
| dGPU | NVIDIA GeForce RTX 4090 Laptop GPU (16GB GDDR6) |
| Display | OLED |
| WiFi | Intel BE200 |
| Audio | Cirrus Logic CS35L56 + Realtek ALC285 |

## Guides

| File | Description |
|------|-------------|
| [ASUS-ROG-Zephyrus-G16-GU605MY-Linux-NVIDIA-Fix.md](ASUS-ROG-Zephyrus-G16-GU605MY-Linux-NVIDIA-Fix.md) | Getting the NVIDIA RTX 4090 Laptop GPU working on Linux. The dGPU is hardware-disabled by ASUS firmware (`dgpu_disable=1`), causing "Unable to change power state from D3cold to D0" and "GPU has fallen off the bus" errors. Includes the systemd service fix, Secure Boot considerations, brightness control, and a detailed log of approaches that didn't work but address similar symptoms. |

## Keywords

ASUS ROG Zephyrus G16, GU605MY, GU605MZ, GU605MI, NVIDIA RTX 4090 Laptop, D3cold, dgpu_disable, "Unable to change power state from D3cold to D0", "GPU has fallen off the bus", Linux, Ubuntu, Kubuntu 24.04, Secure Boot, asus-nb-wmi, Armoury Crate, Intel Meteor Lake, brightness, sleep, suspend
