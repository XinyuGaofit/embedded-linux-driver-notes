# 07. mutex / spinlock / atomic

## 什么时候考虑并发

驱动中的并发来源包括：

```text
read/write/ioctl
IRQ handler
workqueue
threaded IRQ
timer
DMA callback
kthread
另一个 CPU
另一个用户进程
```

如果它们可能同时访问同一份可变状态，就需要检查竞态。

## mutex

```c
struct mutex lock;

mutex_init(&d->lock);

mutex_lock(&d->lock);
/* critical section */
mutex_unlock(&d->lock);
```

适合：

```text
read/write/ioctl
workqueue
kthread
threaded IRQ
```

hard IRQ 中不能用。

## spinlock

```c
spinlock_t lock;
spin_lock_init(&d->lock);
```

进程上下文和 IRQ 共享数据：

```c
unsigned long flags;

spin_lock_irqsave(&d->lock, flags);
/* very short critical section */
spin_unlock_irqrestore(&d->lock, flags);
```

持 spinlock 时不要：

```text
msleep
mutex_lock
copy_to_user
I2C/SPI sync
GFP_KERNEL allocation
```

## atomic

```c
atomic_t ready;

atomic_set(&ready, 0);
atomic_read(&ready);
atomic_inc(&ready);
atomic_dec(&ready);
atomic_xchg(&ready, 0);
```

适合简单计数、flag。

多个字段需要一起保持一致时，atomic 不能代替锁。

## 对照表

|机制|会睡眠吗|典型场景|
|---|---|---|
|mutex|可能|进程上下文共享资源|
|spinlock|否|IRQ 和普通上下文共享短数据|
|atomic|否|简单计数/flag|

## 队列例子

```text
IRQ producer
     ↓
   queue
     ↑
user read consumer
```

双方都改链表：

```c
spin_lock_irqsave(&d->lock, flags);
/* list_add/list_del */
spin_unlock_irqrestore(&d->lock, flags);
```

如果所有队列操作都在线程上下文，则可以考虑 mutex。
