# CMake 工程完整指南

> 本文档基于华大 HC32L190 实际工程，复制给 AI 即可复刻整个 CMake 嵌入式项目。

## 前提条件

- VS Code 已安装 CMake Tools 扩展（`ms-vscode.cmake-tools`）
- ARM GCC 工具链已安装（参见同目录 VScode的EIDE开发环境.md）
- Ninja 已安装（`winget install Ninja-build.Ninja`）

验证环境：
```powershell
arm-none-eabi-gcc --version
ninja --version
cmake --version
```

---

## 一、Ninja 安装

```powershell
winget install Ninja-build.Ninja
```

安装后如果 VS Code 找不到 Ninja，把 ninja.exe 复制到 gcc 的 bin 目录：

```powershell
Copy-Item "C:\Users\10733\AppData\Local\Microsoft\WinGet\Links\ninja.exe" "C:\Users\10733\.eide\tools\gcc_arm_v15\bin\ninja.exe"
```

---

## 二、项目结构

```
project/
├── CMakeLists.txt              <- 主构建文件
├── build.cmake                 <- 一键编译脚本（可选）
├── gcc/
│   ├── arm-none-eabi.cmake     <- 交叉编译工具链文件
│   ├── startup_hc32l19x_gcc.S <- 启动汇编文件
│   ├── app.ld                  <- 链接脚本
│   └── syscalls.c              <- 系统调用桩
├── source/
│   ├── main.c                  <- 主程序
│   └── sys.c                   <- 系统初始化
├── driver/
│   ├── inc/                    <- 外设驱动头文件
│   └── src/                    <- 外设驱动源文件
├── APP/
│   ├── inc/                    <- 应用层头文件
│   └── src/                    <- 应用层源文件
├── common/
│   ├── Include/                <- CMSIS 头文件
│   ├── system_hc32l19x.c      <- 系统初始化
│   └── interrupts_hc32l19x.c  <- 中断处理
└── build/                      <- 编译输出目录
    ├── app.bin                 <- 烧录用
    ├── app.elf                 <- 调试用
    └── app.hex                 <- 烧录用
```

---

## 三、核心文件

### 3.1 gcc/arm-none-eabi.cmake（工具链文件）

```cmake
set(CMAKE_SYSTEM_NAME      Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

if(TOOLCHAIN_DIR)
    set(_prefix "${TOOLCHAIN_DIR}/arm-none-eabi-")
else()
    set(_prefix arm-none-eabi-)
endif()
if(CMAKE_HOST_WIN32)
    set(_ext .exe)
endif()

set(CMAKE_C_COMPILER   "${_prefix}gcc${_ext}")
set(CMAKE_ASM_COMPILER "${_prefix}gcc${_ext}")
set(CMAKE_OBJCOPY      "${_prefix}objcopy${_ext}" CACHE FILEPATH "")
set(CMAKE_OBJDUMP      "${_prefix}objdump${_ext}" CACHE FILEPATH "")
set(CMAKE_SIZE         "${_prefix}size${_ext}"    CACHE FILEPATH "")

set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

### 3.2 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.22)

if(NOT CMAKE_TOOLCHAIN_FILE)
    set(CMAKE_TOOLCHAIN_FILE ${CMAKE_CURRENT_SOURCE_DIR}/gcc/arm-none-eabi.cmake)
endif()
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Release | Debug" FORCE)
endif()

project(app C ASM)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_EXTENSIONS ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_C_FLAGS_RELEASE   "-Os -g")
set(CMAKE_C_FLAGS_DEBUG     "-Og -g3")
set(CMAKE_ASM_FLAGS_RELEASE "-g")
set(CMAKE_ASM_FLAGS_DEBUG   "-g3")

set(MCU_FLAGS -mcpu=cortex-m0plus -mthumb)

add_executable(app)
set_target_properties(app PROPERTIES OUTPUT_NAME app SUFFIX .elf)

target_compile_options(app PRIVATE ${MCU_FLAGS} -ffunction-sections -fdata-sections -Wall -Wno-format -Wno-unknown-pragmas -fdiagnostics-color=always)

target_include_directories(app PRIVATE
    source
    common
    driver/inc
    APP/inc
    common/Include
)

target_sources(app PRIVATE
    source/main.c
    source/sys.c

    common/interrupts_hc32l19x.c
    common/system_hc32l19x.c

    driver/src/adc.c
    driver/src/aes.c
    driver/src/bgr.c
    driver/src/bt.c
    driver/src/ddl.c
    driver/src/flash.c
    driver/src/gpio.c
    driver/src/lpm.c
    driver/src/lptim.c
    driver/src/lpuart.c
    driver/src/rtc.c
    driver/src/sysctrl.c
    driver/src/timer3.c
    driver/src/trim.c
    driver/src/uart.c
    driver/src/wdt.c

    APP/src/Ring_Buf.c
    APP/src/User_AES.c
    APP/src/User_Batt.c
    APP/src/User_BDS.c
    APP/src/User_Boot.c
    APP/src/User_CLK.c
    APP/src/User_Cmd.c
    APP/src/User_Debug.c
    APP/src/User_Flash.c
    APP/src/User_Info.c
    APP/src/User_LED.c
    APP/src/User_Lptimer.c
    APP/src/User_NetNR.c
    APP/src/User_NR_LED.c
    APP/src/User_SysTask.c

    gcc/startup_hc32l19x_gcc.S
    gcc/syscalls.c
)

set(LINKER_SCRIPT ${CMAKE_CURRENT_SOURCE_DIR}/gcc/app.ld)

target_link_options(app PRIVATE
    ${MCU_FLAGS}
    -T${LINKER_SCRIPT}
    --specs=nano.specs
    --specs=nosys.specs
    -Wl,-u,_printf_float
    -Wl,--gc-sections
    -Wl,--no-warn-rwx-segments
    -Wl,-Map=app.map
    -Wl,--print-memory-usage
)
set_target_properties(app PROPERTIES LINK_DEPENDS ${LINKER_SCRIPT})

add_custom_command(TARGET app POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE_NAME:app> app.bin
    COMMAND ${CMAKE_OBJCOPY} -O ihex   $<TARGET_FILE_NAME:app> app.hex
    COMMAND ${CMAKE_COMMAND} -E echo ""
    COMMAND ${CMAKE_SIZE} $<TARGET_FILE_NAME:app>
    VERBATIM
)
```

