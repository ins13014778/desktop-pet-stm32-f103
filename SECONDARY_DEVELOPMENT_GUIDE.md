# 桌面宠物项目二开说明

## 1. 项目概览

这是一个基于 `STM32F103C8T6` 的桌面宠物项目，工程文件是：

- `Project.uvprojx`

这个项目本身是 `Keil MDK` 工程，最适合直接用 `Keil uVision5` 打开和编译。

目录作用大致如下：

- `User/`
  - 主程序入口、中断文件、工程配置
- `HardWare/`
  - 舵机、蓝牙、语音、OLED、动作、PWM 等业务驱动
- `Start/`
  - 启动文件、芯片头文件、系统时钟
- `Library/`
  - STM32 标准外设库
- `Objects/`
  - 编译输出目录
- `Listings/`
  - 列表文件输出目录

## 2. 二开时优先改哪些文件

如果只是普通二开，优先改下面这些文件：

### 2.1 主流程入口

- `User/main.c`

作用：

- 程序入口 `main()`
- 主循环根据 `Action_Mode` 执行动作
- `TIM3_IRQHandler()` 负责灯光呼吸效果

适合修改：

- 动作模式切换逻辑
- 主循环行为
- 灯光效果

### 2.2 语音和蓝牙命令入口

- `HardWare/BlueTooth.c`

作用：

- 初始化 `USART1` 和 `USART3`
- `USART1_IRQHandler()` 处理语音模块命令
- `USART3_IRQHandler()` 处理蓝牙模块命令

适合修改：

- 命令字节和功能的映射关系
- 收到命令后切换的动作、表情、灯光
- 语音和蓝牙控制逻辑

### 2.3 动作库

- `HardWare/PetAction.c`

作用：

- 站立、坐下、趴下、前进、后退、转向、摆尾、打招呼、伸懒腰等动作

适合修改：

- 每个动作的姿态
- 动作顺序
- 动作速度
- 新增自定义动作

### 2.4 舵机控制

- `HardWare/Servo.c`
- `HardWare/PWM.c`

作用：

- `Servo.c` 负责角度到 PWM 脉宽的换算
- `PWM.c` 负责 TIM2/TIM3 的 PWM 输出和灯光通道

适合修改：

- 舵机角度校准
- 舵机方向修正
- 引脚和 PWM 通道配置

### 2.5 表情显示

- `HardWare/Face_Config.c`
- `HardWare/OLED_Data.c`

作用：

- `Face_Config.c` 决定 `Face_Mode` 对应显示哪张脸
- `OLED_Data.c` 存放表情图片数据

适合修改：

- 表情切换逻辑
- 替换已有表情
- 新增表情数据

## 3. 哪些文件一般不要改

普通二开通常不建议一开始就改这些文件：

- `Library/*.c`
- `Library/*.h`
- `Start/*`

原因：

- `Library/` 是 STM32 标准外设库
- `Start/` 是启动文件、系统时钟和芯片头文件
- 如果不是底层驱动开发，一般不需要动

## 4. 语音控制从哪里改

语音相关主要改这里：

- `HardWare/BlueTooth.c`

项目里语音和蓝牙是分开的：

- `USART1`：语音模块
- `USART3`：蓝牙模块

对应入口函数：

- `USART1_IRQHandler()`：语音命令处理
- `USART3_IRQHandler()`：蓝牙命令处理

如果你要修改“语音识别后执行什么动作”，主要就是改 `USART1_IRQHandler()` 里的命令判断。

## 5. 蓝牙控制从哪里改

蓝牙控制同样在：

- `HardWare/BlueTooth.c`

如果你要修改：

- 手机蓝牙发什么字节
- 收到蓝牙命令后执行什么动作
- 蓝牙控制灯光或速度

就改 `USART3_IRQHandler()`。

## 6. 命令字节和功能对照

下面是代码里目前的命令映射，`USART1_IRQHandler()` 和 `USART3_IRQHandler()` 基本一致。

| 命令值 | 当前功能 |
| --- | --- |
| `0x29` | 放松趴下 |
| `0x30` | 坐下 |
| `0x31` | 站立 |
| `0x32` | 趴下 |
| `0x33` | 前进 |
| `0x34` | 后退 |
| `0x35` | 左转 |
| `0x36` | 右转 |
| `0x37` | 摇摆 |
| `0x38` | 调整移动速度 |
| `0x39` | 调整摇摆速度 |
| `0x40` | 摆尾开关 |
| `0x41` | 向前跳 |
| `0x42` | 向后跳 |
| `0x43` | 打招呼 |
| `0x44` | 开灯 |
| `0x45` | 关灯 |
| `0x46` | 开启呼吸灯 |
| `0x47` | 关闭呼吸灯 |
| `0x48` | 伸懒腰 |
| `0x49` | 拉伸后腿 |

如果你想改语音命令，有两种常见方式：

### 6.1 保持语音模块发出的命令不变

只改命令对应的功能，比如把：

