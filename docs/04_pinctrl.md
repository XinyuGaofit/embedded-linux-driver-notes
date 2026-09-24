# 04. pinctrl

## pinctrl 管什么

同一个 SoC 引脚通常可以连接到多个内部功能：

```text
PIN_X
 ├─ GPIO
 ├─ UART_TX
 ├─ I2C_SCL
 └─ PWM
```

pinctrl 负责：

- pin mux：选择引脚功能
- pin configuration：上下拉、驱动能力、开漏等

它不能创造芯片里不存在的连接。

## 设备树

```dts
mydev {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&pinctrl_mydev_default>;
    pinctrl-1 = <&pinctrl_mydev_sleep>;
};
```

i.MX6ULL BSP 常见：

```dts
pinctrl_mydev_default: mydevgrp {
    fsl,pins = <
        /* MX6UL_PAD_xxx__GPIOx_IOxx  PAD_CONFIG */
    >;
};
```

具体宏和值应从当前 BSP 的 pinfunc 头文件、板级 DTS 和参考手册中确认。

## 常用 API

很多简单设备只声明 `default` state，驱动不一定显式操作 pinctrl。

需要手动切状态时：

```c
struct pinctrl *p;
struct pinctrl_state *default_state;
struct pinctrl_state *sleep_state;
```

|API|输入|返回|作用|
|---|---|---|---|
|`devm_pinctrl_get(dev)`|device|pinctrl*/ERR_PTR|获取 handle|
|`pinctrl_lookup_state(p,"default")`|handle、状态名|state*/ERR_PTR|找 state|
|`pinctrl_select_state(p,state)`|handle、state|0/负errno|切换状态|

## 和 GPIO 的关系

```text
pinctrl
  ↓
把 PIN 复用成 GPIO
  ↓
GPIO subsystem
  ↓
输入/输出/读写
```

或者：

```text
pinctrl
  ↓
把 PIN 复用成 I2C_SCL/SDA
  ↓
I2C controller
```

## 为什么硬件 I2C 只能用部分引脚

SoC 内部复用网络是固定设计的：

```text
I2C1_SCL signal
      ↓
内部 mux
      ↓
PIN_A / PIN_C
```

如果 PIN_B 的复用选项里没有 `I2C1_SCL`，设备树不能把它强行变成 I2C1_SCL。

软件 bit-bang I2C 是另一回事，它可以用普通 GPIO 模拟时序。
