# VS Code 的 EIDE 开发环境搭建指南

> 本文档包含完整的 VS Code 嵌入式开发环境配置。复制给 AI 即可复刻整个开发环境。

## 一、安装 VS Code

下载地址：https://code.visualstudio.com/
安装时勾选：
- 添加到 PATH
- 从命令行启动
- 将 Code 注册为支持的文件类型的编辑器

## 二、安装扩展

### 方式一：批量安装（推荐）

打开 PowerShell，粘贴执行：

```powershell
@(
    'anthropic.claude-code',
    'cl.eide',
    'marus25.cortex-debug',
    'mcu-debug.debug-tracker-vscode',
    'mcu-debug.memory-view',
    'mcu-debug.peripheral-viewer',
    'mcu-debug.rtos-views',
    'ms-ceintl.vscode-language-pack-zh-hans',
    'ms-vscode-remote.remote-ssh',
    'ms-vscode.cpptools',
    'ms-vscode.cmake-tools',
    'sst-dev.opencode'
) | ForEach-Object { code --install-extension $_ }
```

### 方式二：手动安装

Ctrl+Shift+X 打开扩展面板，逐个搜索安装：

| 扩展 ID | 名称 | 用途 |
|---------|------|------|
| `cl.eide` | Embedded IDE | 嵌入式项目管理、编译、烧录（点按钮代替 Keil） |
| `marus25.cortex-debug` | Cortex-Debug | ARM Cortex 芯片调试（断点、变量、内存） |
| `mcu-debug.debug-tracker-vscode` | debug-tracker-vscode | 调试会话跟踪 |
| `mcu-debug.memory-view` | MemoryView | 调试时查看芯片 RAM/Flash 内容 |
| `mcu-debug.peripheral-viewer` | Peripheral Viewer | 调试时查看外设寄存器值（需 SVD 文件） |
| `mcu-debug.rtos-views` | RTOS Views | 调试 FreeRTOS/RT-Thread 任务和堆栈 |
| `ms-ceintl.vscode-language-pack-zh-hans` | 中文语言包 | 界面汉化 |
| `ms-vscode-remote.remote-ssh` | Remote - SSH | 远程开发（SSH 到 Linux 编译） |
| `ms-vscode.cpptools` | C/C++ | C/C++ 语法高亮、自动补全、跳转定义 |
| `ms-vscode.cmake-tools` | CMake Tools | CMake 构建支持（Configure/Build/Clean） |
| `anthropic.claude-code` | Claude Code for VS Code | Claude AI 编程助手 |
| `sst-dev.opencode` | opencode | opencode AI 编程助手 |

安装后重启 VS Code。

## 三、下载工具链

### 3.1 ARM GCC 工具链

下载地址：https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads
选择：arm-none-eabi -> Windows -> .zip 或 .exe

安装/解压后，bin 目录下应有：
- arm-none-eabi-gcc.exe
- arm-none-eabi-g++.exe
- arm-none-eabi-objcopy.exe
- arm-none-eabi-objdump.exe
- arm-none-eabi-size.exe
- arm-none-eabi-gdb.exe

放到：C:\Users\10733\.eide\tools\gcc_arm\
目录结构：
```
C:\Users\10733\.eide\tools\gcc_arm\
└── bin\
    ├── arm-none-eabi-gcc.exe
    ├── arm-none-eabi-g++.exe
    ├── arm-none-eabi-objcopy.exe
    ├── arm-none-eabi-objdump.exe
    ├── arm-none-eabi-size.exe
    └── arm-none-eabi-gdb.exe
```

### 3.2 J-Link 软件包

下载地址：https://www.segger.com/downloads/jlink/
选择：J-Link Software and Documentation Pack -> Windows

安装后找到：
- JLinkGDBServerCL.exe
- JLink.exe
- JLinkDevices.xml

放到：C:\Users\10733\.eide\tools\jlink\
目录结构：
```
C:\Users\10733\.eide\tools\jlink\
├── JLinkGDBServerCL.exe
├── JLink.exe
└── JLinkDevices.xml
```

### 3.3 Keil ARMCC v5（可选，EIDE 用）

如果你用 Keil 编译而不是 CMake/GCC：
- 从 Keil MDK 安装目录复制 ARMCC 文件夹
- 或下载 cracked 版本

放到：C:\Users\10733\.eide\tools\armcc_v5_cracked\

### 3.4 工具链总览

