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

Platform Phantom 是一套面向 Qualcomm Snapdragon 平台的 **Android 内核调度器增强系统**。通过 GKI-compatible 的 vendor hook 机制，在不破坏内核 KMI 的前提下注入高级调度与性能控制组件，显著提升 UI 流畅度与游戏帧率稳定性。

核心能力涵盖 Slim WALT 负载跟踪、Phase Lite AMU 采样调度、GLK/iAware 帧感知加速、IPC Peer 感知协同、Binder 优先级继承、内嵌引导加载等，全部以可加载内核模块（`.ko`）形式交付。

### 工作流程

```
渲染帧开始 ──→ GLK 追踪帧生命周期，检测丢帧
    │
    ├──→ iAware 计算 VLoad 虚拟负载紧迫度，关联 Binder 事务线程
    │
    ├──→ IPC Peer 追踪跨进程通信频次，识别高频对端 → 提升调度优先级
    │
    ├──→ Render RT 找到渲染线程，沿唤醒链提升至 RT 调度类
    │
    ├──→ Slim WALT 提供 20ms 窗口的精确 CPU 利用率
    │
    └──→ Phase Lite 采样 AMU 硬件计数器（指令吞吐 + 内存停顿）
              │
              └──→ FreqQoS 将 IPC 目标映射为最佳频率点
                       │
                       └──→ PerfCtl 汇集所有信号，做统一决策
                                │
                                ├──→ Slim WALT Governor 驱动 cpufreq
                                └──→ Binder Prio 继承提升关键服务
```

系统中多个模块存在循环依赖风险（如 ph_glk ↔ slim_walt ↔ perfctl），通过 `phantom_common` 模块内置的 **IFH（Inter-Function Hook）** 回调槽打破加载顺序死锁：各模块向 `phantom_common` 注册回调指针，调用方通过 `phantom_ifh_call_*()` 间接投递事件，无需 load-time 符号依赖。

## 特性

### 调度器增强

- ⚡ **Slim WALT** — 20ms 窗口辅助负载跟踪，16 级 bucket 平滑预测算法，替代内核 PELT 提供更精确的 CPU 利用率。支持运行时热路径开关（`core_hooks_enabled` 参数，A/B 性能对比无需重刷）。自动通过 `phantom_swalt_skip_rvh` jump label 压制 Qualcomm 闭源 WALT 的 RVH 探针，避免双轨并行导致任务记账错乱
- 🎯 **Phase Lite** — 基于 ARM AMU（Activity Monitor Unit）硬件计数器的 IPC 采样策略。每 CPU 独立采集指令增量 + 周期增量 + 内存停顿增量，计算实时 IPC（Instructions Per Cycle）和 stall 百分比。结合帧渲染状态与触控信号，通过 `phantom_freq_qos` 后端将性能需求映射为精确的 cpufreq 频率目标
- 🖼️ **GLK 帧感知调度** — 追踪 SurfaceFlinger 帧生命周期六态（queue/dequeue/vsync/touch/doframe/drawframes），计算帧预算（frame budget）与实际耗时的偏差检测丢帧。维护 Jank 历史窗口（8 帧），分级输出 NONE/LOW/MID/HIGH 四档 boost 级别。支持场景切换（idle/bench/schedule/camera/game）与自定义关键线程类别
- 📐 **iAware 多帧并行** — 最多 8 帧并发追踪，每帧维护独立的渲染线程组、时间窗口和 VLoad（虚拟负载）紧迫度曲线。VLoad 随帧时间线非线性增长，在帧预算耗尽前提供渐进式压力信号。帧内 Binder 事务线程通过 TransThread 机制自动编入同一 RTG 组，协同 Frame Boost 向调度器注频
- 🚀 **Render RT** — 通过 comm 名称匹配识别渲染关键线程（RenderThread/GLThread/hwuiTask/GPU completion/kgsl/mali 等），辅以 PID bloom filter 快速去重。在唤醒链上递归提升渲染相关任务至 SCHED_RR 或 SCHED_FIFO 实时调度类，确保 GPU 命令提交路径不被 CPU 争抢

### IPC 协同

- 📡 **IPC Peer 感知** — 实时追踪前台应用（top-app）与最多 32 个对端进程的 Binder 事务次数 + wakeup 唤醒频次。高频交互对端标记为 MVP Peer（IPC_PEER/BINDER 两级），自动编入 RTG（Runtime Task Group）优先调度组。2 秒 decay 衰减窗口 + 5 秒过期淘汰，自适应应用使用模式变化。通过 RCU 保护的 MVP notify 回调链向 Slim WALT 投递优先级信号
- 🔗 **Binder Priority Inheritance** — 在 Binder 事务上下文中，将 system_server、surfaceflinger、cameraserver 等 30+ 关键服务的调度策略安全提升至 SCHED_RR 或 SCHED_FIFO。通过专属 workqueue 将 `sched_setscheduler_nocheck` 延后到进程上下文执行，规避 tracepoint 回调的 preempt-disabled 上下文导致的 "scheduling while atomic" 内核 panic

### 性能控制

- 🧠 **PerfCtl 决策中枢** — 汇集 Phase Lite 的 AMU IPC 指标、GLK 的帧状态与 boost 级别、iAware 的 VLoad 压力值、FreqQoS 的频率-效率映射表，做统一的 IPC→频率决策。内置四级 escape 递进机制：当低 IPC 持续多帧时，逐级提升 probe 探针频率百分比（probe_level 0→3），防止渲染卡顿进入不可逆的恶化螺旋。决策结果以 freq_cap 形式注入 Phase Lite，再由 FreqQoS 驱动 Slim WALT Governor 调整 cpufreq

### 基础设施

- 📦 **PHBT 内嵌引导** — 构建时将 vendor .ko 等文件打包为 PHBT 二进制 blob，链接进 vmlinux `.rodata` 段。`phantom_early_loader` 在 `late_initcall` 阶段解析 PHBT 头 → 创建 `/phantom_boot` 目录树 → 按表顺序加载 .ko 模块，全程无文件系统挂载、无 loop device、无 `s_bdev`
- 💾 **Phantom VDisk** — 内核侧提供 sysfs 被动接口（`/sys/kernel/phantom_vdisk/`），由用户态 daemon 通过 `trigger` 节点触发 erofs 镜像挂载。支持 Ed25519 签名校验 + 原子回滚更新（新镜像挂载失败自动回退），用于安全交付用户态组件与 daemon 更新
- ⏱️ **Phantom Clock** — 私有 monotonic 时钟源，基于 ktime 加 offset + scale (ppm) 两级校准。所有 phantom 子系统共享统一时间戳基准，不受系统时间跳变、NTP 校时、suspend/resume 影响。提供 sysfs 和 generic netlink 双接口
- 🔒 **GKI KMI 安全注入** — 全模块通过 vendor hook（`trace_android_vh_*` + `trace_android_rvh_*`）接入内核调度路径，不修改 GKI 内核导出符号。Hook 指针注册采用单次 cmpxchg（ARM64 全屏障语义），确保指针在 NULL 和有效函数之间原子切换——杜绝中间态被误调用导致 EL1 panic
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
│  phantom_vh → phantom_common → slim_walt → phase_lite   │
│  → ph_glk → phantom_iaware → ph_ipc → binder_prio       │
│  → render_rt → phantom_perfctl → phantom_vdisk          │
│  通过 ph_vh_reg_xxx() 注册 hook，IFH 回调打破循环依赖     │
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
