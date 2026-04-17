# STM32 Desktop Pet

基于 `STM32F103C8` 的桌面宠物嵌入式项目，使用 `Keil MDK` 与 `STM32 Standard Peripheral Library` 开发。

这个仓库保留了原始工程结构，方便直接用 `Keil uVision5` 打开 `Project.uvprojx` 编译，也补充了适合二次开发的源码说明文档。

## 项目特点

- 通过语音模块和蓝牙模块接收控制命令
- 驱动 4 个主舵机和 1 个尾巴舵机完成宠物动作
- 在 `0.96"` OLED 上显示表情
- 支持 LED 常亮与呼吸灯效果
- 适合做动作扩展、命令扩展、表情扩展和硬件移植

## 目录说明

- `User/`：主入口、动作分发、系统中断模板
- `HardWare/`：蓝牙、语音、舵机、PWM、OLED、动作、表情等业务逻辑
- `Start/`：启动文件、系统时钟、CMSIS 头文件
- `Library/`：STM32 标准外设库
- `docs/`：补充的源码解析与二开文档

## 如何编译

1. 使用 `Keil uVision5` 打开 `Project.uvprojx`
2. 选择 `Target 1`
3. 直接编译
4. 编译结果会生成到 `Objects/Project.hex`

## 二开文档

- [详细二开说明](docs/SECONDARY_DEVELOPMENT_DETAILED.md)
- [AI 对接方案（小智 AI / 网关 / STM32）](docs/AI_INTEGRATION_PLAN.md)

## 当前二开重点文件

- `HardWare/BlueTooth.c`
- `User/main.c`
- `HardWare/PetAction.c`
- `HardWare/Face_Config.c`
- `HardWare/OLED_Data.c`
- `HardWare/Servo.c`
- `HardWare/PWM.c`

## 说明

- 仓库保留了源码内已有的作者说明与第三方注释。
- `Start/` 与 `Library/` 包含芯片启动层与 STM32 标准库代码，通常不作为日常二开的重点区域。
- 已忽略 `Objects/`、`Listings/`、`DebugConfig/` 等编译输出和本机配置目录。
