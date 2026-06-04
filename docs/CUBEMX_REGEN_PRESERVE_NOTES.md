# CubeMX Regeneration Preserve Notes

This project contains several hand-written STM32N6 boot-chain changes. Some are already inside `USER CODE` blocks and should survive CubeMX regeneration. Some are generated-code replacements and must be checked after regenerating code from `TEST_CMAKE.ioc`.

## Already Protected In USER CODE

### AppliSecure/Core/Src/main.c

- Secure debug LED helper definitions, prototypes, variables, and helper functions are in `USER CODE` blocks.
- Non-Secure vector capture helpers are in `USER CODE` blocks.
- Fault/debug behavior in `Error_Handler()` is in `USER CODE`.
- The critical RIF/GPIO Non-Secure permission setup is re-applied in `USER CODE BEGIN RIF_Init 2`:
  - `XSPI2` and `XSPIM` remain Secure.
  - `USART3`, `GPIOD`, and `GPIOO` are Non-Secure.
  - `PD8`, `PD9`, and `PO1` are configured as Non-Secure GPIO pins.

This duplicate RIF setup is intentional. If CubeMX rewrites the generated RIF code above it, the USER block still restores the permissions needed by the current working boot chain.

### AppliNonSecure/Core/Src/main.c

- The LED toggle and USART3 log output are inside `USER CODE` blocks.
- Current expected serial output on every LED toggle:
  - `[AppNS] LED toggle <count>, PO1=<state>`

### FSBL/Core/Src/main.c

- The FSBL boot call path is inside `USER CODE` blocks.

## Source Settings Updated For CubeMX

### TEST_CMAKE.ioc

USART3 has been changed from MSI to PCLK1:

- `RCC.USART3ClockSelection=RCC_USART3CLKSOURCE_PCLK1`
- `RCC.USART3Freq_Value=200000000`

This is important because the working generated C file uses `RCC_USART3CLKSOURCE_PCLK1`. If CubeMX keeps USART3 on MSI, the regenerated `AppliNonSecure/Core/Src/stm32n6xx_hal_msp.c` may break UART output again.

## Must Re-check After CubeMX Regeneration

### AppliSecure/Core/Src/main.c: `NonSecure_Init()`

This function body is outside `USER CODE` and contains important custom logic:

- Sets `SCB_NS->VTOR` to the Non-Secure vector table address.
- Reads the Non-Secure MSP and reset handler.
- Falls back to the secure alias vector if needed.
- Validates the Non-Secure MSP and reset handler before jumping.
- Calls `__TZ_set_MSP_NS(ns_msp)` and then jumps to the Non-Secure reset handler.

If CubeMX regenerates this function, compare it with Git history and restore the custom validation/jump logic if needed.

### AppliNonSecure/Core/Src/stm32n6xx_hal_msp.c: USART3 MSP Init

This file still contains generated code outside `USER CODE`.

After regeneration, check:

- `PeriphClkInitStruct.Usart3ClockSelection` should be `RCC_USART3CLKSOURCE_PCLK1`.
- The USART3 peripheral clock config failure path was changed to `return;` instead of entering `Error_Handler()`.

The `.ioc` update should preserve the PCLK1 selection, but still verify the generated C file.

### FSBL/Core/Inc/stm32_extmem_conf.h

This file has no `USER CODE` markers. The whole external-memory LRUN boot configuration is therefore at risk during regeneration or middleware reconfiguration.

Important current values:

- `EXTMEM_DRIVER_NOR_SFDP` is enabled.
- `EXTMEM_SAL_XSPI` is enabled.
- `EXTMEM_LRUN_DESTINATION_ADDRESS` is `0x34000000`.
- Secure image source is `EXTMEMORY_1` at `0x00100000`, size `0x00010000`.
- Non-Secure LRUN is enabled.
- Non-Secure destination is `0x34100000`.
- Non-Secure source is `0x180000`.
- `EXTMEM_HEADER_OFFSET` is `0x400`.
- `extmem_list_config` currently contains one NOR SFDP XSPI2 device.

After CubeMX regeneration, this file should be compared carefully before flashing.

### AppliSecure/Core/Inc/partition_stm32n657xx.h

The custom `TZ_SAU_Setup()` implementation is inside `USER CODE`, but many SAU region macro definitions are outside the lower `USER CODE` block.

Current important SAU regions:

- Region 0: NSC veneer, using `_sNSCVeneer` to `_eNSCVeneer`.
- Region 1: Non-Secure memory `0x24100000` to `0x241FFFFF`.
- Region 2: Non-Secure peripherals `0x40000000` to `0x4FFFFFFF`.

After regeneration, verify these regions still match the boot-chain memory map.

### FSBL/Core/Src/stm32n6xx_hal_msp.c

The XSPI2 MSP init contains user-protected VDDIO/HSLV preparation code, but generated XSPI clock code around it should still be checked after regeneration. If external flash stops booting after camera configuration, inspect XSPI2 clock source/divider and the external-memory config first.

## Quick Post-Regeneration Test

After configuring the camera in CubeMX and regenerating code:

1. Build FSBL, AppliSecure, and AppliNonSecure.
2. Flash the full boot chain: FSBL -> Secure -> NonSecure.
3. Confirm LED `PO1` blinks.
4. Confirm USART3 prints `[AppNS] LED toggle ...`.
5. If LED does not blink, first check Secure RIF/GPIO permissions and `NonSecure_Init()`.
6. If LED blinks but UART is silent, first check USART3 clock source and PD8/PD9 Non-Secure permissions.
