---
name: build_oh_mini
description: 构建 OpenHarmony mini 系统（LiteOS-M 内核）的完整指南。涵盖环境检查、系统构建、配置修改和常见问题解决。当用户需要：(1) 构建 LiteOS-M 内核镜像，(2) 配置内核选项，(3) 启用内核测试套件，(4) 解决构建过程中的问题，(5) 生成 QEMU 可运行镜像时使用此技能。
---

# OpenHarmony Mini 系统构建指南

## 概述

本 skill 用于构建 OpenHarmony mini 系统（LiteOS-M 内核）。构建产物可在 QEMU 模拟器中运行。

| 属性 | 值 |
|------|-----|
| 内核 | LiteOS-M |
| 架构 | ARM Cortex-M4 |
| 目标产品 | qemu_mini_system_demo |
| QEMU 机器 | mps2-an386 |

## 前置条件检查

### 1. 检查源码是否存在

```bash
WORK_DIR="${WORK_DIR:-$HOME/myws/ohos_mini}"

if [ ! -d "$WORK_DIR/.repo" ]; then
    echo "源码不存在，请先使用 get-oh-code skill 下载代码"
    exit 1
fi
```

如果源码不存在，请使用 **get-oh-code** skill 下载 mini 系统代码：
- 分组参数：`-g ohos:mini`

### 2. 检查 prebuilts

```bash
cd "$WORK_DIR"
if [ ! -d "prebuilts" ]; then
    ./build/prebuilts_download.sh
fi
```

### 3. 检查 QEMU

```bash
if ! command -v qemu-system-arm &> /dev/null; then
    sudo apt-get install -y qemu-system-arm
fi
```

## 构建系统

### 设置产品配置

```bash
cd "$WORK_DIR"
hb set
# 选择: qemu_mini_system_demo
```

### 执行构建

```bash
hb build -f
```

### 构建输出

```
out/arm_mps2_an386/qemu_mini_system_demo/
├── OHOS_Image        # 内核镜像（ELF格式）
├── OHOS_Image.bin    # 内核镜像（二进制格式）
├── OHOS_Image.map    # 符号映射表
└── OHOS_Image.sym.sorted
```

## 内核配置

配置文件路径：`vendor/ohemu/qemu_mini_system_demo/kernel_configs/debug.config`

### 基础配置

```
LOSCFG_PLATFORM_QEMU_ARM_VIRT_CM4=y
LOSCFG_SHELL=y
LOSCFG_ARCH_UNALIGNED_EXC=n
```

### 启用测试套件

```
LOSCFG_KERNEL_TEST=y
```

### 配置说明

| 配置项 | 说明 |
|--------|------|
| `LOSCFG_PLATFORM_QEMU_ARM_VIRT_CM4` | QEMU Cortex-M4 平台支持 |
| `LOSCFG_SHELL` | 启用 Shell 命令行 |
| `LOSCFG_ARCH_UNALIGNED_EXC` | 未对齐访问异常（建议关闭） |
| `LOSCFG_KERNEL_TEST` | 内核测试套件 |
| `LOSCFG_NET_LWIP` | LWIP 网络协议栈 |

## 镜像类型

| 镜像 | 大小 | 用途 |
|------|------|------|
| `OHOS_Image` | ~116KB | 基础镜像（无测试） |
| `OHOS_Image_with_test` | ~687KB | 带测试套件 |

### 保存不同配置的镜像

```bash
# 保存带测试的镜像
cp out/arm_mps2_an386/qemu_mini_system_demo/OHOS_Image \
   out/arm_mps2_an386/qemu_mini_system_demo/OHOS_Image_with_test
```

## 运行镜像

构建完成后，使用 **run_oh_qemu** skill 运行镜像：

```bash
qemu-system-arm -M mps2-an386 -m 16M \
  -kernel out/arm_mps2_an386/qemu_mini_system_demo/OHOS_Image \
  -nographic
```

## 故障排查

遇到问题时，请参考 [troubleshooting.md](references/troubleshooting.md)。

常见问题包括：
- **QEMU Lockup 错误** - UNALIGNFAULT 导致的系统崩溃
- **网络测试编译错误** - LWIP 未启用时的链接问题
- **网络驱动编译错误** - 缺少 LWIP 头文件
