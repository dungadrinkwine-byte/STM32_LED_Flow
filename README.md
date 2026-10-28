# 电控组第二次培训作业

- 姓名：**黄予涵**
- 学号：***20253008098***

## 实现思路

在 CubeMX 中把 PA0、PA2、PA4 配置为推挽输出，并分别打上 User Label `LED0`、`LED1`、`LED2`，生成代码后即可用 `LED0_Pin` / `LED0_GPIO_Port` 这类宏操作引脚。

三盏灯在 `while (1)` 中顺序执行同一套动作：**点亮 → 延时 500 ms → 熄灭**。三段首尾相接，由主循环回到第一段形成循环，即 LED0 → LED1 → LED2 → LED0，一轮 1.5 秒。

这里需要注意的是，每盏灯只需要**一个** `HAL_Delay(500)`。因为上一盏灯熄灭的下一瞬间下一盏就点亮，中间没有全灭的空档，所以延时只放在"点亮"和"熄灭"之间。如果每盏灯后都再补一段延时，周期会变成 2000 ms，比要求慢一倍。

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
