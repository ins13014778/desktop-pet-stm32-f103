# STM32 桌面宠物 (Desktop Pet)

基于 **STM32F103C8T6** 的四足桌面宠物嵌入式项目，使用 **Keil MDK** 与 **STM32 标准外设库 V3.5** 开发。

> 🐾 本仓库保留了原始 Keil 工程结构，可直接用 `Keil uVision5` 打开 `Project.uvprojx` 编译；同时补充了完整的中文二次开发文档。

## 🌟 项目特点

- 🎤 **双通道控制**：语音模块（USART1）+ 手机蓝牙（USART3）同时在线
- 🐕 **15 种动作**：前进/后退/转向/坐下/趴下/跳跃/伸懒腰/打招呼等
- 😊 **7 种表情**：睡觉/瞪眼/快乐/狂热/眼睛/打招呼等，128×64 OLED 显示
- 💡 **呼吸灯效果**：TIM3 硬件 PWM 驱动 LED，支持常亮/呼吸/息屏
- 🦴 **5 路舵机协同**：4 条腿 + 1 条尾巴，SG90 标配
- 🧩 **易扩展**：命令/动作/表情三者解耦，适合新增传感器、AI 对接、WiFi 控制等二开方向

## 🛠️ 硬件规格

| 模块 | 说明 |
| --- | --- |
| 主控 | STM32F103C8T6（Cortex-M3 @ 72 MHz，64KB Flash）|
| 舵机 | SG90 180° × 5（4 腿 + 1 尾）|
| 显示 | 0.96" OLED (SSD1306 I²C) |
| 通信 | HC-05/06 蓝牙 + 语音识别模块 |
| 灯光 | LED × 2（PWM 可调）|

详细接线见 [docs/FULL_SECONDARY_DEV_GUIDE.md §2](docs/FULL_SECONDARY_DEV_GUIDE.md#2-硬件清单与接线图)

## 📁 目录结构

```
桌面宠物代码/
├── User/           主入口 main.c + 中断 + 标准库配置
├── HardWare/       ★ 业务层（舵机/串口/OLED/动作/表情）
├── Start/          启动文件 + 时钟 + 内核头
├── Library/        STM32F10x 标准外设库（V3.5）
├── docs/           二开文档
├── Project.uvprojx Keil 工程主文件
└── CLAUDE.md       给 AI 编程助手的工程导览
```

## 🚀 快速上手

### 编译与烧录

1. 用 `Keil uVision5` 打开 `Project.uvprojx`
2. 选中 `Target 1` → **F7** 编译
3. 用 ST-Link 或串口 ISP 烧录 `Objects/Project.hex` 到板子

### 控制测试

接好蓝牙模块后用手机蓝牙助手（HEX 模式）发送命令：
- `0x31` → 站立
- `0x33` → 前进
- `0x43` → 打招呼
- `0x37` → 摇摆

完整命令表见 [docs/FULL_SECONDARY_DEV_GUIDE.md §7](docs/FULL_SECONDARY_DEV_GUIDE.md#7-命令协议与字节码对照表)

## 📚 二开文档

| 文档 | 说明 | 适合阅读对象 |
| --- | --- | --- |
| **[📖 FULL_SECONDARY_DEV_GUIDE.md](docs/FULL_SECONDARY_DEV_GUIDE.md)** | **最全面的中文二开指南**（1000 行，架构图/命令表/每文件详解/重构建议/扩展方案/路线图）| **推荐首选** |
| [SECONDARY_DEVELOPMENT_DETAILED.md](docs/SECONDARY_DEVELOPMENT_DETAILED.md) | 早期版本详细二开说明 | 历史参考 |
| [AI_INTEGRATION_PLAN.md](docs/AI_INTEGRATION_PLAN.md) | 对接小智 AI / 云端语义的方案 | AI 爱好者 |
| [SECONDARY_DEVELOPMENT_GUIDE.md](SECONDARY_DEVELOPMENT_GUIDE.md) | 顶层二开概述 | 快速入门 |
| [CLAUDE.md](CLAUDE.md) | AI 编程助手使用的工程导览 | AI 工具 |

## 🎯 常见二开方向

| 方向 | 核心文件 | 难度 |
| --- | --- | --- |
| 改语音/蓝牙命令映射 | `HardWare/BlueTooth.c` | 🟢 简单 |
| 新增自定义动作 | `PetAction.c` + `main.c` + `BlueTooth.c` | 🟡 中等 |
| 新增表情图案 | `OLED_Data.c` + `Face_Config.c` | 🟢 简单 |
| 加独立按键/电池检测 | 新增 `Key.c` / `Battery.c` | 🟡 中等 |
| 接入 MPU6050 姿态感知 | 新增 `MPU6050.c` | 🔴 较难 |
| WiFi/云端 AI 控制 | 替换蓝牙模块为 ESP8266/ESP32 | 🔴 较难 |

每种方向的完整代码模板见 [FULL_SECONDARY_DEV_GUIDE.md §10-§11](docs/FULL_SECONDARY_DEV_GUIDE.md#10-二开常见场景)。

## 💡 核心技术要点

- **架构**：USART 中断写全局变量 → 主循环轮询分发 → 动作函数可打断
- **PWM**：TIM2/TIM3 预分频 72 → 1MHz → 20ms 周期 → 50Hz 标准舵机信号
- **舵机角度**：`CCR = Angle/180 × 2000 + 500`，即 500µs~2500µs 对应 0°~180°
- **OLED**：软件 I²C 驱动 + 1KB 显存数组 + `OLED_Update()` 才真正刷屏

## ⚠️ 注意事项

- `Start/` 与 `Library/` 是 STM32 启动层和标准库，**一般不要改动**
- `Objects/`、`Listings/`、`DebugConfig/` 已在 `.gitignore` 中忽略
- 源码使用 **GBK 编码**（Keil Windows 默认），用 VSCode 打开时注意选对编码
- `USART1_IRQHandler` 和 `USART3_IRQHandler` 是镜像复制代码，修改命令必须同步改两处

## 🙏 致谢

- 原作者：**Sngels_wyh**（CSDN / B 站 / 抖音），原项目地址：[oshwhub.com/sngelswyh/stm32-smart-desktop-pet](https://oshwhub.com/sngelswyh/stm32-smart-desktop-pet)
- OLED 驱动：**江协科技**（jiangxiekeji.com）
- 本仓库：[ins13014778/desktop-pet-stm32-f103](https://github.com/ins13014778/desktop-pet-stm32-f103) 在原项目基础上补充了完整中文二开文档与工程导览。

## 📜 许可

保留原作者版权声明。仓库内新增的文档以 MIT 协议共享。
