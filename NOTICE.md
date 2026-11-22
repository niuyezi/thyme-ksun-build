# NOTICE — 上游来源、修改点与许可声明

本项目 `niuyezi/thyme-ksun-build` 是个人设备（Xiaomi Mi 10S / `thyme` / SM8250）的
内核构建工作流与配置仓库，**不是 KernelSU / KernelSU-Next 官方项目，未获得上游授权或背书**。
本文件说明构建过程中引用的上游组件、本项目所做修改，以及分发 GPL 二进制时的义务。

## 1. 上游组件

| 组件 | 来源 | 版本 / commit | 许可证 |
|---|---|---|---|
| 内核源码 | `Flicker-Android-Devices/kernel_xiaomi_sm8250` | `e7dc99d5350d3f443946af44b196c9fa6f2cb02a`（由 boot 镜像 `Linux version` 版本串派生） | GPL-2.0-only（以源码文件头为准） |
| KernelSU-Next | `KernelSU-Next/KernelSU-Next` | `legacy` 分支，`kernel/setup.sh` 在 CI 中远程注入 | `kernel/` 为 GPL-2.0-only，其余 GPL-3.0-or-later（以上游文件头为准） |
| AOSP Clang | `prebuilts/clang/host/linux-x86` | `clang-r596125` | Apache-2.0（LLVM 各子项目许可见上游） |
| `mkdtboimg.py` | `PixelOS-AOSP/android_system_libufdt` | `seventeen` 分支 `utils/src/mkdtboimg.py` | 见文件头 |
| 设备配置 / boot 镜像 | 用户本地的 PixelOS Android 17 `thyme` 原版 `boot.img` | 由 `boot_image_url` 指定或手工提供 | 仅用于提取内嵌 `config.gz` 与重打包，不随本仓库分发 |

## 2. 本项目对内核源码的修改

所有修改都在 CI 工作流 `.github/workflows/build.yml` 中以脚本形式完成，不把上游源码
vendored 进本仓库。当前生效的插桩点共两处：

1. **`kernel/kernel/reboot.c`** — 在 reboot syscall 的能力检查之前插入
   `ksu_handle_sys_reboot(magic1, magic2, cmd, &arg)` inline hook。
   原因：`CONFIG_KSU_SYSCALL_TABLE_HOOK=y` 下 syscall table 只挂载
   execve / execveat / faccessat / newfstatat，不含 reboot；没有这个 hook 时管理器
   的 reboot supercall 无法工作，表现为「su 授权成功但功能异常」。

2. **`fs/stat.c`** — 在 `SYSCALL_DEFINE2(newfstat, ...)` 中、复制 stat 结果之后插入
   `ksu_handle_newfstat_ret(&fd, &statbuf)`（`extern` 原型必须放在文件作用域，
   内核构建启用 `-Werror=declaration-after-statement`）。
   原因：init 通过 `fstat()` 取 `st_size` 决定读取长度，若不放大该值，
   `KERNEL_SU_RC` 永远不会被追加到 ramdisk 脚本末尾，导致**所有模块无法加载**
   （`on post-fs-data` 分支不可达）。`COMPAT_SYSCALL_DEFINE2(newfstat)` 刻意不修改。

配套配置：`CONFIG_KSU=y`、`CONFIG_KSU_SYSCALL_TABLE_HOOK=y`，并显式关闭
`CONFIG_KSU_MANUAL_HOOK` 与 `CONFIG_KSU_KPROBES_HOOK`。工作流还会先移除 legacy
自带的 inline hook，再按上述方式重新注入，避免重复符号。

## 3. 分发 GPL 二进制时的义务

本仓库的产物是替换 kernel 后重打包的 `boot.img`，其中包含 GPL-2.0 内核二进制。
对外发布这类产物时必须同时：

- 提供能够重建该二进制的完整源码：**上表内核仓库 + 准确的 commit**（
  `e7dc99d5350d3f443946af44b196c9fa6f2cb02a`）+ 本仓库工作流中的全部补丁步骤；
- 保留各上游许可证文本（KernelSU-Next 的 `kernel/` 与其余部分许可证不同，不可混写）；
- 说明上表第 2 节的修改点，并保存可审计的 patch / 构建日志。

## 4. 声明

本项目自身编写的文档、脚本与工作流采用 MIT 许可证（见 `LICENSE`）；**MIT 不覆盖
任何上游代码或二进制**。商标 "KernelSU"、"KernelSU-Next" 归各自权利人所有，
本文中的引用仅用于说明构建来源，不构成官方关系或背书。

如权利人认为存在权利冲突，请在本仓库提 issue，我会立即下架相关内容。

*本文件是基于许可证文本与开源惯例的工程说明，不构成法律意见。*
