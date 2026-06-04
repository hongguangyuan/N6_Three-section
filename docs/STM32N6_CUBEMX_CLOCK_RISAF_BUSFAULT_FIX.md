# STM32N6 CubeMX Regeneration Clock/RISAF BusFault Fix

本文记录一次 CubeMX 重新生成代码并修改时钟后，工程出现 LED 变慢、串口无输出、LED 快闪 8 次后暂停的排查和修复过程。

## Final Result

修复后，`Flash: BootChain EXT (FSBL->Secure->NonSecure)` 烧录完整三段镜像，NonSecure LED 闪烁恢复正常。

最终确认：8 次快闪不是普通 LED 主循环，而是 Secure 侧 `BusFault_Handler()` 的故障码。

对应代码：

- `AppliSecure/Core/Src/stm32n6xx_it.c`
- `FAULT_LED_BUSFAULT = 8UL`
- `BusFault_Handler()` 调用 `FaultBlink(FAULT_LED_BUSFAULT)`

## Symptom

CubeMX 重新生成并修改时钟后，出现以下现象：

- LED 闪烁明显变慢。
- USART3 没有输出。
- LED 快速闪烁约 8 次，然后停顿，再重复。

其中 8 次快闪说明程序并没有稳定跑在 `AppliNonSecure/Core/Src/main.c` 的普通 LED 循环中，而是进入了 Secure fault handler。

## Root Cause

问题不只是 USART3 或单个时钟源。

CubeMX 重新生成后，Secure 侧 `SystemIsolation_Config()` 中原本工作正常的 RIF/RISAF 隔离配置被改掉或丢失，导致 Secure 跳转到 NonSecure 后，访问 NonSecure SRAM / 外设安全属性不匹配，从而触发 Secure BusFault。

对本工程尤其关键的是 RISAF 配置。

FSBL 使用 LRUN：

- 从外部 Flash 读取 Secure 镜像。
- 从外部 Flash 读取 NonSecure 镜像。
- 将 NonSecure 镜像加载到 SRAM2。
- Secure 再设置 `SCB_NS->VTOR` 并跳转到 NonSecure reset handler。

如果 RISAF 没有把 NonSecure 运行区域正确配置成 NonSecure，Secure 跳转或 NonSecure 访问内存时就可能 BusFault。

## Clock Alignment

排查时也对照了 GitHub 工作版本，将 FSBL 关键时钟恢复到已验证配置。

`FSBL/Core/Src/main.c` 中的工作配置：

```c
RCC_OscInitStruct.PLL1.PLLSource = RCC_PLLSOURCE_HSI;
RCC_OscInitStruct.PLL1.PLLM = 4;
RCC_OscInitStruct.PLL1.PLLN = 75;
RCC_ClkInitStruct.AHBCLKDivider = RCC_HCLK_DIV2;
```

对应频率：

```text
HSI = 64 MHz
PLL1 input = 64 / 4 = 16 MHz
PLL1 output = 16 * 75 = 1200 MHz
CPUCLK = IC1 = 1200 / 2 = 600 MHz
SYSBCLK = IC2 = 1200 / 3 = 400 MHz
SYSCCLK = IC6 = 1200 / 4 = 300 MHz
SYSDCLK = IC11 = 1200 / 3 = 400 MHz
HCLK/PCLK = 400 / 2 = 200 MHz
```

`TEST_CMAKE.ioc` 中同步保留：

```text
RCC.PLL1Source=RCC_PLLSOURCE_HSI
RCC.FREFDIV1=4
RCC.HPRE_Div=RCC_HCLK_DIV2
RCC.AHB1234Freq_Value=200000000
RCC.APB1Freq_Value=200000000
RCC.APB2Freq_Value=200000000
RCC.APB4Freq_Value=200000000
RCC.APB5Freq_Value=200000000
RCC.USART3Freq_Value=200000000
RCC.XSPI2Freq_Value=200000000
```

注意：GitHub 旧 `.ioc` 中 USART3 曾显示 MSI，但实际工作的 C 代码里 `AppliNonSecure/Core/Src/stm32n6xx_hal_msp.c` 使用的是：

```c
PeriphClkInitStruct.Usart3ClockSelection = RCC_USART3CLKSOURCE_PCLK1;
```

所以当前以实际 C 代码为准，USART3 使用 PCLK1。

## XSPI2 Alignment

FSBL 的 XSPI2 初始化也恢复到 GitHub 工作版。

`FSBL/Core/Src/main.c`：

```c
hxspi2.Init.MemorySize = HAL_XSPI_SIZE_1GB;
hxspi2.Init.DelayHoldQuarterCycle = HAL_XSPI_DHQC_ENABLE;
```

`TEST_CMAKE.ioc`：

```text
XSPI2.MemorySize=HAL_XSPI_SIZE_1GB
```

这些设置会影响外部 Flash 访问稳定性。即使本次最终 BusFault 主因是 RISAF，也建议 CubeMX 再生成后继续检查这些值。

## Critical Fix: Restore Secure RIF/RISAF

