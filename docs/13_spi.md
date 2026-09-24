# 13. SPI

## 分层

```text
SPI device driver
      ↓
    SPI Core
      ↓
SPI controller driver
      ↓
SoC SPI controller
      ↓
SCLK/MOSI/MISO/CS
      ↓
device
```

## 设备树

```dts
&ecspi1 {
    status = "okay";

    dev@0 {
        compatible = "demo,my-spi";
        reg = <0>;
        spi-max-frequency = <10000000>;
    };
};
```

`reg=<0>` 通常表示 chip-select 编号。

## spi_driver

```c
static struct spi_driver my_driver = {
    .driver = {
        .name = "my_spi",
        .of_match_table = my_of_match,
    },
    .probe = my_probe,
    .remove = my_remove,
};
```

注册：

```c
spi_register_driver(&my_driver);
```

注销：

```c
spi_unregister_driver(&my_driver);
```

## spi_device

probe：

```c
static int my_probe(struct spi_device *spi)
{
    ...
}
```

常用字段：

```c
spi->max_speed_hz
spi->mode
spi->bits_per_word
spi->chip_select
spi->dev
```

参数改完后：

```c
ret = spi_setup(spi);
```

## 常用同步 API

|API|用途|
|---|---|
|`spi_write()`|只写|
|`spi_read()`|只读|
|`spi_write_then_read()`|先写命令/地址再读|
|`spi_sync()`|复杂同步事务|

这些同步接口可以睡眠，不应放 hard IRQ。

## spi_message / spi_transfer

```c
struct spi_message msg;
struct spi_transfer xfer = {
    .tx_buf = tx,
    .rx_buf = rx,
    .len = len,
};

spi_message_init(&msg);
spi_message_add_tail(&xfer, &msg);

ret = spi_sync(spi, &msg);
```

一个 message 可以包含多个 transfer。

## 私有数据

```c
spi_set_drvdata(spi, d);
d = spi_get_drvdata(spi);
```

## SPI 和 DMA

高吞吐 SPI controller 驱动可能内部自动使用 DMA。

上层 spi_device driver 一般仍只调用 SPI Core API。
