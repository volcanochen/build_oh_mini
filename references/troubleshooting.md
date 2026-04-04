# Mini 系统常见问题解决

## 目录

1. [QEMU Lockup 错误](#问题-1-qemu-lockup-错误)
2. [网络测试编译错误](#问题-2-网络测试编译错误)
3. [网络驱动编译错误](#问题-3-网络驱动编译错误)

---

## 问题 1: QEMU Lockup 错误

### 症状

```
qemu: fatal: Lockup: can't escalate 3 to HardFault (current priority -1)
```

### 原因分析

UNALIGNFAULT（未对齐访问故障）被启用，导致 `memset` 等函数在执行未对齐内存访问时触发 UsageFault，由于优先级无法升级到 HardFault，系统崩溃。

LiteOS-M 默认配置 `LOSCFG_ARCH_UNALIGNED_EXC=y` 会启用未对齐访问异常检测，但 QEMU 模拟的 Cortex-M4 在处理这种情况时与真实硬件有差异。

### 解决方案

修改 `kernel/liteos_m/arch/arm/cortex-m4/gcc/los_interrupt.c`：

```c
// 在 HalArchInit 函数中添加:
*(volatile UINT32 *)OS_NVIC_CCR |= (DIV0FAULT);
*(volatile UINT32 *)OS_NVIC_CCR &= ~(UNALIGNFAULT);
```

### 完整修改示例

```c
LITE_OS_SEC_TEXT_INIT VOID HalArchInit(VOID)
{
    HalHwiInit();

    /* Enable DIV 0, disable unaligned exception for QEMU compatibility */
    *(volatile UINT32 *)OS_NVIC_CCR |= (DIV0FAULT);
    *(volatile UINT32 *)OS_NVIC_CCR &= ~(UNALIGNFAULT);
}
```

### 替代方案

在配置文件中禁用：

```
LOSCFG_ARCH_UNALIGNED_EXC=n
```

---

## 问题 2: 网络测试编译错误

### 症状

```
undefined reference to 'ActsNetTest'
```

### 原因分析

启用了内核测试 (`LOSCFG_KERNEL_TEST=y`)，但未启用 LWIP 网络协议栈 (`LOSCFG_NET_LWIP=n`)。测试套件代码无条件调用 `ActsNetTest()`，导致链接错误。

### 解决方案

#### 步骤 1: 修改 xts_test.c

文件：`kernel/liteos_m/testsuites/unittest/xts/xts_test.c`

```c
#include "los_config.h"

void XtsTestSuite(void)
{
    IpcSemApiTest();
    IoFuncTest();
    MathFuncTest();
    MemFuncTest();
#if (LOSCFG_NET_LWIP == 1)
    ActsNetTest();
#endif
    PthreadFuncTest();
    SchedApiFuncTest();
    SysApiFuncTest();
    TimeFuncTest();
    CmsisFuncTest();
}
```

#### 步骤 2: 修改 BUILD.gn

文件：`kernel/liteos_m/testsuites/unittest/xts/BUILD.gn`

```gn
deps = [
  "cmsis:cmsis_test",
  "io:io_test",
  "ipc:ipc_test",
  "math:math_test",
  "mem:mem_test",
  "process:pthread_test",
  "sched:sched_test",
  "sys:system_test",
  "time:time_test",
]
if (defined(LOSCFG_NET_LWIP) && LOSCFG_NET_LWIP) {
  deps += [ "net:net_test" ]
}
```

---

## 问题 3: 网络驱动编译错误

### 症状

```
lwip/opt.h: No such file or directory
```

### 原因分析

禁用 LWIP 后，网络驱动代码仍然被编译，但缺少 LWIP 头文件。

### 解决方案

#### 步骤 1: 修改 BUILD.gn

文件：`device/qemu/arm_mps2_an386/liteos_m/board/BUILD.gn`

```gn
sources = [
  "driver/arm_uart_drv.c",
  "driver/uart.c",
  "fs/ff_gen_drv.c",
  "libc/dprintf.c",
  "main.c",
  "startup.s",
]
if (defined(LOSCFG_NET_LWIP) && LOSCFG_NET_LWIP) {
  sources += [ "driver/net/lan9118_eth_drv.c" ]
  include_dirs += [ "driver/net/" ]
}
```

#### 步骤 2: 修改 main.c

文件：`device/qemu/arm_mps2_an386/liteos_m/board/main.c`

```c
#include "los_config.h"
#include "uart.h"
#include "los_debug.h"

#if (LOSCFG_NET_LWIP == 1)
#include "lan9118_eth_drv.h"
#endif

// ... 其他代码 ...

int main(void)
{
    // ... 初始化代码 ...
    
    Uart0RxIrqRegister();

#if (LOSCFG_NET_LWIP == 1)
    NetInit();
#endif

    // ... 其他代码 ...
}
```

---

## 其他常见问题

### prebuilts 下载失败

**症状**：`./build/prebuilts_download.sh` 执行失败

**解决方案**：
```bash
# 手动下载
./build/prebuilts_download.sh --skip-ssl-verify
```

### hb 命令不存在

**症状**：`hb: command not found`

**解决方案**：
```bash
pip3 install build/lite
# 或
pip3 install ohos-build
```

### 构建缓存问题

**症状**：修改配置后构建结果未变化

**解决方案**：
```bash
# 清理构建输出
rm -rf out/arm_mps2_an386/qemu_mini_system_demo

# 重新构建
hb build -f
```
