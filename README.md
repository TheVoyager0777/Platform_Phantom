# Platform Phantom

<div align="center">

**面向 Qualcomm 平台的 Android 内核增强方案**

*Qualcomm Snapdragon Android Kernel Enhancement Platform*

[![Release](https://img.shields.io/github/v/release/TheVoyager0777/Platform_Phantom?style=for-the-badge)](https://github.com/TheVoyager0777/Platform_Phantom/releases)
[![License](https://img.shields.io/github/license/TheVoyager0777/Platform_Phantom?style=for-the-badge)](LICENSE)
[![Platforms](https://img.shields.io/badge/SoC-SM8450%20%7C%20SM8550%20%7C%20SM8650-blue?style=for-the-badge)](#supported-devices)

</div>

---

## 这是什么？

Platform Phantom 是一套面向 Qualcomm Snapdragon 平台的 **Android 内核调度器增强系统**，通过 GKI-compatible 的 vendor hook 机制，在不破坏内核 KMI 的前提下注入高级调度策略。

核心组件包括 WALT 负载跟踪、MIGT 迁移引导、Metis 调度反馈、Phase 优先级调度、IPC 感知、Binder 优先级继承等模块，全部以可加载内核模块（`.ko`）形式交付。

## 特性

- ⚡ **WALT (Window-Assisted Load Tracking)** — 窗口辅助负载跟踪，比 PELT 更激进的调度响应
- 🧭 **MIGT (Migration Guidance)** — 智能任务迁移引导，优化大小核负载分布
- 🔄 **Metis Scheduler Feedback** — 逆向工程自 Qualcomm 闭源调度器的反馈环路
- 🎯 **Phase Priority Scheduler** — 前台/后台/渲染多级优先级调度
- 📡 **IPC-Aware Scheduling** — 跨进程通信感知的调度器协同
- 🔗 **Binder Priority Inheritance** — Binder 事务优先级继承，防止优先级反转
- 🔒 **GKI KMI 兼容** — 全部通过 vendor hook 注入，不修改 GKI 内核导出符号
- 🛡️ **KernelSU / SukiSU 集成** — 内置 root 方案，SukiSU 支持 SUSFS
- 📦 **AnyKernel3 发布** — 即刷即用，无需完整 ROM 替换

## 支持设备

| SoC | 代号 | 内核版本 | 状态 |
|-----|------|---------|:----:|
| **SM8650** (8 Gen 3) | PINEAPPLE / PINEAPPLE_NEW | 6.1 | ✅ |
| **SM8550** (8 Gen 2) | KALAMA-167 / KALAMA-180 | 5.15 | ✅ |
| **SM8450** (8 Gen 1) | WAIPIO | 5.10 | ✅ |
| **SM8350** (888) | LAHAINA | 5.4 | ✅ |
| **SM8250** (865) | KONA | 4.19 | 🚧 |

> 具体设备型号兼容性请参考每个 Release 的说明。不同厂商/地区的设备可能需要不同的 dtb/dtbo 配置。

## 快速安装

### 前提条件

- 解锁的 Bootloader
- 已备份原厂 boot.img
- 安装了 `fastboot` 的 PC

### 安装步骤

1. 从 [Releases](https://github.com/TheVoyager0777/Platform_Phantom/releases) 下载对应你设备的 zip 包
2. 解压获取 `Image` 文件
3. 进入 fastboot 模式后刷入：

```bash
# 临时启动测试（推荐先试，重启即还原）
fastboot boot Image

# 确认正常后永久刷入
fastboot flash boot Image
fastboot reboot
```

> ⚠️ **免责声明**：刷写内核有风险。请确保已备份原厂 boot.img 并了解如何恢复。使用本内核即表示你自行承担设备损坏、数据丢失等风险。作者不提供任何明示或暗示的担保。

## 下载

所有发布包在 [GitHub Releases](https://github.com/TheVoyager0777/Platform_Phantom/releases) 页面。

文件命名规则：`PHANTOM_<项目>_<Profile>-<Root方案>-<日期>.zip`

示例：`PHANTOM_PINEAPPLE_NEW_PRIME-SukiSU-14.06.26.zip`
- **PINEAPPLE_NEW**: SM8650 设备
- **PRIME**: 包含完整调度器增强的 profile
- **SukiSU**: 内置 SukiSU root 方案
- **14.06.26**: 构建日期 (2026年6月14日)

## 架构概览

```
┌─────────────────────────────────────────────────────────┐
│  用户态 (ph-userspace)                                    │
│  phantom_perfctl daemon + native test CLI                │
├─────────────────────────────────────────────────────────┤
│  Vendor 模块 (.ko)                                       │
│  phantom_vh → slim_walt → phase → glk → iaware → ...   │
│  通过 ph_vh_reg_xxx() 注册 hook                          │
├─────────────────────────────────────────────────────────┤
│  vmlinux (phantom_stubs.c)                               │
│  函数指针定义 + EXPORT_SYMBOL                             │
│  调度器核心代码通过 if (hook) hook(...) 调用              │
├─────────────────────────────────────────────────────────┤
│  GKI 内核 (不修改 KMI)                                   │
│  标准 Android GKI 内核 + phantom_stubs.c 补丁            │
└─────────────────────────────────────────────────────────┘
```

详细架构文档见 [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)。

## 从源码构建

完整的构建文档见 [`docs/BUILD.md`](docs/BUILD.md)。

```bash
# 交互式配置（推荐）
./phantom-build --preset PINEAPPLE_NEW -y

# 或直接用 build.sh
cd build
BUILD_VENDOR_TREE=1 INCLUDE_PH_USERSPACE=1 \
  ./build.sh --project KALAMA-180 --profile prime --with-ksu --anykernel
```

## 常见问题

**Q: 刷入后无法开机？**
A: 首先用 `fastboot boot Image` 临时测试。如果无法开机，重启即可恢复原厂内核。检查你下载的包是否匹配设备型号和系统版本。

**Q: 支持哪些 ROM？**
A: 理论上兼容所有基于原厂/类原厂内核的 ROM（包括 MIUI、ColorOS、OxygenOS 等）。AOSP 定制 ROM 可能需要额外适配。

**Q: 如何卸载？**
A: 刷回你备份的原厂 boot.img：`fastboot flash boot stock_boot.img`

## 致谢

- [KernelSU](https://github.com/tiann/KernelSU) — GKI 内核 root 方案
- [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) — SukiSU (KernelSU fork)
- [SUSFS](https://gitlab.com/simonpunk/susfs4ksu) — KernelSU SUSFS 支持
- [AnyKernel3](https://github.com/osm0sis/AnyKernel3) — 通用内核刷写格式

## 协议

本项目内核源码将会在审查结束后基于 GPL-2.0 协议发布。详见 [LICENSE](LICENSE)。

---

<div align="center">

**⚡ 刷写自由，责任自负 ⚡**

</div>
