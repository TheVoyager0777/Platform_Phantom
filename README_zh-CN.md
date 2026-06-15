# Platform Phantom

<div align="center">

**面向 Qualcomm 平台的 Android 内核增强方案**

[English](README.md) | **中文**

[![Release](https://img.shields.io/github/v/release/TheVoyager0777/Platform_Phantom?style=for-the-badge)](https://github.com/TheVoyager0777/Platform_Phantom/releases)
[![License](https://img.shields.io/github/license/TheVoyager0777/Platform_Phantom?style=for-the-badge)](LICENSE)

</div>

---

## 这是什么？

Platform Phantom 是一套面向 Qualcomm Snapdragon 平台的 **Android 内核调度器增强系统**。通过 GKI 兼容的 vendor hook 机制，在不破坏内核 KMI 的前提下注入高级调度策略。

## 特性

- ⚡ **WALT** — 窗口辅助负载跟踪，比 PELT 更激进
- 🧭 **MIGT** — 智能任务迁移引导
- 🔄 **Metis** — 逆向 Qualcomm 闭源调度反馈环路
- 🎯 **Phase Priority** — 多级优先级调度（前台/后台/渲染）
- 📡 **IPC-Aware** — 跨进程通信感知调度
- 🔗 **Binder Priority Inheritance** — 防优先级反转
- 🔒 **GKI KMI 兼容** — 全部 vendor hook 注入
- 🛡️ **KernelSU / SukiSU** — 内置 root，SukiSU 支持 SUSFS
- 📦 **AnyKernel3** — 即刷即用

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

GPL-2.0。详见 [LICENSE](LICENSE)。
