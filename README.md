# 电控组第二次培训作业

- 姓名：**黄予涵**
- 学号：***20253008098***

---

## 一、任务完成情况

| 项目 | 要求 | 完成情况 |
|---|---|---|
| 基础任务 | CubeMX 建立工程，三个 LED 的 GPIO 用 User Label 命名为 LED0 / LED1 / LED2，CLion 中编写代码实现 500ms 间隔三灯流水 | ✅ 已完成 |
| 代码规范 | 直接调用宏定义操作引脚 | ✅ 使用 `LED0_Pin` / `LED0_GPIO_Port` 等宏 |
| README.md | 按要求格式编写 | ✅ 本文件 |
| 仓库 | GitHub 公开仓库，包含工程代码 + 录屏 + README | ✅ |

演示视频：[video/led_flow_simple_demo.mp4](video/led_flow_simple_demo.mp4)

---

## 二、硬件与开发环境

| 项目 | 说明 |
|---|---|
| 开发板 | STM32F103C8T6 最小系统板 |
| 系统时钟 | 72 MHz（HSE 8 MHz 晶振 × PLL9） |
| 下载调试 | SWD（PA13 = SWDIO，PA14 = SWCLK） |
| 配置工具 | STM32CubeMX + STM32Cube FW_F1 V1.8.7 |
| 开发环境 | CLion 2026.2 + CMake + arm-none-eabi-gcc |
| 库 | STM32F1xx HAL Driver |

### 引脚分配

| 灯 | 引脚 | User Label | 端口宏 | 引脚宏 |
|---|---|---|---|---|
| LED0 | PA0 | `LED0` | `LED0_GPIO_Port`（= GPIOA） | `LED0_Pin`（= GPIO_PIN_0） |
| LED1 | PA2 | `LED1` | `LED1_GPIO_Port`（= GPIOA） | `LED1_Pin`（= GPIO_PIN_2） |
| LED2 | PA4 | `LED2` | `LED2_GPIO_Port`（= GPIOA） | `LED2_Pin`（= GPIO_PIN_4） |

### 接线方式

```
STM32 PA0 ──[ 1kΩ ]──▶|── GND      （LED0，阳极接 GPIO，阴极接 GND）
STM32 PA2 ──[ 1kΩ ]──▶|── GND      （LED1）
STM32 PA4 ──[ 1kΩ ]──▶|── GND      （LED2）
```

三个引脚均配置为 **推挽输出（Output Push Pull）**、无上下拉、低速。
LED 阳极管脚接 GPIO，阴极经限流电阻接 GND，因此为 **高电平点亮**。

---

## 三、实现思路

### 1. 整体逻辑

三盏灯在 `while (1)` 中顺序执行同一套动作：**点亮 → 延时 500ms → 熄灭**，
三段首尾相接，靠主循环回到第一段形成循环，即：

```
LED0 亮 500ms → LED1 亮 500ms → LED2 亮 500ms → LED0 亮 500ms → …
```

一轮完整流水耗时 1.5 s。

### 2. 关键点：每盏灯只配一个延时

流水灯**不需要"全灭"阶段**。因为 LED0 熄灭的下一瞬间 LED1 就点亮，视觉上就是"灭一盏、亮一盏"，中间没有空档。

如果照搬"单灯闪烁"的写法（亮 → 延时 → 灭 → 再延时），节奏会变成：

```
LED0 亮 500ms → 全灭 500ms → LED1 亮 500ms → 全灭 500ms → …
```

实际周期变成 2000ms，比要求慢一倍。因此代码中 `HAL_Delay()` **只出现 3 次**，
位置都在"点亮"与"熄灭"之间。

### 3. 为什么直接调用宏而不是写死引脚号

CubeMX 中把三个引脚分别打了 User Label `LED0` / `LED1` / `LED2`，
生成代码时会在 `Core/Inc/main.h` 中自动生成宏：

```c
#define LED0_Pin          GPIO_PIN_0
#define LED0_GPIO_Port    GPIOA
#define LED1_Pin          GPIO_PIN_2
#define LED1_GPIO_Port    GPIOA
#define LED2_Pin          GPIO_PIN_4
#define LED2_GPIO_Port    GPIOA
```

