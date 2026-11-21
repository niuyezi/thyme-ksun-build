# thyme-ksun-build

中文 | [English](README.en.md)

为小米 Mi 10S（`thyme`，SM8250 / 骁龙 870）的 **PixelOS / Android 17** 系统，通过 GitHub Actions **云端编译**集成 **KernelSU-Next** 的内核。无需本地 Linux、交叉工具链或大内存。

## 这是干什么的

- 拉取与设备精确匹配的内核源码提交（`fd144b4b...`，内核 `4.19.325-perf`）
- 使用设备自身 `/proc/config.gz` 导出的配置编译
- 将原版 KernelSU 替换为 KernelSU-Next（`legacy` 分支，内置驱动 —— 这是非 GKI 的 4.19 内核）
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

## 引用的文件与来源

### 仓库内文件

| 文件 | 作用 | 来源 |
|---|---|---|
| `.github/workflows/build.yml` | 编译工作流 | 本仓库 |
| `input/device.config` | 编译时使用的内核配置 | 从设备 `/proc/config.gz` 导出（6378 行） |
| `input/device-ksun.config` | 叠加 KSU 选项后的配置 | 每次运行由 `device.config` 自动生成 |
| `LICENSE` | 本仓库授权 | 见下方「开源与许可」 |

### 开发所依据的系统包

- **目标 ROM**：PixelOS（非官方）for thyme / Mi 10S，Android 17，构建号 `CP2A.260605.016`（2026-11-10）
  - ROM 作者主页（XDA）：https://xdaforums.com/m/alcatraz323.10682425/
  - ROM 下载目录（SourceForge）：https://sourceforge.net/projects/alcatrazdev-aosp/files/Mi%2010S/
  - PixelOS 官网：https://pixelos.net ；源码：https://github.com/PixelOS-AOSP
- **底包固件**（ROM 要求的基线）：Xiaomi HyperOS `1.0.4.0`（thyme）

### 上游源码与工具

| 项目 | 用途 | 链接 | 许可 |
|---|---|---|---|
| Flicker 内核树 | `seventeen` 分支内核源码 | https://github.com/Flicker-Android-Devices/kernel_xiaomi_sm8250 | GPL-2.0 |
| KernelSU-Next | `legacy` 分支内置驱动 | https://github.com/KernelSU-Next/KernelSU-Next | GPL-3.0 |
| PixelOS libufdt | Python3 版 `mkdtboimg.py` | https://github.com/PixelOS-AOSP/android_system_libufdt | Apache-2.0 |
| AOSP Clang | `clang-r596125` 交叉工具链 | https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86 | Apache-2.0 / LLVM 例外 |

## 开源与许可

本仓库自身的工作流与脚本以 [MIT](LICENSE) 授权。引用的上游源码、二进制与配置分别遵循其各自许可证（见上表）；编译产物中的内核与 KernelSU-Next 代码受对应 GPL 许可约束。`input/device.config` 源自设备内核配置，其权利归相应内核版权方所有。

## 注意

产物是**裸内核镜像**（`Image` / `dtbo.img`），**不是可直接刷入的 `boot.img`**。要启动或刷入，需用原 `boot.img` 的头信息（cmdline、pagesize、base、各偏移、OS 版本/补丁级别）和 ramdisk 重新打包。

本 workflow 只负责编译，不会接触你的设备。解锁、临时启动、刷入都需要你单独明确授权。

## 致谢

- **Alcatraz323** —— 提供本设备所用 PixelOS ROM（thyme 维护者）
- **Flicker372** —— 维护内核树 `kernel_xiaomi_sm8250`，并参与 ROM 贡献
- **johnmart19** 及所有为 sm8250 平台工作的开发者 —— 内核支持
- **LineageOS** 的 sm8250 官方维护者 —— 设备与内核基线
- **PixelOS 团队** —— ROM 基础
- **KernelSU / KernelSU-Next 团队与社区**（tiann 及贡献者）—— Root 方案
- **AOSP / Google** —— Android 源码、Clang 工具链与 `libufdt`
