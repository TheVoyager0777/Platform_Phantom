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

### 关键指标

| 维度 | 数值 |
|------|------|
| 调度窗口 | **20ms** (WALT 窗口粒度) |
| 负载平滑 | **16 级** bucket 预测 |
| 帧追踪 | **8 帧** 并发 (iAware) |
| Jank 检测 | **8 帧** 滑动窗口 (GLK) |
| IPC 对端 | **32 个** 进程 (IPC Peer) |
| Binder 优先级 | **30+** 关键服务 |
| Escape 级别 | **4 级** 递进探针 |
| 时间戳精度 | **ns** 级 monotonic |
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
            └──────┬──────┘
                   │
        ┌──────────┼──────────┐
   ┌────▼─────┐    │    ┌─────▼────┐
   │ phantom_ │    │    │ phantom_ │
   │vdisk.ko  │    │    │netlink.ko│
   │虚拟磁盘   │    │    │Netlink通信│
   └──────────┘    │    └──────────┘
              ┌────▼─────┐
              │phantom_  │
              │early_    │
              │loader    │
              │(built-in)│
              └──────────┘
```

模块间存在循环依赖风险（如 `ph_glk ↔ slim_walt ↔ perfctl`），通过 `phantom_common` 内置的 **IFH（Inter-Function Hook）** 回调槽打破加载顺序死锁：各模块向 `phantom_common` 注册回调指针，调用方通过 `phantom_ifh_call_*()` 间接投递事件，无需 load-time 符号依赖。

### 数据流：从帧渲染到频率决策

```
渲染帧开始
  │
  ├─ ① GLK 追踪帧六态 (queue→dequeue→vsync→touch→doframe→drawframes)
  │     输出: Jank 检测 + NONE/LOW/MID/HIGH boost 级别
  │
  ├─ ② iAware 计算 VLoad (虚拟负载) 紧迫度曲线 + TransThread Binder 编组
  │     输出: 逐帧 util 压力值 + RTG 组建议
  │
  ├─ ③ IPC Peer 追踪 Binder 事务 + wakeup 频次
  │     输出: MVP Peer 标记 + RTG 优先级注入
  │
  ├─ ④ Render RT 识别渲染线程，沿唤醒链递归提升调度类
  │     输出: SCHED_RR/FIFO 实时调度
  │
  ├─ ⑤ Slim WALT 以 20ms 窗口维护每任务负载
  │     输出: bucket 平滑后的 CPU 利用率
  │
  └─ ⑥ Phase Lite 采样 AMU (指令/周期/内存停顿)
         │
         ├─→ FreqQoS: IPC 目标 → 频率-效率表查表 → 最佳 OPP
         │
         └─→ PerfCtl: 汇集 ①~⑥ 全部信号
              │
              ├─ IPC 正常 → 维持当前频率
              ├─ IPC 偏低 2 帧 → probe_level=1 轻度提频
              ├─ IPC 偏低 4 帧 → probe_level=2 中度提频
              ├─ IPC 偏低 6 帧 → probe_level=3 高度提频
              └─ IPC 持续低   → probe_level=4 全速冲刺
                   │
                   └─→ Slim WALT Governor → cpufreq driver
