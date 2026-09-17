# CMake 裸机构建完整指南（通用模板）

> 本文档是一个 **可复用的 CMake 裸机模板**，适用于任何 ARM Cortex-M 芯片（HC32、STM32、GD32、CH32、AT32 等）。复制给 AI 即可复刻整个工程。

## 使用方法

1. 找到你的芯片型号，在模板中搜索 `TODO` 标记，全部替换
2. 下载对应芯片的启动文件和链接脚本（芯片厂商提供）
3. 编译

## 前提条件

- VS Code 已安装 CMake Tools 扩展（`ms-vscode.cmake-tools`）
- ARM GCC 工具链（`arm-none-eabi-gcc`）已安装（参见同目录 vscode裸机开发环境.md）
- Ninja 已安装（见下方）

### 安装 Ninja

```powershell
winget install Ninja-build.Ninja
```

或从 https://github.com/ninja-build/ninja/releases 下载 ninja-win.zip，解压后把 ninja.exe 放到 PATH 目录下。

验证：
```powershell
ninja --version
```

---

## 目录结构

```
project/
├── CMakeLists.txt              <- 主构建文件
├── toolchain.cmake             <- 交叉编译工具链（唯一需要改路径的文件）
├── .vscode/
│   ├── settings.json
│   ├── launch.json
│   └── tasks.json
├── src/
│   ├── main.c
│   └── system.c                <- 系统初始化（时钟、中断）
├── bsp/
│   └── gpio.c                  <- 板级支持包（可选）
├── drivers/
│   └── HC32L190/               <- 芯片 CMSIS 头文件（从厂商 SDK 复制）
├── startup/
│   └── startup_hc32l190.s      <- 启动文件（从厂商 SDK 复制）
└── linker/
    └── hc32l190_flash.ld       <- 链接脚本（从厂商 SDK 复制）
```

---

## 文件 1：toolchain.cmake

```cmake
# ============================================
# 交叉编译工具链文件 - ARM GCC（通用）
# ============================================

set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

# ---- TODO: 改成你的 ARM GCC 路径 ----
set(TOOLCHAIN_DIR "C:/Users/10733/.eide/tools/gcc_arm")

set(CMAKE_C_COMPILER   "${TOOLCHAIN_DIR}/bin/arm-none-eabi-gcc.exe")
set(CMAKE_CXX_COMPILER "${TOOLCHAIN_DIR}/bin/arm-none-eabi-g++.exe")
set(CMAKE_ASM_COMPILER "${TOOLCHAIN_DIR}/bin/arm-none-eabi-gcc.exe")
set(CMAKE_OBJCOPY      "${TOOLCHAIN_DIR}/bin/arm-none-eabi-objcopy.exe")
set(CMAKE_OBJDUMP      "${TOOLCHAIN_DIR}/bin/arm-none-eabi-objdump.exe")
set(CMAKE_SIZE         "${TOOLCHAIN_DIR}/bin/arm-none-eabi-size.exe")

# 交叉编译必须
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

# ---- TODO: 根据芯片改 CPU 参数 ----
# Cortex-M0:  -mcpu=cortex-m0  -mthumb
# Cortex-M3:  -mcpu=cortex-m3  -mthumb
# Cortex-M4:  -mcpu=cortex-m4  -mthumb -mfloat-abi=soft -mfpu=fpv4-sp-d16
# Cortex-M7:  -mcpu=cortex-m7  -mthumb -mfloat-abi=hard -mfpu=fpv5-d16
set(CPU_FLAGS "-mcpu=cortex-m4 -mthumb -mfloat-abi=soft")

set(CMAKE_C_FLAGS_INIT
    "${CPU_FLAGS} -fdata-sections -ffunction-sections -Wall -fno-common"
)
set(CMAKE_CXX_FLAGS_INIT
    "${CPU_FLAGS} -fdata-sections -ffunction-sections -fno-exceptions -fno-rtti -Wall -fno-common"
)
set(CMAKE_ASM_FLAGS_INIT
    "${CPU_FLAGS} -x assembler-with-cpp"
)
set(CMAKE_EXE_LINKER_FLAGS_INIT
    "${CPU_FLAGS} -Wl,--gc-sections --specs=nano.specs --specs=nosys.specs -Wl,--print-memory-usage"
)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

---

## 文件 2：CMakeLists.txt

```cmake
# ============================================
# 主构建文件（通用模板）
# ============================================
cmake_minimum_required(VERSION 3.20)
project(my_project C ASM)

