# Changelog

## [2026.06.14] — Initial Public Release

### 新增

**调度器增强**
- Slim WALT：20ms 窗口负载跟踪 + 16 级 bucket 平滑预测
- Phase Lite：ARM AMU IPC 采样 → CPU 频率目标
- GLK 帧感知调度：帧生命周期追踪 + 丢帧检测 + 分级提升
- iAware 多帧并行：VLoad 虚拟负载 + TransThread 自动编组
- Render RT：渲染线程识别 + 唤醒链 RR/FIFO 提升

**IPC 协同**
- IPC Peer 感知：Binder + wakeup 频次追踪，高频对端自动 RTG
- Binder Priority Inheritance：30+ 关键服务 RT/FIFO 提升

**性能控制**
- PerfCtl 决策中枢：AMU/帧/VLoad → IPC→频率统一决策，四级 escape

**基础设施**
- PHBT 内嵌引导：blob 直接从 vmlinux 落地，无需挂载
- Phantom VDisk：sysfs 被动挂载 + Ed25519 签名 + 原子更新
- Phantom Clock：私有 monotonic 时钟源
- SukiSU root + SUSFS 文件系统伪装
- AnyKernel3 打包格式

### 已知问题
- KONA (SM8250/4.19) 平台仍在适配中
- 部分厂商定制 ROM 可能需要额外 dtb 配置

---

> 格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.0.0/)。
