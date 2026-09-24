# 03. Character Device

## 字符设备解决什么问题

字符设备让用户空间通过标准文件接口访问驱动：

```text
open
read
write
ioctl
poll
mmap
close
```

“字符设备”描述的是 Linux 的访问接口，不代表底层没有 I2C/SPI/USB。

## 设备号

`dev_t` 保存 major + minor：

```c
dev_t devt;

ret = alloc_chrdev_region(&devt, 0, 1, "mydev");
```

查看：

```c
MAJOR(devt);
MINOR(devt);
```

|API|参数|返回|作用|
|---|---|---|---|
|`alloc_chrdev_region(&devt,base_minor,count,name)`|输出devt、起始minor、数量、名字|0/负errno|动态申请设备号|
|`unregister_chrdev_region(devt,count)`|起始devt、数量|void|释放设备号|

申请设备号后不会自动出现 `/dev/mydev`。

## cdev

```c
struct cdev cdev;
```

初始化：

```c
cdev_init(&cdev, &my_fops);
```

注册：

```c
cdev_add(&cdev, devt, 1);
```

关系：

```text
设备号
  ↓
cdev
  ↓
file_operations
```

|API|作用|
|---|---|
|`cdev_init()`|把 cdev 和 fops 关联|
|`cdev_add()`|注册到字符设备表|
|`cdev_del()`|删除 cdev|

## class 和 device

Linux 4.9：

```c
class = class_create(THIS_MODULE, "myclass");
```

创建设备：

```c
dev = device_create(class, parent, devt, drvdata, "mydev0");
```

常见结果：

```text
/sys/class/myclass/mydev0
```

随后 devtmpfs/udev/mdev 可以创建：

```text
/dev/mydev0
```

手工 `mknod` 也能创建设备节点，因此 class 并不是 cdev 工作的绝对前提。

## file_operations

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

|用户态|驱动回调|用途|
|---|---|---|
|`open()`|`.open`|建立本次 open 上下文|
|`read()`|`.read`|取数据|
|`write()`|`.write`|下发数据/命令|
|`ioctl()`|`.unlocked_ioctl`|专用控制|
|`poll()`|`.poll`|就绪查询|
|`mmap()`|`.mmap`|共享大缓冲|
|`close()`|`.release`|释放本次 open 状态|

## open 如何找到当前设备

```c
struct mydev {
    struct cdev cdev;
    ...
};
```

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
struct mydev *d = file->private_data;
```

## copy_to_user / copy_from_user

```c
copy_to_user(user_buf, kernel_buf, len);
copy_from_user(kernel_buf, user_buf, len);
```

返回值是“未成功拷贝的字节数”。

常见：

```c
if (copy_to_user(buf, &data, sizeof(data)))
    return -EFAULT;
```

这类函数可能触发缺页和睡眠，不应放在 hard IRQ 或持 spinlock 时调用。

## ioctl

```c
#define MY_IOC_MAGIC 'M'
#define MY_GET_VALUE _IOR(MY_IOC_MAGIC, 0, int)
#define MY_SET_VALUE _IOW(MY_IOC_MAGIC, 1, int)
```

```c
static long my_ioctl(struct file *file,
                     unsigned int cmd,
                     unsigned long arg)
{
    switch (cmd) {
    case MY_GET_VALUE:
        break;
    case MY_SET_VALUE:
        break;
    default:
        return -ENOTTY;
    }

    return 0;
}
```

## poll

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

事件到来：

```c
atomic_set(&d->data_ready, 1);
wake_up_interruptible(&d->wq);
```

## 完整注册链

```text
alloc_chrdev_region
      ↓
cdev_init
      ↓
cdev_add
      ↓
class_create
      ↓
device_create
      ↓
/dev/mydev0
      ↓
open
      ↓
inode->i_cdev
      ↓
container_of
      ↓
file->private_data
      ↓
read/write/ioctl/poll
```
