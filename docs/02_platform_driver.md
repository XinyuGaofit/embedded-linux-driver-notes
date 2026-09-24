# 02. Platform Driver

## Platform 是什么

`platform` 不是一种通信协议。

它主要用于描述：

- SoC 内部控制器
- 板级固定设备
- 没有 USB/PCI 那种自动枚举机制的设备

典型：

```text
GPIO Controller
I2C Controller
SPI Controller
UART
PWM
LCD Controller
DMA Controller
```

外部 I2C 传感器通常是 `i2c_driver`；SoC 内部的 I2C 控制器本身则经常是 `platform_driver`。

## 设备树和 platform_device

```dts
mydev@20000000 {
    compatible = "demo,mydev";
    reg = <0x02000000 0x1000>;
    interrupts = <0 32 IRQ_TYPE_LEVEL_HIGH>;
    status = "okay";
};
```

driver：

```c
static const struct of_device_id my_of_match[] = {
    { .compatible = "demo,mydev" },
    { }
};

MODULE_DEVICE_TABLE(of, my_of_match);
```

## platform_driver

```c
static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "mydev",
        .of_match_table = my_of_match,
    },
};
```

注册：

```c
platform_driver_register(&my_driver);
```

注销：

```c
platform_driver_unregister(&my_driver);
```

## 常用 API

|API|关键输入|返回|作用|
|---|---|---|---|
|`platform_driver_register(&drv)`|driver|0/负errno|注册|
|`platform_driver_unregister(&drv)`|driver|void|注销|
|`platform_set_drvdata(pdev,data)`|pdev、私有数据|void|保存每设备状态|
|`platform_get_drvdata(pdev)`|pdev|void *|读取私有状态|
|`platform_get_resource(pdev,type,index)`|资源类型、索引|resource*/NULL|获取 MMIO 等资源|
|`platform_get_irq(pdev,index)`|IRQ索引|IRQ号/负errno|获取 IRQ|

## probe 常见结构

```c
static int my_probe(struct platform_device *pdev)
{
    struct mydev *d;
    int ret;

    d = devm_kzalloc(&pdev->dev, sizeof(*d), GFP_KERNEL);
    if (!d)
        return -ENOMEM;

    d->dev = &pdev->dev;
    platform_set_drvdata(pdev, d);

    /* 获取 GPIO / clock / reset / regulator / MMIO */
    /* 初始化硬件 */
    /* 注册 IRQ */
    /* 注册字符设备 / Input / V4L2 等 */

    return 0;
}
```

真实顺序要服从硬件手册。有些芯片要求先上电，再开时钟，再解除 reset。

## of_node

```c
struct device_node *np = pdev->dev.of_node;
```

常用：

|API|输入|输出|
|---|---|---|
|`of_property_read_u32(np,name,&val)`|节点、属性名|u32|
|`of_property_read_string(np,name,&str)`|节点、属性名|字符串指针|
|`of_property_read_bool(np,name)`|节点、属性名|bool|
|`of_get_named_gpio_flags(np,name,index,&flags)`|GPIO属性|GPIO号/负错误|

## remove

Linux 4.9 常见：

```c
static int my_remove(struct platform_device *pdev)
{
    struct mydev *d = platform_get_drvdata(pdev);

    /* 停 DMA / IRQ / work */
    /* 注销上层接口 */
    /* 关 clock / regulator */

    return 0;
}
```

一般按照 probe 中资源启用的反方向收尾。

## 一个 driver 对多个设备

```dts
dev0 { compatible = "demo,mydev"; };
dev1 { compatible = "demo,mydev"; };
```

结果可以是：

```text
probe(pdev0) → private_data0
probe(pdev1) → private_data1
```

代码共享，数据分开。
