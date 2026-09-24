# 06. IRQ

## 中断路径

```text
硬件事件
   ↓
Interrupt Controller
   ↓
Linux IRQ subsystem
   ↓
irq_handler
   ↓
清状态 / 更新状态 / 唤醒 / 安排后续工作
```

## request_irq

```c
ret = request_irq(irq,
                  my_handler,
                  flags,
                  "mydev",
                  d);
```

|参数|含义|
|---|---|
|`irq`|Linux IRQ号|
|`handler`|hard IRQ handler|
|`flags`|触发、共享等标志|
|`name`|`/proc/interrupts` 中的名字|
|`dev_id`|私有指针，handler 会收到|

```c
static irqreturn_t my_handler(int irq, void *dev_id)
{
    struct mydev *d = dev_id;

    ...
    return IRQ_HANDLED;
}
```

## 常见 flags

```text
IRQF_TRIGGER_RISING
IRQF_TRIGGER_FALLING
IRQF_TRIGGER_HIGH
IRQF_TRIGGER_LOW
IRQF_SHARED
IRQF_ONESHOT
```

触发类型最好和设备树/硬件设计一致。

## hard IRQ 不能睡

不要在 hard IRQ 中：

```text
mutex_lock()
msleep()
copy_to_user()
copy_from_user()
同步 I2C/SPI
*_cansleep GPIO
```

常见操作：

```text
读/清状态寄存器
atomic
短 spinlock
complete()
wake_up_interruptible()
schedule_work()
```

## threaded IRQ

```c
request_threaded_irq(irq,
                     top_handler,
                     thread_handler,
                     IRQF_ONESHOT,
                     "mydev",
                     d);
```

```c
static irqreturn_t top_handler(int irq, void *data)
{
    return IRQ_WAKE_THREAD;
}
```

```c
static irqreturn_t thread_handler(int irq, void *data)
{
    /* 线程上下文，可睡眠 */
    return IRQ_HANDLED;
}
```

I2C 触摸屏、传感器一类设备很常见。

## devm_request_irq

```c
devm_request_irq(dev, irq, handler, flags, name, d);
```

解绑时自动注销，但 remove 中仍要先停掉依赖 IRQ 的 DMA/work/buffer，防止 use-after-free。

## 其他 API

|API|作用|注意|
|---|---|---|
|`free_irq(irq,dev_id)`|注销|dev_id要对应|
|`disable_irq(irq)`|禁用|可能等待 handler 完成|
|`disable_irq_nosync(irq)`|禁用但不等待|竞态风险更高|
|`enable_irq(irq)`|恢复|和disable配合|
|`synchronize_irq(irq)`|等待 handler 结束|释放相关资源前有用|

## 调试

```bash
cat /proc/interrupts
```

可以检查 IRQ 号、CPU 计数、handler 名字等。
