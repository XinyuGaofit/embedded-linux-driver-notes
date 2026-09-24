# 05. GPIO

## GPIO framework 的位置

```text
device driver
     ↓
GPIO framework
     ↓
GPIO controller driver
     ↓
SoC GPIO registers
```

设备驱动因此不需要知道每颗 SoC 的 GPIO 寄存器布局。

## Linux 4.9 常见整数 GPIO API

|API|参数|返回|作用|
|---|---|---|---|
|`gpio_request(gpio,label)`|GPIO号、标签|0/负errno|申请|
|`gpio_free(gpio)`|GPIO号|void|释放|
|`gpio_direction_input(gpio)`|GPIO号|0/负errno|输入|
|`gpio_direction_output(gpio,val)`|GPIO号、初值|0/负errno|输出|
|`gpio_get_value(gpio)`|GPIO号|0/1|读|
|`gpio_set_value(gpio,val)`|GPIO号、0/1|void|写|
|`gpio_get_value_cansleep(gpio)`|GPIO号|值|可睡眠读取|
|`gpio_set_value_cansleep(gpio,val)`|GPIO号、值|void|可睡眠写入|
|`gpio_to_irq(gpio)`|GPIO号|IRQ号/负错误|转IRQ|

## descriptor API

```c
struct gpio_desc *reset_gpio;
```

获取：

```c
reset_gpio = devm_gpiod_get(dev, "reset", GPIOD_OUT_LOW);
if (IS_ERR(reset_gpio))
    return PTR_ERR(reset_gpio);
```

对应 DT：

```dts
reset-gpios = <&gpio1 3 GPIO_ACTIVE_LOW>;
```

`con_id = "reset"` 对应 `reset-gpios`。

## gpiod 常用 API

|API|输入|返回|作用|
|---|---|---|---|
|`devm_gpiod_get(dev,id,flags)`|device、连接名、flag|desc/ERR_PTR|获取GPIO|
|`gpiod_direction_input(desc)`|desc|0/负errno|输入|
|`gpiod_direction_output(desc,val)`|desc、逻辑值|0/负errno|输出|
|`gpiod_get_value(desc)`|desc|0/1/负错误|不可睡眠路径读取|
|`gpiod_set_value(desc,val)`|desc、值|void|不可睡眠路径写|
|`gpiod_get_value_cansleep(desc)`|desc|0/1/负错误|可睡眠读取|
|`gpiod_set_value_cansleep(desc,val)`|desc、值|void|可睡眠写|
|`gpiod_to_irq(desc)`|desc|IRQ号/负错误|转IRQ|

## active-low

```dts
reset-gpios = <&gpio1 3 GPIO_ACTIVE_LOW>;
```

descriptor API 会处理逻辑极性。

```c
gpiod_set_value_cansleep(reset_gpio, 1);
```

这里的 `1` 表示逻辑 active。底层可转换为物理低电平。

## GPIO + IRQ

```c
irq = gpiod_to_irq(d->key_gpio);
if (irq < 0)
    return irq;

ret = devm_request_irq(dev, irq, handler,
                       IRQF_TRIGGER_FALLING,
                       "my-key", d);
```

如果 DT 本身已经描述 `interrupts`，platform 设备通常更适合：

```c
irq = platform_get_irq(pdev, 0);
```

## 上下文

带 `_cansleep` 的 API 可能睡眠：

```text
process context  → 可以
hard IRQ         → 不要调用可能睡眠的 GPIO API
```

GPIO controller 如果本身挂在 I2C/SPI 上，读一个 GPIO 也可能需要总线事务。
