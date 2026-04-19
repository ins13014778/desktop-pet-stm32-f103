# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

STM32F103C8T6 四足桌面宠物（quadruped desktop pet），使用 Keil MDK + STM32 标准外设库 V3.5。通过语音模块（USART1）或手机蓝牙（USART3）接收 1 字节命令，驱动 5 个 SG90 舵机做 15 种动作，并在 0.96" I²C OLED 上显示表情。

**这是一个 Keil 工程，不是 CMake/Make 工程。所有"编译"相关操作都在 Keil uVision 里做。**

## Build / Flash Commands

| 动作 | 操作 |
| --- | --- |
| 打开工程 | 双击 `Project.uvprojx`（Keil uVision 5）|
| 编译 | Keil 菜单 `Project → Build Target`，或 **F7** |
| 清理 | Keil 菜单 `Project → Clean Target` |
| 烧录（SWD）| ST-Link 接好后点 Keil 工具栏 **Download**，或 **F8** |
| 烧录（ISP）| BOOT0=1 复位，FlyMcu 打开 `Objects/Project.hex` 从 PA9/PA10 烧录 |
| 调试 | **Ctrl+F5** 进入仿真，**F10/F11** 单步 |

产物路径：`Objects/Project.hex` 和 `Objects/Project.axf`（已在 .gitignore 中排除）。

**没有命令行 `make`/单元测试/CI** —— 这是典型的 IDE-based 嵌入式工程。

## High-Level Architecture

### 控制模型：串口中断 → 全局状态 → 主循环分发

```
外部命令字节 →→→ USART1_IRQ / USART3_IRQ  (BlueTooth.c)
                        ↓ 写
                全局变量 Action_Mode / Face_Mode
                        ↓ 读
             main() while(1) 分发             (main.c)
                        ↓ 调用
             Action_XXX()                    (PetAction.c)
                        ↓ 改 PWM CCR
                舵机/LED/OLED
```

整个固件是**共享全局变量的极简状态机**，无 RTOS。动作函数内部主动检查 `Action_Mode` 是否变化来实现可打断性。

### 关键"合约"（跨文件约束）

1. **`Action_Mode` 的值域是 0~15**，必须与 `main.c` 的 `if/else` 分发以及 `PetAction.c` 的动作函数一一对应。新增动作必须同时改三个文件：
   - `PetAction.c` 实现
   - `PetAction.h` 声明
   - `main.c` 新增分发分支
   - `BlueTooth.c` 的**两个** USART IRQ 同步添加命令字节

2. **`Face_Mode` 的值域是 0~6**，由 `Face_Config.c` 映射到 OLED 图像数组。新表情需要：
   - `OLED_Data.c` 添加字模数据
   - `OLED_Data.h` 声明 extern
   - `Face_Config.c` 添加分支

3. **`USART1_IRQHandler` 和 `USART3_IRQHandler` 逻辑几乎完全重复**，**修改命令映射时必须同步修改两处**。唯一差别是 `Sustainedmove`：USART1（语音）置 0（做完即停），USART3（蓝牙）置 1（持续运动）。

4. **舵机编号约定**：1=左上腿，2=右上腿，3=左下腿，4=右下腿。`Servo.c` 中 2/4 号做了 `180-Angle` 镜像，保证所有动作都用"0°=向前甩，90°=站立"的统一坐标系。

5. **PWM 定时器分配不可随意改动**：
   - TIM2 CH1~CH4 = 4 条腿（PA0~PA3）
   - TIM3 CH1 = 尾巴（PA6）
   - TIM3 CH3/CH4 = 呼吸灯（PB0/PB1）
   - TIM3 Update IRQ 同时驱动呼吸灯状态机（`main.c` 的 `TIM3_IRQHandler`）—— 所以舵机周期和呼吸节奏是同步的。

### 目录职责

| 目录 | 职责 | 二开频率 |
| --- | --- | --- |
| `User/` | 主入口 + 异常模板 + 标准库配置 | 高（`main.c`）|
| `HardWare/` | 舵机/串口/OLED/动作/表情业务层 | **非常高** |
| `Start/` | 启动文件 + 时钟配置 + 核心头文件 | 极少 |
| `Library/` | STM32F10x 标准外设库 V3.5 | **不要改** |
| `docs/` | 二开中文文档 | 按需 |
| `Objects/` / `Listings/` / `DebugConfig/` | 编译产物 | 已 gitignore |