# ---- TODO: 改成你的芯片 ----
set(MCU_LINE    HC32L190)          # 芯片系列（编译宏用）
set(MCU_DEVICE  HC32L190FP6TA)     # 完整型号（看数据手册）

# ---- 源文件 ----
file(GLOB_RECURSE SOURCES
    "${CMAKE_SOURCE_DIR}/src/*.c"
    "${CMAKE_SOURCE_DIR}/bsp/*.c"
    "${CMAKE_SOURCE_DIR}/drivers/*.c"       # 如果用厂商固件库
    "${CMAKE_SOURCE_DIR}/startup/*.s"
)

# ---- 头文件路径 ----
set(INCLUDE_DIRS
    "${CMAKE_SOURCE_DIR}/src"
    "${CMAKE_SOURCE_DIR}/bsp"
    "${CMAKE_SOURCE_DIR}/drivers/${MCU_LINE}"
)

# ---- TODO: 改成你的链接脚本路径 ----
set(LINKER_SCRIPT "${CMAKE_SOURCE_DIR}/linker/hc32l190_flash.ld")

# ---- 编译宏 ----
add_definitions(
    -D${MCU_LINE}
    -DHSE_VALUE=32000000           # 外部晶振频率，按实际改
)

# ---- 编译目标 ----
add_executable(${PROJECT_NAME}.elf
    ${SOURCES}
)

target_include_directories(${PROJECT_NAME}.elf PRIVATE
    ${INCLUDE_DIRS}
)

target_link_options(${PROJECT_NAME}.elf PRIVATE
    -T${LINKER_SCRIPT}
    -Wl,-Map=${PROJECT_NAME}.map
    -Wl,--start-group
    -lc -lm -lnosys
    -Wl,--end-group
)

# ---- 后处理：生成 .bin .hex ----
add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O binary
        $<TARGET_FILE:${PROJECT_NAME}.elf>
        ${PROJECT_NAME}.bin
    COMMAND ${CMAKE_OBJCOPY} -O ihex
        $<TARGET_FILE:${PROJECT_NAME}.elf>
        ${PROJECT_NAME}.hex
    COMMAND ${CMAKE_SIZE} $<TARGET_FILE:${PROJECT_NAME}.elf>
    COMMENT ">> Generating ${PROJECT_NAME}.bin / .hex"
)
```

---

## 文件 3：linker/hc32l190_flash.ld（模板）

> 此为通用模板，**必须替换为厂商提供的链接脚本**。不同芯片的 Flash/RAM 大小和地址不同。

```ld
/* ============================================
 * 通用链接脚本模板
 * TODO: 改 MEMORY 块中的地址和大小
 * ============================================ */

ENTRY(Reset_Handler)

_Min_Heap_Size  = 0x200;
_Min_Stack_Size = 0x400;

/* TODO: 改 Flash 和 RAM 地址/大小 */
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x00000000, LENGTH = 256K    /* 华大通常从 0x00000000 */
    SRAM  (rwx) : ORIGIN = 0x20000000, LENGTH = 32K
}

SECTIONS
{
    .isr_vector :
    {
        . = ALIGN(4);
        KEEP(*(.isr_vector))
        . = ALIGN(4);
    } > FLASH

    .text :
    {
        . = ALIGN(4);
        *(.text)
        *(.text*)
        *(.glue_7)
        *(.glue_7t)
        KEEP(*(.init))
        KEEP(*(.fini))
        . = ALIGN(4);
        _etext = .;
    } > FLASH

    .rodata :
    {
        . = ALIGN(4);
        *(.rodata)
        *(.rodata*)
        . = ALIGN(4);
    } > FLASH

    _sidata = LOADADDR(.data);

    .data :
    {
        . = ALIGN(4);
        _sdata = .;
        *(.data)
        *(.data*)
        . = ALIGN(4);
        _edata = .;
    } > SRAM AT> FLASH

    .bss :
    {
        . = ALIGN(4);
        _sbss = .;
        *(.bss)
        *(.bss*)
        *(COMMON)
        . = ALIGN(4);
        _ebss = .;
    } > SRAM

    ._user_heap_stack :
    {
        . = ALIGN(8);
        . = . + _Min_Heap_Size;
        . = . + _Min_Stack_Size;
        . = ALIGN(8);
    } > SRAM
}
```

---

## 文件 4：startup/startup_hc32l190.s（模板）

> 此为通用模板，**必须替换为厂商提供的启动文件**。不同芯片的中断向量表不同。

```asm
/**
 * 通用启动文件模板
 * TODO: 替换为厂商 SDK 中的 startup_xxx.s
 * 通常在 SDK 的 device/arm/ 目录下
 */

