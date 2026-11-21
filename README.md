# thyme-ksun-build

为小米 Mi 10S（codename `thyme`）的 **PixelOS / Android 17** 类原生 ROM，通过 GitHub Actions **云端编译**集成 **KernelSU-Next** 的内核。

本机无需 Linux 环境、无需交叉工具链、无需大内存——编译全部在 GitHub 托管的 `ubuntu-22.04`  runner 上完成，本地只负责触发和下载产物。

---

## 1. 目标设备与匹配依据

| 项 | 值 | 来源 |
|---|---|---|
| 机型 | 小米 Mi 10S（M2102J2SC，SM8250 / 骁龙 870） | `adb getprop ro.product.model` |
| Codename | `thyme` | `adb devices -l` |
| 系统 | Android 17，PixelOS（alcatraz323 构建） | 用户提供 |
| 构建号 | `CP2A.260605.016`，2026-11-10 | `adb getprop ro.build.*` |
| 内核版本 | `4.19.325-perf-gfd144b4b278e` | `adb shell uname -a` |
| 内核源码提交 | `fd144b4b278e61e0389cc72e07247bc919eb1be3` | 由 `uname` 中的 `g<sha>` 反查确认 |
| 上游内核仓库 | `Flicker-Android-Devices/kernel_xiaomi_sm8250`（分支 `seventeen`） | XDA 帖子列出的 ROM 内核树 |
| 编译配置 | 设备 `/proc/config.gz` 原样导出（6378 行） | `adb exec-out gzip -dc /proc/config.gz` |

> **为什么用设备导出配置而不是 defconfig**：`thyme_defconfig` 只是模板，与手机实际运行的配置可能有差异。直接复用 `/proc/config.gz` 能最大化复现当前内核，避免硬件失效。

---

## 2. 它做了什么

workflow（`.github/workflows/build.yml`）共 16 步，核心流程：

1. 安装依赖，并建立 `python2 → python3` 兼容 shim（4.19 的 Makefile 硬编码 `python2`）
2. 按精确 commit SHA 拉取内核源码（用 `git init + fetch <ref>`，支持 SHA 而非仅分支名）
3. 稀疏克隆 AOSP `clang-r596125` 工具链
4. 下载 Python3 版 `mkdtboimg.py`，通过 `MKDTBO_EXT` 传给 make
5. 由 `input/device.config` **每次运行自动重新生成** KSU 配置，防止手改坏
6. （可选）先编未修改基线并记录 SHA-256
7. `git reset --hard` + 删除原版 `drivers/kernelsu`
8. **清除 Flicker 内核源码里提交态的原版 KernelSU 手动 hook**（见踩坑记录）
9. 集成 KernelSU-Next `legacy` 驱动
10. 打 4.19 兼容补丁（见踩坑记录）
11. 用设备配置编最终内核
12. 无论成功失败都上传日志与诊断摘要
13. 上传产物

关键设计：**所有 KSU 相关配置都在运行时从设备原始配置生成**，不依赖仓库里可能被改坏的静态文件。

---

## 3. 如何使用

1. 进入仓库 **Actions** 页
2. 左侧选择 **thyme PixelOS KernelSU-Next kernel**
3. 点 **Run workflow** → **Run workflow**
4. 参数全部保持默认即可（已锁定本机的精确源码提交与配置）
5. 运行约 15–30 分钟，完成后在 run 页面底部下载 artifact

### 输入参数

| 参数 | 默认值 | 说明 |
|---|---|---|
| `kernel_repo` | `Flicker-Android-Devices/kernel_xiaomi_sm8250` | 内核源码仓库 |
| `kernel_ref` | `fd144b4b278e...`（40 位 SHA） | 精确提交，来自设备 `uname` |
| `defconfig` | `thyme_defconfig` | 备用 defconfig；实际优先用 `input/device.config` |
| `enable_ksu` | `true` | 是否替换原版 KernelSU 为 KSU-Next |
| `ksun_ref` | `legacy` | KernelSU-Next 分支（4.19 非 GKI 必须用 legacy） |
| `clang_version` | `clang-r596125` | AOSP 预编译 clang 目录名 |
| `build_baseline` | `false` | 是否先编未修改基线（已验证，默认关闭省约 18 分钟） |

### 产物

