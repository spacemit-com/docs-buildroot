# PWM

介绍 K3 小核心（ESOS/RTT）平台 PWM 模块的定位、软件层次、设备树写法以及应用侧使用方式。

## 模块介绍

PWM（Pulse Width Modulation）用于输出可调周期、可调占空比的脉冲波形。在 K3 小核心 ESOS 中，PWM 模块主要承担以下职责：

- 提供 PWM 输出使能与关闭能力
- 提供周期和脉宽配置能力
- 支持通过 DTS `compatible` 自动探测并注册 PWM 设备
- 通过 RT-Thread PWM 设备模型向上提供统一的 PWM 接口
- 与 pinctrl 子系统配合，将 PWM 功能复用到目标 PAD

当前平台 PWM 控制器使用 `spacemit,k1x-rpwm0` ~ `spacemit,k1x-rpwm9` 兼容串。

## 软件结构

从应用到硬件的大致调用关系如下：

```text
应用/组件驱动
├─ rt_device_find("rpwm0")
├─ rt_pwm_set()
├─ rt_pwm_enable()
├─ rt_pwm_disable()
└─ rt_pwm_get()

                │
                ▼
RT-Thread PWM 框架
├─ components/drivers/misc/rt_drv_pwm.c
└─ components/drivers/include/drivers/rt_drv_pwm.h

                │
                ▼
SoC PWM 驱动层
└─ bsp/spacemit/drivers/pwm/pwm-pxa.c

                │
                ▼
PWM 控制器硬件
└─ rpwm0 ~ rpwm9

                │
                ▼
pinctrl / PAD 复用
└─ 通过 pinctrl-0 选择 PWM 输出 PAD
```

## 接口介绍

### RT-Thread PWM API 介绍

在使用 RT-Thread PWM 接口时，通常先通过设备名获取 PWM 设备句柄：

- `rpwm0` ~ `rpwm9`：K3 小核心 PWM 设备名

**注册 PWM 设备**

```c
rt_err_t rt_device_pwm_register(struct rt_device_pwm *device,
                                const char *name,
                                const struct rt_pwm_ops *ops,
                                const void *user_data);
```

参数说明：

- `device`：PWM 设备对象指针
- `name`：设备名，例如 `"rpwm0"`
- `ops`：PWM 操作集合
- `user_data`：用户私有数据指针

返回值说明：

- `RT_EOK`：注册成功
- 其他错误码：注册失败

**使能 PWM 输出**

```c
rt_err_t rt_pwm_enable(struct rt_device_pwm *device, int channel);
```

参数说明：

- `device`：PWM 设备句柄
- `channel`：PWM 通道号

返回值说明：

- `RT_EOK`：使能成功
- 其他错误码：使能失败

**关闭 PWM 输出**

```c
rt_err_t rt_pwm_disable(struct rt_device_pwm *device, int channel);
```

参数说明：

- `device`：PWM 设备句柄
- `channel`：PWM 通道号

返回值说明：

- `RT_EOK`：关闭成功
- 其他错误码：关闭失败

**设置 PWM 周期和脉宽**

```c
rt_err_t rt_pwm_set(struct rt_device_pwm *device,
                    int channel,
                    rt_uint32_t period,
                    rt_uint32_t pulse);
```

参数说明：

- `device`：PWM 设备句柄
- `channel`：PWM 通道号
- `period`：PWM 周期，单位 ns
- `pulse`：PWM 脉宽，单位 ns，且必须满足 `pulse <= period`

返回值说明：

- `RT_EOK`：设置成功
- 其他错误码：设置失败

### PWM 控制命令说明

**使能命令**

```c
#define PWM_CMD_ENABLE      (128 + 0)
```

参数说明：

- 用于使能 PWM 输出

返回值说明：

- 无

**关闭命令**

```c
#define PWM_CMD_DISABLE     (128 + 1)
```

参数说明：

- 用于关闭 PWM 输出

返回值说明：

- 无

**设置命令**

```c
#define PWM_CMD_SET         (128 + 2)
```

参数说明：

- 用于设置 PWM 周期和脉宽

返回值说明：

- 无

**获取命令**

```c
#define PWM_CMD_GET         (128 + 3)
```

参数说明：

- 用于获取 PWM 配置

返回值说明：

- 无

## 使用示例

### 设备树节点示例

```dts
&rpwm0 {
    pinctrl-names = "default";
    pinctrl-0 = <&rpwm0_0_cfg>;
    status = "okay";
};
```

### 驱动侧使用示例

```c
struct rt_device_pwm *pwm_dev;

pwm_dev = (struct rt_device_pwm *)rt_device_find("rpwm0");
if (pwm_dev == RT_NULL)
    return -RT_ERROR;

rt_pwm_set(pwm_dev, 1, 100000, 50000);
rt_pwm_enable(pwm_dev, 1);
```

### 关闭输出示例

```c
rt_pwm_disable(pwm_dev, 1);
```

### shell 命令示例

```sh
pwm_set rpwm0 1 100000 50000
pwm_enable rpwm0 1
```
