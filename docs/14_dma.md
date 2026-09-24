# 14. DMA

DMA 常见有两层：

```text
DMA mapping API
    CPU buffer ↔ device DMA address

DMAEngine
    使用 SoC DMA controller 搬数据
```

## coherent buffer

```c
void *cpu_addr;
dma_addr_t dma_addr;

cpu_addr = dma_alloc_coherent(dev,
                              size,
                              &dma_addr,
                              GFP_KERNEL);
```

得到：

```text
cpu_addr → CPU 使用
dma_addr → 硬件/DMA 使用
```

释放：

```c
dma_free_coherent(dev, size, cpu_addr, dma_addr);
```

适合 descriptor ring、长期共享控制结构等。

## streaming mapping

```c
dma_addr = dma_map_single(dev,
                          buf,
                          len,
                          DMA_TO_DEVICE);
```

检查：

```c
if (dma_mapping_error(dev, dma_addr))
    ...
```

完成：

```c
dma_unmap_single(dev,
                 dma_addr,
                 len,
                 DMA_TO_DEVICE);
```

direction：

```text
DMA_TO_DEVICE
DMA_FROM_DEVICE
DMA_BIDIRECTIONAL
```

## DMA 地址不是 CPU 指针

`dma_addr_t` 不是普通 CPU 虚拟地址，不能直接解引用。

## DMAEngine

4.9 常见请求：

```c
chan = dma_request_slave_channel(dev, "rx");
if (!chan)
    return -ENODEV;
```

配置：

```c
struct dma_slave_config cfg = { ... };
ret = dmaengine_slave_config(chan, &cfg);
```

准备 descriptor：

```c
desc = dmaengine_prep_slave_single(chan,
                                   dma_addr,
                                   len,
                                   DMA_DEV_TO_MEM,
                                   DMA_PREP_INTERRUPT);
```

callback：

```c
desc->callback = dma_done;
desc->callback_param = d;
```

提交并启动：

```c
cookie = dmaengine_submit(desc);
dma_async_issue_pending(chan);
```

## 常用 API

|API|作用|
|---|---|
|`dma_alloc_coherent()`|一致性缓冲|
|`dma_free_coherent()`|释放|
|`dma_map_single()`|映射普通buffer|
|`dma_unmap_single()`|结束映射|
|`dma_mapping_error()`|检查错误|
|`dma_request_slave_channel()`|获取channel|
|`dmaengine_slave_config()`|配置外设DMA|
|`dmaengine_prep_slave_single()`|准备传输|
|`dmaengine_submit()`|提交|
|`dma_async_issue_pending()`|启动|
|`dmaengine_terminate_all()`|终止|
|`dma_release_channel()`|释放channel|

## DMA callback + completion

```c
static void dma_done(void *arg)
{
    struct mydev *d = arg;
    complete(&d->dma_done);
}
```

等待：

```c
reinit_completion(&d->dma_done);

if (!wait_for_completion_timeout(&d->dma_done,
                                 msecs_to_jiffies(1000)))
    return -ETIMEDOUT;
```

## 常见错误

- 混淆 CPU 虚拟地址、物理地址、DMA 地址
- streaming mapping 后忘记 unmap
- direction 填错
- buffer 生命周期早于 DMA 完成
- remove 时 DMA 仍访问已释放内存