| Artifact | 内容 |
|---|---|
| `thyme-pixelos-ksun` | `Image`、`Image.gz`、`.dtb`、`dtbo.img`、`.config`、日志、SHA-256 清单 |
| `thyme-build-logs`（失败时也有） | `build-final.log`、`diagnostics.txt`、`final-defconfig.config` 等 |

成功构建的哈希示例：

```
Image     sha256 316f64752c23b940baa82a93339547d1e92a645719cb728028f1ce7856183508
dtbo.img  sha256 37f8fa51b3c74d8425b6b09b516b33eda1d646f7e2d259dce4269d7c8358c8b6
```

---

## 4. 已解决的技术问题（踩坑记录）

这些是调试过程中逐个定位并修掉的真实问题，改动前请务必了解：

| 问题 | 表现 | 修法 |
|---|---|---|
| Node 20 弃用 | Actions 持续警告；GitHub 2026-09-23 移除 Node 20 | `checkout@v7.0.1` / `upload-artifact@v7.0.2`（node24）+ `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24='true'` |
| clang PATH 不生效 | exit 127，`clang: not found` | `$GITHUB_PATH` 只对后续步骤生效，同一步内必须 `export PATH` |
| `python2` 缺失 | `dtbo.img` 构建 `python2: not found` | 建 `python2 → python3` shim + 用 `MKDTBO_EXT` 指向 Python3 版脚本 |
| KSU hook 选错 | `-- No hooks were defined` | 4.19 非 GKI 用 `CONFIG_KSU_SYSCALL_TABLE_HOOK=y`，**不用** kprobes hook（Kconfig 要求 ≥5.10） |
| `can_umount` 重复定义 | `redefinition of 'can_umount'` | KSU 的 Kbuild 检测 `^static int`，本内核是 `static inline int` → 检测正则改为 `^static (inline )?int` |
| `copy_to_kernel_nofault` 破坏声明 | `uaccess.h: type specifier missing` | 4.19 已 backport 此 API，把 KSU 的 `<5.8` shim `#if` 改为 `#if 0` |
| 原版 KSU 残留 hook | 链接期 `undefined symbol: ksu_handle_devpts` | Flicker 把原版 KSU hook **提交进源码**（`pty.c` 等），`git reset` 删不掉 → 通用扫描删除所有纯 `#ifdef CONFIG_KSU` hook 块 |
| 补丁验证误杀 | 补丁成功但步骤 exit 1 | `set -e` 下 shell `grep` 验证因字符串漂移误判 → 验证全部收进 Python 内做回读断言 |

> 教训：把「删除原版 KSU 驱动目录」等同于「移除原版 KSU」是错的，必须同时清理内核源码里提交态的手动 hook；这类残留只会在链接期暴露。

---

## 5. 已知限制与下一步

**产物还不能直接刷入。** `Image` / `dtbo.img` 只是裸内核与 dtbo，需要用原 `boot.img` 的头信息重打包成可刷镜像：

- `cmdline`、`pagesize`、`base`
- `kerneloff` / `ramdiskoff` / `dtboff`
- `os_version` / `os_patch_level`
- 原 ramdisk

可行路径：

- **A**：把原 `boot.img` 或解包出的 `split_img/` 小文本放进 `input/`，加一个 `repack.yml` 自动产出可刷 `boot.img`
- **B**：通过 adb **只读**导出 boot 分区（需 root，先列命令再执行）
- **C**：用 Android Image Kitchen 手动重打包

其他提醒：

- LOS 官方无 thyme 的 `lineage-24.0` 设备树；本项目基于 alcatraz323 实际 ROM 所用的 Flicker 内核树，非官方 LineageOS 构建
- KernelSU-Next 管理器版本需与编入的 `legacy` 驱动匹配
- 刷机前务必备份原始 `boot.img` 并验证恢复路径

---

## 6. 安全说明

- 本仓库的 workflow **只编译，不接触设备**
- 解锁 / `fastboot boot` 临时启动 / 刷入 / 回锁，每一步都需单独明确授权
- 用于推送文件的 GitHub token 为短期凭证，任务完成后应立即在 https://github.com/settings/personal-access-tokens 撤销

---

## 7. 目录结构

```
.github/workflows/build.yml   # 主编译工作流（16 步，node24 actions）
input/
  device.config              # 从设备 /proc/config.gz 导出的原始配置
  device-ksun.config         # 运行时自动生成的 KSU-Next 配置（无需手改）
README.md
```
