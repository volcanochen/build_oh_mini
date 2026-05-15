---
name: build_oh_mini
description: 构建 OpenHarmony mini 系统（LiteOS-M 内核）的完整指南。涵盖环境检查、系统构建、配置修改和常见问题解决。当用户需要：(1) 构建 LiteOS-M 内核镜像，(2) 配置内核选项，(3) 启用内核测试套件，(4) 解决构建过程中的问题，(5) 生成 QEMU 可运行镜像时使用此技能。
---

# OpenHarmony Mini 系统构建指南

## ⚠️ 重要提醒

1. **删除编译产物需管理员权限时**：如需删除 `out` 目录等编译产物但权限不足，可进入 Docker 容器内以 root 用户删除：
   ```bash
   docker run --rm -v $(pwd):/home/openharmony -w /home/openharmony \
     swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_mini:3.2 \
     rm -rf out/
   ```

2. **问题分析必须具体**：遇到问题时，必须具体分析原因，不要猜测。可通过增加日志或打开日志等级来收集更多信息，例如在 BUILD.gn 中添加 `defines = [ "DEBUG_VERBOSE" ]`，或在代码中添加打印语句。

3. **记录决策过程**：选择某个方案后，请记录到文档（markdown 格式），记录好决策的原因和过程，便于后续回溯。

4. **先最小化验证**：先最小化验证方案改动是否有效，再进行更大范围的任务。例如，构建过程中某个模块编译出错，修改后先单独验证该模块，确认通过后再回到完整构建流程。

5. **增量编译使用 `--fast-rebuild`**：`hb build` 的增量编译应使用 `hb build --fast-rebuild`，跳过 preloader 和 GN gen，只运行 ninja 增量编译。

6. **⚠️ `hb set` 和 `hb build -f` 会导致增量失效**：`hb set xxx` 会重新生成产品配置，`hb build -f` 会强制全量编译，两者都会使增量编译失效，请谨慎选择。

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

### Docker 构建（推荐）

拉取 Docker 镜像：

```bash
docker pull swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_mini:3.2
```

进入 Docker 构建环境：

```bash
cd "$WORK_DIR"
docker run -it -v $(pwd):/home/openharmony swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_mini:3.2
```

进入容器后执行构建：

```bash
python3 build.py -p qemu_mini_system_demo@ohemu
```

### 本地构建

#### 首次完整构建

```bash
cd "$WORK_DIR"
hb set
# 选择: qemu_mini_system_demo
hb build -f
```

### 增量构建

```bash
hb build --fast-rebuild
```

> ⚠️ `hb set` 和 `hb build -f` 会重新生成产品配置，导致增量失效。

### 清理构建产物

如需清理但无管理员权限，可在 Docker 中执行：

```bash
docker run --rm -v $(pwd):/home/openharmony -w /home/openharmony \
  swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_mini:3.2 \
  rm -rf out/
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
