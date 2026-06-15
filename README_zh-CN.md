# Platform Phantom

<div align="center">

**面向 Qualcomm 平台的 Android 内核增强方案**

[English](README.md) | **中文**

[![Release](https://img.shields.io/github/v/release/TheVoyager0777/Platform_Phantom?style=for-the-badge)](https://github.com/TheVoyager0777/Platform_Phantom/releases)
[![License](https://img.shields.io/github/license/TheVoyager0777/Platform_Phantom?style=for-the-badge)](LICENSE)

</div>

---

## 这是什么？

Platform Phantom 是一套面向 Qualcomm Snapdragon 平台的 **Android 内核调度器增强系统**。通过 GKI 兼容的 vendor hook 机制，在不破坏内核 KMI 的前提下注入高级调度策略与性能控制组件。

## 特性

### 调度器增强

- ⚡ **Slim WALT** — 20ms 窗口辅助负载跟踪，16 级 bucket 平滑预测，自动压制高通闭源 WALT 双轨记账
- 🎯 **Phase Lite** — ARM AMU 硬件计数器 IPC 采样，实时指令吞吐+内存停顿→CPU 频率目标
- 🖼️ **GLK 帧感知** — 追踪帧生命周期，丢帧检测+分级提升，场景切换（idle/bench/camera/game）
- 📐 **iAware 多帧并行** — 8 帧并发追踪，虚拟负载（VLoad）紧迫度曲线，帧内 Binder 事务自动编组
- 🚀 **Render RT** — 渲染线程识别+唤醒链 SCHED_RR/FIFO 提升，防 GPU 提交路径被 CPU 争抢

### IPC 协同

- 📡 **IPC Peer 感知** — 追踪前台与 32 对端 Binder+wakeup 频次，高频对端自动编入 RTG 优先调度
- 🔗 **Binder Priority Inheritance** — 30+ 关键服务（system_server/surfaceflinger 等）调度策略提升至 RT/FIFO

### 性能控制

- 🧠 **PerfCtl 决策中枢** — 汇集 AMU/帧状态/VLoad 做统一 IPC→频率决策，四级 escape 递进防渲染卡顿恶化
- 💉 **内核态 ELF 注入器** — execve hook 捕获→内核 ELF 解析/重定位，零 ptrace，零用户态痕迹

### 基础设施

- 📦 **PHBT 内嵌引导** — 构建产物打包为二进制 blob，`late_initcall` 直接从 vmlinux 落地，无需挂载文件系统
- 💾 **Phantom VDisk** — sysfs 被动接口+用户态触发 erofs 挂载，Ed25519 签名+原子回滚更新
- ⏱️ **Phantom Clock** — 私有 monotonic 时钟，offset+scale(ppm) 校准，统一时间戳不受系统跳变影响
- 🔒 **GKI KMI 安全注入** — 全模块 vendor hook 接入，cmpxchg 单次注册杜绝中间态指针 EL1 panic
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
