# PINCTRL

介绍 K3 小核心（ESOS/RTT）平台 pinctrl 模块的定位、软件层次、设备树写法以及应用侧使用方式。

## 模块介绍

pinctrl 用于完成 SoC PAD 的复用选择和电气配置。在 K3 小核心 ESOS 中，pinctrl 模块主要承担以下职责：

- 根据设备树中的 `pinctrl-*` 属性查找设备对应的 state
- 解析 pinctrl 子节点中的 `pinctrl-single,pins`
- 将 `<寄存器偏移, 配置值>` 形式的数据写入 PAD 寄存器
- 支持 consumer 按 state 名称切换引脚配置
- 通过 GPIO range 机制与 GPIO 子系统协同工作

当前平台 pinctrl 控制器使用 `pinctrl-single` 驱动模型。

## 软件结构

从应用到硬件的大致调用关系如下：

```text
应用/外设驱动
├─ pinctrl_get()
├─ pinctrl_lookup_state()
└─ pinctrl_select_state()

                │
                ▼
pinctrl core
├─ components/drivers/pinctrl/core.c
├─ components/drivers/pinctrl/devicetree.c
└─ components/drivers/pinctrl/pinmux.c

                │
                ▼
SoC pinctrl 驱动
└─ bsp/spacemit/drivers/pinctrl/pinctrl-single.c

                │
                ▼
PAD / pinmux 寄存器
└─ 通过 pinctrl-single,pins 将配置写入寄存器
```

## 接口介绍

### API 介绍

- **获取设备对应的 pinctrl 句柄**

```c
struct pinctrl *pinctrl_get(struct dtb_node *dev);
```

参数说明：

- `dev`：设备树节点指针

返回值说明：

- 成功：返回 `struct pinctrl *` 句柄
- 失败：返回错误指针

- **按名称查找 pinctrl state**

```c
struct pinctrl_state *pinctrl_lookup_state(struct pinctrl *p,
                                           const char *name);
```

参数说明：

- `p`：`pinctrl_get()` 返回的 pinctrl 句柄
- `name`：state 名称，例如 `"default"`、`"sleep"`

返回值说明：

- 成功：返回 `struct pinctrl_state *` 句柄
- 失败：返回错误指针

- **选择并应用指定的 pinctrl state**

```c
int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *state);
```

参数说明：

- `p`：`pinctrl_get()` 返回的 pinctrl 句柄
- `state`：`pinctrl_lookup_state()` 返回的 state 句柄

返回值说明：

- `0`：应用成功
- 负值：应用失败

- **申请指定 GPIO 对应的 pinctrl GPIO 功能**

```c
int pinctrl_apply_state(struct dtb_node *node, const char *state);
```

参数说明：

- `node`：设备树节点指针
- `state`：要应用的 state 名称，例如 `"default"`、`"sleep"`

返回值说明：

- `0`：应用成功
- 负值：应用失败

- **应用 default 状态**

```c
int pinctrl_apply_default(struct dtb_node *node);
```

参数说明：

- `node`：设备树节点指针

返回值说明：

- `0`：应用成功
- 负值：应用失败

- **应用 sleep 状态**

```c
int pinctrl_apply_sleep(struct dtb_node *node);
```

参数说明：

- `node`：设备树节点指针

返回值说明：

- `0`：应用成功
- 负值：应用失败

### 设备树属性说明

- **pinctrl 控制器 compatible**

```dts
compatible = "pinctrl-single";
```

参数说明：

- `compatible`：pinctrl 控制器驱动匹配字符串

返回值说明：

- 无

- **pinctrl 状态名称**

```dts
pinctrl-names = "default";
```

参数说明：

- `pinctrl-names`：state 名称列表

返回值说明：

- 无

- **pinctrl 状态引用**

```dts
pinctrl-0 = <&ruart0_0_cfg>;
```

参数说明：

- `pinctrl-0`：第 0 个 state 对应的 pinctrl 配置节点引用

返回值说明：

- 无

- **pinctrl-single 寄存器配置项**

```dts
pinctrl-single,pins = <offset value offset value ...>;
```

参数说明：

- `offset`：PAD 配置寄存器偏移
- `value`：写入该寄存器的配置值

返回值说明：

- 无

### K3_PADCONF 写法说明

- **K3_PADCONF 宏写法**

```dts
K3_PADCONF(GPIO_134, (MUX_MODE4 | EDGE_NONE | PULL_UP | PAD_DS8))
```

参数说明：

- `GPIO_134`：PAD 编号
- `MUX_MODE4`：复用功能选择
- `EDGE_NONE`：边沿检测配置
- `PULL_UP`：上下拉配置
- `PAD_DS8`：驱动能力配置

返回值说明：

- 生成一组 `<offset value>` 数据，供 `pinctrl-single,pins` 使用

## 使用示例

### 设备树状态定义示例

```dts
ruart0_0_cfg: ruart0-0-cfg {
    pinctrl-single,pins = <
        K3_PADCONF(GPIO_134, (MUX_MODE4 | EDGE_NONE | PULL_UP | PAD_DS8))
        K3_PADCONF(GPIO_135, (MUX_MODE4 | EDGE_NONE | PULL_UP | PAD_DS8))
    >;
};
```

### 设备节点引用示例

```dts
&ruart0 {
    pinctrl-names = "default";
    pinctrl-0 = <&ruart0_0_cfg>;
    status = "okay";
};
```
