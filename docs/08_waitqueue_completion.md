# 08. wait_queue / completion

## wait_queue

适合“条件没满足就睡，条件满足再继续”。

```text
read()
  ↓
没有数据
  ↓
wait_event_interruptible()
  ↓
睡眠

IRQ
  ↓
data_ready = 1
  ↓
wake_up_interruptible()

read 被唤醒
  ↓
重新检查条件
```

初始化：

```c
wait_queue_head_t wq;
init_waitqueue_head(&wq);
```

等待：

```c
ret = wait_event_interruptible(d->wq,
                               atomic_read(&d->data_ready));
```

唤醒：

```c
wake_up_interruptible(&d->wq);
```

|API|返回|说明|
|---|---|---|
|`wait_event_interruptible(wq,cond)`|0/负值|可被信号打断|
|`wait_event_interruptible_timeout(...)`|>0/0/<0|成功/超时/信号|
|`wake_up(&wq)`|void|唤醒|
|`wake_up_interruptible(&wq)`|void|唤醒interruptible等待者|

## wait_queue + poll

```c
static unsigned int my_poll(struct file *file, poll_table *wait)
{
    struct mydev *d = file->private_data;
    unsigned int mask = 0;

    poll_wait(file, &d->wq, wait);

    if (atomic_read(&d->data_ready))
        mask |= POLLIN | POLLRDNORM;

    return mask;
}
```

IRQ：

```c
atomic_set(&d->data_ready, 1);
wake_up_interruptible(&d->wq);
```

## completion

适合一次操作完成通知：

```text
发起命令/DMA
    ↓
等待完成
    ↓
IRQ/callback
    ↓
complete()
```

```c
struct completion done;

init_completion(&done);
wait_for_completion(&done);
complete(&done);
```

常用 API：

|API|作用|
|---|---|
|`init_completion()`|初始化|
|`reinit_completion()`|重新置未完成|
|`wait_for_completion()`|等待|
|`wait_for_completion_timeout()`|带超时|
|`wait_for_completion_interruptible()`|可被信号打断|
|`complete()`|完成一次|
|`complete_all()`|唤醒所有|

## 怎么选

```text
等待一个条件持续变化 → wait_queue
等待某次操作完成     → completion
```
