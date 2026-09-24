# Embedded Linux Driver Notes

基于 **Linux 4.9.88 / i.MX6ULL** 整理的一套驱动学习笔记。

这套笔记不是按内核源码目录硬列 API，而是按实际写驱动时常见的顺序整理：

```text
设备树描述硬件
        ↓
内核创建设备对象
        ↓
driver 与 device 匹配
        ↓
probe()
        ↓
获取 GPIO / IRQ / clock / reset / regulator / MMIO
        ↓
初始化硬件
        ↓
接入字符设备 / Input / V4L2 / ALSA 等上层框架
        ↓
用户空间使用
```

当前内容以 Linux 4.9.88 为主。新内核接口变化较大的地方会单独提醒。

## 目录

|章节|内容|
|---|---|
|[01 Linux Driver Model](docs/01_linux_driver_model.md)|device、driver、bus、probe、私有数据|
|[02 Platform Driver](docs/02_platform_driver.md)|platform_device / platform_driver / Device Tree|
|[03 Character Device](docs/03_character_device.md)|设备号、cdev、class、device、file_operations|
|[04 pinctrl](docs/04_pinctrl.md)|引脚复用、default/sleep state|
|[05 GPIO](docs/05_gpio.md)|整数 GPIO API 与 gpiod API|
|[06 IRQ](docs/06_irq.md)|request_irq、threaded IRQ、上下文限制|
|[07 mutex / spinlock / atomic](docs/07_mutex_spinlock_atomic.md)|驱动并发与锁|
|[08 waitqueue / completion](docs/08_waitqueue_completion.md)|阻塞、唤醒、事件完成|
|[09 workqueue / kthread / timer](docs/09_workqueue_kthread_timer.md)|延后执行和后台任务|
|[10 clock / reset / regulator](docs/10_clock_reset_regulator.md)|时钟、复位、电源|
|[11 MMIO](docs/11_mmio.md)|reg、resource、ioremap、readl/writel|
|[12 I2C](docs/12_i2c.md)|i2c_client、i2c_driver、i2c_transfer|
|[13 SPI](docs/13_spi.md)|spi_device、spi_driver、transfer/message|
|[14 DMA](docs/14_dma.md)|DMA mapping 与 DMAEngine|
|[15 Input](docs/15_input.md)|按键、触摸、eventX|
|[16 V4L2](docs/16_v4l2.md)|video_device、ioctl、vb2 基本结构|
|[17 ALSA](docs/17_alsa.md)|PCM、ASoC、DAI、Codec 基本结构|

## 建议学习顺序

前 8 章先打通。它们基本覆盖普通字符设备、GPIO 按键、I2C 传感器这类驱动的核心骨架。

随后按设备类型继续：

```text
SoC 内部控制器 → platform + MMIO + clock/reset
GPIO 按键      → pinctrl + GPIO + IRQ + Input
I2C 传感器     → I2C + IRQ + regulator
SPI 外设       → SPI + GPIO/IRQ
视频设备       → V4L2
音频设备       → ALSA / ASoC
```

## 怎么使用这套笔记

不建议背函数名。写驱动时先确认：

1. 设备挂在哪种总线或框架上。
2. probe 里需要获取哪些硬件资源。
3. 哪些代码运行在进程上下文，哪些代码可能运行在中断上下文。
4. 最终通过什么方式把功能交给用户空间。

确定这些以后，再查对应章节的 API 表。

## 环境

```text
Kernel: Linux 4.9.88
Board/SoC: i.MX6ULL
Language: C
Build style: out-of-tree kernel module / BSP kernel tree
```

部分 BSP 会带厂商补丁，同一个 API 在主线 4.9 和厂商 4.9 中可能有差异。实际开发时以当前内核树头文件和同目录已有驱动写法为准。
