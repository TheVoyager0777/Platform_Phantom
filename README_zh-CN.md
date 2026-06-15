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

模块间通过 `phantom_common` 内置的 **IFH（Inter-Function Hook）** 回调槽打破循环依赖，各模块注册回调指针、调用方间接投递事件，无需 load-time 符号依赖。

## 特性

### 调度器增强

- ⚡ **Slim WALT** — 20ms 窗口辅助负载跟踪，16 级 bucket 平滑预测替代内核 PELT。支持运行时热路径开关（A/B 对比无需重刷），自动压制高通闭源 WALT RVH 避免双轨记账错乱
- 🎯 **Phase Lite** — ARM AMU 硬件计数器 IPC 采样，每 CPU 独立采集指令增量+周期增量+内存停顿增量，计算实时 IPC 与 stall 百分比。结合帧渲染/触控状态通过 FreqQoS 后端驱动 cpufreq
- 🖼️ **GLK 帧感知调度** — 追踪帧生命周期六态（queue/dequeue/vsync/touch/doframe/drawframes），维护 8 帧 Jank 历史窗口，输出 NONE/LOW/MID/HIGH 四档 boost。支持场景切换（idle/bench/schedule/camera/game）
- 📐 **iAware 多帧并行** — 8 帧并发追踪，每帧独立 VLoad 紧迫度曲线（帧预算耗尽前渐进加压）。帧内 Binder 事务线程自动编入同一 RTG 组，协同 Frame Boost 注频
- 🚀 **Render RT** — comm 名称识别渲染线程（RenderThread/GLThread/hwuiTask/GPU completion/kgsl/mali 等），PID bloom filter 去重，唤醒链递归提升至 SCHED_RR/FIFO 防 GPU 提交路径被争抢

### IPC 协同

- 📡 **IPC Peer 感知** — 追踪前台与 32 对端的 Binder 事务+wakeup 频次，高频对端标记 MVP Peer（IPC_PEER/BINDER 两级）自动编入 RTG。2 秒 decay + 5 秒过期，RCU 保护回调链向 Slim WALT 投递优先级
- 🔗 **Binder Priority Inheritance** — 30+ 关键服务（system_server/surfaceflinger/cameraserver 等）调度策略提升至 SCHED_RR/FIFO。专属 workqueue 延后规避 tracepoint 上下文 "scheduling while atomic" 内核 panic

### 性能控制

- 🧠 **PerfCtl 决策中枢** — 汇集 AMU IPC 指标、GLK 帧状态与 boost 级别、iAware VLoad 压力、FreqQoS 频率-效率映射表，统一 IPC→频率决策。四级 escape 递进：低 IPC 持续多帧逐级提频探针，防渲染卡顿不可逆恶化

### 基础设施

- 📦 **PHBT 内嵌引导** — 构建产物打包为 PHBT 二进制 blob 链接进 vmlinux `.rodata`。`phantom_early_loader` 在 `late_initcall` 解析→建目录树→按序加载 .ko，全程无文件系统挂载、无 loop device
- 💾 **Phantom VDisk** — sysfs 被动接口，用户态 daemon 通过 `trigger` 节点触发 erofs 挂载。Ed25519 签名校验 + 原子回滚更新
- ⏱️ **Phantom Clock** — 私有 monotonic 时钟，offset+scale(ppm) 校准。全子系统统一时间戳，不受系统时间跳变影响。sysfs + generic netlink 双接口
- 🔒 **GKI KMI 安全注入** — 全模块 vendor hook 接入内核调度路径。单次 cmpxchg 原子注册，杜绝中间态指针被误调用导致 EL1 panic
- 🛡️ **KernelSU / SukiSU** — 内置 root，SukiSU 支持 SUSFS 文件系统伪装

## 支持设备

| SoC | 代号 | 内核版本 | 状态 |
|-----|------|---------|:----:|
| SM8650 (8 Gen 3) | PINEAPPLE / PINEAPPLE_NEW | 6.1 | ✅ |
| SM8550 (8 Gen 2) | KALAMA-167 / KALAMA-180 | 5.15 | ✅ |
| SM8450 (8 Gen 1) | WAIPIO | 5.10 | ✅ |
| SM8350 (888) | LAHAINA | 5.4 | ✅ |
| SM8250 (865) | KONA | 4.19 | 🚧 |

## 快速安装

```bash
# 临启测试
fastboot boot Image

# 永久刷入
fastboot flash boot Image
fastboot reboot
```

> ⚠️ **免责声明**：请备份原厂 boot.img。使用本内核即表示你自行承担所有风险。

## 下载

所有发布包在 [GitHub Releases](https://github.com/TheVoyager0777/Platform_Phantom/releases)。

文件命名：`PHANTOM_<项目>_<Profile>-<Root方案>-<日期>.zip`

## 致谢

- [KernelSU](https://github.com/tiann/KernelSU) / [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU)
- [SUSFS](https://gitlab.com/simonpunk/susfs4ksu)
- [AnyKernel3](https://github.com/osm0sis/AnyKernel3)

## 协议

本项目内核源码将会在审查结束后基于 GPL-2.0 协议发布。详见 [LICENSE](LICENSE)。
