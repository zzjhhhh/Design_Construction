# 按键抖动计数（STM8 + IAR Watch）

## 一、功能说明

不按键码消抖，直接统计按键电平**跳变的次数**，也就是机械按键在按下/松开瞬间的抖动次数。计数结果保存在全局变量里，用 IAR 的 Watch 窗口实时查看；LED 每次跳变翻转一次，肉眼也能看到抖动。

| 操作 | 现象 |
| --- | --- |
| 上电 | `bounce_cnt = 0`，LED 熄灭 |
| 按一下按键 | `bounce_cnt` 增加若干（含按下、松开两个过程），LED 快速闪几下 |
| 看数值 | IAR `Live Watch` 里读 `bounce_cnt` |

## 二、硬件接线

| STM8 引脚 | 接到 | 说明 |
| --- | --- | --- |
| PI1 | 按键 | 按键另一端接 GND，内部上拉，按下为低电平 |
| PB0 | 板载 LED | 推挽输出，高电平点亮 |

## 三、程序思路

程序只有一个机制：**边沿检测**。

```
last = 当前电平
while(1)
{
    读电平
    if (电平 != last)   →  发生了一次跳变
    {
        last = 电平;
        bounce_cnt++;   记录一次抖动
        LED 翻转;        可视化
    }
}
```

`last` 保存上一次采样到的电平，只要本次读到的和它不一样，就说明按键在这两个采样点之间发生了跳变。循环里没有延时，采样速度极快（一条循环只有几个微秒），所以连很窄的毛刺也能抓到。

## 四、完整代码

```c
#include "stm8s.h"

unsigned int bounce_cnt = 0;      /* 抖动次数，加到 IAR Watch 里看 */

void main(void)
{
    unsigned char last;

    GPIO_Init(GPIOI, GPIO_PIN_1, GPIO_MODE_IN_PU_NO_IT);       /* 按键 */
    GPIO_Init(GPIOB, GPIO_PIN_0, GPIO_MODE_OUT_PP_LOW_FAST);   /* LED */

    last = GPIO_ReadInputPin(GPIOI, GPIO_PIN_1);

    while(1)
    {
        if(GPIO_ReadInputPin(GPIOI, GPIO_PIN_1) != last)    /* 电平一变 = 抖了一下 */
        {
            last = GPIO_ReadInputPin(GPIOI, GPIO_PIN_1);
            bounce_cnt++;                                   /* 计数 */
            GPIO_WriteReverse(GPIOB, GPIO_PIN_0);           /* LED 跟着翻 */
        }
    }
}
```

## 五、在 IAR 里看变量

1. 点击 **Download and Debug** 进入调试模式；
2. 菜单 **View → Live Watch**，在表达式框里输入 `bounce_cnt` 后回车；
3. 按 **F5** 全速运行；
4. 按一下按键，观察数字的变化；
5. 每测一组数据，按板上的**复位键**清零后重新开始。

> 如果数字不刷新：把 `Live Watch` 的刷新周期调小，或者改用普通 **Watch**（暂停后查看）。

## 六、读数怎么理解

| 读数 | 含义 |
| --- | --- |
| 按一次 = 2 | 理想开关：按下 1 次跳变、松开 1 次跳变，完全没有抖动 |
| 按一次 = 4 ~ 12 | **典型机械按键**，多出来的部分就是抖动 |
| 按一次 = 几十 | 按键老化、接触不良，或导线过长引入干扰 |
| 数字一直缓慢增长 | 引脚悬空或干扰，检查按键接线和接地 |

需要注意：本程序统计的是**按下和松开两个过程的总跳变数**，所以要先减去 2 次正常边沿，剩下的才是抖动次数。

## 七、实验数据记录表（可直接抄进报告）

| 次数 | 按下(次) | 抖动 `bounce_cnt` | 备注 |
| --- | --- | --- | --- |
| 1 | | | |
| 2 | | | |
| 3 | | | |
| 4 | | | |
| 5 | | | |

## 八、可选扩展

**1. 分开统计按下和松开方向的抖动**

```c
unsigned int press_cnt = 0;      /* 按下方向的边沿 */

/* 在 bounce_cnt++ 后面加一行 */
if(GPIO_ReadInputPin(GPIOI, GPIO_PIN_1) == RESET) press_cnt++;
```

最后 `press_cnt - 1` 是按下瞬间的抖动次数，`(bounce_cnt - press_cnt) - 1` 是松开瞬间的抖动次数。

**2. 和消抖程序做对照**

同一只手、同一个按键，分别跑一遍本程序和带 10 ms 延时的消抖程序，比较“识别到的按键次数”，就能直观看到抖动的危害。

## 九、注意事项

1. 变量的值靠**复位**清零，按复位键比在 Watch 里手动改更可靠；
2. 若编译器开了高等级优化，把变量改成 `volatile unsigned int bounce_cnt = 0;`，避免 Watch 里看到的数值不刷新；
3. `bounce_cnt` 是 16 位无符号数（0 ~ 65535），正常实验不会溢出，但连续按很久要注意上限；
4. LED 每次跳变都会翻转，抖动多的时候会看到明显的“闪”，这是正常现象，不是程序出错。
