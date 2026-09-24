# Embedded Linux 驱动学习笔记：常用子系统与 API（Linux 4.9 / i.MX6ULL）

> 这份笔记按“实际写驱动时会遇到的顺序”整理，而不是按内核源码目录罗列 API。目标是：看到一个硬件需求时，知道该找哪个子系统、先拿什么对象、函数参数填什么、返回值怎么判断，以及哪些函数能睡眠、哪些不能放进中断。
>
> 当前环境以 **Linux 4.9.88 / i.MX6ULL** 为主。新内核接口有变化的地方会单独提醒。

---

## 目录

- [0. 先搞清楚：为什么 Linux 要有“子系统”](#0-先搞清楚为什么-linux-要有子系统)
- [1. platform 驱动、设备树和私有数据](#1-platform-驱动设备树和私有数据)
- [2. 字符设备：设备号、cdev、class、device、file_operations](#2-字符设备设备号cdevclassdevicefile_operations)
- [3. pinctrl：引脚复用到底在管什么](#3-pinctrl引脚复用到底在管什么)
- [4. GPIO 子系统](#4-gpio-子系统)
- [5. IRQ 中断子系统](#5-irq-中断子系统)
- [6. 并发：mutex、spinlock、atomic](#6-并发mutexspinlockatomic)
- [7. 等待和通知：wait_queue、completion](#7-等待和通知wait_queuecompletion)
- [8. 延后处理和内核线程：workqueue、threaded IRQ、kthread、timer](#8-延后处理和内核线程workqueuethreaded-irqkthreadtimer)
- [9. clock / reset / regulator](#9-clock--reset--regulator)
- [10. MMIO：寄存器映射与 readl/writel](#10-mmio寄存器映射与-readlwritel)
- [11. I2C 子系统](#11-i2c-子系统)
- [12. SPI 子系统](#12-spi-子系统)
- [13. PWM 子系统](#13-pwm-子系统)
- [14. DMA：mapping API 和 DMAEngine](#14-dmamapping-api-和-dmaengine)
- [15. Input 子系统](#15-input-子系统)
- [16. USB 子系统](#16-usb-子系统)
- [17. V4L2 / ALSA / DRM / Network / Block / MTD 怎么看](#17-v4l2--alsa--drm--network--block--mtd-怎么看)
- [18. devm、ERR_PTR 和错误回滚](#18-devmerr_ptr-和错误回滚)
- [19. 一个完整 probe 应该怎么看](#19-一个完整-probe-应该怎么看)
- [20. 写驱动时怎么快速判断该用什么](#20-写驱动时怎么快速判断该用什么)
- [21. 建议的学习顺序](#21-建议的学习顺序)

---

# 0. 先搞清楚：为什么 Linux 要有“子系统”

如果没有子系统，每个驱动都要直接碰 SoC 寄存器。

比如一个按键驱动，如果什么框架都不用，可能要自己做：

```text
找到 IOMUX 寄存器
  ↓
把 PIN 切成 GPIO
  ↓
找到 GPIO 方向寄存器
  ↓
设置输入
  ↓
找到 GPIO 中断寄存器
  ↓
配置边沿触发
  ↓
找到中断控制器
  ↓
处理 IRQ
```

这当然能写，但问题很明显：换一个 SoC，寄存器地址、位定义、控制器结构都变了，上层按键驱动几乎要重写。

Linux 的做法是把重复问题拆出来：

```text
你的设备驱动
   │
   ├── pinctrl：PIN 复用和电气属性
   ├── GPIO：输入、输出、读写电平
   ├── IRQ：中断号、handler、屏蔽/同步
   ├── clock：时钟
   ├── reset：复位
   ├── regulator：电源
   ├── DMA：搬数据
   └── I2C/SPI/USB：总线传输
        │
        ↓
具体控制器驱动
        │
        ↓
SoC 寄存器 / 硬件
```

所以写设备驱动时，通常不应该先问“这个寄存器是多少”，而应该先问：

> 这个资源是不是已经有内核子系统管理？如果有，优先走子系统 API。

子系统最大的价值不是“代码好看”，而是把**设备功能**和**SoC 实现细节**拆开。

---

# 1. platform 驱动、设备树和私有数据

## 1.1 platform 是什么

`platform` 不是一种通信协议。它更像 Linux 对“板级 / SoC 内部设备”的一种设备模型。

常见的 platform 设备包括：

```text
GPIO 控制器
I2C 控制器
SPI 控制器
UART 控制器
PWM 控制器
LCD 控制器
DMA 控制器
以及一些没有 I2C/SPI/USB 这类标准总线的板级设备
```

典型关系：

```text
Device Tree
   ↓
platform_device
   ↓ compatible 匹配
platform_driver
   ↓
probe(pdev)
```

## 1.2 `struct platform_driver`

最常见骨架：

```c
static int my_probe(struct platform_device *pdev)
{
    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    return 0;
}

static const struct of_device_id my_of_match[] = {
    { .compatible = "demo,my-device" },
    { }
};
MODULE_DEVICE_TABLE(of, my_of_match);

static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name           = "my_device",
        .of_match_table = my_of_match,
    },
};
```

注册和注销：

```c
platform_driver_register(&my_driver);
platform_driver_unregister(&my_driver);
```

| API | 输入 | 返回 | 作用 |
|---|---|---|---|
| `platform_driver_register(&drv)` | `struct platform_driver *` | `0` 成功，负 errno 失败 | 把驱动挂到 platform bus |
| `platform_driver_unregister(&drv)` | 同一个 driver | `void` | 注销驱动，已绑定设备会走 remove |
| `.probe(pdev)` | 内核传入的具体设备 | `int` | 初始化这一个设备 |
| `.remove(pdev)` | 这一个具体设备 | Linux 4.9 常见 `int` | 释放这一个设备的资源 |

注意：`probe()` 不是用户 `open()` 触发的。它发生在**设备和驱动匹配成功之后**。

## 1.3 `pdev->dev.of_node`

设备树节点入口：

```c
struct device_node *np = pdev->dev.of_node;
```

之后很多 OF API 都从这个 `np` 开始。

例如：

```c
u32 debounce_ms;

ret = of_property_read_u32(np, "debounce-ms", &debounce_ms);
if (ret)
    return ret;
```

设备树：

```dts
mykey {
    compatible = "demo,my-key";
    debounce-ms = <10>;
};
```

常用 API：

| API | 关键参数 | 输出 / 返回 |
|---|---|---|
| `of_property_read_u32(np, name, &val)` | 节点、属性名、输出变量地址 | `0` 成功 |
| `of_property_read_string(np, name, &str)` | 字符串属性 | `0` 成功，`str` 指向 DT 内部字符串 |
| `of_property_read_bool(np, name)` | 判断属性是否存在 | `bool` |
| `platform_get_irq(pdev, index)` | 第几个 IRQ，0 开始 | IRQ 号或负 errno |

## 1.4 每个设备都应该有自己的私有结构体

同一个驱动可能管理多个设备，代码可以共享，但状态不能都写成全局变量。

```c
struct mydev {
    struct device *dev;
    int irq;
    int state;
    struct mutex lock;
};
```

在 `probe()` 中为当前设备创建一份：

```c
struct mydev *d;

d = devm_kzalloc(&pdev->dev, sizeof(*d), GFP_KERNEL);
if (!d)
    return -ENOMEM;

d->dev = &pdev->dev;
platform_set_drvdata(pdev, d);
```

在 `remove()` 中取回来：

```c
struct mydev *d = platform_get_drvdata(pdev);
```

对应关系：

```text
pdev0 → private data 0
pdev1 → private data 1
pdev2 → private data 2
```

函数只有一份，传进去的对象不同。

---

# 2. 字符设备：设备号、cdev、class、device、file_operations

这一部分要把几个经常混在一起的对象拆开。

```text
设备号 dev_t
   ↓
cdev
   ↓
file_operations
   ↓
驱动回调函数
```

而 `class + device` 是另一条线，主要属于 Linux device model / sysfs / uevent，用来组织设备并方便生成 `/dev` 节点。

## 2.1 申请设备号

```c
dev_t devt;

ret = alloc_chrdev_region(&devt, 0, 1, "mydev");
if (ret)
    return ret;
```

参数：

```c
alloc_chrdev_region(&devt, base_minor, count, name);
```

- `&devt`：输出参数，内核把申请到的 major/minor 写回来
- `base_minor`：从哪个 minor 开始
- `count`：要多少个连续设备号
- `name`：在内核设备号管理中的名字

成功后：

```c
MAJOR(devt)
MINOR(devt)
```

可以得到主次设备号。

**只申请设备号不会自动生成 `/dev/mydev`。**

## 2.2 `cdev_init()` 和 `cdev_add()`

```c
struct cdev cdev;

cdev_init(&cdev, &my_fops);
cdev.owner = THIS_MODULE;

ret = cdev_add(&cdev, devt, 1);
```

这里完成的是：

```text
devt
 ↓
cdev
 ↓
file_operations
```

`cdev_init()` 只是初始化并绑定 `file_operations`；`cdev_add()` 才把它正式注册到字符设备体系里。

## 2.3 `class_create()` 到底干什么

Linux 4.9 常见写法：

```c
struct class *class;

class = class_create(THIS_MODULE, "my_class");
if (IS_ERR(class))
    return PTR_ERR(class);
```

它主要建立一个 class，例如：

```text
/sys/class/my_class/
```

这个 class 不是为了把 `file_operations` 再注册一次，也不是设备号的一部分。

它解决的是**设备模型中的分类和设备对象管理**。

## 2.4 `device_create()`

```c
struct device *device;

device = device_create(class,
                       &pdev->dev,
                       devt,
                       d,
                       "mydev0");
if (IS_ERR(device))
    return PTR_ERR(device);
```

它会在 device model 中创建一个设备对象，并建立 sysfs 信息、发送 uevent。系统中的 `udev`、`mdev` 或 `devtmpfs` 可以据此让 `/dev/mydev0` 出现。

所以这几步的职责不同：

```text
alloc_chrdev_region
    负责：设备号

cdev_init + cdev_add
    负责：设备号 ↔ file_operations

class_create + device_create
    负责：device model / sysfs / uevent / 自动设备节点管理
```

如果不用 `class_create/device_create`，理论上也可以手工：

```bash
mknod /dev/mydev c <major> <minor>
```

## 2.5 `file_operations` 就是普通字符设备暴露给用户态的主要接口表

```c
static const struct file_operations my_fops = {
    .owner          = THIS_MODULE,
    .open           = my_open,
    .release        = my_release,
    .read           = my_read,
    .write          = my_write,
    .poll           = my_poll,
    .unlocked_ioctl = my_ioctl,
    .mmap           = my_mmap,
    .llseek         = no_llseek,
};
```

用户态和驱动回调大致对应：

| 用户态 | `file_operations` |
|---|---|
| `open()` | `.open` |
| `read()` | `.read` |
| `write()` | `.write` |
| `ioctl()` | `.unlocked_ioctl` |
| `poll()/select()/epoll()` | `.poll` |
| `mmap()` | `.mmap` |
| `close()` | 最终对应 `.release` |

## 2.6 `file->private_data`

如果 `cdev` 放在你的私有结构体里：

```c
struct mydev {
    struct cdev cdev;
    int irq;
    ...
};
```

`open()` 可以通过 `inode->i_cdev` 找回整个对象：

```c
static int my_open(struct inode *inode, struct file *file)
{
    struct mydev *d;

    d = container_of(inode->i_cdev, struct mydev, cdev);
    file->private_data = d;

    return nonseekable_open(inode, file);
}
```

之后：

```c
static ssize_t my_read(...)
{
    struct mydev *d = file->private_data;
    ...
}
```

这就是“当前这个 fd 到底对应哪一个具体设备”的关键。

## 2.7 用户内存不能直接当内核内存用

内核 → 用户：

```c
if (copy_to_user(buf, &data, sizeof(data)))
    return -EFAULT;
```

用户 → 内核：

```c
if (copy_from_user(&data, buf, sizeof(data)))
    return -EFAULT;
```

返回值不是 `0/-1` 风格，而是“还有多少字节没有拷贝成功”。因此通常判断是否非 0。

`copy_to_user/copy_from_user` 可能涉及缺页，不能放在 hard IRQ 中，也不要拿着 spinlock 调。

---

# 3. pinctrl：引脚复用到底在管什么

一个物理 PIN 往往有多个功能：

```text
某个 PIN
 ├─ GPIO
 ├─ UART_TX
 ├─ I2C_SCL
 └─ PWM_OUT
```

SoC 内部已经设计好了这些可选连接，`pinctrl` 只是从可选项中选择，并配置电气属性。

它不能把一个本来不支持 I2C 的 PIN “变成” I2C。

## 3.1 pinctrl 管两件事

第一类：**mux**

```text
这个物理 PIN 当前连接到 GPIO、UART、I2C 还是 SPI？
```

第二类：**pin configuration**

```text
上拉 / 下拉
驱动能力
开漏
施密特输入
slew rate
```

具体支持什么，取决于 SoC pinctrl 驱动。

## 3.2 设备树里常见写法

```dts
pinctrl-names = "default", "sleep";
pinctrl-0 = <&pinctrl_mydev_default>;
pinctrl-1 = <&pinctrl_mydev_sleep>;
```

必要时驱动可以手动切：

```c
struct pinctrl *p;
struct pinctrl_state *default_state;

p = devm_pinctrl_get(&pdev->dev);
if (IS_ERR(p))
    return PTR_ERR(p);

default_state = pinctrl_lookup_state(p, "default");
if (IS_ERR(default_state))
    return PTR_ERR(default_state);

ret = pinctrl_select_state(p, default_state);
```

很多普通设备只需要在设备树写好 `default`，不需要自己频繁调 pinctrl API。

可以这样区分：

```text
pinctrl：决定“这个 PIN 是谁”
GPIO：决定“当它已经是 GPIO 后，输入/输出/电平怎么操作”
```

---

# 4. GPIO 子系统

Linux 4.9 BSP 里会同时看到老的整数 GPIO API 和 descriptor API。

## 4.1 老式整数 GPIO API

申请：

```c
ret = gpio_request(gpio, "my_gpio");
if (ret)
    return ret;
```

输入：

```c
ret = gpio_direction_input(gpio);
```

输出：

```c
ret = gpio_direction_output(gpio, 0);
```

读取：

```c
value = gpio_get_value_cansleep(gpio);
```

写入：

```c
gpio_set_value_cansleep(gpio, 1);
```

转 IRQ：

```c
irq = gpio_to_irq(gpio);
```

释放：

```c
gpio_free(gpio);
```

## 4.2 `devm_gpio_request_one()`

Linux 4.9 很实用：

```c
ret = devm_gpio_request_one(&pdev->dev,
                            gpio,
                            GPIOF_IN,
                            "my_key");
```

或者：

```c
GPIOF_OUT_INIT_LOW
GPIOF_OUT_INIT_HIGH
```

`devm_` 版本在设备解绑时自动释放。

## 4.3 descriptor / gpiod API

更现代、语义更完整：

设备树：

```dts
reset-gpios = <&gpio1 3 GPIO_ACTIVE_LOW>;
```

驱动：

```c
struct gpio_desc *reset_gpio;

reset_gpio = devm_gpiod_get(&pdev->dev,
                            "reset",
                            GPIOD_OUT_LOW);
if (IS_ERR(reset_gpio))
    return PTR_ERR(reset_gpio);
```

这里 `"reset"` 会对应 `reset-gpios`。

常用：

```c
gpiod_direction_input(desc);
gpiod_direction_output(desc, value);
gpiod_get_value_cansleep(desc);
gpiod_set_value_cansleep(desc, value);
gpiod_to_irq(desc);
```

descriptor API 对 `GPIO_ACTIVE_LOW` 的逻辑语义处理更自然。

## 4.4 `*_cansleep` 是什么意思

如果 GPIO 控制器本身挂在 I2C/SPI 等慢总线上，读写 GPIO 可能需要睡眠等待总线事务。

所以：

```c
gpiod_get_value_cansleep()
gpiod_set_value_cansleep()
```

不要放进 hard IRQ。

---

# 5. IRQ 中断子系统

## 5.1 `request_irq()`

```c
ret = request_irq(irq,
                  my_irq_handler,
                  IRQF_TRIGGER_FALLING,
                  "mydev",
                  d);
```

参数：

```c
request_irq(irq, handler, flags, name, dev_id)
```

- `irq`：Linux IRQ number
- `handler`：中断函数
- `flags`：触发方式、共享等标志
- `name`：`/proc/interrupts` 等地方显示的名字
- `dev_id`：你自己的指针，IRQ 发生后会原样传回来

handler：

```c
static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct mydev *d = dev_id;

    ...

    return IRQ_HANDLED;
}
```

释放：

```c
free_irq(irq, d);
```

第二个参数要和注册时的 `dev_id` 对应。

## 5.2 hard IRQ 里能做什么

适合：

```text
读中断状态
清硬件中断标志
更新简单原子状态
短时间 spinlock 临界区
wake_up
complete
schedule_work
返回
```

不适合：

```text
mutex_lock
msleep
I2C/SPI 同步传输
copy_to_user / copy_from_user
可能睡眠的 GPIO cansleep API
长时间循环处理
```

## 5.3 `request_threaded_irq()`

如果处理中断时需要做比较慢、甚至可能睡眠的工作，可以使用 threaded IRQ。

```c
ret = request_threaded_irq(irq,
                           my_top,
                           my_thread,
                           IRQF_ONESHOT,
                           "mydev",
                           d);
```

top half：

```c
static irqreturn_t my_top(int irq, void *data)
{
    return IRQ_WAKE_THREAD;
}
```

thread handler：

```c
static irqreturn_t my_thread(int irq, void *data)
{
    struct mydev *d = data;

    /* 这里是线程上下文，可以做允许睡眠的工作 */

    return IRQ_HANDLED;
}
```

如果硬件不要求先在 hard IRQ 里快速确认状态，有些驱动会直接把 top handler 传 `NULL`。

---

# 6. 并发：mutex、spinlock、atomic

不要看到共享变量就机械加锁。先把访问路径画出来。

例如：

```text
read() --------┐
               ├── dev->queue
IRQ handler ---┘
```

两条路径能同时访问 `queue`，就要同步。

## 6.1 mutex

```c
struct mutex lock;

mutex_init(&lock);

mutex_lock(&lock);
/* 临界区 */
mutex_unlock(&lock);
```

mutex 可能让当前线程睡眠，因此只用于可睡眠上下文。

典型：

```text
read/write/ioctl
workqueue
kthread
threaded IRQ
probe/remove 的普通流程
```

hard IRQ 里不能 `mutex_lock()`。

## 6.2 spinlock

```c
spinlock_t lock;
spin_lock_init(&lock);
```

如果同一份数据既会在 IRQ 中访问，也会在进程上下文访问，常见写法：

```c
unsigned long flags;

spin_lock_irqsave(&d->lock, flags);
/* 非常短的临界区 */
spin_unlock_irqrestore(&d->lock, flags);
```

spinlock 的临界区不能睡眠，也不要放耗时操作。

## 6.3 atomic

适合简单计数器 / flag：

```c
atomic_t ready;

atomic_set(&ready, 0);
atomic_read(&ready);
atomic_inc(&ready);
atomic_dec(&ready);
atomic_xchg(&ready, 0);
```

它适合“单个简单状态的原子操作”，不等于可以替代对复杂结构体、链表、多字段一致性的锁。

## 6.4 怎么选

```text
只有简单计数/flag？
    → atomic

多个线程/进程上下文共享复杂状态？
    → mutex

IRQ 和其他上下文共享短小数据结构？
    → spinlock
```

真正判断标准还是：**哪些执行路径会同时碰这份数据，以及这些上下文能不能睡眠。**

---

# 7. 等待和通知：wait_queue、completion

锁解决“不能同时改”的问题；等待队列解决“条件还没满足，我先睡，等事件来再叫醒我”的问题。

## 7.1 wait_queue

私有结构体：

```c
wait_queue_head_t wq;
atomic_t ready;
```

初始化：

```c
init_waitqueue_head(&d->wq);
atomic_set(&d->ready, 0);
```

`read()`：

```c
ret = wait_event_interruptible(d->wq,
                               atomic_read(&d->ready));
if (ret)
    return ret;
```

IRQ：

```c
atomic_set(&d->ready, 1);
wake_up_interruptible(&d->wq);
```

关键点：`wake_up` 只是叫醒等待者，等待者醒后还会重新判断 condition。

带超时：

```c
ret = wait_event_interruptible_timeout(
        d->wq,
        atomic_read(&d->ready),
        msecs_to_jiffies(500));
```

通常：

- `> 0`：条件满足，返回剩余 jiffies
- `0`：超时
- `< 0`：被信号打断

## 7.2 `poll()` 和 wait_queue

```c
static unsigned int my_poll(struct file *file,
                            poll_table *wait)
{
    struct mydev *d = file->private_data;
    unsigned int mask = 0;

    poll_wait(file, &d->wq, wait);

    if (atomic_read(&d->ready))
        mask |= POLLIN | POLLRDNORM;

    return mask;
}
```

`poll_wait()` 只是登记“如果这个等待队列被唤醒，请重新检查我”。

真正是否 ready，还得你自己判断状态并返回 `POLLIN/POLLOUT/...`。

## 7.3 completion

completion 更像“一次任务完成事件”。

```c
struct completion done;
init_completion(&done);
```

等待：

```c
wait_for_completion(&done);
```

通知：

```c
complete(&done);
```

适合：

```text
DMA 完成
硬件命令完成
某个初始化步骤完成
线程之间一次性同步
```

和 wait_queue 的区别不是谁“更高级”，而是语义不同：

```text
wait_queue：等待某个条件
completion：等待某件事情完成
```

---

# 8. 延后处理和内核线程：workqueue、threaded IRQ、kthread、timer

## 8.1 workqueue

如果 hard IRQ 里只想快速记录状态，真正耗时工作放后面：

```c
struct work_struct work;
```

初始化：

```c
INIT_WORK(&d->work, my_work);
```

IRQ 中：

```c
schedule_work(&d->work);
```

worker：

```c
static void my_work(struct work_struct *work)
{
    struct mydev *d;

    d = container_of(work, struct mydev, work);

    /* 线程上下文，可以睡眠 */
}
```

remove 前：

```c
cancel_work_sync(&d->work);
```

否则驱动资源已经释放，work 还可能跑进来。

## 8.2 delayed work

```c
INIT_DELAYED_WORK(&d->dwork, my_delayed_work);

schedule_delayed_work(&d->dwork,
                      msecs_to_jiffies(100));

cancel_delayed_work_sync(&d->dwork);
```

适合延迟执行、周期检查等。

## 8.3 kthread

需要一个长期存在、自己循环的专用内核线程时：

```c
d->thread = kthread_run(my_thread,
                        d,
                        "mydev_thread");
if (IS_ERR(d->thread))
    return PTR_ERR(d->thread);
```

线程函数：

```c
static int my_thread(void *data)
{
    struct mydev *d = data;

    while (!kthread_should_stop()) {
        /* 长期后台工作 */
        msleep(100);
    }

    return 0;
}
```

停止：

```c
kthread_stop(d->thread);
```

普通驱动不要一上来就创建线程。很多“中断后做点慢工作”的需求，用 workqueue 或 threaded IRQ 更合适。

## 8.4 timer（Linux 4.9）

4.9 常见旧式写法：

```c
setup_timer(&d->timer, my_timer_fn, (unsigned long)d);
```

启动 / 重设：

```c
mod_timer(&d->timer,
          jiffies + msecs_to_jiffies(100));
```

删除：

```c
del_timer_sync(&d->timer);
```

timer 回调运行在原子 / softirq 类上下文，不能睡眠。

新内核常用 `timer_setup()`，看到不同写法时先确认内核版本。

---

# 9. clock / reset / regulator

很多 SoC 外设不是“映射寄存器就能工作”。常见初始化顺序是：

```text
打开电源
  ↓
打开时钟
  ↓
解除复位
  ↓
配置寄存器
```

具体顺序一定以芯片手册为准。

## 9.1 clock

设备树通常有：

```dts
clocks = <...>;
clock-names = "ipg";
```

驱动：

```c
struct clk *clk;

clk = devm_clk_get(&pdev->dev, "ipg");
if (IS_ERR(clk))
    return PTR_ERR(clk);

ret = clk_prepare_enable(clk);
if (ret)
    return ret;
```

关闭：

```c
clk_disable_unprepare(clk);
```

频率：

```c
clk_set_rate(clk, rate);
rate = clk_get_rate(clk);
```

## 9.2 reset

```c
struct reset_control *rst;

rst = devm_reset_control_get(&pdev->dev, NULL);
if (IS_ERR(rst))
    return PTR_ERR(rst);

ret = reset_control_deassert(rst);
```

重新拉复位：

```c
reset_control_assert(rst);
```

有些控制器支持：

```c
reset_control_reset(rst);
```

## 9.3 regulator

设备树常见：

```dts
vdd-supply = <&reg_xxx>;
```

驱动：

```c
struct regulator *vdd;

vdd = devm_regulator_get(&pdev->dev, "vdd");
if (IS_ERR(vdd))
    return PTR_ERR(vdd);

ret = regulator_enable(vdd);
```

关闭：

```c
regulator_disable(vdd);
```

设置电压：

```c
regulator_set_voltage(vdd, min_uV, max_uV);
```

不要随便绕过板级 DT 中的电源约束。驱动表达“需要什么”，实际 PMIC / 电源树由 regulator framework 管理。

---

# 10. MMIO：寄存器映射与 readl/writel

SoC 内部控制器最典型的资源是 MMIO。

设备树：

```dts
mydev@20000000 {
    compatible = "demo,mydev";
    reg = <0x20000000 0x1000>;
};
```

`probe()`：

```c
struct resource *res;
void __iomem *base;

res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
if (!res)
    return -ENODEV;

base = devm_ioremap_resource(&pdev->dev, res);
if (IS_ERR(base))
    return PTR_ERR(base);
```

访问：

```c
u32 val;

val = readl(base + REG_STATUS);
writel(val | BIT(3), base + REG_CTRL);
```

不要把 `void __iomem *` 当普通内存指针直接 `*ptr` 解引用。

常用：

```c
readb / readw / readl
writeb / writew / writel
```

宽度要和硬件寄存器定义匹配。

---

# 11. I2C 子系统

先把两层分开：

```text
MPU6050 / EEPROM / 温度传感器驱动
        ↓
      I2C Core
        ↓
I2C Controller Driver
        ↓
SoC I2C 控制器寄存器
        ↓
      SCL/SDA
```

上层设备驱动不应该自己去生成 START/STOP/ACK。

## 11.1 `i2c_driver`

```c
static int my_i2c_probe(struct i2c_client *client,
                        const struct i2c_device_id *id)
{
    ...
    return 0;
}

static int my_i2c_remove(struct i2c_client *client)
{
    ...
    return 0;
}

static const struct of_device_id my_of_match[] = {
    { .compatible = "demo,my-sensor" },
    { }
};

static struct i2c_driver my_i2c_driver = {
    .driver = {
        .name           = "my_sensor",
        .of_match_table = my_of_match,
    },
    .probe  = my_i2c_probe,
    .remove = my_i2c_remove,
};
```

注册：

```c
i2c_add_driver(&my_i2c_driver);
i2c_del_driver(&my_i2c_driver);
```

`client` 里常看：

```c
client->addr
client->adapter
client->irq
client->dev
```

设备树中的：

```dts
sensor@68 {
    compatible = "demo,my-sensor";
    reg = <0x68>;
};
```

这里的 `reg = <0x68>` 是 I2C 从机地址。

## 11.2 SMBus helper

读一字节寄存器：

```c
ret = i2c_smbus_read_byte_data(client, reg);
if (ret < 0)
    return ret;

value = ret & 0xff;
```

写一字节：

```c
ret = i2c_smbus_write_byte_data(client, reg, value);
```

连续读：

```c
ret = i2c_smbus_read_i2c_block_data(client,
                                    start_reg,
                                    len,
                                    buf);
```

是否可用取决于 adapter / 芯片事务能力。

## 11.3 `i2c_transfer()`

更通用：

```c
u8 reg = 0x3B;
u8 buf[6];

struct i2c_msg msgs[2] = {
    {
        .addr  = client->addr,
        .flags = 0,
        .len   = 1,
        .buf   = &reg,
    },
    {
        .addr  = client->addr,
        .flags = I2C_M_RD,
        .len   = sizeof(buf),
        .buf   = buf,
    },
};

ret = i2c_transfer(client->adapter, msgs, 2);
if (ret != 2)
    return ret < 0 ? ret : -EIO;
```

这里“成功返回 2”是因为提交了两条 message。它不是普通的 `0=成功` 形式。

I2C 同步传输可能睡眠，不要放 hard IRQ。

---

# 12. SPI 子系统

关系和 I2C 类似：

```text
具体 SPI 芯片驱动
   ↓
SPI Core
   ↓
SPI Controller Driver
   ↓
SoC SPI 控制器
```

## 12.1 `spi_driver`

`probe()` 拿到：

```c
struct spi_device *spi
```

常看字段：

```c
spi->max_speed_hz
spi->mode
spi->bits_per_word
spi->chip_select
```

修改参数后：

```c
ret = spi_setup(spi);
```

## 12.2 简单同步 API

```c
spi_write(spi, tx_buf, len);
spi_read(spi, rx_buf, len);
```

寄存器型设备常见：

```c
ret = spi_write_then_read(spi,
                          &reg,
                          1,
                          buf,
                          len);
```

复杂事务：

```c
struct spi_message m;
struct spi_transfer xfer = { ... };

spi_message_init(&m);
spi_message_add_tail(&xfer, &m);
ret = spi_sync(spi, &m);
```

这些同步接口通常可以睡眠，同样不要塞进 hard IRQ。

---

# 13. PWM 子系统

Linux 4.9 BSP 常见旧接口：

```c
struct pwm_device *pwm;

pwm = devm_pwm_get(&pdev->dev, NULL);
if (IS_ERR(pwm))
    return PTR_ERR(pwm);
```

配置：

```c
ret = pwm_config(pwm, duty_ns, period_ns);
```

例如 1 kHz：

```text
period = 1 ms = 1,000,000 ns
```

50% 占空比：

```text
duty = 500,000 ns
```

启动：

```c
pwm_enable(pwm);
```

停止：

```c
pwm_disable(pwm);
```

常见场景：背光、蜂鸣器、LED 调光、电机控制。

新内核 PWM API 越来越偏向 state/apply 模型，读新代码时要注意版本差异。

---

# 14. DMA：mapping API 和 DMAEngine

DMA 初学最容易把两个概念混起来。

## 14.1 DMA mapping API

解决的是：

> 一块内存怎么安全地让设备 DMA 访问？CPU 地址和设备看到的 DMA 地址怎么对应？缓存一致性怎么办？

### coherent buffer

```c
dma_addr_t dma_handle;
void *cpu_addr;

cpu_addr = dma_alloc_coherent(dev,
                              size,
                              &dma_handle,
                              GFP_KERNEL);
if (!cpu_addr)
    return -ENOMEM;
```

得到两个地址：

```text
cpu_addr   → CPU 在内核里访问

dma_handle → 写给 DMA/设备硬件
```

释放：

```c
dma_free_coherent(dev,
                  size,
                  cpu_addr,
                  dma_handle);
```

### streaming mapping

```c
dma_addr = dma_map_single(dev,
                          cpu_buf,
                          size,
                          DMA_TO_DEVICE);

if (dma_mapping_error(dev, dma_addr))
    return -EIO;
```

完成后：

```c
dma_unmap_single(dev,
                 dma_addr,
                 size,
                 DMA_TO_DEVICE);
```

## 14.2 DMAEngine

DMAEngine 解决的是：

> 我想让 SoC 的 DMA 控制器真正搬这段数据，怎么申请 channel、配置、提交 descriptor？

Linux 4.9 常见：

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

准备：

```c
desc = dmaengine_prep_slave_single(chan,
                                   dma_addr,
                                   len,
                                   DMA_DEV_TO_MEM,
                                   DMA_PREP_INTERRUPT);
if (!desc)
    return -EIO;
```

回调：

```c
desc->callback = my_dma_done;
desc->callback_param = d;
```

提交：

```c
cookie = dmaengine_submit(desc);
dma_async_issue_pending(chan);
```

停止：

```c
dmaengine_terminate_all(chan);
dma_release_channel(chan);
```

真正写 DMA 驱动时，一定要先确认硬件方向、外设 FIFO 地址、总线宽度、burst、缓存一致性和 descriptor 生命周期。

---

# 15. Input 子系统

如果设备本质是“输入事件”，例如：

```text
按键
鼠标
触摸屏
旋钮
```

通常不需要自己造一套 `/dev/key` + `cdev`。

Input Core 已经定义好了事件模型。

分配：

```c
struct input_dev *input;

input = devm_input_allocate_device(&pdev->dev);
if (!input)
    return -ENOMEM;
```

声明能力：

```c
input->name = "my-key";
input_set_capability(input, EV_KEY, KEY_ENTER);
```

注册：

```c
ret = input_register_device(input);
```

上报：

```c
input_report_key(input, KEY_ENTER, 1);
input_sync(input);

input_report_key(input, KEY_ENTER, 0);
input_sync(input);
```

最后用户空间通常看到：

```text
/dev/input/eventX
```

这就是“大子系统替你封装字符设备接口”的一个典型例子。

---

# 16. USB 子系统

USB 设备一般不是 `platform_device`。

典型对象：

```text
usb_device
usb_interface
usb_driver
usb_device_id
URB
```

匹配可能基于：

```text
VID / PID
USB class
interface class/subclass/protocol
```

`probe()` 常见签名：

```c
static int my_probe(struct usb_interface *intf,
                    const struct usb_device_id *id)
```

拔设备时：

```c
static void my_disconnect(struct usb_interface *intf)
```

同步控制传输：

```c
usb_control_msg(...)
```

简单 bulk：

```c
usb_bulk_msg(...)
```

高吞吐 / 异步传输更多使用 URB：

```text
usb_alloc_urb
usb_fill_*_urb
usb_submit_urb
completion callback
usb_kill_urb
```

USB 驱动最大的一个现实问题是：**设备可能随时拔出**。因此 disconnect、URB 停止、引用计数、并发退出非常重要。

你以前用的 USB 摄像头路径可以理解为：

```text
用户程序
  ↓
V4L2
  ↓
uvcvideo
  ↓
USB Core / URB
  ↓
USB host controller driver
  ↓
硬件
```

---

# 17. V4L2 / ALSA / DRM / Network / Block / MTD 怎么看

这些都不是“背十几个 API 就算会”的小框架。先知道它们接管了什么。

## V4L2 / Media

解决视频设备统一模型。

常见入口：

```c
v4l2_device_register()
video_register_device()
vb2_queue_init()
```

用户态常见：

```text
/dev/videoX
VIDIOC_*
mmap
QBUF / DQBUF
poll
```

## ALSA / ASoC

解决音频：

```text
PCM
DAI
codec
machine/card
mixer/control
```

用户通常看到：

```text
/dev/snd/*
```

## DRM/KMS

显示子系统，核心概念包括：

```text
plane
crtc
encoder
connector
framebuffer
```

## Network

网卡主要走：

```text
net_device
net_device_ops
sk_buff
NAPI
socket
```

不是普通的 `/dev/net0 + file_operations` 思路。

## Block

磁盘、eMMC、SSD 等块设备：

```text
gendisk
request_queue
bio
```

用户可能看到：

```text
/dev/mmcblk0
/dev/sda
```

但它不是普通字符设备。

## MTD

面向裸 NOR/NAND Flash。

NAND 还会涉及：

```text
坏块
ECC
页/块擦写
```

和普通 block device 思路不同。

---

# 18. devm、ERR_PTR 和错误回滚

## 18.1 为什么很多 API 不返回 NULL，而返回 `ERR_PTR`

例如：

```c
clk = devm_clk_get(dev, NULL);
```

正确判断：

```c
if (IS_ERR(clk))
    return PTR_ERR(clk);
```

不是：

```c
if (!clk)
    ...
```

常见辅助：

```c
IS_ERR(ptr)
PTR_ERR(ptr)
ERR_PTR(err)
IS_ERR_OR_NULL(ptr)
```

## 18.2 `devm_*` 的意义

例如：

```c
devm_kzalloc
devm_ioremap_resource
devm_clk_get
devm_gpiod_get
devm_request_irq
```

这些资源会跟 `struct device` 生命周期绑定，probe 失败或设备解绑时自动清理。

但是不要把 `devm` 理解成“remove 可以完全不写”。

例如：

```text
DMA 还在跑
workqueue 还没停
kthread 还没退出
clock 还开着
regulator 还开着
```

这些“硬件活动和执行路径的停机顺序”仍然要你自己处理。

## 18.3 错误回滚为什么常用 goto

内核驱动里 `goto` 很常见，不是因为代码落后，而是资源通常按顺序申请、按相反顺序释放。

```c
ret = regulator_enable(d->vdd);
if (ret)
    return ret;

ret = clk_prepare_enable(d->clk);
if (ret)
    goto err_regulator;

ret = request_irq(...);
if (ret)
    goto err_clk;

return 0;

err_clk:
    clk_disable_unprepare(d->clk);
err_regulator:
    regulator_disable(d->vdd);
    return ret;
```

这种结构比在每个失败分支里复制一堆 cleanup 更不容易漏。

---

# 19. 一个完整 probe 应该怎么看

不要逐行背 `probe()`。把它拆成阶段。

```c
struct mydev {
    struct device *dev;

    void __iomem *base;

    struct clk *clk;
    struct reset_control *rst;
    struct regulator *vdd;
    struct gpio_desc *reset_gpio;

    int irq;

    struct mutex lock;
    spinlock_t irq_lock;
    wait_queue_head_t wq;
    atomic_t ready;

    struct cdev cdev;
    dev_t devt;
};
```

一个典型 `probe()`：

```c
static int my_probe(struct platform_device *pdev)
{
    struct mydev *d;
    struct resource *res;
    int ret;

    /* 1. 创建每设备私有数据 */
    d = devm_kzalloc(&pdev->dev, sizeof(*d), GFP_KERNEL);
    if (!d)
        return -ENOMEM;

    d->dev = &pdev->dev;
    platform_set_drvdata(pdev, d);

    /* 2. 初始化软件同步对象 */
    mutex_init(&d->lock);
    spin_lock_init(&d->irq_lock);
    init_waitqueue_head(&d->wq);
    atomic_set(&d->ready, 0);

    /* 3. 获取硬件资源 */
    d->vdd = devm_regulator_get(&pdev->dev, "vdd");
    if (IS_ERR(d->vdd))
        return PTR_ERR(d->vdd);

    d->clk = devm_clk_get(&pdev->dev, NULL);
    if (IS_ERR(d->clk))
        return PTR_ERR(d->clk);

    d->rst = devm_reset_control_get(&pdev->dev, NULL);
    if (IS_ERR(d->rst))
        return PTR_ERR(d->rst);

    d->reset_gpio = devm_gpiod_get(&pdev->dev,
                                   "reset",
                                   GPIOD_OUT_LOW);
    if (IS_ERR(d->reset_gpio))
        return PTR_ERR(d->reset_gpio);

    /* 4. 按芯片手册要求做上电时序 */
    ret = regulator_enable(d->vdd);
    if (ret)
        return ret;

    ret = clk_prepare_enable(d->clk);
    if (ret)
        goto err_regulator;

    ret = reset_control_deassert(d->rst);
    if (ret)
        goto err_clk;

    /* 5. MMIO */
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    d->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(d->base)) {
        ret = PTR_ERR(d->base);
        goto err_reset;
    }

    /* 6. IRQ */
    d->irq = platform_get_irq(pdev, 0);
    if (d->irq < 0) {
        ret = d->irq;
        goto err_reset;
    }

    ret = devm_request_irq(&pdev->dev,
                           d->irq,
                           my_irq,
                           0,
                           dev_name(&pdev->dev),
                           d);
    if (ret)
        goto err_reset;

    /* 7. 初始化设备寄存器 */
    writel(..., d->base + REG_CTRL);

    /* 8. 接入上层接口 */
    ret = my_register_userspace_interface(d);
    if (ret)
        goto err_reset;

    return 0;

err_reset:
    reset_control_assert(d->rst);
err_clk:
    clk_disable_unprepare(d->clk);
err_regulator:
    regulator_disable(d->vdd);
    return ret;
}
```

读一个陌生驱动的 `probe()` 时，可以按下面顺序找：

```text
1. private data 在哪分配？
2. 设备树 / 总线资源怎么拿？
3. pinctrl / gpio / clock / regulator / reset 怎么处理？
4. MMIO 或 I2C/SPI 通信怎么建立？
5. IRQ / DMA 在哪注册？
6. 硬件寄存器初始化在哪？
7. 最后接入了哪个上层子系统？cdev / input / V4L2 / ALSA / DRM？
8. remove 和 error path 怎么停止所有并发执行路径？
```

这比从第一行一直背到最后一行有效得多。

---

# 20. 写驱动时怎么快速判断该用什么

| 需求 | 优先想到 |
|---|---|
| 物理 PIN 切成 GPIO/UART/I2C/SPI | pinctrl |
| GPIO 输入输出、电平 | GPIO / gpiod |
| 硬件异步事件 | IRQ |
| `read()` 没数据时睡眠、事件来再醒 | wait_queue |
| `poll/select/epoll` | `.poll + poll_wait + wait_queue` |
| 两个进程上下文改共享状态 | mutex |
| IRQ 和进程共享短小数据 | spinlock |
| 简单计数 / flag | atomic |
| IRQ 后需要慢处理 | threaded IRQ / workqueue |
| 长期后台循环 | kthread |
| 等一次硬件/DMA操作完成 | completion |
| 外设需要时钟 | clock framework |
| 外设需要解除复位 | reset controller |
| 设备需要供电 | regulator |
| SoC 内部寄存器 | platform resource + ioremap + readl/writel |
| 外部 I2C 芯片 | i2c_driver / i2c_transfer |
| 外部 SPI 芯片 | spi_driver / spi_sync |
| PWM 输出 | PWM framework |
| 大数据搬运 | DMA mapping + DMAEngine |
| 按键 / 鼠标 / 触摸 | Input |
| 摄像头 / 视频 | V4L2 / Media |
| 音频 | ALSA / ASoC |
| 显示 | DRM/KMS |
| 网卡 | Network subsystem |
| eMMC/SSD/磁盘 | Block |
| NOR/NAND 裸 Flash | MTD |

---

# 21. 建议的学习顺序

不要按“子系统大全”从头背到尾。

比较适合嵌入式 Linux 驱动入门的路线是：

```text
platform + Device Tree
        ↓
字符设备 + file_operations
        ↓
pinctrl + GPIO
        ↓
IRQ
        ↓
wait_queue + poll
        ↓
mutex / spinlock / atomic
        ↓
workqueue / threaded IRQ
        ↓
clock + reset + regulator + MMIO
        ↓
I2C 设备驱动
        ↓
SPI 设备驱动
        ↓
DMA
        ↓
Input / V4L2 / ALSA / DRM 等大型子系统
```

最重要的是每学一块就写一个能跑的设备：

```text
GPIO LED
→ GPIO 按键
→ 按键 + IRQ
→ IRQ + 阻塞 read
→ IRQ + poll
→ platform + Device Tree
→ I2C 传感器
→ SPI 设备
→ DMA
```

API 忘了就查，重点是能回答这几个问题：

```text
这个对象是谁创建的？
这个指针指向什么？
资源从设备树哪里来？
这个 API 会不会睡眠？
谁可能和它并发？
失败以后要释放什么？
remove 时谁还可能继续访问这份内存？
```

这些问题能回答清楚，驱动代码就不再是一堆函数名。

---

## 版本说明

本文按 Linux 4.9.88 / i.MX6ULL 学习环境整理。Linux 内核 API 会持续变化，例如：

```text
GPIO：整数 API → descriptor/gpiod API
Timer：setup_timer → timer_setup
PWM：旧 config/enable → state/apply 模型
DMA：部分 channel 获取 API 在新内核有变化
class / remove 等接口在更高版本中也有签名变化
```

读 BSP 驱动时，优先以你当前内核源码中的头文件和同版本驱动为准，不要直接把新内核示例复制到 4.9。
