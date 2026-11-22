# thyme-ksun-build

[中文](README.md) | English

Build a **KernelSU-Next** kernel for the Xiaomi Mi 10S (`thyme`, SM8250 / Snapdragon 870) running **PixelOS / Android 17** — entirely in the cloud via GitHub Actions. No local Linux, cross-toolchain, or large RAM needed.

## What it does

- Pulls the exact kernel source commit matching the device (`fd144b4b...`, kernel `4.19.325-perf`)
- Builds with the device's own `/proc/config.gz` config
- Replaces the stock KernelSU with KernelSU-Next (`legacy` branch, built in — this is a non-GKI 4.19 kernel)
- Adds a `reboot` supercall hook for 4.19 so a UAPI 6 manager can obtain the driver fd and grant root
- Adds a `newfstat` return hook in `fs/stat.c` so init can see the injected `on post-fs-data` block, which is required for modules (overlay mounts, module scripts, metamodules)
- Outputs the kernel image, build log, and SHA-256 checksums

## How to use

1. Open the **Actions** tab of this repo.
2. Select **thyme PixelOS KernelSU-Next kernel** on the left.
3. Click **Run workflow** → **Run workflow**.
4. Leave every parameter at its default — they are locked to this device's source commit and config.
5. Wait ~15–30 min, then download the artifacts from the run page.

### Parameters

| Parameter | Default | Meaning |
|---|---|---|
| `kernel_repo` | `Flicker-Android-Devices/kernel_xiaomi_sm8250` | Kernel source repo |
| `kernel_ref` | `fd144b4b278e...` | Exact source commit (from device `uname`) |
| `defconfig` | `thyme_defconfig` | Fallback defconfig (device config takes priority) |
| `enable_ksu` | `true` | Replace stock KernelSU with KernelSU-Next |
| `ksun_ref` | `legacy` | KernelSU-Next branch (required for 4.19 non-GKI) |
| `clang_version` | `clang-r596125` | AOSP prebuilt clang version |
| `build_baseline` | `false` | Also build an unmodified baseline first |

### Outputs

| Artifact | Contents |
|---|---|
| `thyme-pixelos-ksun` | `Image`, `*.dtb`, `dtbo.img`, `final-defconfig.config`, `artifacts.sha256`, build log |
| `thyme-build-logs` | Build log + diagnostics (also uploaded on failure) |

## Companion manager

This kernel bundles the KernelSU-Next `legacy` driver, which declares **UAPI 6**. The manager must also be UAPI 6 for root grants to work:

- Use the regular (not spoofed) manager from a **dev-branch CI build** of KernelSU-Next (e.g. `v3.4.2-200-g32e25d58`, versionCode `33555-6`).
- Do **not** use the Release managers `v3.4.0` / `v3.4.1` / `v3.4.2` (UAPI 4 / 5 / 5 respectively) — they report a version mismatch and cannot grant root.
- Download from: KernelSU-Next repo → **Actions** → a dev-branch run → the **`manager`** artifact (not `manager-spoofed`).

## Referenced files and sources

### Files in this repo

| File | Role | Origin |
|---|---|---|
| `.github/workflows/build.yml` | Build workflow | This repo |
| `input/device.config` | Kernel config used for the build | Exported from the device's `/proc/config.gz` (6378 lines) |
| `input/device-ksun.config` | Config with KSU options layered on | Regenerated from `device.config` on every run |
| `LICENSE` | Repo license | See "Open source & licensing" below |

### Base system package

- **Target ROM**: PixelOS (unofficial) for thyme / Mi 10S, Android 17, build `CP2A.260605.016` (2026-11-10)
  - ROM author profile (XDA): https://xdaforums.com/m/alcatraz323.10682425/
  - ROM downloads (SourceForge): https://sourceforge.net/projects/alcatrazdev-aosp/files/Mi%2010S/
  - PixelOS site: https://pixelos.net ; source: https://github.com/PixelOS-AOSP
- **Firmware base** (baseline required by the ROM): Xiaomi HyperOS `1.0.4.0` (thyme)

### Upstream source and tools

| Project | Use | Link | License |
|---|---|---|---|
| Flicker kernel tree | Kernel source, `seventeen` branch | https://github.com/Flicker-Android-Devices/kernel_xiaomi_sm8250 | GPL-2.0 |
| KernelSU-Next | Built-in driver, `legacy` branch | https://github.com/KernelSU-Next/KernelSU-Next | GPL-3.0 |
| PixelOS libufdt | Python3 `mkdtboimg.py` | https://github.com/PixelOS-AOSP/android_system_libufdt | Apache-2.0 |
| AOSP Clang | `clang-r596125` cross toolchain | https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86 | Apache-2.0 / LLVM exception |

## Open source & licensing

The workflow and scripts in this repo are licensed under [MIT](LICENSE). Referenced upstream source, binaries, and configs each follow their own licenses (see table above); the kernel and KernelSU-Next code in the build output remain under their respective GPL licenses. `input/device.config` is derived from the device kernel config and its rights belong to the respective kernel copyright holders.

## Note

The output is a **bare kernel image** (`Image` / `dtbo.img`), **not a flashable `boot.img`**. To boot or flash it, repack using your current system's original `boot.img`: replace the kernel region in place (start is normally pagesize `4096`, length is `kernel_size` from the boot header); if the new image is smaller than the original `kernel_size`, zero-fill the remainder; keep the boot header, ramdisk, and trailing AVB footer byte-for-byte identical, and keep the total file size unchanged.

Modules depend on the kernel injecting an `on post-fs-data` block into init.rc and running `ksud post-fs-data`; this kernel includes the required patch. Root grant and modules are separate paths — working root does not imply working modules, and vice versa. If modules do not work, first verify the manager is UAPI 6 and built from the same source generation as the kernel.

This workflow only builds — it never touches your device. Unlocking, temporary boot, and flashing each need your separate authorization.

## Credits

- **Alcatraz323** — author of the PixelOS ROM used on this device (thyme maintainer)
- **Flicker372** — maintainer of the `kernel_xiaomi_sm8250` kernel tree and ROM contributor
- **johnmart19** and everyone who worked on the sm8250 platform — kernel support
- **LineageOS** official maintainers for sm8250 — device and kernel baseline
- **PixelOS team** — ROM foundation
- **KernelSU / KernelSU-Next team and community** (tiann, rifsxd, and contributors) — root solution; the reboot hook is based on official issue #1053 and the backslashxx manual-hooks work
- **AOSP / Google** — Android source, Clang toolchain, and `libufdt`
