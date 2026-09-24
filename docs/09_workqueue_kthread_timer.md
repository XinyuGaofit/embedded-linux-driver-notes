# 09. workqueue / kthread / timer

## workqueue

很多异步任务只需要“稍后在线程上下文执行”。

```text
IRQ
 ↓
schedule_work()
 ↓
worker thread
 ↓
my_work()
```

定义：

```c
struct work_struct work;
```

初始化：

```c
INIT_WORK(&d->work, my_work);
```

安排：

```c
schedule_work(&d->work);
```

回调：

```c
static void my_work(struct work_struct *work)
{
    struct mydev *d;

    d = container_of(work, struct mydev, work);

    /* 可做允许睡眠的工作 */
}
```

remove：

```c
cancel_work_sync(&d->work);
```

## delayed_work

```c
struct delayed_work dwork;

INIT_DELAYED_WORK(&d->dwork, my_work);

schedule_delayed_work(&d->dwork,
                      msecs_to_jiffies(100));

cancel_delayed_work_sync(&d->dwork);
```

## kthread

长期独立循环：

```c
static int my_thread(void *data)
{
    struct mydev *d = data;

    while (!kthread_should_stop()) {
        ...
        msleep(100);
    }

    return 0;
}
```

启动：

```c
d->task = kthread_run(my_thread, d, "mydev");
if (IS_ERR(d->task))
    return PTR_ERR(d->task);
```

停止：

```c
kthread_stop(d->task);
```

## timer

Linux 4.9 常见：

```c
struct timer_list timer;

setup_timer(&d->timer, my_timer_fn, (unsigned long)d);
```

启动/重设：

```c
mod_timer(&d->timer,
          jiffies + msecs_to_jiffies(100));
```

删除：

```c
del_timer_sync(&d->timer);
```

timer callback 在原子/软中断环境，不能睡眠。

如果超时后需要 I2C/SPI 等操作：

```text
timer
  ↓
schedule_work
  ↓
workqueue
```

## 选择

|需求|建议|
|---|---|
|IRQ 后慢处理|workqueue / threaded IRQ|
|延迟执行|delayed_work|
|长期后台循环|kthread|
|原子上下文定时|timer|
