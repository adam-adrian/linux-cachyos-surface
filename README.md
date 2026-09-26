# CachyOS Kernel for Surface Devices

This repository contains the PKGBUILD and patches needed to build an optimized [CachyOS](https://github.com/CachyOS/linux-cachyos) kernel with [linux-surface](https://github.com/linux-surface/linux-surface) support, tailored with custom hardware fixes for Microsoft Surface devices (particularly tested on the Surface Pro 5 / model 1796).

This branch (`7.2`) targets the **Linux 7.2** mainline series (currently `7.2.7-1`), pulling upstream CachyOS base optimizations (BORE scheduler, BBR3, ntsync, zstd) together with hardware patches for Surface devices.

---

## Hardware Patches Included

In addition to upstream `linux-surface` patches, this branch includes three dedicated hardware fixes:

1. **`0001-surface-button-eprobe-defer.patch` (Physical Button Race Fix)**:
   - Fixes a boot-time race condition where volume and power buttons fail to register on Surface devices.
   - Converts `-ENOENT` return to `-EPROBE_DEFER` in `soc_button_array.c` and adds a soft dependency on `pinctrl_sunrisepoint` so GPIO pin controllers are ready before probing.

2. **`0002-ov8865-stale-mode-fix.patch` (Rear Camera Greenscreen & Stall Fix)**:
   - Fixes an issue where the OmniVision OV8865 rear camera fails to stream (producing a solid green frame or hanging) when restarted or when used with the DW9719 voice-coil focus motor.
   - Forces full sensor register reprogramming on stream start whenever `hw_mode != mode`.

3. **`0003-ov8865-surface-hflip-polarity.patch` (Rear Camera Horizontal Mirror Fix)**:
   - Reverses the horizontal flip polarity in `ov8865_flip_horz_configure()` specifically for Microsoft Surface DMI matches (`Microsoft Corporation`).
   - Fixes horizontally inverted video at the driver/register level without breaking `libcamera`'s automatic transform calculations (`VFLIP=1, HFLIP=0`).

---

## Build & CI Configuration

Building kernels locally on tablet hardware like the Surface Pro can be slow and painful. Compilation is designed to be offloaded to **GitHub Actions CI**.

Key build configurations:
- **Clang ThinLTO (`_use_llvm_lto:=thin`)**: Configured with ThinLTO instead of Full LTO. Full LTO exceeds 20 GB of memory during `ld.lld vmlinux.o` on 16 GB CI runners, triggering Out-Of-Memory (OOM Error 137). ThinLTO runs multi-threaded within ~3–4 GB of RAM while preserving link-time optimizations.
- **Compiler Caching (`ccache`)**: Configured with persistent GitHub Actions caching, significantly reducing incremental build times.
- **Landlock Compatibility**: Pacman 7.0 download sandbox restrictions are handled in the container runtime (`--security-opt seccomp=unconfined`).

---

## Installation Instructions

### 1. Build via GitHub Actions CI (Recommended)

1. Fork or push to your branch (`7.2`).
2. Trigger the `Build CachyOS Surface Kernel` workflow from the **Actions** tab (or via `workflow_dispatch`).
3. Download the built `.pkg.tar.zst` artifacts from the completed run.

### 2. Build Locally from Source

To build the packages locally using `makepkg`:

```bash
sudo pacman -S base-devel
git clone -b 7.2 https://github.com/adam-adrian/linux-cachyos-surface.git
cd linux-cachyos-surface/linux-cachyos-surface
makepkg -si
```

_**NOTE:** Ensure your machine has at least 16 GB of available RAM/swap. ThinLTO is enabled by default in the PKGBUILD._

### 3. Install Prebuilt Packages

After downloading or compiling the packages, install them using `pacman`:

```bash
sudo pacman -U linux-cachyos-surface-7.2.*.pkg.tar.zst linux-cachyos-surface-headers-7.2.*.pkg.tar.zst
```

If using a Unified Kernel Image (UKI) or systemd-boot / rEFInd:

```bash
sudo mkinitcpio -P
```

---

## Acknowledgements

- **Maximilian Luz & contributors**: [linux-surface/linux-surface](https://github.com/linux-surface/linux-surface)
- **Peter Jung & CachyOS team**: [CachyOS/linux-cachyos](https://github.com/CachyOS/linux-cachyos)
- **Apiznel**: Initial physical button probe deferral fix (PR #2233)
