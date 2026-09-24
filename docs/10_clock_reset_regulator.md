# 10. clock / reset / regulator

真实 SoC 外设初始化常见顺序：

```text
regulator enable
      ↓
clock enable
      ↓
reset deassert
      ↓
寄存器初始化
```

实际顺序以芯片手册为准。

## Clock

设备树：

```dts
clocks = <&clks SOME_CLK>;
clock-names = "core";
```

获取：

```c
d->clk = devm_clk_get(dev, "core");
if (IS_ERR(d->clk))
    return PTR_ERR(d->clk);
```

开启：

```c
ret = clk_prepare_enable(d->clk);
```

关闭：

```c
clk_disable_unprepare(d->clk);
```

|API|返回|用途|
|---|---|---|
|`devm_clk_get()`|clk/ERR_PTR|获取|
|`clk_prepare_enable()`|0/负errno|开启|
|`clk_disable_unprepare()`|void|关闭|
|`clk_set_rate()`|0/负errno|请求频率|
|`clk_get_rate()`|Hz|查询|

## Reset

设备树：

```dts
resets = <&src SOME_RESET>;
reset-names = "core";
```

获取：

```c
d->rst = devm_reset_control_get(dev, "core");
```

常用：

```c
reset_control_assert(d->rst);
reset_control_deassert(d->rst);
reset_control_reset(d->rst);
```

## Regulator

设备树：

```dts
vdd-supply = <&reg_3v3>;
```

获取：

```c
d->vdd = devm_regulator_get(dev, "vdd");
```

开启：

```c
ret = regulator_enable(d->vdd);
```

关闭：

```c
regulator_disable(d->vdd);
```

电压：

```c
regulator_set_voltage(d->vdd, 1800000, 1800000);
```

## 错误回滚

```c
ret = regulator_enable(d->vdd);
if (ret)
    return ret;

ret = clk_prepare_enable(d->clk);
if (ret)
    goto err_regulator;

ret = reset_control_deassert(d->rst);
if (ret)
    goto err_clk;

return 0;

err_clk:
    clk_disable_unprepare(d->clk);
err_regulator:
    regulator_disable(d->vdd);
    return ret;
```

devm 解决对象释放，但硬件 enable/deassert 状态仍需正确恢复。
