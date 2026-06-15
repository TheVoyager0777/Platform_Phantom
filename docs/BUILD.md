# 从源码构建

## 前置条件

- Linux 构建环境 (推荐 Ubuntu 22.04+)
- 对应内核版本的 Clang 工具链（自动检测）
- 足够的磁盘空间（单次构建约 5-10 GB）

## 快速构建

### 方式一：交互式配置器（推荐）

```bash
cd Platform_Phantom

# 全交互模式
./phantom-build

# 指定项目，跳过问答
./phantom-build --preset PINEAPPLE_NEW -y

# 查看所有可用项目和能力
./phantom-build --list
```

### 方式二：直接调用 build.sh

```bash
cd Platform_Phantom/build

# 标准构建（无 KSU）
./build.sh --project PINEAPPLE_NEW --profile prime --variant noksu

# 带 KernelSU + Vendor Tree + 用户态
BUILD_VENDOR_TREE=1 INCLUDE_PH_USERSPACE=1 \
  ./build.sh --project KALAMA-180 --profile prime --with-ksu

# 生成 AnyKernel3 zip
./build.sh --project PINEAPPLE_NEW --profile prime --with-ksu --anykernel
```

### 方式三：ph 统一入口 (ninja 引擎)

```bash
cd Platform_Phantom/build

# ninja 构建（默认引擎）
./ph build --project KALAMA-180 --profile prime --with-ksu

# 回退到 legacy 引擎
./ph build --engine=legacy --project PINEAPPLE_NEW
```

## 构建 Profile

| Profile | KSU | 说明 |
|---------|:---:|------|
| `normal` | ❌ | 默认构建 |
| `prime` | ✅ | 完整增强（推荐） |
| `public` | ❌ | 公开发布用 |

## 环境变量

| 变量 | 说明 |
|------|------|
| `BUILD_VENDOR_TREE=1` | 编译 Vendor 模块栈 |
| `INCLUDE_PH_USERSPACE=1` | 嵌入用户态 daemon |
| `ENABLE_DEBUG=1` | 合并 debug 配置 |
| `WITH_KSU=1` | 启用 KernelSU |

## 输出产物

```
product_out/<项目名>/
├── Image/           # 内核 Image + dtb/dtbo
└── Anykernel/       # AnyKernel3 zip 包
```

## 分布式编译 (icecc)

支持 icecream 分布式编译加速：

```bash
cd Platform_Phantom/build
./setup-icecc.sh           # 本地配置
./setup-remote.sh          # 远程节点 SSH 配置
```
