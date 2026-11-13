<div align="center">

# ⚡ PROJECT EXTREME++
### BMW Edition · POCO F4 / Redmi K40S (`munch` / `munch-in`)
**Linux 4.19.325-cip135-st19 · Snapdragon 870 5G · HyperOS + AOSP**

[![Build Status](https://img.shields.io/github/actions/workflow/status/PandeyJI-9/Extreme-PLUS-kernel-munch/build.yml?branch=main&style=for-the-badge&logo=github&label=Build%20%26%20Release&color=00c853)](https://github.com/PandeyJI-9/Extreme-PLUS-kernel-munch/actions)
[![Kernel](https://img.shields.io/badge/Linux-4.19.325--cip135--st19-007acc?style=for-the-badge&logo=linux&logoColor=white)](https://github.com/PandeyJI-9/Extreme-PLUS-kernel-munch)
[![Device](https://img.shields.io/badge/POCO_F4-munch%20%2F%20munch--in-ff6f00?style=for-the-badge&logo=xiaomi&logoColor=white)](https://github.com/PandeyJI-9/Extreme-PLUS-kernel-munch)
[![SoC](https://img.shields.io/badge/Snapdragon_870-SM8250--AC-red?style=for-the-badge&logo=qualcomm&logoColor=white)](https://www.qualcomm.com/products/application/smartphones/snapdragon-8-series-mobile-platforms/snapdragon-870-5g-mobile-platform)
[![Targets](https://img.shields.io/badge/Targets-HyperOS%201.0--4.0%20%7C%20AOSP%2013--17-success?style=for-the-badge&logo=android&logoColor=white)](https://github.com/PandeyJI-9/Extreme-PLUS-kernel-munch)
[![Root](https://img.shields.io/badge/Root-KowSU%20%2B%20SuSFS%20v2.3.0-9c27b0?style=for-the-badge&logo=android&logoColor=white)](https://github.com/ReSukiSU/ReSukiSU)
[![Community](https://img.shields.io/badge/Community-Telegram_Support-29b6f6?style=for-the-badge&logo=telegram&logoColor=white)](https://t.me/Extremeplus_Support)

> *"Tune the silicon. Do not fight it."*

</div>

---

## 📌 Release Status

This README documents the current EXTREME++ source line, not older builds.

| Item | Current state |
| :--- | :--- |
| **Source line** | **Build #132** (`a035ee8`) |
| **Kernel base** | AstideLabs SM8250 4.19, PR #74 merge (`fcf3667`) |
| **Kernel release** | `4.19.325-cip135-st19` |
| **Default governor** | `extreme+` |
| **Default ZRAM compressor** | **ZSTD level 3** |
| **Build system** | GitHub Actions · ZyC-Clang 16.0.6 · LLD · ccache |
| **Primary target** | Xiaomi HyperOS |
| **Secondary target** | AOSP / custom ROMs |

### Target matrix

| Target | Supported range | Status | Notes |
| :--- | :---: | :---: | :--- |
| **Xiaomi HyperOS** | HyperOS 1.0–4.0 · Android 14–17 | 🟢 **Primary** | Xiaomi subsystem path, MIGT/MIHW/RTMM/MILLET compatibility |
| **AOSP / custom ROMs** | Android 13–17 · A16/A17 target | 🟡 **Supported** | Clean AOSP path; ZSTD ZRAM verified; verify governor/profile on your ROM |

> [!NOTE]
> The kernel ships as separate **HyperOS** and **AOSP** AnyKernel ZIPs. Flash the ZIP that matches your ROM. Do not mix them.

---

## 🧭 Engineering Rules

EXTREME++ is built around a small set of non-negotiable rules:

1. **Keep the proven scheduling algorithm.** EXTREME+ is a stock AstideLabs CAF/WALT schedutil governor with a profile ceiling layer, not a rewritten scheduler.
2. **Never force a frequency floor.** No profile sets `policy->min`. Every cluster can reach its lowest OPP; Little cores may drop to 300 MHz when userspace/ROM selects it.
3. **Make EXTREME+ the boot default.** `CONFIG_CPU_FREQ_GOV_EXTREME_PLUS=y` and `CONFIG_CPU_FREQ_DEFAULT_GOV_EXTREME_PLUS=y`.
4. **Keep ZSTD as the ZRAM default.** LZO/LZO-RLE/LZ4 remain fallback compressors only.
5. **Do not fight userspace.** The kernel exposes the profile ceiling; EKM, FKM, PowerHAL, and thermal services may still lower limits.
6. **Preserve stock vendor-module ABI.** The kernel is built to remain compatible with the stock vendor module set.
7. **Flash the AnyKernel ZIP, not raw images.** The Raw Images ZIP is for inspection and HyperOS repacking only.

---

## 🧠 EXTREME+ Governor & EKM Profiles

EXTREME+ keeps the stock CAF/WALT schedutil utilization path and adds a **maximum-frequency ceiling plus matching schedutil rate-limit tuning** for the three CPU clusters.

```
policy0  CPU 0-3   Little   stock schedutil + optional ceiling
policy4  CPU 4-6   Gold     stock schedutil + profile ceiling
policy7  CPU 7     Prime    stock schedutil + profile ceiling
```

### Profile table

| Profile | sysfs value | Prime (CPU 7) | Gold (CPU 4-6) | Little (CPU 0-3) |
| :--- | :---: | :---: | :---: | :---: |
| **Gaming** | `0` | `3187200` kHz | `2419200` kHz | Stock range |
| **Balanced** | `1` | `2841600` kHz | `2246400` kHz | Stock range |
| **Battery** | `2` | `2419200` kHz | `1958400` kHz | Stock range |

- **Gaming** is the default profile at boot.
- A profile applies a maximum ceiling and its schedutil rate-limit/hispeed settings. It never forces a frequency or sets `policy->min`.
- Thermal and QoS limits can still lower the ceiling below the selected profile.
- There is **no 576 MHz Little-core floor** and no forced Gold/Prime performance lock.
- Profiles are not persisted across reboot; the default returns to Gaming.

### Control interface

Both paths are root-writable `0664`:

```bash
# Read current profile
cat /sys/kernel/extreme_profile

# Numeric form
echo 1 > /sys/kernel/extreme_profile     # balanced

# Named aliases are accepted too
echo battery > /sys/kernel/profile
echo gaming  > /sys/kernel/profile
```

Accepted names:

| Value | Aliases |
| :---: | :--- |
| `0` | `gaming`, `performance`, `perf` |
| `1` | `balanced`, `bal`, `default` |
| `2` | `battery`, `powersave`, `bat` |

### Boot governor behavior

The kernel image defaults to `extreme+`. If a ROM, EKM/FKM profile, KernelSU module, or `/data/adb` script writes `schedutil` after boot, userspace is overriding the kernel default. Check those components before changing kernel code.

---

## 🏎️ Adreno 650: FakeDreamer OPP & Undervolt

The GPU uses the FakeDreamer Adreno 650 OPP table with seven active states:

| Frequency | RPMh level | Typical use |
| :---: | :---: | :--- |
| **670 MHz** | `SVS_L2` (225) | Peak gaming |
| **587 MHz** | `SVS_L1` (129) | Sustained heavy rendering |
| **510 MHz** | `SVS` (66) | Efficient gaming |
| **400 MHz** | `LOW_SVS` (1) | UI and multitasking |
| **330 MHz** | `LOW_SVS` (1) | Video and light composition |
| **205 MHz** | `MIN_SVS` (1) | Low-power display workloads |
| **150 MHz** | `MIN_SVS` (1) | Idle / low-power state |

- Native Qualcomm **`msm-adreno-tz`** devfreq governor is preserved.
- The top three GPU states use DDR bus tier **11** for maximum memory bandwidth.
- This is an **undervolt and OPP-table integration**, not a 670 MHz overclock. 670 MHz remains the Adreno 650 stock peak.
- The GPU table is injected through the DTB/vendor_boot path; the AnyKernel ZIP writes the matching DTB.

---

## 💾 Memory, ZRAM & VM Behavior

### ZRAM

- Kernel default compressor: **ZSTD**
- Default compression level: **3**
- Runtime tunable: `/sys/module/zstd/parameters/compression_level` (`1`–`3`)
- LZO remains compiled as a vendor-ABI fallback. The build explicitly disables LZO, LZO-RLE, LZ4, LZ4HC, and 842 as **default** compressor choices.
- **The kernel does not choose the ZRAM disk size.** The size is set by ROM/userspace. An AOSP report showed a 4 GiB zram0 device with `[zstd]` active; that size is not hardcoded by this kernel.

### VM anti-stutter

The kernel includes a direct-reclaim throttle bypass for critical tasks:

- real-time tasks
- tasks with `oom_score_adj <= -900`

This keeps latency-sensitive UI/compositor work from being put to sleep during direct reclaim. It is a targeted bypass, not a global reclaim disable.

### Networking

- **BBR** congestion control is the default TCP congestion algorithm.
- **FQ** and **FQ-CoDel** packet schedulers are enabled.
- Useful for low-latency gaming and stable throughput under load.

---

## 🔌 Charging & Power

- **PD/PPS minimum adapter requirements:** 8.5 V / 1.5 A, improving compatibility with third-party PD/PPS chargers.
- **Stock Xiaomi 67 W Mi Turbo authentication path is preserved.** The BMS digest and `mi_thermald` handshake are not replaced by this kernel.
- **Hardware bypass charging:** exposed through the battery `input_suspend` interface; the charger can power the system rail while battery charging is inhibited.
- **JEITA thermal steps:** the charger current is stepped down as battery temperature rises instead of running a single unrestricted current.
- **Thermal and battery safety behavior remains active.** Charging speed depends on the charger, cable, battery temperature, and ROM power stack.

---

## 🛡️ Root, Compatibility & Security

### Root variants

| Build mode | Contents | Best for |
| :--- | :--- | :--- |
| **`ksu_susfs`** | KowSU/ReSukiSU multi-manager + SuSFS v2.3.0 | Default rooted build |
| **`ksu_no_susfs`** | KowSU multi-manager, standard KSU path | Root without SuSFS features |
| **`noroot`** | Pure clean kernel | No root binaries |
| **`both`** | `noroot` + `ksu_susfs` | Two-package release |
| **`all`** | `noroot` + `ksu_no_susfs` + `ksu_susfs` | Complete suite |

The `ksu_susfs` build uses the ReSukiSU/KowSU non-GKI multi-manager path and SuSFS v2.3.0 features, including path/mount/kstat hiding, uname spoofing, open redirect, and SUS map support. The `ksu_no_susfs` path keeps the same manager with SuSFS features disabled.

### Bootloader command-line spoof

The kernel patches `/proc/cmdline` for normal post-boot processes with a **PID-1/AVB safety guard**:

- PID 1 (`init`) and `ueventd` receive the genuine command line so Android can mount verified partitions.
- Normal post-boot processes see spoofed `verifiedbootstate` / `device_state` values.

> [!IMPORTANT]
> This is a command-line-level spoof, not a universal Play Integrity guarantee. App-level integrity depends on the ROM, Play Integrity fixes, key attestation, and the apps being tested.

### HyperOS and AOSP paths

| Area | HyperOS build | AOSP build |
| :--- | :--- | :--- |
| Xiaomi MIGT/MIHW/RTMM/MILLET | Enabled | Disabled |
| Millet dual-ABI netlink compatibility | Enabled | Compiled out |
| ReKernel | Disabled | Enabled |
| OverlayFS | Enabled | Enabled |
| Static user-mode helper | `/system/bin/micd` | Empty |
| Root | Optional | Optional |

AOSP builds are intentionally clean: Xiaomi-only subsystems are not forced into a non-Xiaomi userspace.

---

## 📥 Installation

> [!WARNING]
> **Never flash the Raw Images ZIP.** It is not an AnyKernel package and is intended for inspection/HyperOS repacking only. Use the main AnyKernel ZIP for normal installation.

### Before flashing

- Device: **POCO F4 / Redmi K40S** (`munch` / `munch-in`)
- Back up your current **boot** and **vendor_boot** partitions.
- Use the ZIP that matches your ROM:
  - `..._HyperOS_...zip`
  - `..._AOSP_...zip`
- Keep a working recovery or fastboot image available.

### Recovery installation

1. Download the correct AnyKernel ZIP from **GitHub Actions** or **Releases**.
2. Boot into OrangeFox, TWRP, or another compatible recovery.
3. Flash the ZIP.
4. Reboot to system.
5. If you use a rooted build, install a supported KernelSU/KowSU manager.

### Kernel-manager installation

1. Open FKM, SmartPack, or a compatible kernel flasher.
2. Select the AnyKernel ZIP.
3. Flash and reboot.

### What the ZIP does

- Writes the kernel `Image` to the active **boot** partition.
- Writes the matching **DTB** to the active **vendor_boot** partition.
- Purges known legacy `/data/adb` scripts from older EXTREME++ builds.
- Does **not** install a boot-time governor script or a ZRAM-size script.

### HyperOS image repacking

The `hyperos4_images/` output and the Raw Images ZIP are HyperOS-only artifacts. They are not AOSP packages. Normal users should use the main HyperOS AnyKernel ZIP.

---

## 🔍 Post-Boot Verification

Run these from Termux or ADB shell.

### Kernel identity

```bash
uname -r
# HyperOS: 4.19.325-cip135-st19-Extreme++Gaming-HyperOS-BMW-gfcf366797808
# AOSP:    4.19.325-cip135-st19-Extreme++Gaming-AOSP-BMW-gfcf366797808
```

### Active governor

```bash
for p in /sys/devices/system/cpu/cpufreq/policy*; do
  echo "${p##*/}: $(cat "$p/scaling_governor")"
done
# Expected: extreme+ on policy0, policy4, and policy7
```

If you see `schedutil`, inspect userspace writers before blaming the kernel. See Troubleshooting below.

### Active profile

```bash
cat /sys/kernel/extreme_profile
# 0 = gaming, 1 = balanced, 2 = battery
```

### CPU limits

```bash
for p in /sys/devices/system/cpu/cpufreq/policy*; do
  echo "${p##*/} max=$(cat "$p/scaling_max_freq") min=$(cat "$p/scaling_min_freq")"
done
```

A profile changes the maximum ceiling and rate-limit tuning. It does not force `scaling_min_freq` to a high value; lowest OPPs remain available.

### ZRAM

```bash
cat /sys/block/zram0/comp_algorithm
# Expected active entry: [zstd]

cat /sys/block/zram0/disksize
cat /proc/swaps
```

The compressor is a kernel/runtime feature; the **disk size** is selected by ROM/userspace.

### GPU

```bash
cat /sys/class/kgsl/kgsl-3d0/gpu_available_frequencies
# Expected set: 150000000, 205000000, 330000000, 400000000,
#               510000000, 587000000, 670000000 (order may vary)
```

### Command-line spoof

```bash
cat /proc/cmdline
```

From a normal post-boot shell, the spoofed values should be visible. PID 1 intentionally receives the genuine command line.

---

## 🧰 Troubleshooting

### Governor changes to `schedutil` after boot

The kernel default is `extreme+`. A delayed change is almost always userspace:

1. Check FKM/EKM **apply-on-boot** profiles.
2. Check KernelSU modules and `/data/adb/service.d`.
3. Search for scripts that write `scaling_governor`.
4. Set the governor to `extreme+` in your manager.
5. Reboot and re-check immediately and after ~20 seconds.

The kernel does not add a background governor-enforcement hook, because that would fight ROM/PowerHAL and can cause policy churn.

### ZRAM is not using ZSTD

Check the active compressor:

```bash
cat /sys/block/zram0/comp_algorithm
```

If `[zstd]` is not active, check the ROM's ZRAM setup. The kernel supports ZSTD; userspace selects the active compressor. The kernel also does not force a 4 GiB or 6 GiB ZRAM size.

### Profile write is denied

Use root:

```bash
su -c 'echo 1 > /sys/kernel/extreme_profile'
```

Both `/sys/kernel/extreme_profile` and `/sys/kernel/profile` are root-writable `0664`.

### AOSP-specific notes

- Use the AOSP AnyKernel ZIP, not the HyperOS ZIP.
- AOSP ROMs and PowerHAL implementations may change the governor after boot. Check userspace first.
- ZRAM size and compressor selection are AOSP userspace decisions.

### Do not flash raw images

If you downloaded a ZIP containing `Image`, `dtb`, `boot.img`, or `vendor_boot.img`, verify its purpose first. The Raw Images ZIP is not a flashable package.

---

## ⚙️ Build Pipeline

The kernel is built by **GitHub Actions** using the workflow in `.github/workflows/build.yml`.

- **Source base:** AstideLabs SM8250 4.19, pinned to PR #74 merge `fcf3667`.
- **Compiler:** ZyC-Clang 16.0.6 with LLD and ccache.
- **Build host:** Ubuntu 22.04 runner with a 10 GB swap workspace to prevent compiler OOM.
- **Targets:** HyperOS and/or AOSP.
- **Root modes:** `ksu_susfs`, `ksu_no_susfs`, `noroot`, `both`, `all`.
- **Outputs:** AnyKernel ZIPs, a HyperOS Raw Images ZIP for inspection/repacking, and HyperOS repacked images.
- **Telegram delivery:** configured CI notifications and artifact delivery remain part of the release workflow.

### Build guards

The build fails early if any of these are missing:

- `CONFIG_CPU_FREQ_GOV_EXTREME_PLUS=y`
- `CONFIG_CPU_FREQ_DEFAULT_GOV_EXTREME_PLUS=y`
- `CONFIG_ZRAM=y`
- `CONFIG_CRYPTO_ZSTD=y`
- `CONFIG_ZSTD_COMPRESS=y`
- `CONFIG_ZSTD_DECOMPRESS=y`
- `CONFIG_ZRAM_DEF_COMP_ZSTD=y`
- ZSTD level-3 default and safe workspace allocation

This prevents a release from silently shipping the wrong governor or compressor.

---

## 🤝 Community & Support

- 💬 **Telegram Support:** [@Extremeplus_Support](https://t.me/Extremeplus_Support)
- 👨‍💻 **Maintainer:** **Ayush Pandey JI** ([@pandey_ji_8](https://t.me/pandey_ji_8))
- 🐞 **Issues:** [GitHub Issues](https://github.com/PandeyJI-9/Extreme-PLUS-kernel-munch/issues)

When reporting a problem, include:

```bash
uname -r
cat /sys/devices/system/cpu/cpufreq/policy*/scaling_governor
cat /sys/kernel/extreme_profile
cat /sys/block/zram0/comp_algorithm
```

Also mention your ROM, Android version, root manager, and whether you flashed the HyperOS or AOSP ZIP.

---

## 🏆 Credits

- **Ayush Pandey JI** — Lead architect and maintainer of EXTREME++.
- **FakeDreamer / re-noroi** — Adreno 650 RPMh voltage stepping and undervolt tables.
- **ReSukiSU, KowSU, and KOWX712** — Non-GKI multi-manager root and SuSFS v2.3.0 integration.
- **AstideLabs** — SM8250 Linux 4.19 kernel foundation.
- **osm0sis** — AnyKernel3 backend.

---

<details>
<summary><strong>📜 Disclaimer</strong></summary>

```c
/*
 * Your warranty is now void.
 * I am not responsible for bricked devices, dead SD cards, thermonuclear war,
 * or you getting fired because the alarm app failed.
 *
 * Do your research before flashing. YOU choose to modify your device.
 */
```

Kernel source follows the licensing terms of the upstream Linux kernel and included projects. All upstream copyrights remain with their respective owners.

</details>

<div align="center">
<b>Built for the POCO F4 community. Keep it stable, keep it fast. 🚀</b>
</div>
