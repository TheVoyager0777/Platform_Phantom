# 安装指南

## 前提条件

1. **解锁的 Bootloader** — 这是刷写自定义内核的前置条件
2. **已备份原厂 boot.img** — 出问题时可以恢复
3. **安装了 `fastboot` 的 PC** — Android Platform Tools

## 下载

从 [GitHub Releases](https://github.com/TheVoyager0777/Platform_Phantom/releases) 下载对应你设备的 zip 包。

文件命名规则：`PHANTOM_<项目>_<Profile>-<Root方案>-<日期>.zip`

找到你设备的 SoC 对应的项目名：

| 设备 SoC | 项目名 | 示例文件 |
|----------|--------|---------|
| SM8650 (8 Gen 3) | PINEAPPLE_NEW | `PHANTOM_PINEAPPLE_NEW_PRIME-*.zip` |
| SM8550 (8 Gen 2) | KALAMA-180 / KALAMA-167 | `PHANTOM_KALAMA_180_PRIME-*.zip` |
| SM8450 (8 Gen 1) | WAIPIO | `PHANTOM_WAIPIO_PRIME-*.zip` |
| SM8350 (888) | LAHAINA | `PHANTOM_LAHAINA_PRIME-*.zip` |

## 安装步骤

### 1. 解压

```bash
unzip PHANTOM_PINEAPPLE_NEW_PRIME-SukiSU-14.06.26.zip -d phantom-release/
cd phantom-release/
```

### 2. 进入 fastboot 模式

```bash
adb reboot bootloader
# 或关机状态下按 电源 + 音量下
```

### 3. 临时启动测试（强烈推荐）

```bash
fastboot boot Image
```

设备会使用新内核启动。如果出现问题，重启即可恢复原厂内核。

### 4. 永久刷入

确认临时启动正常后：

```bash
fastboot flash boot Image
fastboot reboot
```

## 卸载

刷回原厂 boot.img：

```bash
fastboot flash boot /path/to/stock_boot.img
fastboot reboot
```

## 故障排查

### 无法开机

1. 长按电源键 10 秒强制关机
2. 按住 电源 + 音量下 进入 fastboot
3. 刷回原厂 boot.img
4. 检查：是否下载了正确的设备型号对应的包

### WiFi / 蓝牙不工作

可能是模块未正确加载。尝试：

```bash
# 确认模块已加载
lsmod | grep phantom

# 手动加载
insmod /vendor/lib/modules/phantom_vh.ko
```

### KernelSU 无 root

1. 确认下载的是 `PRIME` profile（非 `noksu` 变体）
2. 安装 KernelSU Manager APK
3. 检查 `CONFIG_KSU` 是否启用：`zcat /proc/config.gz | grep KSU`

---

> ⚠️ **警告**：刷写操作不可逆。请确保理解每一步操作，并自行承担风险。