最终让 LED 恢复正常的关键修复在：

```text
AppliSecure/Core/Src/main.c
SystemIsolation_Config()
```

需要保留的配置：

```c
HAL_RIF_RISC_SetSlaveSecureAttributes(RIF_RISC_PERIPH_INDEX_XSPI2,
                                      RIF_ATTRIBUTE_SEC | RIF_ATTRIBUTE_NPRIV);
HAL_RIF_RISC_SetSlaveSecureAttributes(RIF_RISC_PERIPH_INDEX_XSPIM,
                                      RIF_ATTRIBUTE_SEC | RIF_ATTRIBUTE_NPRIV);
HAL_RIF_RISC_SetSlaveSecureAttributes(RIF_RISC_PERIPH_INDEX_USART3,
                                      RIF_ATTRIBUTE_NSEC | RIF_ATTRIBUTE_NPRIV);
HAL_RIF_RISC_SetSlaveSecureAttributes(RIF_RCC_PERIPH_INDEX_GPIOD,
                                      RIF_ATTRIBUTE_NSEC | RIF_ATTRIBUTE_NPRIV);
HAL_RIF_RISC_SetSlaveSecureAttributes(RIF_RCC_PERIPH_INDEX_GPIOO,
                                      RIF_ATTRIBUTE_NSEC | RIF_ATTRIBUTE_NPRIV);
```

RISAF base region：

```c
RISAF_BaseRegionConfig_t risaf_base_config = {0};
__HAL_RCC_RISAF_CLK_ENABLE();

risaf_base_config.EndAddress = 0xfffff;
risaf_base_config.Filtering = RISAF_FILTER_ENABLE;
risaf_base_config.ReadWhitelist = 255;
risaf_base_config.WriteWhitelist = 255;
risaf_base_config.Secure = RIF_ATTRIBUTE_NSEC;
risaf_base_config.PrivWhitelist = RIF_CID_NONE;
risaf_base_config.StartAddress = 0x0000;
HAL_RIF_RISAF_ConfigBaseRegion(RISAF3, RISAF_REGION_1, &risaf_base_config);

risaf_base_config.EndAddress = 0x9bfff;
risaf_base_config.Secure = RIF_ATTRIBUTE_SEC;
HAL_RIF_RISAF_ConfigBaseRegion(RISAF2, RISAF_REGION_1, &risaf_base_config);

risaf_base_config.EndAddress = 0x63fff;
HAL_RIF_RISAF_ConfigBaseRegion(RISAF7, RISAF_REGION_1, &risaf_base_config);
```

GPIO NonSecure pin attributes：

```c
__HAL_RCC_GPIOD_CLK_ENABLE();
__HAL_RCC_GPIOO_CLK_ENABLE();

HAL_GPIO_ConfigPinAttributes(GPIOD, GPIO_PIN_8, GPIO_PIN_NSEC);
HAL_GPIO_ConfigPinAttributes(GPIOD, GPIO_PIN_9, GPIO_PIN_NSEC);
HAL_GPIO_ConfigPinAttributes(GPIOO, GPIO_PIN_1, GPIO_PIN_NSEC);
```

## Diagnostic Notes

8 次快闪的含义：

```text
5 pulses = NMI
6 pulses = HardFault
7 pulses = MemManage
8 pulses = BusFault
9 pulses = UsageFault
```

如果看到 8 次快闪，应优先检查：

1. Secure `SystemIsolation_Config()` 中 RISAF 配置是否仍在。
2. `AppliSecure/Core/Inc/partition_stm32n657xx.h` 中 SAU 是否仍把 NonSecure SRAM 配出来。
3. `NonSecure_Init()` 中 `SCB_NS->VTOR`、MSP、Reset_Handler 地址是否仍和 LRUN 地址一致。
4. FSBL 外部 Flash / LRUN 配置是否被 CubeMX 或 middleware 更新覆盖。
5. XSPI2 clock / memory size / DHQC 是否仍和工作版一致。

## Files To Re-check After CubeMX Regeneration

每次 CubeMX 重新生成后，至少检查以下文件：

```text
AppliSecure/Core/Src/main.c
AppliSecure/Core/Inc/partition_stm32n657xx.h
AppliNonSecure/Core/Src/stm32n6xx_hal_msp.c
FSBL/Core/Src/main.c
FSBL/Core/Src/stm32n6xx_hal_msp.c
FSBL/Core/Inc/stm32_extmem_conf.h
TEST_CMAKE.ioc
```

重点不是只看是否能编译，而是确认 Secure/NonSecure 的安全隔离、LRUN 地址和外部 Flash 初始化仍然匹配。

## Verification

修复后执行：

```powershell
make -C Makefile
```

编译通过。

然后执行 VSCode 任务：

```text
Flash: BootChain EXT (FSBL->Secure->NonSecure)
```

验证标准：

- LED 正常按 NonSecure 主循环闪烁。
- 不再出现 8 次快闪后暂停。
- USART3 应输出 NonSecure 启动/heartbeat/LED toggle 日志。

