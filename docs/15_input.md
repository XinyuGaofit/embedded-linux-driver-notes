# 15. Input Subsystem

设备本质是输入事件时，优先考虑 Input 子系统，而不是自己创建 `/dev/key0`。

典型：

- 按键
- 键盘
- 鼠标
- 触摸屏
- 摇杆

用户常看到：

```text
/dev/input/event0
```

## input_dev

```c
struct input_dev *input;
```

分配：

```c
input = devm_input_allocate_device(dev);
if (!input)
    return -ENOMEM;
```

名字：

```c
input->name = "my-key";
```

## 声明能力

```c
input_set_capability(input, EV_KEY, KEY_ENTER);
```

绝对坐标：

```c
input_set_abs_params(input, ABS_X, 0, 1023, 0, 0);
```

## 注册

```c
ret = input_register_device(input);
```

## 上报

按键：

```c
input_report_key(input, KEY_ENTER, 1);
input_sync(input);

input_report_key(input, KEY_ENTER, 0);
input_sync(input);
```

触摸：

```c
input_report_abs(input, ABS_X, x);
input_report_abs(input, ABS_Y, y);
input_sync(input);
```

## GPIO key 路径

```text
Device Tree
    ↓
pinctrl / GPIO
    ↓
IRQ
    ↓
handler / threaded irq
    ↓
input_report_key
    ↓
input_sync
    ↓
Input Core
    ↓
/dev/input/eventX
```

## 常用 API

|API|作用|
|---|---|
|`devm_input_allocate_device()`|分配 input_dev|
|`input_set_capability()`|声明 event|
|`input_set_abs_params()`|声明 ABS 范围|
|`input_register_device()`|注册|
|`input_report_key()`|上报按键|
|`input_report_abs()`|上报绝对坐标|
|`input_sync()`|提交一组事件|

## 为什么不自己做字符设备

Input 已经统一了 event 格式、key code、用户态生态和工具支持。像 `evtest` 可以直接使用。