```
C:\Users\10733\.eide\tools\
├── gcc_arm\              <- ARM GCC 编译器（CMake 编译用）
│   └── bin\
│       ├── arm-none-eabi-gcc.exe
│       ├── arm-none-eabi-g++.exe
│       ├── arm-none-eabi-objcopy.exe
│       ├── arm-none-eabi-objdump.exe
│       ├── arm-none-eabi-size.exe
│       └── arm-none-eabi-gdb.exe
├── jlink\                <- J-Link 调试器软件
│   ├── JLinkGDBServerCL.exe
│   └── JLink.exe
└── armcc_v5_cracked\     <- Keil 编译器（可选）
```

## 四、VS Code 全局 Settings

Ctrl+Shift+P -> Preferences: Open Settings (JSON)，粘贴：

```json
{
    "EIDE.ARM.ARMCC5.InstallDirectory": "C:\\Users\\10733\\.eide\\tools\\armcc_v5_cracked",
    "EIDE.JLink.InstallDirectory": "C:\\Users\\10733\\.eide\\tools\\jlink",
    "files.encoding": "gbk",
    "files.autoGuessEncoding": true,
    "extensions.ignoreRecommendations": true,
    "cortex-debug.armToolchainPath": "C:\\Users\\10733\\.eide\\tools\\gcc_arm\\bin",
    "cortex-debug.JLinkGDBServerPath": "C:\\Users\\10733\\.eide\\tools\\jlink\\JLinkGDBServerCL.exe",
    "editor.defaultFormatter": "ms-vscode.cpptools",
    "workbench.startupEditor": "none",
    "workbench.secondarySideBar.defaultVisibility": "hidden",
    "diffEditor.renderSideBySide": false,
    "editor.stickyScroll.enabled": false
}
```

各项说明：

| 配置项 | 作用 |
|--------|------|
| `EIDE.ARM.ARMCC5.InstallDirectory` | Keil ARMCC v5 编译器路径（EIDE 用） |
| `EIDE.JLink.InstallDirectory` | J-Link 工具路径（EIDE 烧录用） |
| `files.encoding` | 文件编码 GBK（兼容中文注释） |
| `files.autoGuessEncoding` | 自动识别文件编码 |
| `cortex-debug.armToolchainPath` | ARM GCC bin 目录（Cortex-Debug 调试用） |
| `cortex-debug.JLinkGDBServerPath` | J-Link GDB Server 路径 |
| `editor.defaultFormatter` | C/C++ 文件默认格式化器 |

## 五、验证安装

打开 VS Code 终端（Ctrl+`），执行：

```powershell
# 验证 ARM GCC
arm-none-eabi-gcc --version

# 验证 J-Link
& "C:\Users\10733\.eide\tools\jlink\JLink.exe" --version
```

两条都输出版本号 = 环境正常。

## 六、创建项目

### 方式一：CMake 项目（推荐）

参见同目录下 CMake构建方式.md。

### 方式二：EIDE 项目

1. Ctrl+Shift+P -> EIDE: New Project
2. 选芯片型号
3. 导入 Keil 工程（如有 .uvprojx 文件）
4. 点 Build 编译，点 Flash 烧录

### 方式三：纯 C/C++ 项目

1. File -> Open Folder -> 新建文件夹
2. 创建 src/main.c
3. Ctrl+Shift+P -> C/C++: Edit Configurations 生成 .vscode/c_cpp_properties.json

## 七、调试

1. 连接 J-Link 到开发板（SWD 接口：SWDIO、SWCLK、GND、3.3V）
2. 打开项目，编译生成 .elf 文件
3. F5 启动调试（或侧边栏 Run and Debug -> 选 Debug (J-Link)）
4. 调试面板：断点、单步、变量监视、内存查看、寄存器查看

## 八、常见问题

Q: code 命令找不到
A: 重新安装 VS Code，安装时勾选"添加到 PATH"。或手动添加：C:\Users\你的用户名\AppData\Local\Programs\Microsoft VS Code\bin

Q: 扩展安装后不显示中文
A: Ctrl+Shift+P -> Configure Display Language -> 选 zh-cn -> 重启

Q: EIDE 找不到编译器
A: Settings 里 EIDE.ARM.ARMCC5.InstallDirectory 路径不对，改成实际路径

Q: 调试连不上 J-Link
A: 检查 J-Link 驱动是否安装，cortex-debug.JLinkGDBServerPath 路径是否正确

Q: 编码乱码
A: 确认 files.encoding 为 gbk，files.autoGuessEncoding 为 true