代码中直接使用这些宏，好处是**将来改引脚只需在 CubeMX 里重新配置，
业务代码一行都不用动**。

需要注意 `HAL_GPIO_WritePin(port, pin, state)` 要同时传入两个参数：

- `LEDx_GPIO_Port` 决定操作**哪个端口**（哪一组寄存器）；
- `LEDx_Pin` 决定操作**这一组的第几位**（位掩码）。

本工程三盏灯都在 GPIOA 上，所以三个端口宏展开后完全相同，**真正起区分作用的是 `pin` 参数**。

---

## 四、核心代码

完整代码见 [`Core/Src/main.c`](Core/Src/main.c)，核心部分是 `while (1)` 循环体：

```c
while (1)
{
  HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_SET);
  HAL_Delay(500);
  HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_RESET);

  HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_SET);
  HAL_Delay(500);
  HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_RESET);

  HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_SET);
  HAL_Delay(500);
  HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_RESET);
}
```

引脚初始化由 CubeMX 生成在 [`Core/Src/gpio.c`](Core/Src/gpio.c) 的 `MX_GPIO_Init()` 中：

```c
HAL_GPIO_WritePin(GPIOA, LED0_Pin|LED1_Pin|LED2_Pin, GPIO_PIN_RESET);

GPIO_InitStruct.Pin = LED0_Pin|LED1_Pin|LED2_Pin;
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
GPIO_InitStruct.Pull = GPIO_NOPULL;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
```

---

## 五、目录结构

```
.
├── Core
│   ├── Inc
│   │   ├── gpio.h
│   │   ├── main.h                    # LED0/1/2 的宏定义在此
│   │   ├── stm32f1xx_hal_conf.h
│   │   └── stm32f1xx_it.h
│   ├── Src
│   │   ├── gpio.c                    # MX_GPIO_Init()：三个 LED 引脚初始化
│   │   ├── main.c                    # 流水灯主逻辑
│   │   ├── stm32f1xx_hal_msp.c
│   │   ├── stm32f1xx_it.c            # SysTick / 中断入口
│   │   ├── syscalls.c
│   │   ├── sysmem.c
│   │   └── system_stm32f1xx.c
│   └── Startup
│       └── startup_stm32f103c8tx.s
├── Drivers                           # CMSIS + STM32F1xx HAL Driver
├── video
│   └── led_flow_simple_demo.mp4      # 演示视频
├── CMakeLists.txt                    # CMake 工程文件（CLion 使用）
├── STM32F103C8TX_FLASH.ld            # 链接脚本
├── LED_WATER3.ioc                    # CubeMX 工程配置
└── README.md
```

---

## 六、编译与烧录

### 编译

用 CLion 打开本仓库根目录（含 `CMakeLists.txt` 与 `.ioc` 的那一层），
等待 CMake 加载完成后按 `Ctrl+F9` 编译，产物为 `cmake-build-debug/LED_WATER3.elf`。

命令行方式：

```bash
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build build
```

### 烧录

方式一：CLion 中配置 OpenOCD（`Settings → Build, Execution, Deployment → Embedded Development`），
运行配置选择 `OpenOCD Download & Run`，`Shift+F10` 一键编译下载。

方式二：用 STM32CubeProgrammer 直接烧写 `LED_WATER3.elf`。

### 上电现象

下载完成后板子复位，三盏灯依次点亮，每盏持续 500 ms，一轮 1.5 s，循环往复。

---

## 七、演示视频

[video/led_flow_simple_demo.mp4](video/led_flow_simple_demo.mp4)

视频内容：CLion 编译通过 → 程序烧录到开发板 → 三盏 LED 以 500ms 间隔依次流水。

---

## 八、重新生成代码说明

若需要修改引脚或时钟配置，请在 `LED_WATER3.ioc` 中修改后执行 `GENERATE CODE`。
CubeMX 只会保留成对 `USER CODE BEGIN xxx` / `USER CODE END xxx` 标记之间的内容，
其余部分会按 .ioc 配置重新生成，因此自定义代码务必写在用户区内。
