# Platform Phantom

<div align="center">

**面向 Qualcomm 平台的 Android 内核增强方案**

[English](README.md) | **中文**

[![Release](https://img.shields.io/github/v/release/TheVoyager0777/Platform_Phantom?style=for-the-badge)](https://github.com/TheVoyager0777/Platform_Phantom/releases)
[![License](https://img.shields.io/github/license/TheVoyager0777/Platform_Phantom?style=for-the-badge)](LICENSE)

</div>

---

## 这是什么？

Platform Phantom 是一套面向 Qualcomm Snapdragon 平台的 **Android 内核调度器增强系统**。通过 GKI 兼容的 vendor hook 机制，在不破坏内核 KMI 的前提下注入高级调度与性能控制组件，显著提升 UI 流畅度与游戏帧率稳定性。

### 关键指标

| 维度 | 数值 |
|------|------|
| 调度窗口 | **20ms** (WALT) |
| 负载平滑 | **16 级** bucket |
| 帧并发追踪 | **8 帧** (iAware) |
| Jank 检测窗口 | **8 帧** (GLK) |
| IPC 对端 | **32 个** 进程 |
| Binder 优先级 | **30+** 关键服务 |
| Escape 递进 | **4 级** 探针 |
| 跨内核版本 | **5.10 / 5.15 / 6.1** |
| 模块总数 | **14 个** vendor .ko |

## 模块协作全景

```
                       ┌──────────────────────────┐
                       │     phantom_vh.ko        │
                       │   Hook 注册中心 + 构建信息  │
                       └──────┬───────────────────┘
                              │ (首个加载)
                       ┌──────▼───────────────────┐
                       │   phantom_common.ko      │
                       │   IFH 回调槽 + 共享 WQ    │
                       └──────┬───────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   ┌────▼─────┐         ┌────▼─────┐         ┌─────▼────┐
   │ slim_walt│         │phase_lite│         │ phantom_  │
   │   .ko    │         │   .ko    │         │  clock.ko │
   │ 负载跟踪  │         │ AMU 采样  │         │ 私有时钟   │
   └────┬─────┘         └────┬─────┘         └──────────┘
        │                    │
   ┌────▼─────┐         ┌────▼─────┐
   │slim_walt │         │ phantom_ │
   │ _gov.ko  │         │freq_qos.ko│
   │cpufreq调速│        │ IPC→频率表 │
   └────┬─────┘         └────┬─────┘
        │                    │
   ┌────▼─────┐              │
   │ ph_glk   │              │
   │  .ko     │              │
   │帧感知调度 │              │
   └────┬─────┘              │
        │                    │
   ┌────▼─────┐              │
   │phantom_  │              │
   │iaware.ko │              │
   │多帧并行   │              │
   └────┬─────┘              │
        │                    │
   ┌────▼─────┐         ┌────▼─────┐
   │  ph_ipc  │         │render_rt │
   │  .ko     │         │  .ko     │
   │IPC Peer感知│        │渲染 RT 提升│
   └────┬─────┘         └────┬─────┘
        │                    │
   ┌────▼─────┐              │
   │binder_   │              │
   │prio_mod.ko│             │
   │Binder优先级│             │
   └────┬─────┘              │
        │                    │
        └──────────┬─────────┘
                   │
            ┌──────▼──────┐
            │phantom_perfctl│
            │   .ko        │
            │ 决策中枢       │
            └──────────────┘
```

模块间循环依赖通过 `phantom_common` 内置的 **IFH（Inter-Function Hook）** 回调槽打破：各模块注册回调指针，调用方通过 `phantom_ifh_call_*()` 间接投递事件。

### 数据流：从帧渲染到频率决策

```
渲染帧开始
  │
  ├─ ① GLK 追踪帧六态 → Jank 检测 + NONE/LOW/MID/HIGH boost
  ├─ ② iAware 计算 VLoad 紧迫度 + TransThread Binder 编组
  ├─ ③ IPC Peer 追踪 Binder+wakeup 频次 → MVP Peer → RTG
  ├─ ④ Render RT 识别渲染线程 → 唤醒链 SCHED_RR/FIFO
  ├─ ⑤ Slim WALT 20ms 窗口维护每任务负载
  └─ ⑥ Phase Lite 采样 AMU (指令/周期/内存停顿)
         │
         ├─→ FreqQoS: IPC 目标 → 频率-效率表查表 → 最佳 OPP
         └─→ PerfCtl 汇集全部信号
              ├─ IPC 正常 → 维持
              ├─ 低 IPC 2 帧 → probe_level=1
              ├─ 低 IPC 4 帧 → probe_level=2
              ├─ 低 IPC 6 帧 → probe_level=3
              └─ 持续低 → probe_level=4
                   └─→ Slim WALT Governor → cpufreq
```

## 特性详解

### 一、调度器增强

| | 模块 | 说明 |
|---|------|------|
| ⚡ | **Slim WALT** | 20ms 窗口负载跟踪，16 级 bucket 平滑预测替代内核 PELT。支持运行时热路径开关(A/B 对比无需重刷)。自动压制高通闭源 WALT RVH 避免双轨记账错乱 |
| 🎯 | **Phase Lite** | ARM AMU 硬件计数器 IPC 采样。每 CPU 采集指令/周期/内存停顿增量，计算实时 IPC 和 stall%。结合帧渲染/触控通过 FreqQoS 驱动 cpufreq |
| 🖼️ | **GLK 帧感知** | 追踪帧六态(queue/dequeue/vsync/touch/doframe/drawframes)。8 帧 Jank 历史→四档 boost。支持场景切换(idle/bench/schedule/camera/game) |
| 📐 | **iAware 多帧** | 8 帧并发，VLoad 紧迫度曲线随帧时间线非线性增长。帧内 Binder 事务自动编入同一 RTG，协同 Frame Boost 注频 |
| 🚀 | **Render RT** | 渲染线程识别(RenderThread/GLThread/hwuiTask/GPU completion/kgsl/mali)+PID bloom filter。唤醒链递归提升至 SCHED_RR/FIFO |

### 二、IPC 协同

| | 模块 | 说明 |
|---|------|------|
| 📡 | **IPC Peer** | 追踪前台与 32 对端 Binder+wakeup 频次。MVP Peer(IPC_PEER/BINDER 两级)自动编入 RTG。2s decay+5s 过期，RCU 回调链投递优先级 |
| 🔗 | **Binder Prio** | 30+ 关键服务(system_server/surfaceflinger/cameraserver 等)提升至 SCHED_RR/FIFO。专属 workqueue 延后规避 "scheduling while atomic" panic |

### 三、性能控制中枢

| | 模块 | 说明 |
|---|------|------|
| 🧠 | **PerfCtl** | 汇集 AMU IPC + GLK boost + iAware VLoad + FreqQoS 频率表，统一 IPC→频率决策。**四级 escape 递进：**低 IPC 2/4/6 帧逐级提频探针，防卡顿不可逆恶化 |

### 四、基础设施

| | 模块 | 说明 |
|---|------|------|
| 📦 | **PHBT 引导** | 构建产物打包为 blob 链接进 vmlinux `.rodata`。late_initcall 解析→建目录→按序加载 .ko，无文件系统挂载 |
| 💾 | **Phantom VDisk** | sysfs 被动接口，用户态触发 erofs 挂载。Ed25519 签名+原子回滚 |
| ⏱️ | **Phantom Clock** | 私有 monotonic 时钟，offset+scale(ppm)校准。全子系统统一时间戳不受 NTP/suspend 影响 |
| 🔒 | **GKI KMI 注入** | 全模块 vendor hook 接入。cmpxchg 单次原子注册，杜绝中间态指针 EL1 panic |
| 🛡️ | **KSU / SukiSU** | 内置 root，SukiSU 支持 SUSFS 文件系统伪装 |

## 支持设备

| SoC | 代号 | 内核版本 | Clang | GKI | 状态 |
|-----|------|---------|-------|:---:|:----:|
| SM8650 (8 Gen 3) | PINEAPPLE / PINEAPPLE_NEW | 6.1 | r487747c | ✅ | ✅ |
| SM8550 (8 Gen 2) | KALAMA-167 / KALAMA-180 | 5.15 | r450784e | ✅ | ✅ |
| SM8450 (8 Gen 1) | WAIPIO | 5.10 | r416183b | ✅ | ✅ |
| SM8350 (888) | LAHAINA | 5.4 | r416183b | ❌ | ✅ |
| SM8250 (865) | KONA | 4.19 | r416183b | ❌ | 🚧 |

## 快速安装

```bash
# 临启测试（推荐，重启即还原）
fastboot boot Image

# 永久刷入
fastboot flash boot Image
fastboot reboot
```

> ⚠️ **免责声明**：请备份原厂 boot.img。使用本内核即表示你自行承担所有风险。

## 下载

[GitHub Releases](https://github.com/TheVoyager0777/Platform_Phantom/releases)

命名：`PHANTOM_<项目>_<Profile>-<Root方案>-<日期>.zip`

## 致谢

- [KernelSU](https://github.com/tiann/KernelSU) / [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU)
- [SUSFS](https://gitlab.com/simonpunk/susfs4ksu)
- [AnyKernel3](https://github.com/osm0sis/AnyKernel3)

## 协议

本项目内核源码将会在审查结束后基于 GPL-2.0 协议发布。详见 [LICENSE](LICENSE)。