```

## 特性详解

### 一、调度器增强

<table>
<tr>
<td width="24"><b>⚡</b></td>
<td width="120"><b>Slim WALT</b></td>
<td>
20ms 窗口辅助负载跟踪，16 级 bucket 平滑预测，替代内核 PELT。<br>
<sub>支持运行时热路径开关（<code>core_hooks_enabled</code>），A/B 性能对比无需重刷。自动通过 <code>phantom_swalt_skip_rvh</code> jump label 压制 Qualcomm 闭源 WALT RVH 探针，避免双轨并行记账错乱。</sub>
</td>
</tr>
<tr>
<td><b>🎯</b></td>
<td><b>Phase Lite</b></td>
<td>
ARM AMU 硬件计数器 IPC 采样策略。每 CPU 独立采集指令/周期/内存停顿增量，计算实时 IPC 和 stall%。<br>
<sub>结合帧渲染/触控状态，通过 <code>phantom_freq_qos</code> 将性能需求映射为精确的 cpufreq 频率目标。</sub>
</td>
</tr>
<tr>
<td><b>🖼️</b></td>
<td><b>GLK 帧感知</b></td>
<td>
追踪 SurfaceFlinger 帧六态(queue/dequeue/vsync/touch/doframe/drawframes)，计算帧预算偏差检测丢帧。<br>
<sub>8 帧 Jank 历史窗口 → NONE/LOW/MID/HIGH 四档 boost。支持场景切换(idle/bench/schedule/camera/game)与自定义关键线程类别。</sub>
</td>
</tr>
<tr>
<td><b>📐</b></td>
<td><b>iAware 多帧并行</b></td>
<td>
8 帧并发追踪，每帧独立渲染线程组 + VLoad(虚拟负载)紧迫度曲线。VLoad 随帧时间线非线性增长，预算耗尽前渐进施压。<br>
<sub>帧内 Binder 事务线程通过 TransThread 机制自动编入同一 RTG 组，协同 Frame Boost 注频。</sub>
</td>
</tr>
<tr>
<td><b>🚀</b></td>
<td><b>Render RT</b></td>
<td>
comm 名称匹配识别关键渲染线程(RenderThread/GLThread/hwuiTask/GPU completion/kgsl/mali) + PID bloom filter 去重。<br>
<sub>唤醒链递归提升至 SCHED_RR 或 SCHED_FIFO，确保 GPU 提交路径不被 CPU 争抢。</sub>
</td>
</tr>
</table>

### 二、IPC 协同

<table>
<tr>
<td width="24"><b>📡</b></td>
<td width="120"><b>IPC Peer 感知</b></td>
<td>
追踪前台 top-app 与 32 个对端进程的 Binder 事务 + wakeup 频次。高频对端标记 MVP Peer(IPC_PEER/BINDER 两级)自动编入 RTG 优先调度组。<br>
<sub>2s decay 衰减 + 5s 过期淘汰。RCU 保护的 MVP notify 回调链向 Slim WALT 投递优先级信号。</sub>
</td>
</tr>
<tr>
<td><b>🔗</b></td>
<td><b>Binder Prio</b></td>
<td>
在 Binder 事务上下文中，将 system_server、surfaceflinger、cameraserver 等 30+ 关键服务调度策略安全提升至 SCHED_RR/FIFO。<br>
<sub>专属 workqueue 延后 <code>sched_setscheduler_nocheck</code> 到进程上下文，规避 tracepoint 的 preempt-disabled 上下文 "scheduling while atomic" panic。</sub>
</td>
</tr>
</table>

### 三、性能控制中枢

<table>
<tr>
<td width="24"><b>🧠</b></td>
<td width="120"><b>PerfCtl</b></td>
<td>
汇集 Phase Lite AMU IPC + GLK 帧 boost + iAware VLoad + FreqQoS 频率-效率表，统一 IPC→频率决策。<br>
<sub><b>四级 escape 递进：</b>低 IPC 持续 2 帧 → probe_level 1(轻); 4 帧 → level 2(中); 6 帧 → level 3(重); 持续低 → level 4(全速)。逐级提频探针防止渲染卡顿不可逆恶化。决策以 freq_cap 注入 Phase Lite → FreqQoS → Slim WALT Governor → cpufreq。</sub>
</td>
</tr>
</table>

### 四、基础设施

<table>
<tr>
<td width="24"><b>📦</b></td>
<td width="120"><b>PHBT 内嵌引导</b></td>
<td>
构建时打包为 PHBT 二进制 blob 链接进 vmlinux <code>.rodata</code>。late_initcall 解析头→建 /phantom_boot 目录树→按表顺序加载 .ko。<br>
<sub>全程无文件系统挂载、无 loop device、无 <code>s_bdev</code>。每步失败非致命：跳过继续，不 panic。</sub>
</td>
</tr>
<tr>
<td><b>💾</b></td>
<td><b>Phantom VDisk</b></td>
<td>
内核侧 sysfs 被动接口(<code>/sys/kernel/phantom_vdisk/</code>)，用户态 daemon 经 trigger 节点触发 erofs 镜像挂载。<br>
<sub>Ed25519 签名校验 + 原子回滚(新镜像挂载失败自动回退)。用于安全交付用户态组件与 daemon 更新。</sub>
</td>
</tr>
<tr>
<td><b>⏱️</b></td>
<td><b>Phantom Clock</b></td>
<td>
私有 monotonic 时钟源，ktime + offset + scale(ppm) 两级校准。全子系统共享统一时间戳，不受系统时间跳变/NTP/suspend 影响。<br>
<sub>sysfs + generic netlink 双接口。</sub>
</td>
</tr>
<tr>
<td><b>🔒</b></td>
<td><b>GKI KMI 安全注入</b></td>
<td>
全模块通过 vendor hook(<code>trace_android_vh_*</code> + <code>trace_android_rvh_*</code>)接入调度路径，不修改 GKI 导出符号。<br>
<sub>Hook 注册采用单次 cmpxchg(ARM64 全屏障)，指针仅在 NULL 与有效函数间原子切换——杜绝中间态被误调用 EL1 panic。</sub>
</td>
</tr>
<tr>
<td><b>🛡️</b></td>
<td><b>KernelSU / SukiSU</b></td>
<td>
内置 root 方案。SukiSU 额外支持 SUSFS 文件系统伪装。
</td>
</tr>
</table>

## 支持设备

| SoC | 代号 | 内核版本 | Clang | GKI | 状态 |
|-----|------|---------|-------|:---:|:----:|
| **SM8650** (8 Gen 3) | PINEAPPLE / PINEAPPLE_NEW | 6.1 | r487747c | ✅ | ✅ |
| **SM8550** (8 Gen 2) | KALAMA-167 / KALAMA-180 | 5.15 | r450784e | ✅ | ✅ |
| **SM8450** (8 Gen 1) | WAIPIO | 5.10 | r416183b | ✅ | ✅ |
| **SM8350** (888) | LAHAINA | 5.4 | r416183b | ❌ | ✅ |
| **SM8250** (865) | KONA | 4.19 | r416183b | ❌ | 🚧 |

> 具体设备型号兼容性请参考每个 Release 的说明。不同厂商/地区可能需要不同的 dtb/dtbo。

## 快速安装

```bash
# 临时启动测试（推荐，重启即还原）
fastboot boot Image

