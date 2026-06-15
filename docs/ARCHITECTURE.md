# 架构概览

## 设计理念

Platform Phantom 的核心设计目标是：**在不修改 GKI 内核 KMI 的前提下，注入高级调度器增强功能**。

实现方式：

1. 在 vmlinux 中嵌入轻量 `phantom_stubs.c`（函数指针定义 + EXPORT_SYMBOL）
2. 调度器核心代码通过 `if (hook) hook(...)` 模式调用
3. 实际逻辑以 vendor `.ko` 模块形式存在，通过 `ph_vh_reg_xxx()` 注册 hook
4. 同一份 vendor 源码跨 5.10 / 5.15 / 6.1 内核版本编译

## 架构分层

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
├─────────────────────────────────────────────────────────┤
│  GKI 内核 (不修改 KMI)                                   │
└─────────────────────────────────────────────────────────┘
```

## Vendor 模块栈

模块按拓扑顺序加载（由 `modules.load` 确定）：

```
phantom_vh        — Vendor Hook 注册中心
phantom_common    — 公共基础设施
phantom_clock     — CPU 时钟管理
phantom_freq_qos  — 频率 QoS 控制
slim_walt_gov     — WALT Governor
phantom_slim_walt — WALT 调度器增强
phantom_render_rt — 渲染实时调度
ph_glk            — GLK 调度器接口
phase_lite        — 轻量 Phase 优先级
phantom_iaware    — iAware 负载感知
ph_ipc            — IPC 感知调度
binder_prio_mod   — Binder 优先级继承
phantom_perfctl   — 性能控制接口
phantom_netlink   — Netlink 通信通道
```

## 跨版本兼容

同一份 vendor 源码需在 5.10 / 5.15 / 6.1 上编译。差异处理：

1. **`LINUX_VERSION_CODE` 条件编译** — 结构体差异、API 变更
2. **`phantom_compat.h`** — 统一 shim 层
3. **内核树侧 include guard** — 防止头文件重复包含
4. **KCFLAGS / subdir-ccflags-y** — 确保 include 路径在所有构建模式下可达

## 工具链

| 内核版本 | Clang 版本 |
|----------|-----------|
| 6.1 | clang-r487747c |
| 5.15 | clang-r450784e |
| 5.10 | clang-r416183b |

## KernelSU 集成

- **GKI 项目 (5.10+)**：Tracepoint hook 方式，`CONFIG_KSU=y` + `CONFIG_KSU_TRACEPOINT_HOOK=y`
- **非 GKI 项目 (LAHAINA 5.4)**：Manual hook 方式，需指定 `--device`
- **SukiSU**：KernelSU fork，额外支持 SUSFS 文件系统伪装