---

## 四、VS Code 操作

### 4.1 打开项目

File → Open Folder → 选项目根目录（含 CMakeLists.txt 的目录）

### 4.2 选择 Kit

Ctrl+Shift+P → `CMake: Select a Kit` → 选 `GCC 15.2.1 arm-none-eabi`

或点左侧 CMake 面板 → 配置 → `[未选择工具包]` → 选择

### 4.3 配置

Ctrl+Shift+P → `CMake: Configure`

或点左侧 CMake 面板 → 配置旁边的齿轮图标

成功标志：
```
[cmake] -- Configuring done (4.8s)
[cmake] -- Generating done (0.1s)
[cmake] -- Build files have been written to: .../build
```

### 4.4 编译

- 点底部状态栏 **Build**
- 或 Ctrl+Shift+B
- 或 Ctrl+Shift+P → `CMake: Build`

成功标志：
```
[build] [38/38 100%] Linking C executable app.elf
[build] Memory region         Used Size  Region Size  %age Used
[build]            FLASH:       54304 B       116 KB     45.72%
[build]              RAM:        9968 B        11 KB     88.49%
[build] 生成已完成，退出代码为 0
```

### 4.5 产物

编译成功后在 `build/` 目录下生成：

| 文件 | 大小 | 用途 |
|------|------|------|
| `app.bin` | ~55 KB | 烧录用（J-Link / ISP） |
| `app.elf` | ~677 KB | 调试用（GDB 加载） |
| `app.hex` | ~155 KB | 烧录用（HEX 格式） |
| `app.map` | ~598 KB | 内存映射分析 |

---

## 五、一键编译脚本（不依赖 VS Code）

```powershell
cd "项目根目录"
cmake -P build.cmake
```

产物在 `build/gcc/app.bin`。

手动编译：
```powershell
cmake -G Ninja -S . -B build/gcc -DCMAKE_BUILD_TYPE=Release
cmake --build build/gcc
```

---

## 六、常见问题

**Q: CMake Tools 找不到 Ninja**
A: ninja.exe 不在 PATH。复制到 gcc 的 bin 目录：`Copy-Item "...\WinGet\Links\ninja.exe" "...\gcc_arm_v15\bin\ninja.exe"`

**Q: 编译报 warning: 'x' may be used uninitialized**
A: 变量未初始化，不影响编译。在声明时赋值：`uint8_t x = 0;`

**Q: 生成的文件在哪**
A: `build/app.bin`（烧录）、`build/app.elf`（调试）

**Q: 如何切换 Debug/Release**
A: Ctrl+Shift+P → `CMake: Set Build Type` → 选 Debug 或 Release

**Q: 编译后想全量重编**
A: Ctrl+Shift+P → `CMake: Clean Rebuild` 或手动删除 `build/` 目录后重新 Configure