# 确认正常后永久刷入
fastboot flash boot Image
fastboot reboot
```

> ⚠️ **免责声明**：刷写内核有风险。请备份原厂 boot.img 并了解如何恢复。使用本内核即表示你自行承担设备损坏、数据丢失等风险。

## 下载

所有发布包在 [GitHub Releases](https://github.com/TheVoyager0777/Platform_Phantom/releases)。

文件命名：`PHANTOM_<项目>_<Profile>-<Root方案>-<日期>.zip`

示例 `PHANTOM_PINEAPPLE_NEW_PRIME-SukiSU-14.06.26.zip`：

| 字段 | 含义 |
|------|------|
| PINEAPPLE_NEW | SM8650 设备 |
| PRIME | 完整调度器增强 profile |
| SukiSU | 内置 SukiSU root |
| 14.06.26 | 构建日期 (2026-06-14) |

## 架构概览

```
┌─────────────────────────────────────────────────────────┐
│  用户态 (ph-userspace)                                    │
│  phantom_perfctl daemon + native test CLI                │
├─────────────────────────────────────────────────────────┤
│  Vendor 模块栈 (14 个 .ko)                                │
│  phantom_vh → phantom_common → slim_walt → phase_lite   │
│  → ph_glk → phantom_iaware → ph_ipc → binder_prio       │
│  → render_rt → phantom_perfctl → phantom_vdisk          │
│  IFH 回调打破循环依赖 | cmpxchg 原子注册 hook              │
├─────────────────────────────────────────────────────────┤
│  vmlinux (phantom_stubs.c)                               │
│  函数指针定义 + EXPORT_SYMBOL | if(hook) hook(...) 模式  │
├─────────────────────────────────────────────────────────┤
│  GKI 内核 (KMI 不变)                                     │
│  标准 Android GKI + phantom_stubs 补丁                   │
└─────────────────────────────────────────────────────────┘
```

详见 [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)。

## 从源码构建

详见 [`docs/BUILD.md`](docs/BUILD.md)。

```bash
# 交互式配置（推荐）
./phantom-build --preset PINEAPPLE_NEW -y

# 直接调用
cd build
BUILD_VENDOR_TREE=1 INCLUDE_PH_USERSPACE=1 \
  ./build.sh --project KALAMA-180 --profile prime --with-ksu --anykernel
```

## 常见问题

**Q: 刷入后无法开机？**
A: 用 `fastboot boot Image` 临时测试。无法开机则重启恢复原厂内核。检查包是否匹配设备型号。

**Q: 支持哪些 ROM？**
A: 兼容基于原厂/类原厂内核的 ROM（MIUI、ColorOS、OxygenOS 等）。AOSP 定制 ROM 可能需要额外适配。

**Q: 如何卸载？**
A: 刷回原厂 boot.img：`fastboot flash boot stock_boot.img`

## 致谢

- [KernelSU](https://github.com/tiann/KernelSU) — GKI 内核 root
- [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) — SukiSU (KernelSU fork)
- [SUSFS](https://gitlab.com/simonpunk/susfs4ksu) — KernelSU SUSFS 支持
- [AnyKernel3](https://github.com/osm0sis/AnyKernel3) — 通用内核刷写格式

## 协议

本项目内核源码将会在审查结束后基于 GPL-2.0 协议发布。详见 [LICENSE](LICENSE)。

---

<div align="center">

**⚡ 刷写自由，责任自负 ⚡**

</div>
