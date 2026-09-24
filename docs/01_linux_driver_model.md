# 01. Linux Driver Model

## device、driver、bus

Linux 驱动模型里经常反复出现三个对象：

```text
device
driver
bus
```

可以先按这个关系理解：

```text
device = 一个具体硬件实例
driver = 能操作这一类设备的代码
bus    = 组织并匹配 device 和 driver
```

例如 I2C：

```text
MPU6050 @ 0x68          mpu6050_driver
      \                    /
       \                  /
             I2C bus
                |
             匹配成功
                |
              probe()
```

一个 driver 可以匹配多个 device。函数代码只有一份，但每个设备实例都需要自己的状态。

## probe 和 module_init 不是一回事

模块加载函数表示“这个 driver 进入内核”。

```c
module_init(my_init);
```

常见的 `my_init()` 只是注册 driver：

```c
static int __init my_init(void)
{
    return platform_driver_register(&my_driver);
}
```

真正和某个具体设备相关的初始化通常在：

```c
static int my_probe(struct platform_device *pdev)
{
    ...
}
```

典型执行链：

```text
insmod
  ↓
module_init
  ↓
注册 driver
  ↓
bus 尝试匹配已有 device
  ↓
匹配成功
  ↓
probe(device)
```

所以 `probe()` 不是 `open()`，也不是模块入口函数。

## struct device

很多总线专用对象内部都嵌了 `struct device`：

```c
struct platform_device {
    ...
    struct device dev;
};

struct i2c_client {
    ...
    struct device dev;
};

struct spi_device {
    ...
    struct device dev;
};
```

因此很多资源 API 都接收：

```c
struct device *dev
```

例如：

```c
devm_clk_get(&pdev->dev, NULL);
devm_regulator_get(&client->dev, "vdd");
devm_gpiod_get(&spi->dev, "reset", GPIOD_OUT_LOW);
```

## 每设备私有数据

不要把所有设备状态都写成全局变量。

常见做法：

```c
struct mydev {
    struct device *dev;
    int irq;
    struct mutex lock;
    ...
};
```

probe 中：

```c
struct mydev *d;

d = devm_kzalloc(&pdev->dev, sizeof(*d), GFP_KERNEL);
if (!d)
    return -ENOMEM;

d->dev = &pdev->dev;
platform_set_drvdata(pdev, d);
```

remove 中：

```c
d = platform_get_drvdata(pdev);
```

常见保存/读取函数：

|类型|保存|读取|
|---|---|---|
|platform|`platform_set_drvdata()`|`platform_get_drvdata()`|
|I2C|`i2c_set_clientdata()`|`i2c_get_clientdata()`|
|SPI|`spi_set_drvdata()`|`spi_get_drvdata()`|
|USB interface|`usb_set_intfdata()`|`usb_get_intfdata()`|

## devm 资源管理

`devm_*` 表示 device-managed resource：

```c
d = devm_kzalloc(dev, sizeof(*d), GFP_KERNEL);
clk = devm_clk_get(dev, NULL);
base = devm_ioremap_resource(dev, res);
```

这些对象的回收会跟随 `struct device` 生命周期。

但“对象自动释放”不等于“硬件状态自动恢复”。

例如：

```c
clk_prepare_enable(clk);
regulator_enable(vdd);
```

即便对象是 devm 获取，remove/error path 仍通常需要：

```c
clk_disable_unprepare(clk);
regulator_disable(vdd);
```

## 常用 API

|API|输入|返回|作用|
|---|---|---|---|
|`devm_kzalloc(dev,size,GFP_KERNEL)`|device、大小、flag|指针/NULL|分配每设备私有数据|
|`dev_set_drvdata(dev,data)`|device、私有指针|void|保存私有数据|
|`dev_get_drvdata(dev)`|device|void *|读取私有数据|
|`dev_info(dev,fmt,...)`|device、格式串|void|打印设备日志|
|`dev_err(dev,fmt,...)`|device、格式串|void|打印错误日志|
|`container_of(ptr,type,member)`|成员指针、外层类型、成员名|外层对象指针|从成员找到所属对象|

## container_of

```c
struct mydev {
    struct cdev cdev;
    int value;
};
```

手里只有 `struct cdev *`：

```c
struct mydev *d;

d = container_of(cdev, struct mydev, cdev);
```

字符设备 `open()` 常这样写：

```c
d = container_of(inode->i_cdev, struct mydev, cdev);
file->private_data = d;
```

之后 `read/write/ioctl/poll` 都可以：

```c
d = file->private_data;
```

## 一条完整主线

```text
Device Tree
   ↓
生成 / 注册 device
   ↓
driver 注册
   ↓
bus.match()
   ↓
probe(device)
   ↓
分配 private_data
   ↓
获取 GPIO/IRQ/clock/MMIO...
   ↓
初始化硬件
   ↓
注册上层接口
   ↓
用户空间
```
