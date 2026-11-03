# 电控组第二次培训作业

- 姓名：**黄予涵**
- 学号：***20253008098***

## 实现思路

在 CubeMX 中把 PA0、PA2、PA4 配置为推挽输出，并分别打上 User Label `LED0`、`LED1`、`LED2`，生成代码后即可用 `LED0_Pin` / `LED0_GPIO_Port` 这类宏操作引脚。

三盏灯依次点亮，每盏保持 500 ms：点亮 LED0 → 等 500 ms → 熄灭 → 点亮 LED1 → …… 首尾相接形成 LED0 → LED1 → LED2 → LED0 的循环，一轮 1.5 秒。每盏灯只需要一个 500 ms 延时，因为上一盏熄灭的下一瞬间下一盏就点亮，中间没有全灭的空档。

为了不让"等待"占住 CPU，流水逻辑用非阻塞延时实现，原理见下一节。

## 加分项：非阻塞延时

### 一、原方案的问题

`HAL_Delay()` 的内部实现可以简化为：

```c
uint32_t tickstart = HAL_GetTick();
while ((HAL_GetTick() - tickstart) < Delay)
{
}
```

问题不在于"用 tick 计时"，而在于这个 `while` 把 CPU 锁死在函数里整整 500 ms。这段时间里按键不响应、串口来不及接收，也没法再写第二件事。

### 二、实现原理：把"等"改成"查"

非阻塞的思路是：不再停在原地等，而是**记住上一次换灯的时刻**，每次进函数只问一句"距离上次够 500 ms 了吗"。

- 不够：立即 `return`，CPU 回去处理别的事情；
- 够了：执行换灯，并把"上次时刻"更新为现在。

计时的活儿交给 SysTick 中断维护的 `HAL_GetTick()` 毫秒计数——它一直在后台自己走，不需要 CPU 参与。

这样主循环里只剩一句 `LedFlow_Task()`，绝大多数时候 1~2 微秒就返回，"等待"这件事从"独占 CPU"变成了"每次路过问一句"。

### 三、三个关键细节

1. **时间差用无符号相减**：`now - s_lastTick`。`HAL_GetTick()` 是 `uint32_t` 毫秒计数，约 49.7 天回绕一次；无符号减法按模 2³² 计算，回绕时仍能得到正确的差值。不能写成 `now > s_lastTick + 500`，那样加法本身会溢出。
2. **`s_lastTick = now` 而不是 `+= 500`**：以"实际换灯那一刻"作为新基准，误差不会累积；写成 `+= 500` 会把基准锁在理想网格上，循环一旦被拖慢就会连续补跳。
3. **`% LED_NUM` 实现序号循环**：把 0→1→2→0 表达成一个表达式，改灯数只需要改宏。

### 四、核心代码

灯表，把"第几盏灯"和"哪个端口、哪个引脚"解耦：

```c
typedef struct
{
  GPIO_TypeDef *port;
  uint16_t      pin;
} LedDef_t;

static const LedDef_t s_ledTable[LED_NUM] =
{
  { LED0_GPIO_Port, LED0_Pin },
  { LED1_GPIO_Port, LED1_Pin },
  { LED2_GPIO_Port, LED2_Pin },
};
```

非阻塞的核心任务：

```c
static void LedFlow_Task(void)
{
  uint32_t now = HAL_GetTick();

  if ((now - s_lastTick) < LED_FLOW_INTERVAL_MS)
  {
    return;
  }
  s_lastTick = now;

  Led_Write(s_step, 0U);
  s_step = (uint8_t)((s_step + 1U) % LED_NUM);
  Led_Write(s_step, 1U);
}
```

换灯函数：

```c
static void Led_Write(uint8_t idx, uint8_t on)
{
  GPIO_PinState level = (on != 0U) ? LED_LEVEL_ON : LED_LEVEL_OFF;
  HAL_GPIO_WritePin(s_ledTable[idx].port, s_ledTable[idx].pin, level);
}
```

主循环，已经没有任何 `HAL_Delay()`：

```c
while (1)
{
  LedFlow_Task();
}
```

## 演示视频

| 版本 | 文件 |
|---|---|
| 基础版（`HAL_Delay` 阻塞实现） | [video/led_flow_simple_demo.mp4](video/led_flow_simple_demo.mp4) |
| 升级版（非阻塞延时） | [video/led_flow_nonblocking_demo.mp4](video/led_flow_nonblocking_demo.mp4) |

基础版的完整代码保留在 [基础版参考/main_basic.c](基础版参考/main_basic.c)，不参与当前工程编译。
