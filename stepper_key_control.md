# 按键启停步进电机（STM8 + TB6600）

## 一、功能说明

用一个按键控制步进电机的启停，按一下启动、再按一下停止、再按一下继续运行；板载 LED 指示当前状态。

| 操作 | 现象 |
| --- | --- |
| 上电 | 电机停止，LED 熄灭 |
| 按一下按键 | 电机连续转动，LED 点亮 |
| 再按一下 | 电机停止（仍有保持力矩），LED 熄灭 |
| 再按一下 | 电机继续转动 |

按键按住不放只切换一次，松开后才允许下一次触发。

## 二、硬件接线

| STM8 引脚 | 接到 | 说明 |
| --- | --- | --- |
| PI1 | 按键 | 按键另一端接 GND，内部上拉，按下为低电平 |
| PB0 | 板载 LED | 推挽输出，高电平点亮 |
| PB2 | 驱动器 `PUL-` | 共阳接法，`PUL+` 接 5V |
| GND | 驱动器信号 GND | — |

驱动器 `DIR+ / DIR- / ENA+ / ENA-` 本程序不使用，可不接。

## 三、程序结构

| 函数 | 作用 |
| --- | --- |
| `delay()` | 步间延时，决定转速；同时兼作按键消抖 |
| `delay_pulse()` | 脉冲宽度，保证驱动器能识别（> 2.5 μs 即可） |
| `Key_Scan()` | 按键扫描，只在"刚按下"那一下翻转 `run_flag` |
| `Step_One()` | 输出一个完整脉冲，电机走一个细分步 |
| `main()` | 初始化 + 主循环 |

主循环流程：

```
初始化 → while(1) ┬─ Key_Scan()  扫描按键，翻转 run_flag
                  └─ run_flag == 1 ? ── 是 → Step_One() + delay()
                                     └─ 否 → 空转，等待按键
```

## 四、完整代码

```c
#include "stm8s.h"

/* ---------------------------------------------------------
   接线：
   PI1  <- 按键（另一端接 GND，内部上拉，按下为低）
   PB0  -> 板载 LED（高电平点亮，用来指示运行/停止）
   PB2  -> 驱动器 PUL-（共阳接法：PUL+ 接 5V）
   DIR+ / DIR- / ENA+ / ENA- 不用接
   --------------------------------------------------------- */

unsigned char run_flag = 0;   // 0:停止   1:运行
unsigned char key_flag = 1;   // 1:按键已松开   0:按键按住还没松

/* 步间延时：数值越大转得越慢（这个值也当按键消抖用） */
void delay()
{
  unsigned int a, i;
  for (i = 0; i < 5000; i++)
    a = 0;
}

/* 脉冲宽度：几十微秒，保证驱动器能稳稳识别（>2.5us 即可） */
void delay_pulse()
{
  unsigned int i;
  for (i = 0; i < 50; i++);
}

/* 按键扫描：只在"刚按下"那一下翻转 run_flag */
void Key_Scan(void)
{
    if (GPIO_ReadInputPin(GPIOI, GPIO_PIN_1) == RESET)   // 现在是按下
    {
        if (key_flag == 1)                               // 上一次是松开的 → 新的一次按下
        {
            delay();                                     // 消抖
            if (GPIO_ReadInputPin(GPIOI, GPIO_PIN_1) == RESET)
            {
                key_flag = 0;                            // 锁住，按住不放不再触发
                run_flag = !run_flag;                    // 启动 <-> 停止 切换

                if (run_flag == 1)
                    GPIO_WriteHigh(GPIOB, GPIO_PIN_0);   // 运行：LED 点亮
                else
                    GPIO_WriteLow(GPIOB, GPIO_PIN_0);    // 停止：LED 熄灭
            }
        }
    }
    else                                                 // 松手了
    {
        key_flag = 1;                                    // 允许下一次按下
    }
}

/* 发一个脉冲 = 电机走一个细分步 */
void Step_One(void)
{
    GPIO_WriteHigh(GPIOB, GPIO_PIN_2);
    delay_pulse();
    GPIO_WriteLow(GPIOB, GPIO_PIN_2);
}

void main(void)
{
    GPIO_Init(GPIOI, GPIO_PIN_1, GPIO_MODE_IN_PU_NO_IT);      // 按键：上拉输入，不开中断
    GPIO_Init(GPIOB, GPIO_PIN_0, GPIO_MODE_OUT_PP_LOW_FAST);  // LED：推挽输出，初始低电平（熄灭）
    GPIO_Init(GPIOB, GPIO_PIN_2, GPIO_MODE_OUT_PP_LOW_FAST);  // PUL-：推挽输出，初始低

    while (1)
    {
        Key_Scan();              // 一直扫按键，停止状态下也能响应

        if (run_flag == 1)       // 只有运行时才发脉冲
        {
            Step_One();          // 走一步
            delay();             // 步间停顿 -> 决定转速
        }
    }
}
```

## 五、关键点说明

### 1. 按键边沿检测（不会连续切换）

用 `key_flag` 记录"上一次是否已经按下"：

- 松开状态（`key_flag = 1`）下检测到低电平 → 认为是一次新的按下，翻转 `run_flag`，并把 `key_flag` 置 0；
- 只要还按着，`key_flag` 一直是 0，不会再进判断；
- 松开后检测到高电平 → `key_flag` 恢复 1，允许下一次触发。

`delay()` 在"刚按下"这一瞬间做一次消抖，避免抖动导致误翻转。

### 2. 脉冲与转速

`Step_One()` 输出的高—低电平变化对应一个脉冲，驱动器每收到一个脉冲走一个细分步。

```
转速(RPM) = 60 x 步频 / (200 x 细分数)
```

其中 `步频` 由 `delay()` 的循环次数和系统时钟决定（复位后默认 2 MHz）。同一份代码，细分不同转速差很多：

| 细分（拨码 S1 S2 S3） | 每圈脉冲数 | 大致转速 |
| --- | --- | --- |
| 1/32（OFF OFF OFF） | 6400 | 极慢，几乎看不出 |
| 1/8（OFF ON OFF） | 1600 | 慢 |
| 1/2（OFF ON ON） | 400 | 明显转动 |
| 1/1（ON ON OFF） | 200 | 最快 |

想提速有三个办法：减小 `delay()` 里的 5000、把细分调低、提高系统时钟（`CLK_HSIPrescalerConfig(CLK_PRESCALER_HSIDIV1)` 切到 16 MHz）。

### 3. LED 指示

本板载 LED 为**高电平点亮**：`GPIO_WriteHigh()` 亮，`GPIO_WriteLow()` 灭。初始化用 `GPIO_MODE_OUT_PP_LOW_FAST`，上电时是熄灭状态。

### 4. 停止时电机仍被锁住

驱动器没有接 `ENA`，所以即使 `run_flag = 0`（不发送脉冲），驱动器仍在给电机通电保持力矩，手拧不动属于正常现象。

## 六、注意事项

1. **拨码要匹配电机**：电流档按电机额定值设（本电机 1.5 A），改完必须**断电重新上电**才生效；
2. **PB2 接 `PUL-`**，不要接 `PUL+`（共阳接法下 `PUL+` 接 5V，`PUL-` 被拉低才出脉冲）；
3. **不要带电插拔电机线**，容易损坏驱动器；
4. 细分调得过细、`delay()` 调得过小时，电机会"嗡嗡"响但不转，说明丢步，需要把参数调回去；
5. 按键使用内部上拉，按键另一端必须可靠接地。
