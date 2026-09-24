# 11. MMIO

## 基本路径

设备树：

```dts
mydev@20000000 {
    reg = <0x02000000 0x1000>;
};
```

驱动：

```text
DT reg
  ↓
resource
  ↓
ioremap
  ↓
void __iomem *
  ↓
readl/writel
```

## 获取 resource

```c
struct resource *res;

res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
if (!res)
    return -ENODEV;
```

## 映射

```c
d->base = devm_ioremap_resource(&pdev->dev, res);
if (IS_ERR(d->base))
    return PTR_ERR(d->base);
```

## 访问寄存器

```c
u32 val;

val = readl(d->base + REG_STATUS);
writel(val | BIT(3), d->base + REG_CTRL);
```

|宽度|读|写|
|---|---|---|
|8bit|`readb()`|`writeb()`|
|16bit|`readw()`|`writew()`|
|32bit|`readl()`|`writel()`|

不要把 `__iomem` 当普通 RAM 指针直接解引用。

## resource_size

```c
size = resource_size(res);
```

用于边界检查和调试。

## 位操作

```c
#define CTRL_ENABLE BIT(0)
#define CTRL_MODE_MASK GENMASK(3, 1)
```

```c
val = readl(base + REG_CTRL);
val |= CTRL_ENABLE;
writel(val, base + REG_CTRL);
```

如果寄存器有 W1C、自清零、reserved bit 等特殊语义，不能机械 read-modify-write。

## 并发

多个执行路径同时改同一寄存器时，要考虑：

- 是否需要 spinlock
- 是否有硬件 set/clear 寄存器
- 是否只有一个上下文应该操作它