- `0x43` 从“打招呼”改成“原地摇摆”

这种做法最简单，只需要改 `BlueTooth.c`。

### 6.2 改语音模块本身输出的命令字节

如果你的语音模块支持自定义返回码，也可以在语音模块那边改发出的字节，再同步修改 `BlueTooth.c` 的判断。

## 7. 动作要从哪里改

动作内容主要在：

- `HardWare/PetAction.c`

你会看到这些动作函数：

- `Action_relaxed_getdowm()`
- `Action_sit()`
- `Action_upright()`
- `Action_getdowm()`
- `Action_advance()`
- `Action_back()`
- `Action_Lrotation()`
- `Action_Rrotation()`
- `Action_Swing()`
- `Action_SwingTail()`
- `Action_JumpU()`
- `Action_JumpD()`
- `Action_Hello()`
- `Action_stretch()`
- `Action_Lstretch()`

如果你想二开一个新动作，通常做法是：

1. 在 `PetAction.c` 里新增一个动作函数
2. 在 `PetAction.h` 里声明这个函数
3. 在 `main.c` 里给新的 `Action_Mode` 增加分支
4. 在 `BlueTooth.c` 里加新的命令触发条件

## 8. 表情从哪里改

表情切换逻辑：

- `HardWare/Face_Config.c`

表情图片数据：

- `HardWare/OLED_Data.c`

目前表情模式大致是：

- `Face_Mode = 0`：睡觉
- `Face_Mode = 1`：瞪眼
- `Face_Mode = 2`：开心
- `Face_Mode = 3`：狂热
- `Face_Mode = 4`：非常开心
- `Face_Mode = 5`：眼睛
- `Face_Mode = 6`：打招呼

如果你只是想改某个状态显示哪张脸：

- 改 `Face_Config.c`

如果你想换图案本身：

- 改 `OLED_Data.c`

## 9. 灯光从哪里改

灯光控制主要看：

- `User/main.c`
- `HardWare/PWM.c`

说明：

- `TIM3_IRQHandler()` 在 `main.c` 里
- `PWM_LED1()` 和 `PWM_LED2()` 在 `PWM.c` 里
- `AllLed` 和 `BreatheLed` 控制灯是否开启、是否呼吸

如果你想改：

- 呼吸灯亮灭节奏
- 灯光亮度变化速度
- 灯光默认开关状态

就从 `main.c` 里的 `TIM3_IRQHandler()` 开始改。

## 10. 二开建议顺序

如果你是第一次改这个项目，建议按这个顺序来：

1. 先看 `BlueTooth.c`
   - 搞清楚命令字节和功能映射
2. 再看 `main.c`
   - 搞清楚 `Action_Mode` 如何分发动作
3. 再看 `PetAction.c`
   - 搞清楚每个动作函数具体做了什么
4. 最后再看 `Face_Config.c` 和 `OLED_Data.c`
   - 修改表情显示

这个顺序最容易快速上手。

## 11. 编译方式建议

这个项目自带 `Keil` 工程，最推荐：

1. 用 `Keil uVision5` 打开 `Project.uvprojx`
2. 直接编译
3. 生成 `Objects/Project.hex`
4. 再用下载工具或烧录软件把 `hex` 烧进板子

如果使用图形化搭积木软件，通常只能烧它自己生成的程序，不适合直接接管这个现成的 `Keil` C 工程。

## 12. 修改时的注意事项

### 12.1 不要一开始就改底层库

优先改：

- `User/`
- `HardWare/`

尽量不要先改：

- `Library/`
- `Start/`

### 12.2 动作修改时先小改

建议先只改一两个角度和延时，观察效果，不要一次把整套动作都改掉。

### 12.3 舵机角度需要注意机械极限

代码里常用的角度大多在安全范围内。如果你自己改动作，注意不要让舵机超过结构允许范围，否则容易卡住或者抖动。

### 12.4 语音和蓝牙逻辑目前是重复的

`USART1_IRQHandler()` 和 `USART3_IRQHandler()` 现在基本是复制关系。二开时如果改了一个入口，另一个入口通常也要同步修改。

## 13. 最常见的二开需求对应改法

### 13.1 想改语音命令对应的动作

改：

- `HardWare/BlueTooth.c`

### 13.2 想改动作姿态和速度

改：

- `HardWare/PetAction.c`

### 13.3 想改表情

改：

- `HardWare/Face_Config.c`
- `HardWare/OLED_Data.c`

### 13.4 想改灯光效果

改：

- `User/main.c`
- `HardWare/PWM.c`

### 13.5 想新增一个完整功能

通常需要同时改：

- `HardWare/BlueTooth.c`
- `User/main.c`
- `HardWare/PetAction.c`
- `HardWare/PetAction.h`

---

如果后续要继续二开，最推荐的阅读顺序是：

`BlueTooth.c -> main.c -> PetAction.c -> Face_Config.c -> OLED_Data.c`

这样最容易快速理解这个项目的控制链路。
