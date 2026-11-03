# 电控组第二次培训作业

- 姓名：**黄予涵**
- 学号：***20253008098***

## 实现思路

在 CubeMX 中把 PA0、PA2、PA4 配置为推挽输出，并分别打上 User Label `LED0`、`LED1`、`LED2`，生成代码后即可用 `LED0_Pin` / `LED0_GPIO_Port` 这类宏操作引脚。

三盏灯依次点亮，每盏保持 500 ms：点亮 LED0 → 等 500 ms → 熄灭 → 点亮 LED1 → …… 首尾相接形成 LED0 → LED1 → LED2 → LED0 的循环，一轮 1.5 秒。

每盏灯只需要一个 500 ms 延时。因为上一盏熄灭的下一瞬间下一盏就点亮，中间没有全灭的空档；如果每盏灯后面跟两个延时，实际周期会变成 2000 ms，慢一倍。

核心代码（写在 `main()` 的 `while (1)` 里）：

```c
HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_SET);
HAL_Delay(500);
HAL_GPIO_WritePin(LED0_GPIO_Port, LED0_Pin, GPIO_PIN_RESET);

HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_SET);
HAL_Delay(500);
HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_RESET);

HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_SET);
HAL_Delay(500);
HAL_GPIO_WritePin(LED2_GPIO_Port, LED2_Pin, GPIO_PIN_RESET);
```

这个版本用 `HAL_Delay()` 阻塞等待，CPU 在 500 ms 内被锁在延时函数里。改成非阻塞延时的做法见 [main 分支](https://github.com/dungadrinkwine-byte/STM32_LED_Flow/tree/main)，那里有完整实现原理与核心代码片段。

## 演示视频

[video/led_flow_simple_demo.mp4](video/led_flow_simple_demo.mp4)
