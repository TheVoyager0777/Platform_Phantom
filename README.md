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

Platform Phantom 是一套面向 Qualcomm Snapdragon 平台的 **Android 内核调度器增强系统**，通过 GKI-compatible 的 vendor hook 机制，在不破坏内核 KMI 的前提下注入高级调度策略与性能控制组件。

核心能力涵盖 Slim WALT 负载跟踪、Phase Lite AMU 采样调度、GLK/iAware 帧感知加速、IPC Peer 感知协同、Binder 优先级继承、内核态 ELF 注入器、内嵌引导加载等，全部以可加载内核模块（`.ko`）形式交付。

## 特性

### 调度器增强

- ⚡ **Slim WALT** — 20ms 窗口辅助负载跟踪，16 级 bucket 平滑预测，替代内核 PELT 提供更精确的 CPU 利用率估算。支持运行时热路径开关（A/B 性能对比无需重刷），自动压制 Qualcomm 闭源 WALT RVH 避免双轨记账
- 🎯 **Phase Lite** — 基于 ARM AMU 硬件计数器的 IPC（Instructions Per Cycle）采样策略。实时采集指令吞吐 + 内存停顿率，结合帧渲染/触控状态转换为 CPU 频率目标，通过 FreqQoS 后端驱动 cpufreq
- 🖼️ **GLK 帧感知调度** — 追踪 SurfaceFlinger 帧生命周期（queue/dequeue/vsync/touch/doframe/drawframes），检测丢帧（Jank）历史并分级提升渲染线程优先级。支持自定义关键线程类别、场景切换（idle/bench/camera/game）
- 📐 **iAware 多帧并行** — 多帧并发追踪（最多 8 帧），计算虚拟负载（VLoad）随帧时间线增长的紧迫度曲线。关联帧内 Binder 事务线程自动编组（TransThread），协同 Frame Boost 向渲染 RTG 注频
- 🚀 **Render RT** — 识别渲染关键线程（RenderThread/GLThread/hwuiTask/GPU completion 等），在唤醒链上递归提升其调度类至 SCHED_RR/FIFO，确保 GPU 提交路径不被 CPU 争抢

### IPC 协同

- 📡 **IPC Peer 感知** — 实时追踪前台应用与 32 个对端进程的 Binder 事务 + wakeup 频次，自动将高频交互对端提升为 MVP Peer 并编入 RTG 优先调度组。2 秒衰减窗口 + 5 秒过期，自适应应用使用模式变化
- 🔗 **Binder Priority Inheritance** — 在 Binder 事务上下文中，将 system_server、surfaceflinger 等 30+ 关键服务的调度策略安全提升至 SCHED_RR/FIFO。通过 workqueue 延后执行规避 tracepoint 上下文原子睡眠陷阱

### 性能控制

- 🧠 **PerfCtl 决策中枢** — 汇集 Phase Lite 的 AMU IPC 指标、GLK 的帧状态、iAware 的 VLoad 压力、FreqQoS 的频率表，做统一 IPC→频率映射决策。包含四级 escape 递进机制：低 IPC 持续帧数触发逐级提频探针，防止渲染卡顿不可逆恶化
- 💉 **内核态 ELF 注入器** — 通过 execve hook 捕获目标进程启动，内核态直接解析 ELF、分配内存、完成重定位，无需 ptrace 或用户态注入器。支持 denylist/allowlist，零用户态痕迹

### 基础设施

- 📦 **PHBT 内嵌引导** — 构建时将 ART APEX、vendor .ko 等文件打包为 PHBT 二进制 blob，由 `phantom_early_loader` 在 `late_initcall` 直接从 vmlinux `.rodata` 解析落地到 `/phantom_boot`，无需挂载文件系统，无 loop device
- 💾 **Phantom VDisk** — 内核侧提供 sysfs 被动接口，由用户态 daemon 通过 `trigger` 节点触发 erofs 挂载。支持 Ed25519 签名校验 + 原子回滚更新，用于安全交付用户态组件
- ⏱️ **Phantom Clock** — 私有 monotonic 时钟源，带 offset + scale (ppm) 校准。所有 phantom 子系统共享统一时间戳，不受系统时间跳变影响
- 🔒 **GKI KMI 安全注入** — 全模块通过 vendor hook（`trace_android_vh_*` + `trace_android_rvh_*`）接入内核调度路径。Hook 指针注册使用单次 cmpxchg（全屏障），杜绝中间态指针被误调用导致 EL1 panic
- 🛡️ **KernelSU / SukiSU 集成** — 内置 root 方案，SukiSU 额外支持 SUSFS 文件系统伪装

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
