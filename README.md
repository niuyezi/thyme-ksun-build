# thyme-ksun-build

[中文](README.zh-CN.md) | English

Build a **KernelSU-Next** kernel for the Xiaomi Mi 10S (`thyme`, SM8250) running **PixelOS / Android 17** — entirely in the cloud via GitHub Actions. No local Linux, cross-toolchain, or large RAM needed.

## What it does

- Pulls the exact kernel source commit matching the device (`fd144b4b...`, kernel `4.19.325-perf`)
- Builds with the device's own `/proc/config.gz` config
- Replaces the stock KernelSU with KernelSU-Next (`legacy`, built in — this is a non-GKI 4.19 kernel)
- Outputs the kernel image, logs, and SHA-256 checksums

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
| `thyme-pixelos-ksun` | `Image`, `Image.gz`, `*.dtb`, `dtbo.img`, `.config`, checksums, build log |
| `thyme-build-logs` | Build log + diagnostics (also uploaded on failure) |

## Note

The output is a **bare kernel image** (`Image` / `dtbo.img`), **not a flashable `boot.img`**. To boot or flash it, repack it with your original `boot.img` header (cmdline, pagesize, base, offsets, OS version/patch level) and ramdisk.

This workflow only builds — it never touches your device. Unlocking, temporary boot, and flashing each need your separate authorization.