.syntax unified
.cpu cortex-m4
.fpu softvfp
.thumb

.word _estack

.section .isr_vector, "a", %progbits
.type g_pfnVectors, %object

g_pfnVectors:
    .word _estack
    .word Reset_Handler
    .word NMI_Handler
    .word HardFault_Handler
    .word MemManage_Handler
    .word BusFault_Handler
    .word UsageFault_Handler
    .word 0
    .word 0
    .word 0
    .word 0
    .word SVC_Handler
    .word DebugMon_Handler
    .word 0
    .word PendSV_Handler
    .word SysTick_Handler

    /* TODO: 按芯片数据手册添加外部中断 */
    .word 0
    .word 0
    .word 0
    .word 0
    .word 0
    .word 0
    .word 0

    .size g_pfnVectors, . - g_pfnVectors

.section .text.Reset_Handler
.weak Reset_Handler
.type Reset_Handler, %function
Reset_Handler:
    ldr sp, =_estack

    /* 拷贝 .data */
    ldr r0, =_sdata
    ldr r1, =_edata
    ldr r2, =_sidata
    movs r3, #0
    b LoopCopyDataInit

CopyDataInit:
    ldr r4, [r2, r3]
    str r4, [r0, r3]
    adds r3, r3, #4

LoopCopyDataInit:
    adds r4, r0, r3
    cmp r4, r1
    bcc CopyDataInit

    /* 清零 .bss */
    ldr r2, =_sbss
    ldr r4, =_ebss
    movs r3, #0
    b LoopFillZerobss

FillZerobss:
    str r3, [r2]
    adds r2, r2, #4

LoopFillZerobss:
    cmp r2, r4
    bcc FillZerobss

    bl main
    b .

    .size Reset_Handler, . - Reset_Handler

.section .text.Default_Handler, "ax", %progbits
Default_Handler:
Infinite_Loop:
    b Infinite_Loop
    .size Default_Handler, . - Default_Handler

.weak NMI_Handler
.thumb_set NMI_Handler, Default_Handler

.weak HardFault_Handler
.thumb_set HardFault_Handler, Default_Handler

.weak MemManage_Handler
.thumb_set MemManage_Handler, Default_Handler

.weak BusFault_Handler
.thumb_set BusFault_Handler, Default_Handler

.weak UsageFault_Handler
.thumb_set UsageFault_Handler, Default_Handler

.weak SVC_Handler
.thumb_set SVC_Handler, Default_Handler

.weak DebugMon_Handler
.thumb_set DebugMon_Handler, Default_Handler

.weak PendSV_Handler
.thumb_set PendSV_Handler, Default_Handler

.weak SysTick_Handler
.thumb_set SysTick_Handler, Default_Handler
```

---

## 文件 5：src/main.c

```c
#include "hc32l190.h"      /* TODO: 改成你的芯片头文件 */

void delay(volatile unsigned int count)
{
    while (count--)
    {
        __asm("nop");
    }
}

