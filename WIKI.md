# Linux 内核代码仓库 Wiki

## 概述

这是一个 Linux 内核代码仓库，版本为 **7.1.0-rc7**（代号 "Baby Opossum Posse"）。Linux 内核是任何 Linux 操作系统的核心，负责管理硬件、系统资源，并为所有其他软件提供基础服务。

## 目录结构

### 核心目录

| 目录 | 描述 |
|------|------|
| `Documentation/` | 内核文档中心，包含完整的 API 手册和使用指南 |
| `block/` | 块设备子系统，处理磁盘 I/O 调度 |
| `drivers/` | 设备驱动程序集合 |
| `include/` | 头文件目录 |
| `init/` | 内核初始化代码 |
| `lib/` | 内核通用库函数 |
| `mm/` | 内存管理子系统 |
| `rust/` | Rust 语言支持模块 |
| `scripts/` | 构建脚本和工具 |
| `security/` | 安全子系统 |
| `sound/` | 音频子系统 |

### 文档子目录

`Documentation/` 目录包含以下主要类别：

- **开发流程**: `process/` - 提交补丁、代码规范、维护者手册
- **核心 API**: `core-api/` - 内核核心接口文档
- **驱动 API**: `driver-api/` - 设备驱动开发指南
- **内存管理**: `mm/` - 内存管理子系统文档
- **网络**: `networking/` - 网络子系统文档
- **文件系统**: `filesystems/` - 文件系统文档
- **调度器**: `scheduler/` - CPU 调度器文档
- **安全**: `security/` - 安全子系统文档
- **追踪调试**: `trace/` - 追踪和调试工具

## 核心子系统

### 1. 内存管理 (mm/)

负责系统内存的分配、回收和管理：

- **页面分配**: `page_alloc.c` - 物理页面分配器
- **slab 分配器**: `slub.c` - 内核对象缓存
- **内存回收**: `vmscan.c` - 页面回收算法
- **大页支持**: `hugetlb.c` - HugeTLB 支持
- **内存压缩**: `zswap.c`, `zsmalloc.c`

### 2. 块设备 (block/)

处理磁盘 I/O 操作：

- **I/O 调度器**: `bfq-iosched.c`, `mq-deadline.c`, `kyber-iosched.c`
- **多队列支持**: `blk-mq.c` - 高性能块层
- **块加密**: `blk-crypto.c`
- **设备映射**: `blk-map.c`

### 3. 安全子系统 (security/)

提供安全框架和机制：

- **LSM (Linux Security Modules)**: `lsm.c`, `lsm_init.c`
- **能力管理**: `commoncap.c`
- **设备 cgroup**: `device_cgroup.c`

## 开发指南

### 快速开始

```bash
# 克隆仓库
git clone <repo-url>
cd linux

# 配置内核
make menuconfig  # 或 make defconfig

# 构建内核
make -j$(nproc)

# 安装内核模块
make modules_install

# 安装内核
make install
```

### 编译选项

| 选项 | 说明 |
|------|------|
| `V=1` | 显示详细编译命令 |
| `C=1` | 启用静态检查 (sparse) |
| `O=output/` | 指定输出目录 |
| `ARCH=<arch>` | 指定目标架构 |
| `CROSS_COMPILE=<prefix>` | 指定交叉编译工具链 |

### 代码规范

Linux 内核有严格的代码规范，详见：
- `Documentation/process/coding-style.rst`

### 提交补丁

提交补丁流程：
1. 编写代码并遵循编码规范
2. 使用 `scripts/checkpatch.pl` 检查补丁
3. 发送补丁到相关邮件列表
4. 等待审核和反馈

## 维护指南

### MAINTAINERS 文件

`MAINTAINERS` 文件列出了所有子系统的维护者信息，包括：
- **M**: 主要维护者邮件
- **R**: 指定审核者
- **L**: 相关邮件列表
- **S**: 状态（Supported/Maintained/Odd Fixes/Orphan/Obsolete）
- **W**: 相关网页
- **F**: 负责的文件/目录

### 维护状态说明

| 状态 | 说明 |
|------|------|
| **Supported** | 有人专门负责维护 |
| **Maintained** | 有人实际维护 |
| **Odd Fixes** | 有维护者但时间有限 |
| **Orphan** | 暂无维护者 |
| **Obsolete** | 已被替代的旧代码 |

## 配置系统

### Kbuild 系统

内核使用 Kbuild 构建系统，主要文件：
- `Kbuild` - 顶层 Kbuild 配置
- `Makefile` - 顶层 Makefile
- `scripts/Kbuild.include` - Kbuild 包含文件
- `scripts/Makefile.*` - 各种构建脚本

### 配置目标

```bash
make config      # 文本界面配置
make menuconfig  # 菜单界面配置
make xconfig     # QT 图形界面配置
make gconfig     # GTK 图形界面配置
make defconfig   # 默认配置
make oldconfig   # 基于已有配置更新
```

## 测试和调试

### 调试工具

- **kasan**: 内存安全检测
- **kcsan**: 并发安全检测
- **kfence**: 内存安全检测（低开销）
- **kmsan**: 未初始化内存检测
- **trace**: 追踪框架
- **perf**: 性能分析工具

### 自测框架

```bash
# 运行内核自测
make kselftest

# 运行特定测试
make kselftest TARGETS=net
```

## 版本信息

- **版本**: 7.1.0-rc7
- **代号**: Baby Opossum Posse
- **许可证**: GPL-2.0

## 社区资源

- **邮件列表**: https://lore.kernel.org/
- **Bug 追踪**: https://bugzilla.kernel.org/
- **IRC**: #kernelnewbies on irc.oftc.net
- **在线文档**: https://www.kernel.org/doc/html/latest/

## 参考文档

以下是一些核心文档的快速参考：

| 文档路径 | 描述 |
|----------|------|
| `Documentation/process/submitting-patches.rst` | 提交补丁指南 |
| `Documentation/process/coding-style.rst` | 代码风格指南 |
| `Documentation/kbuild/index.rst` | 构建系统文档 |
| `Documentation/dev-tools/index.rst` | 开发工具文档 |
| `Documentation/admin-guide/index.rst` | 管理员指南 |
| `Documentation/core-api/index.rst` | 核心 API 文档 |

---

**最后更新**: 2026-06-12
**内核版本**: 7.1.0-rc7