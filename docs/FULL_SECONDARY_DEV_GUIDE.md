# STM32 桌面宠物 —— 二次开发最大化详细指南

> 目标：让任何人都能零基础上手这个项目，看完这份文档就能完成从基础改动（换表情、改动作）到进阶扩展（新增传感器、WiFi、AI 语音）的全部工作。

---

## 目录

1. [项目一句话总览](#1-项目一句话总览)
2. [硬件清单与接线图](#2-硬件清单与接线图)
3. [软件总体架构](#3-软件总体架构)
4. [完整文件路径树](#4-完整文件路径树)
5. [每个文件详解（路径 + 作用 + 关键符号）](#5-每个文件详解)
6. [核心技术分解](#6-核心技术分解)
7. [命令协议与字节码对照表](#7-命令协议与字节码对照表)
8. [控制流与数据流图示](#8-控制流与数据流图示)
9. [从零编译与烧录](#9-从零编译与烧录)
10. [二开常见场景（含完整代码模板）](#10-二开常见场景)
11. [新增功能建议与实现方案](#11-新增功能建议与实现方案)
12. [代码已知问题与重构建议](#12-代码已知问题与重构建议)
13. [调试技巧](#13-调试技巧)
14. [二开路线图（新手 → 进阶 → 专家）](#14-二开路线图)

---

## 1. 项目一句话总览

**这是一台用 4 + 1 = 5 个 SG90 舵机驱动的桌面四足宠物，主控是 STM32F103C8T6，通过语音模块（USART1）或手机蓝牙（USART3）接收 1 字节命令，在 0.96" I²C OLED 上显示表情，由板载 PWM 输出的 LED 呼吸灯做氛围。**

核心关键词：
- MCU：STM32F103C8T6（Cortex-M3，72 MHz，64KB Flash，20KB SRAM）
- 开发工具：Keil uVision 5 + STM32 标准外设库 (StdPeriph V3.5)
- 舵机控制：TIM2 CH1~CH4（4 条腿）+ TIM3 CH1（尾巴）共 5 路 50 Hz PWM
- 灯光：TIM3 CH3/CH4 两路 PWM，配合 TIM3 更新中断实现呼吸灯
- 显示：软件 I²C 驱动的 0.96" SSD1306 128×64 OLED
- 通信：USART1 (9600 bps，语音) + USART3 (9600 bps，蓝牙)

---

## 2. 硬件清单与接线图

### 2.1 硬件清单

| 模块                   | 说明                             | 数量 |
| ---------------------- | -------------------------------- | ---- |
| STM32F103C8T6 最小系统板 | 主控                             | 1    |
| SG90 180° 舵机         | 4 条腿 + 1 条尾巴                  | 5    |
| 0.96" OLED (SSD1306 I²C) | 表情显示                         | 1    |
| HC-05 / HC-06 蓝牙模块 | 手机蓝牙控制                     | 1    |
| 语音识别模块（SU-03T / LD3320 等） | 离线语音识别，串口输出 1 字节码  | 1    |
| 5V 锂电池 / 3.7V×2     | 舵机供电，稳定 5V 建议单独供应     | 1    |
| LED 灯珠 + 限流电阻    | 氛围灯（项目中的 PWM_LED1/2）    | 2    |

### 2.2 引脚分配（**来源：`HardWare/PWM.c` + `HardWare/BlueTooth.c` + `HardWare/OLED.c`**）

```
STM32F103C8T6 引脚分配
┌────────┬──────────────────────┬──────────────────────────┐
│ 引脚   │ 功能                 │ 来源文件                 │
├────────┼──────────────────────┼──────────────────────────┤
│ PA0    │ TIM2_CH1 → 舵机1 左上 │ PWM.c / Servo.c          │
│ PA1    │ TIM2_CH2 → 舵机2 右上 │ PWM.c / Servo.c          │
│ PA2    │ TIM2_CH3 → 舵机3 左下 │ PWM.c / Servo.c          │
│ PA3    │ TIM2_CH4 → 舵机4 右下 │ PWM.c / Servo.c          │
│ PA6    │ TIM3_CH1 → 尾巴舵机  │ PWM.c / Servo.c          │
│ PB0    │ TIM3_CH3 → LED1 PWM  │ PWM.c (PWM_LED1)         │
│ PB1    │ TIM3_CH4 → LED2 PWM  │ PWM.c (PWM_LED2)         │
│ PA9    │ USART1_TX → 语音模块  │ BlueTooth.c              │
│ PA10   │ USART1_RX ← 语音模块  │ BlueTooth.c              │
│ PB10   │ USART3_TX → 蓝牙模块  │ BlueTooth.c              │
│ PB11   │ USART3_RX ← 蓝牙模块  │ BlueTooth.c              │
│ PB8    │ OLED_SCL（软件 I²C） │ OLED.c                   │
│ PB9    │ OLED_SDA（软件 I²C） │ OLED.c                   │
└────────┴──────────────────────┴──────────────────────────┘
```

### 2.3 舵机编号约定（**关键，别接错！**）

```
           站立方向（头→前）
               ↑
        ┌──────────┐
 左上1  │          │ 右上2
   ─────┤          ├─────
        │  STM32   │
        │          │
   ─────┤          ├─────
 左下3  │          │ 右下4
        └──────────┘
               尾巴(W)
```

**机械约定**：在 `Servo.c` 中，**角度 0° 代表腿向前甩**，**角度 90° 代表站立**。修改动作一定要以此为参考坐标系。

### 2.4 整机连线示意（ASCII）

```
┌─────────────────────────────────┐
│   STM32F103C8T6 最小系统板       │
│                                 │
│  PA0─PA3  ┐                     │         5V 独立供电（舵机）
│   PA6     ├──── PWM (50Hz) ───────→  4×腿 SG90 + 1×尾巴 SG90
│  PB0/PB1  ┘                     │       │
│                                 │       │ 共地
│  PA9/PA10 ←──── USART1 ──────────→  语音模块 SU-03T
│  PB10/PB11 ←─── USART3 ──────────→  蓝牙模块 HC-05
│                                 │
│  PB8/PB9  ←──── 软件 I²C ─────────→  OLED SSD1306
│                                 │
│  PB0/PB1  ←──── PWM ─────────────→  LED×2（带限流电阻）
└─────────────────────────────────┘
```

---

## 3. 软件总体架构

### 3.1 一张图看懂

```
 外部世界                  STM32 内部                  执行器
┌─────────────┐         ┌──────────────────┐        ┌──────────┐
│ 语音模块     │  命令字节 │USART1_IRQHandler │        │ 舵机×5   │
│  (USART1)   ├────────→│  (BlueTooth.c)   │        │          │
└─────────────┘         │        ↓         │        └────↑─────┘
                        │   Action_Mode    │             │PWM
┌─────────────┐         │   Face_Mode      │             │
│ 手机蓝牙     │  命令字节 │USART3_IRQHandler │────────────┘
│  (USART3)   ├────────→│  (BlueTooth.c)   │        ┌──────────┐
└─────────────┘         │        ↓         │   I²C  │  OLED    │
                        │  while(1)主循环  ├───────→│          │
                        │  (main.c)        │        └──────────┘
                        │        ↓         │
                        │  Action_XXX()    │        ┌──────────┐
                        │  (PetAction.c)   │   PWM  │  LED×2   │
                        │        ↓         ├───────→│          │
                        │  TIM3_IRQHandler │        └──────────┘
                        │  (main.c)        │
                        └──────────────────┘
```

### 3.2 状态机视角

整个固件本质是**一个极简状态机**：

```
┌──────────────────────────────────────────┐
│ 全局状态（共享变量，BlueTooth.c 中定义）    │
│                                          │
│  Action_Mode : 当前要执行的动作 ID (0-15)  │
│  Face_Mode   : 当前要显示的表情 ID (0-6)   │
│  SpeedDelay  : 前进/后退每帧延时 (ms)      │
│  SwingDelay  : 摇摆每帧延时 (ms)           │
│  WeiBa       : 是否同时摇尾 (0/1)          │
│  AllLed      : 灯光总开关 (0/1)            │
│  BreatheLed  : 呼吸灯使能 (0/1)            │
│  Sustainedmove: 持续运动标志 (0/1)         │
└──────────────────────────────────────────┘
         ↑ 中断写
         │
  ┌──────┴──────┐           ┌──────────────┐
  │ 串口中断     │  改变状态  │ 主循环轮询     │
  │ (生产者)     │───────────→│ (消费者)      │
  └─────────────┘            └──────────────┘
```

**关键点**：
- 所有"命令接收"都是中断驱动；
- 所有"动作执行"都是主循环轮询；
- 两者通过**全局变量**通信；
- 动作函数内部主动检查 `Action_Mode` 变化，**实现"可打断"动作**。

### 3.3 代码层次图

```
┌─────────────────────────────────────────┐
│ 应用层 (User/)                          │
│   main.c                                │
│     ├ main()          动作分发器        │
│     └ TIM3_IRQHandler 呼吸灯节奏        │
├─────────────────────────────────────────┤
│ 业务层 (HardWare/)                      │
│   BlueTooth.c   命令→状态映射 (核心入口) │
│   PetAction.c   15 种预定义动作姿态      │
│   Face_Config.c Face_Mode → OLED 图像   │
│   OLED_Data.c   所有表情位图数据         │
├─────────────────────────────────────────┤
│ 驱动层 (HardWare/)                      │
│   Servo.c   角度 → PWM 脉宽换算         │
│   PWM.c     TIM2/TIM3 5 路 PWM 初始化   │
│   OLED.c    SSD1306 软件 I²C 驱动       │
│   Delay.c   SysTick 毫秒延时            │
├─────────────────────────────────────────┤
│ HAL 层 (Library/)                       │
│   stm32f10x_xxx.c  STM32 标准外设库     │
├─────────────────────────────────────────┤
│ 启动层 (Start/)                         │
│   startup_stm32f10x_md.s  启动代码      │
│   system_stm32f10x.c      时钟配置      │
│   core_cm3.h              Cortex-M3 核  │
└─────────────────────────────────────────┘
```

---

## 4. 完整文件路径树

```
桌面宠物代码(2月17日更新)/
├── Project.uvprojx                 ← Keil 工程主文件（双击打开）
├── Project.uvoptx                  ← Keil 工程选项
├── Project.uvguix.lenovo           ← Keil UI 布局（本机专属，可忽略）
├── EventRecorderStub.scvd          ← 事件记录桩
├── README.md                       ← 项目说明
├── SECONDARY_DEVELOPMENT_GUIDE.md  ← 初版二开指南
│
├── User/                           ★ 应用层（二开首选区）
│   ├── main.c                      ← 程序入口 + 主循环 + TIM3 中断
│   ├── stm32f10x_it.c              ← Cortex-M3 异常/中断模板
│   ├── stm32f10x_it.h              ← 中断声明
│   └── stm32f10x_conf.h            ← 标准库模块开关
│
├── HardWare/                       ★ 驱动+业务层（二开主战场）
│   ├── BlueTooth.c / .h            ← ★★ 串口命令接收（语音+蓝牙）
│   ├── PetAction.c / .h            ← ★★ 15 种动作姿态函数
│   ├── Face_Config.c / .h          ← 表情模式 → OLED 图像切换
│   ├── OLED.c / .h                 ← SSD1306 驱动（江协科技）
│   ├── OLED_Data.c / .h            ← 所有表情位图字模
│   ├── Servo.c / .h                ← 角度→PWM 脉宽换算
│   ├── PWM.c / .h                  ← TIM2/TIM3 PWM 初始化
│   └── Delay.c / .h                ← SysTick 延时
│
├── Start/                          启动+时钟+核心头文件（一般不改）
│   ├── startup_stm32f10x_md.s      ← 中容量启动文件
│   ├── system_stm32f10x.c          ← 72MHz 时钟配置
│   ├── stm32f10x.h                 ← 片上寄存器定义
│   ├── core_cm3.c / .h             ← Cortex-M3 内核库
│   └── startup_stm32f10x_*.s       ← 其他容量启动模板
│
├── Library/                        STM32 标准外设库 V3.5（一般不改）
│   ├── stm32f10x_gpio.c            ← GPIO 驱动
│   ├── stm32f10x_tim.c             ← 定时器驱动
│   ├── stm32f10x_usart.c           ← USART 驱动
│   ├── stm32f10x_rcc.c             ← 时钟控制
│   └── ...（其余外设驱动）
│
├── Objects/                        ← 编译输出（.hex/.axf/.o），.gitignore 已忽略
├── Listings/                       ← 列表文件（.map/.lst），.gitignore 已忽略
├── DebugConfig/                    ← Keil 调试配置（本机），.gitignore 已忽略
│
├── docs/                           ★ 二开文档
│   ├── SECONDARY_DEVELOPMENT_DETAILED.md  ← 详细二开文档（旧版）
│   ├── AI_INTEGRATION_PLAN.md             ← AI 对接方案
│   └── FULL_SECONDARY_DEV_GUIDE.md        ← 本文档（最全版本）
│
└── project_code_analysis_fixed.docx       ← Word 版分析报告
    secondary_dev_detailed_guide.docx      ← Word 版二开指南
    secondary_dev_detailed_guide_fixed.docx
```

---

## 5. 每个文件详解

### 5.1 `User/main.c` —— **整机入口**

**作用**：
- 调用各模块初始化函数；
- 主循环 `while(1)` 根据 `Action_Mode` 分发到不同动作函数；
- `TIM3_IRQHandler()` 实现呼吸灯状态机（常亮 / 呼吸 / 息屏）。

**关键符号**：
| 符号 | 类型 | 说明 |
| --- | --- | --- |
| `main()` | 函数 | 入口，初始化顺序：`Servo_Init → OLED_Init → BlueTooth_Init → OLED 显示初始表情 → while(1) 分发` |
| `TIM3_IRQHandler()` | 中断 | 每 20ms 触发一次（同 TIM3 溢出周期），驱动 `PWM_LED1/LED2` 的亮度变化 |
| `HuXi` | uint16_t | 呼吸灯当前亮度（0~20000）|
| `PanDuan` | uint16_t | 呼吸灯状态机：1=渐亮 / 2=渐暗 / 3=息屏等待 |
| `Wait` | uint16_t | 呼吸灯息屏计数 |

**初始化顺序解读**：
```c
Servo_Init();       // 1. 舵机 PWM 先起（TIM2+TIM3 5 路 PWM）
OLED_Init();        // 2. 软件 I²C 挂到 PB8/PB9
BlueTooth_Init();   // 3. 打开 USART1/USART3 + 中断 + NVIC
OLED_ShowImage(...) // 4. 显示初始睡眠表情
OLED_Update();      // 5. 把显存推到 OLED
```

> 这个顺序不能乱。尤其不能先开 `BlueTooth_Init` 再初始化 PWM，否则上位机发命令时舵机尚未进入中立位，容易冲击结构。

### 5.2 `HardWare/BlueTooth.c` —— **命令路由（二开最常改！）**

**作用**：
- 初始化 USART1（语音）和 USART3（蓝牙），配置 9600 8N1；
- 配置对应 NVIC 中断（USART1 抢占级 1，USART3 抢占级 2）；
- `USART1_IRQHandler()` / `USART3_IRQHandler()` 把接收到的字节映射到 `Action_Mode`/`Face_Mode`。

**全局变量（在这里 `definition`，在 `BlueTooth.h` 里 `extern`）**：
| 变量 | 默认值 | 含义 |
| --- | --- | --- |
| `Action_Mode` | 0 | 当前动作（0=放松趴下 … 15=后腿拉伸）|
| `Face_Mode` | 0 | 当前表情 |
| `SpeedDelay` | 200 | 前进/后退每帧延时 (ms)，越小越快 |
| `SwingDelay` | 6 | 摇摆每帧延时 (ms) |
| `WeiBa` | 0 | 是否伴随摇尾 |
| `AllLed` | 1 | 灯光总开关 |
| `BreatheLed` | 0 | 是否呼吸效果 |
| `Sustainedmove` | 0 | 持续运动（蓝牙=1 会一直走，语音=0 只走两步）|

**关键区别（重要！）**：
- `USART1_IRQHandler()` 接到命令后会把 `Sustainedmove` 置 0 → 动作函数执行完预定次数后会自动回到站立
- `USART3_IRQHandler()` 接到命令后会把 `Sustainedmove` 置 1 → 动作持续执行直到收到下一个命令

> 这是语音和蓝牙行为差异的根源。设计意图：语音"做一次就停"，蓝牙"按住一直走"。

### 5.3 `HardWare/PetAction.c` —— **15 种动作**

**作用**：实现 15 个 `Action_XXX()` 函数，每个函数调用 `Servo_Angle1~4()` 和 `WServo_Angle()` 改变舵机角度，中间穿插 `Delay_ms()`。

**动作函数清单**（完整来源文件：`PetAction.c`）：
| ID | 函数 | 动作描述 | 典型姿态 |
| --- | --- | --- | --- |
| 0 | `Action_relaxed_getdowm` | 放松趴下 | 上肢 20°，下肢 160° |
| 1 | `Action_sit` | 坐下 | 上肢 90°，下肢 20° |
| 2 | `Action_upright` | 站立（先上后下）| 全部 90° |
| 3 | `Action_getdowm` | 趴下 | 全部 20° |
| 4 | `Action_advance` | 前进 | 斜对角抬腿循环 |
| 5 | `Action_back` | 后退 | 反向斜对角抬腿循环 |
| 6 | `Action_Lrotation` | 左转 | 差速抬腿 |
| 7 | `Action_Rrotation` | 右转 | 差速抬腿 |
| 8 | `Action_Swing` | 全身摇摆 | 4 腿同步扫角 |
| 9 | `Action_SwingTail` | 摇尾 | 仅尾巴舵机扫角 |
| 10 | `Action_JumpU` | 前跳 | 快速抬前腿 |
| 11 | `Action_JumpD` | 后跳 | 快速抬后腿 |
| 12 | `Action_upright2` | 站立（先下后上）| 用于跳跃后归位 |
| 13 | `Action_Hello` | 打招呼 | 一腿反复挥动 |
| 14 | `Action_stretch` | 伸懒腰 | 前腿→后腿依次伸展 |
| 15 | `Action_Lstretch` | 后腿拉伸 | 左右后腿依次拉伸 |

**可打断模式**：持续动作（前进/后退/摇摆等）内部 while 循环每帧都会检查 `Action_Mode != 当前 ID`，一旦变化立刻 break —— 这是 UI 响应性的关键。

### 5.4 `HardWare/Servo.c` —— **角度到 PWM 的换算**

**核心公式**：
```c
PWM_SetCompareX(Angle / 180 * 2000 + 500);
//               └──────────┘   └───┘
//               比例映射         基础脉宽 500us (0°)
// 总脉宽 = 500us (0°) ~ 2500us (180°)
// TIM 周期 20ms = 50Hz，正好是标准舵机规范
```

**重要细节**：舵机 2 和舵机 4（右侧两腿）取了 `180 - Angle` 做镜像，这样写动作时左右两腿都用"90°=站立、0°=向前甩"的统一约定。

### 5.5 `HardWare/PWM.c` —— **5 路舵机 + 2 路 LED 的定时器配置**

**TIM 分配**：
```
TIM2 (APB1, 72MHz → 预分频 72 → 1MHz → 周期 20000 = 20ms = 50Hz)
├── CH1 (PA0) → 舵机1
├── CH2 (PA1) → 舵机2
├── CH3 (PA2) → 舵机3
└── CH4 (PA3) → 舵机4

TIM3 (APB1, 同上 50Hz)
├── CH1 (PA6) → 尾巴舵机
├── CH3 (PB0) → LED1 (PWM 亮度)
├── CH4 (PB1) → LED2 (PWM 亮度)
└── Update IRQ → 呼吸灯状态机驱动
```

**为什么 LED 也挂在 TIM3？** 因为 TIM3 更新中断（每 20ms）同时用来刷新呼吸灯亮度。把 LED 的 CCR 接到同一个定时器，PWM 频率与呼吸节奏天然同步。

### 5.6 `HardWare/Face_Config.c` —— **表情切换**

**单函数**：`Face_Config(void)`。根据全局 `Face_Mode` 清屏 → 显示对应图像 → `OLED_Update()`。

**当前表情映射**：
| Face_Mode | 位图 | 语义 |
| --- | --- | --- |
| 0 | `Face_sleep` | 睡觉 |
| 1 | `Face_stare` | 瞪眼 |
| 2 | `Face_happy` | 快乐 |
| 3 | `Face_mania` | 狂热（速度拉满时显示）|
| 4 | `Face_very_happy` | 非常快乐 |
| 5 | `Face_eyes` | 眼睛 |
| 6 | `Face_hello` | 打招呼 |

### 5.7 `HardWare/OLED_Data.c` —— **字模+位图数据库**

**三类数据**：
- `OLED_F8x16[]` ASCII 8×16 字模
- `OLED_F6x8[]` ASCII 6×8 字模
- `OLED_CF16x16[]` 中文 16×16 字模（UTF-8 或 GB2312）
- `Face_*[]` 七种 128×64 表情位图（共 7 × 1024 字节 ≈ 7KB）

**新增表情方法**：见 §10.3。

### 5.8 `HardWare/OLED.c` —— **SSD1306 驱动**

江协科技开源的软件 I²C 驱动，支持：
- `OLED_ShowImage(X, Y, W, H, img)` 显示位图
- `OLED_ShowChar/String/Num/Chinese` 文字显示
- `OLED_DrawLine/Rectangle/Circle/Arc` 绘图
- `OLED_Clear / OLED_Update` 显存管理

**关键约定**：所有绘制操作都只改 `OLED_DisplayBuf[8][128]`，**必须调用 `OLED_Update()` 才会真正推到 OLED 硬件**。

### 5.9 `HardWare/Delay.c` —— **SysTick 毫秒延时**

```c
Delay_us(xus);   // 0~233015 微秒
Delay_ms(xms);   // 毫秒
Delay_s(xs);     // 秒
```

基于 SysTick，不占用 TIMx 资源。

### 5.10 `User/stm32f10x_conf.h` —— **标准库模块开关**

所有 `#include "stm32f10x_xxx.h"` 都由这个文件统一包含。二开时如果用到某个新外设（比如 `stm32f10x_adc.h` 加 ADC），注释掉对应的 `#include` 即可生效。

### 5.11 `User/stm32f10x_it.c` —— **异常处理模板**

NMI/HardFault/MemManage/BusFault/UsageFault/SysTick 的空模板。**注意**：项目里真正用到的 `USART1_IRQHandler/USART3_IRQHandler/TIM3_IRQHandler` 并没有在这里，而是分散在 `BlueTooth.c` 和 `main.c` 里。**二开时如果想集中管理中断，也可以把它们挪到这里**。

---

## 6. 核心技术分解

### 6.1 STM32 定时器的 PWM 输出

公式（所有 4 路 + 1 路尾巴都一样）：
```
PWM 频率 = 72MHz / Prescaler / Period
         = 72MHz / 72 / 20000
         = 50 Hz  (20ms 周期)

舵机脉宽 = CCR / 72MHz × Prescaler
        = CCR × (72/72MHz)
        = CCR × 1µs     // 因为预分频 72 后计数频率 1MHz
        
所以 CCR=500 → 500µs   → 角度 0°
    CCR=1500 → 1500µs  → 角度 90°
    CCR=2500 → 2500µs  → 角度 180°
```

### 6.2 USART 中断接收模型

每次收到 1 个字节就进中断，在中断里**直接决定**整机下一步行为：
```c
void USART1_IRQHandler(void) {
    if (USART_GetITStatus(USART1, USART_IT_RXNE) == SET) {
        uint8_t cmd = USART_ReceiveData(USART1);   // ← 只读一次！
        switch (cmd) { ... }                        // 映射 Action_Mode/Face_Mode
        USART_ClearITPendingBit(USART1, USART_IT_RXNE);
    }
}
```

> **注意一个隐藏坑**：现有代码连续 20 次调用 `USART_ReceiveData(USART1)`，每次都重新读 DR 寄存器。一般情况下 DR 只被读一次就有效，连续读可能读到 0，侥幸正常是因为第一次判断 `!=0x29` 时寄存器已空。**重构第一步就应该把它改成先 `uint8_t cmd = USART_ReceiveData()` 一次，然后 switch**。见 §12。

### 6.3 主循环 + 全局变量的协作

这是典型的"**轮询+共享内存**"嵌入式模式，不需要 RTOS。
- 优点：实现简单，无优先级反转，代码直观；
- 缺点：动作函数里的 `Delay_ms()` 会阻塞主循环，响应速度 = 每帧延时；
- 解决：动作函数内部显式检查 `Action_Mode`，一旦改变就 `break`。

### 6.4 呼吸灯状态机

```c
PanDuan=1 : HuXi 从 0 → 20000 （渐亮）
PanDuan=2 : HuXi 从 20000 → 0（渐暗）
PanDuan=3 : Wait 累加，到 20000 切回 1（息屏等待）
```

每次 TIM3 更新中断（20ms）增减 100，完成一次渐亮 ≈ 200 帧 ≈ 4 秒，整体呼吸周期 ≈ 12 秒。改呼吸速度只需要改 `HuXi+=100` 的步进和 `Wait` 上限。

### 6.5 软件 I²C

OLED 不用硬件 I²C（节省 I2C1/I2C2 资源），用 GPIO 模拟：
- PB8 = SCL
- PB9 = SDA

优点：I/O 随便挑；缺点：占用 CPU，不过 128×64 位图刷新一次仅 ~10ms，没瓶颈。

---

## 7. 命令协议与字节码对照表

**协议**：1 字节 + 串口 9600 8N1 无校验。无握手、无校验和。

### 7.1 动作命令

| 字节 | USART1(语音) 行为 | USART3(蓝牙) 行为 | Action_Mode | Face_Mode |
| ---- | ----------------- | ----------------- | ----------- | --------- |
| 0x29 | 放松趴下（单次）   | 放松趴下（持续）   | 0 | 0 |
| 0x30 | 坐下（单次）      | 坐下（持续）      | 1 | 1 |
| 0x31 | 站立（单次）      | 站立（持续）      | 2 | 5 |
| 0x32 | 趴下（单次）      | 趴下（持续）      | 3 | 1 |
| 0x33 | 前进（两步停）    | 前进（不停）      | 4 | 2 |
| 0x34 | 后退（两步停）    | 后退（不停）      | 5 | 2 |
| 0x35 | 左转（两步停）    | 左转（不停）      | 6 | 2 |
| 0x36 | 右转（两步停）    | 右转（不停）      | 7 | 2 |
| 0x37 | 摇摆              | 摇摆              | 8 | 4 |
| 0x40 | 摇尾巴开关        | 摇尾巴开关        | 9 | 1 |
| 0x41 | 前跳              | 前跳              | 10 | 2 |
| 0x42 | 后跳              | 后跳              | 11 | 2 |
| 0x43 | 打招呼            | 打招呼            | 13 | 6 |
| 0x48 | 伸懒腰            | 伸懒腰            | 14 | 6 |
| 0x49 | 后腿拉伸          | 后腿拉伸          | 15 | 6 |

### 7.2 参数命令

| 字节 | 行为 | 影响变量 |
| ---- | ---- | -------- |
| 0x38 | 加快移动（SpeedDelay 每次 -20，到 100 归位 200）| SpeedDelay |
| 0x39 | 加快摇摆（SwingDelay 每次 -1，到 3 归位 9）| SwingDelay |
| 0x44 | 开灯 | AllLed=1 |
| 0x45 | 关灯 | AllLed=0 |
| 0x46 | 开呼吸灯 | BreatheLed=1 |
| 0x47 | 关呼吸灯 | BreatheLed=0 |

---

## 8. 控制流与数据流图示

### 8.1 一次"前进"命令的完整生命周期

```
1. 手机发蓝牙: 0x33
         ↓
2. HC-05 串口: 收到 0x33
         ↓
3. STM32 USART3 接收寄存器填入 0x33，触发 RXNE
         ↓
4. USART3_IRQHandler() 被打断执行
         ├─ Sustainedmove = 1
         ├─ 命中 0x33 分支
         ├─ Face_Mode = 2
         ├─ Face_Config()   ← 立刻更新 OLED 显示 Face_happy
         └─ Action_Mode = 4
         ↓
5. 中断返回
         ↓
6. main() while(1) 检测到 Action_Mode==4
         ↓
7. Action_advance() 被调用
         ├─ 进入 while(Action_Mode==4)
         ├─ 每帧 Servo_AngleX(..) → PWM CCR 改变
         │   ├─ 每帧都 Delay_ms(SpeedDelay)
         │   └─ 每帧都检查 Action_Mode!=4 → 如不是就 break
         └─ Sustainedmove=1 → 永远循环
         ↓
8. 手机再发 0x31（站立），重复步骤 3~5
         ↓
9. Action_Mode 变 2，Action_advance 内部 break 出 while
         ↓
10. 回到 main() while(1)，进入 Action_upright() 分支
```

### 8.2 数据总线

```
          ┌──────────────── OLED 显存(1KB) ─────────┐
          │                                        │
Face_Mode ─→ Face_Config() ──→ OLED_ShowImage() ─→ OLED_DisplayBuf ─→ I²C → OLED
                                                                       │
Action_Mode ─→ main while() ──→ Action_XXX() ──→ Servo_Angle() ──→ CCR → PWM → 舵机
                                       │
                                       └─→ Delay_ms() (CPU 空转)
                                       
AllLed/BreatheLed ─→ TIM3_IRQ 20ms ─→ PWM_LED1/2() ──→ CCR → PWM → LED
```

---

## 9. 从零编译与烧录

### 9.1 环境准备（Windows）

1. 安装 Keil MDK-ARM 5.x（官方，需付费或社区免费版）
2. 安装 STM32F1 设备支持包：`Keil → Pack Installer → STMicroelectronics → STM32F1 Series → Install`
3. 下载烧录工具：
   - **ST-Link Utility**（ST 官方，ST-Link V2 用）
   - **FlyMcu**（串口 ISP 用，支持 BOOT0 拉高后从 UART1 下载）

### 9.2 编译

1. 双击 `Project.uvprojx` 打开 Keil
2. 选中左侧 Target 为 `Target 1`
3. 点 **Build** (F7)
4. 看输出 `Build target 'Target 1'` 出现 `0 Error(s)` 表示成功
5. 生成的 hex 在 `Objects/Project.hex`（已自动配置）

### 9.3 烧录

**方案 A：ST-Link SWD（推荐，可断点调试）**
1. ST-Link 接板子的 SWCLK/SWDIO/GND/3V3
2. Keil 菜单 `Flash → Configure Flash Tools → Debug`，选 `ST-Link Debugger`
3. 按 Keil 工具栏的 **Download (F8)** 即可

**方案 B：串口 ISP（最简单，只需 USB-TTL）**
1. 板子 BOOT0 拉高、BOOT1 拉低
2. USB-TTL 接 PA9(TX)/PA10(RX)
3. FlyMcu → 选串口 → 选 `Objects/Project.hex` → "开始编程"
4. 烧完 BOOT0 拉回低，复位即可运行

---

## 10. 二开常见场景

### 场景 10.1：改某个语音命令对应的动作

**需求**：把原本 `0x43` 的"打招呼"改成"原地摇摆"。

**改法**（只改一处）：`HardWare/BlueTooth.c` 的 `USART1_IRQHandler` 和 `USART3_IRQHandler`：

```c
// 原代码
else if(USART_ReceiveData(USART1)==0x43) {
    Face_Mode=6;
    Face_Config();
    Action_Mode=13;  // 打招呼
}

// 改成
else if(USART_ReceiveData(USART1)==0x43) {
    Face_Mode=4;     // 非常快乐
    Face_Config();
    Action_Mode=8;   // 摇摆
}
```

### 场景 10.2：新增一个自定义动作"转圈+摇尾"

**四步走**：

#### Step 1：在 `PetAction.c` 写动作函数
```c
void Action_CircleWithTail(void) {
    while(Action_Mode == 16) {
        // 旋转一圈（4 × 90°）
        for (int step = 0; step < 4; step++) {
            Servo_Angle2(45);  Servo_Angle3(135);
            WServo_Angle(30);
            Delay_ms(SpeedDelay);
            if(Action_Mode != 16) return;
            
            Servo_Angle1(45);  Servo_Angle4(135);
            WServo_Angle(150);
            Delay_ms(SpeedDelay);
            if(Action_Mode != 16) return;
            
            Servo_Angle2(90);  Servo_Angle3(90);
            Servo_Angle1(90);  Servo_Angle4(90);
            Delay_ms(SpeedDelay);
            if(Action_Mode != 16) return;
        }
        if (!Sustainedmove) { Action_Mode = 2; break; }
    }
}
```

#### Step 2：在 `PetAction.h` 声明
```c
void Action_CircleWithTail(void);
```

#### Step 3：在 `main.c` 加分支
```c
else if(Action_Mode==16){Action_CircleWithTail();}
```

#### Step 4：在 `BlueTooth.c` 的两个 IRQ 里加命令
```c
else if(USART_ReceiveData(USART1)==0x4A) {
    Face_Mode=4;
    Face_Config();
    Action_Mode=16;
}
```

完工。

### 场景 10.3：新增一个表情（比如生气脸）

#### Step 1：用 [img2lcd](https://sourceforge.net/projects/img2lcd/) 生成 128×64 取模数据
设置：**逐列式，低位在上，纵向 8 点**。输出的 C 数组复制进 `OLED_Data.c`：
```c
const uint8_t Face_angry[] = {
    0x00, 0x00, ... // 1024 字节
};
```

#### Step 2：在 `OLED_Data.h` 声明
```c
extern const uint8_t Face_angry[];
```

#### Step 3：在 `Face_Config.c` 加分支
```c
else if(Face_Mode==7) {
    OLED_Clear();
    OLED_ShowImage(0,0,128,64,Face_angry);
    OLED_Update();
}
```

#### Step 4：在需要的地方触发
```c
Face_Mode=7;
Face_Config();
```

### 场景 10.4：改舵机方向/校准零点

如果你发现装配好后某条腿反了：
- **方向反**：在 `Servo.c` 里把 `Angle` 改成 `180 - Angle`
- **零点偏**：改公式中的 `500` 这个基础脉宽，比如改成 `480` 就把 0° 位置向一个方向偏 ~2°

```c
// Servo.c
void Servo_Angle1(float Angle) {
    // 原：正向，零点 500us
    PWM_SetCompare1(Angle / 180 * 2000 + 500);
    
    // 改方向：
    // PWM_SetCompare1((180-Angle) / 180 * 2000 + 500);
    
    // 零点微调：
    // PWM_SetCompare1(Angle / 180 * 2000 + 480);
}
```

### 场景 10.5：调整呼吸灯节奏

`main.c` 的 `TIM3_IRQHandler`：
```c
HuXi += 100;   // 改小→更慢；改大→更快
if(HuXi == 20000) PanDuan = 2;   // 改小上限 → 最大亮度变暗
```

---

## 11. 新增功能建议与实现方案

下面每一个都附上"改哪几个文件 + 大致代码框架"，方便你对号入座挑一个开做。

### 11.1 加电池电量检测并在 OLED 上显示

**思路**：用 ADC1 通道读取电池电压分压点，换算成 0~100% 显示在表情角落。

**涉及文件**：
- 新增 `HardWare/Battery.c / .h`
- 修改 `Face_Config.c` 在每次 `OLED_Update` 前叠加电量文字

**关键代码**：
```c
// Battery.c
float Battery_ReadVoltage(void) {
    ADC_RegularChannelConfig(ADC1, ADC_Channel_4, 1, ADC_SampleTime_55Cycles5);
    ADC_SoftwareStartConvCmd(ADC1, ENABLE);
    while(!ADC_GetFlagStatus(ADC1, ADC_FLAG_EOC));
    uint16_t raw = ADC_GetConversionValue(ADC1);
    return raw * 3.3f / 4096 * 2;  // 假设 1:1 分压
}
```

### 11.2 加一个独立按键直接切换动作（免蓝牙）

**思路**：按键挂在 PA11（上拉），中断 EXTI11 里把 `Action_Mode` 循环 +1。

**涉及文件**：
- 新增 `HardWare/Key.c / .h`
- `main.c` 加一次 `Key_Init()`

**关键代码**：
```c
void EXTI15_10_IRQHandler(void) {
    if(EXTI_GetITStatus(EXTI_Line11) == SET) {
        Delay_ms(20);  // 消抖
        Action_Mode = (Action_Mode + 1) % 16;
        Face_Mode = 2; Face_Config();
        EXTI_ClearITPendingBit(EXTI_Line11);
    }
}
```

### 11.3 加 MPU6050 让宠物"感知被拿起来"

**思路**：I²C 读 MPU6050 的加速度，当 Z 轴方向重力突然消失 → 判定"被拿起"，立刻切到"惊讶表情+挣扎动作"。

**涉及文件**：
- 新增 `HardWare/MPU6050.c / .h`
- `main.c` 里 while(1) 开头加一次 `MPU6050_Read()` 状态检查

**伪代码**：
```c
float az = MPU6050_GetAccelZ();
if (az < 0.3f) {   // 正常 ~1g，失重 <0.3g
    Action_Mode = 8;  // 挣扎（暂用摇摆）
    Face_Mode = 3; Face_Config();
}
```

### 11.4 加 ESP8266 / ESP32 实现 WiFi + 小程序控制

**思路**：ESP8266 透传模式，把 USART3 换成 ESP8266，手机小程序发 HTTP → ESP8266 TCP 透传 → STM32 串口。

**几乎不用改 STM32 代码**，只要把蓝牙模块替换成 ESP8266，保持 9600 波特率即可。

### 11.5 加离线 AI 语音（SU-03T 已有，升级到小智 AI）

详细方案见 `docs/AI_INTEGRATION_PLAN.md`。核心思路：
- 高层 AI（云端）判断意图；
- 语音网关（ESP32）把意图压缩成 1 字节发给 STM32；
- STM32 代码几乎零改动。

### 11.6 加"随机空闲动作"让宠物更生动

**思路**：主循环里维护一个"空闲计时器"，超过 30 秒没收到命令就随机播放一个动作。

**改动 `main.c`**：
```c
uint32_t IdleCounter = 0;
// while(1) 内加：
if(++IdleCounter > 300000UL) {   // 主循环每圈大约 100us，300000 圈≈30s
    IdleCounter = 0;
    Action_Mode = 8 + (rand() % 3);   // 8/9/10 随机
    Face_Mode = 2 + (rand() % 3);
    Face_Config();
}
// 在 IRQ 里清零 IdleCounter 即可
```

### 11.7 加"情绪值"系统（RPG 风格）

**思路**：维护一个 `Mood` 全局变量（0~100）。每次成功完成命令 +1，每次被摇晃（MPU6050）-5，低情绪显示悲伤表情、拒绝某些命令。

### 11.8 加 SD 卡 + 语音播放（DFPlayer Mini）

**思路**：挂一个 DFPlayer Mini MP3 模块，USART2 控制。每个动作对应一段叫声。

---

## 12. 代码已知问题与重构建议

以下问题来自对当前 master 分支的源码审查，**修复会显著提升代码质量，但不会改变功能**。

### 12.1 `BlueTooth.c` 的 `USART_ReceiveData` 被连续多次调用

**问题**：一次中断只应读 DR 一次。
**现象**：`USART_ReceiveData(USART1)==0x29 … ==0x30 … ==0x31 …` 每个 `else if` 都重新读一次 DR。
**结果**：第一次读到正确值后 DR 被清空，后续比较读到 0，只是恰好 0x29 是第一个判断所以没出问题。换成 0x49 这种排在最后的命令，实际上走了多次读+清空过程，**是隐蔽 bug**。
**修复**：
```c
void USART1_IRQHandler(void) {
    if (USART_GetITStatus(USART1, USART_IT_RXNE) == SET) {
        uint8_t cmd = USART_ReceiveData(USART1);   // ← 只读一次
        Sustainedmove = 0;
        switch (cmd) {
            case 0x29: Face_Mode=0; Face_Config(); Action_Mode=0; break;
            case 0x30: Face_Mode=1; Face_Config(); Action_Mode=1; break;
            // ... 其余 case
        }
        USART_ClearITPendingBit(USART1, USART_IT_RXNE);
    }
}
```

### 12.2 USART1/USART3 中断代码完全重复

**问题**：220 行几乎一模一样，仅 `Sustainedmove` 默认值不同。
**修复**：抽一个函数：
```c
static void Command_Dispatch(uint8_t cmd, uint8_t sustained) {
    Sustainedmove = sustained;
    switch (cmd) {
        case 0x29: Action_Mode=0; Face_Mode=0; Face_Config(); break;
        // ...
    }
}
void USART1_IRQHandler(void) { /* 读 cmd */ Command_Dispatch(cmd, 0); }
void USART3_IRQHandler(void) { /* 读 cmd */ Command_Dispatch(cmd, 1); }
```

### 12.3 全局变量缺 `volatile`

**问题**：`Action_Mode`、`Face_Mode` 在中断改，在主循环读，编译器可能优化成寄存器缓存。
**修复**：`BlueTooth.c` 中定义改成 `volatile uint16_t Action_Mode=0;`，`BlueTooth.h` 声明同步改 `extern volatile uint16_t Action_Mode;`。

### 12.4 `Delay_ms` 在中断里会导致响应延迟

**问题**：动作函数里的 `Delay_ms(SpeedDelay)` 阻塞主循环，如果正好在动作中收到新命令，最慢要等 `SpeedDelay` ms。
**缓解**：动作函数内部已经用"每帧检查 + break"部分化解决，但彻底方案是把动作做成状态机（如 `Action_Tick()` 每 20ms 调用一次）。

### 12.5 硬件 I²C 可用却用了软件 I²C

**问题**：软件 I²C 会占用 CPU。
**改造**：`OLED.c` 的 `OLED_W_SCL/SDA` 可以替换成 `I2C_SendData(I2C1, ...)`，需要把 PB8/PB9 换成 I2C 专用的 PB6/PB7。

### 12.6 `PetAction.h` 头文件宏名不一致

```c
#ifndef __PET_ACTION_
#define __PET_ACTION        ← 少了后缀下划线
```
应为：
```c
#ifndef __PET_ACTION_H
#define __PET_ACTION_H
```

### 12.7 `Face_sleep` 等变量没在 `OLED_Data.h` 中完全声明 / `BlueTooth.h` 部分声明重复

建议统一维护一个 `AssetTable.h`。

---

## 13. 调试技巧

### 13.1 用 OLED 当 printf

最简单的调试手段，在任何地方写：
```c
OLED_ShowNum(0, 0, Action_Mode, 3, OLED_8X16);
OLED_Update();
```
立刻看到状态变化。

### 13.2 逻辑分析仪看 PWM

把 PA0 接逻辑分析仪或示波器，观察脉宽是否在 500µs~2500µs 之间，占空比 ~5%~12.5%。

### 13.3 串口发命令模拟控制

用 USB-TTL 接 PB10/PB11，打开 SSCOM（或任意串口助手），**HEX 模式**发送 `0x33` 就能触发前进，不需要真蓝牙模块。

### 13.4 Keil 仿真

Keil 直接 **Start/Stop Debug Session (Ctrl+F5)** 就能全速运行/单步；还可以在 `Watch 1` 窗口监视 `Action_Mode`、`Face_Mode`、`SpeedDelay` 实时变化。

---

## 14. 二开路线图

### 🟢 新手阶段（1~3 小时）
- [ ] 成功烧录官方代码
- [ ] 用串口助手发 HEX 命令测试所有动作
- [ ] 改一个表情数据（比如把睡觉脸改成 X_X）
- [ ] 改 `SpeedDelay` 初始值让它默认更快/更慢

### 🟡 进阶阶段（一周）
- [ ] 新增一个自定义动作（§10.2）
- [ ] 新增一个自定义表情（§10.3）
- [ ] 重构 USART1/3 中断（§12.1, §12.2）
- [ ] 加电池检测（§11.1）或按键（§11.2）

### 🔴 专家阶段（一个月）
- [ ] 接入 MPU6050 做姿态感知（§11.3）
- [ ] 把动作重构成非阻塞状态机（§12.4）
- [ ] 对接云 AI 实现语义级控制（§11.5 / `AI_INTEGRATION_PLAN.md`）
- [ ] 迁移到 FreeRTOS，拆分动作/显示/通信三个任务

---

## 附录 A：一键上手 Checklist

工作区打开后第一件事：
1. 📖 通读本文档 §3、§7、§8（20 分钟）
2. 🔌 按 §2.2 接好硬件
3. 🛠️ 按 §9 编译 + 烧录
4. 🧪 用串口发 `0x31` 测试站立
5. ✏️ 按 §10.1 改一个命令验证开发循环
6. 🚀 对着 §14 的路线图开干！

---

**文档版本**：2026-04-19
**适配代码版本**：master（2025-02-17 更新）
**作者**：基于原作者 Sngels_wyh 的工程二次梳理（GitHub: ins13014778/desktop-pet-stm32-f103）