int main(void)
{
    /* TODO: 初始化时钟、GPIO 等 */

    while (1)
    {
        /* TODO: 你的业务逻辑 */
        delay(500000);
    }
}
```

---

## 文件 6：.vscode/settings.json

```json
{
    "cmake.buildDirectory": "${workspaceFolder}/build",
    "cmake.generator": "Ninja",
    "cmake.configureSettings": {
        "CMAKE_TOOLCHAIN_FILE": "${workspaceFolder}/toolchain.cmake"
    },
    "cmake.configureOnOpen": true,
    "cmake.configureArgs": [
        "-DCMAKE_BUILD_TYPE=Debug"
    ]
}
```

---

## 文件 7：.vscode/launch.json

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug (J-Link)",
            "type": "cortex-debug",
            "request": "launch",
            "servertype": "jlink",
            "device": "HC32L190",
            "gdbPath": "C:/Users/10733/.eide/tools/gcc_arm/bin/arm-none-eabi-gdb.exe",
            "executable": "${workspaceFolder}/build/my_project.elf",
            "runToEntryPoint": "main",
            "serverArgs": [
                "-singlerun",
                "-if SWD",
                "-speed 4000"
            ],
            "showDevDebugOutput": "raw"
        }
    ]
}
```

---

## 文件 8：.vscode/tasks.json

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build",
            "type": "shell",
            "command": "cmake",
            "args": [
                "--build", "${workspaceFolder}/build",
                "--config", "Debug"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "problemMatcher": ["$gcc"]
        },
        {
            "label": "clean",
            "type": "shell",
            "command": "cmake",
            "args": [
                "--build", "${workspaceFolder}/build",
                "--target", "clean"
            ],
            "problemMatcher": []
        }
    ]
}
```

---

## 编译命令（终端）

```bash
mkdir build && cd build
cmake -G Ninja -DCMAKE_TOOLCHAIN_FILE=../toolchain.cmake -DCMAKE_BUILD_TYPE=Debug ..
ninja
```

---

## TODO 清单

用之前搜索 `TODO`，逐个替换：

| TODO | 改成什么 |
|------|---------|
| `TOOLCHAIN_DIR` | 你的 arm-none-eabi-gcc 安装路径 |
| `MCU_LINE` | 芯片系列名（如 HC32L190、GD32F303） |
| `MCU_DEVICE` | 完整芯片型号（J-Link 调试用） |
| `CPU_FLAGS` | Cortex-M0/M3/M4/M7 对应参数 |
| `HSE_VALUE` | 外部晶振频率（看原理图） |
| `MEMORY` 块 | Flash/RAM 起始地址和大小（看数据手册） |
| 链接脚本 | 用厂商 SDK 提供的 `.ld` 文件 |
| 启动文件 | 用厂商 SDK 提供的 `startup_xxx.s` |
| 头文件 | 复制厂商 SDK 的 CMSIS 头文件到 `drivers/` |

---

## 厂商 SDK 下载

| 芯片系列 | 厂商 | SDK 地址 |
|---------|------|---------|
| HC32L190/L110 | 华大半导体 | [HDSC 官网](https://www.hdsc.com.cn) |
| HC32F030/F445 | 华大半导体 | 同上 |
| STM32F1/G4/L4 | ST | [STM32CubeMX](https://www.st.com/stm32cubemx) |
| GD32F1/F3/E1 | 兆易创新 | [GD32 官网](https://www.gd32mcu.com) |
| CH32V/F103 | 沁恒 | [WCH 官网](https://www.wch-ic.com) |
| AT32F403/413 | 雅特力 | [AT32 官网](https://www.arterytek.com) |

SDK 里找以下文件，复制到项目对应目录：

```
SDK/
├── device/
│   ├── startup_xxx.s        -> 复制到 startup/
│   └── linker/
│       └── xxx_flash.ld     -> 复制到 linker/
├── CMSIS/
│   ├── xxx.h                -> 复制到 drivers/xxx/
│   ├── system_xxx.h
│   └── core_cmX.h
└── peripheral/
    └── inc/ + src/          -> 复制到 drivers/xxx/（如用固件库）
```

---

## 常见问题

Q: 编译报 "arm-none-eabi-gcc: not found"
A: toolchain.cmake 里的 TOOLCHAIN_DIR 路径不对。

Q: 链接报 "cannot find -lnosys"
A: 把 --specs=nosys.specs 去掉，只保留 --specs=nano.specs。

Q: 烧录后不运行
A: 检查链接脚本 FLASH 起始地址。华大通常是 0x00000000，STM32 是 0x08000000。

Q: 中断不触发
A: startup 文件的中断向量表不完整，从厂商 SDK 重新复制。

Q: 如何切换 Debug/Release
A: .vscode/settings.json 里改 "-DCMAKE_BUILD_TYPE=Release"。