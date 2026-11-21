# thyme-ksun-build

中文 | [English](README.md)

为小米 Mi 10S（`thyme`，SM8250）的 **PixelOS / Android 17** 系统，通过 GitHub Actions **云端编译**集成 **KernelSU-Next** 的内核。无需本地 Linux、交叉工具链或大内存。

## 这是干什么的

- 拉取与设备精确匹配的内核源码提交（`fd144b4b...`，内核 `4.19.325-perf`）
- 使用设备自身 `/proc/config.gz` 导出的配置编译
- 将原版 KernelSU 替换为 KernelSU-Next（`legacy`，内置驱动 —— 这是非 GKI 的 4.19 内核）
- 产出内核镜像、编译日志和 SHA-256 校验清单

## 怎么用

1. 打开本仓库的 **Actions** 页面。
2. 左侧选择 **thyme PixelOS KernelSU-Next kernel**。
3. 点 **Run workflow** → **Run workflow**。
4. 所有参数保持默认即可 —— 已锁定该设备的源码提交和配置。
5. 等待约 15–30 分钟，在 run 页面下载产物。

### 参数

| 参数 | 默认值 | 说明 |
|---|---|---|
| `kernel_repo` | `Flicker-Android-Devices/kernel_xiaomi_sm8250` | 内核源码仓库 |
| `kernel_ref` | `fd144b4b278e...` | 精确源码提交（来自设备 `uname`） |
| `defconfig` | `thyme_defconfig` | 备用 defconfig（设备配置优先） |
| `enable_ksu` | `true` | 是否将原版 KernelSU 替换为 KSU-Next |
| `ksun_ref` | `legacy` | KernelSU-Next 分支（4.19 非 GKI 必须用此分支） |
| `clang_version` | `clang-r596125` | AOSP 预编译 clang 版本 |
| `build_baseline` | `false` | 是否同时先编译未修改的基线 |

### 产物

| Artifact | 内容 |
|---|---|
| `thyme-pixelos-ksun` | `Image`、`Image.gz`、`*.dtb`、`dtbo.img`、`.config`、校验清单、编译日志 |
| `thyme-build-logs` | 编译日志 + 诊断摘要（失败时也会上传） |

## 注意

产物是**裸内核镜像**（`Image` / `dtbo.img`），**不是可直接刷入的 `boot.img`**。要启动或刷入，需用原 `boot.img` 的头信息（cmdline、pagesize、base、各偏移、OS 版本/补丁级别）和 ramdisk 重新打包。

本 workflow 只负责编译，不会接触你的设备。解锁、临时启动、刷入都需要你单独明确授权。