## Twin-IRQ Gotcha

`BlueTooth.c` 中 `USART1_IRQHandler` 和 `USART3_IRQHandler` 是**代码几乎完全复制**的 220 行函数。添加/修改命令字节映射**必须同步改两处**，否则只有语音或只有蓝牙生效，表现为"蓝牙好使语音不行"。

重构成单一 `Command_Dispatch(cmd, sustained)` 是首推优化，但改动前确认不会影响硬件中断优先级。

## USART_ReceiveData Gotcha

现有代码 `USART_ReceiveData(USART1)` 在每个 `else if` 中重复调用。实际上 DR 寄存器读取后会被清空，**只有第一次读取得到真实值**，后续判断读到的是 0。当前能工作只是巧合（0x29 作为第一个判断命中早）。如修改命令映射顺序或新增命令，**必须先修复成"开头读一次存入局部变量"**：

```c
uint8_t cmd = USART_ReceiveData(USART1);
switch (cmd) { ... }
```

## Hardware Pin Map (不要轻易改)

| 引脚 | 功能 |
| --- | --- |
| PA0~PA3 | TIM2 CH1~4 → 四条腿 PWM |
| PA6 | TIM3 CH1 → 尾巴 PWM |
| PA9/PA10 | USART1 TX/RX → 语音模块 |
| PB10/PB11 | USART3 TX/RX → 蓝牙模块 |
| PB8/PB9 | 软件 I²C SCL/SDA → OLED |
| PB0/PB1 | TIM3 CH3/CH4 → LED PWM |

改引脚要同步改 `PWM.c`（GPIO 初始化 + TIM_OCxInit）和 `BlueTooth.c`（USART 引脚配置）。

## Key Files for Secondary Development

优先级由高到低：

1. `HardWare/BlueTooth.c` —— 改命令字节映射（最常见二开点）
2. `User/main.c` —— 改主循环分发、呼吸灯节奏
3. `HardWare/PetAction.c` —— 改动作姿态/速度、新增动作
4. `HardWare/Face_Config.c` —— 切换表情逻辑
5. `HardWare/OLED_Data.c` —— 新增表情位图
6. `HardWare/Servo.c` —— 舵机方向/零点校准

## Global State Variables (定义在 BlueTooth.c)

| 变量 | 默认 | 含义 |
| --- | --- | --- |
| `Action_Mode` | 0 | 当前动作 ID (0~15) |
| `Face_Mode` | 0 | 当前表情 ID (0~6) |
| `SpeedDelay` | 200 | 前进后退每帧 ms |
| `SwingDelay` | 6 | 摇摆每帧 ms |
| `WeiBa` | 0 | 动作中是否联动摇尾 |
| `AllLed` | 1 | 灯光总开关 |
| `BreatheLed` | 0 | 呼吸灯使能 |
| `Sustainedmove` | 0 | 动作是否持续（蓝牙触发才置 1）|

**二开注意**：这些变量缺少 `volatile`，主循环读取时编译器可能缓存。修改任一值的代码路径要谨慎（见 docs/FULL_SECONDARY_DEV_GUIDE.md §12.3）。

## Related Documents

- `README.md` —— 项目简介与快速上手
- `docs/FULL_SECONDARY_DEV_GUIDE.md` —— **最全面的二开文档**（1000 行，架构图/命令表/重构建议/扩展方案）
- `docs/SECONDARY_DEVELOPMENT_DETAILED.md` —— 早期版本二开文档
- `docs/AI_INTEGRATION_PLAN.md` —— 对接小智 AI / 云端语义的方案
- `SECONDARY_DEVELOPMENT_GUIDE.md` —— 顶层二开概述

## Encoding

源文件使用 **GB2312/GBK** 中文编码（Keil Windows 默认）。用 VSCode/其他编辑器打开时确保以 GBK 读取，否则注释乱码。
