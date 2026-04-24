# PMIC

本文介绍 K3 平台基于 RPMI 协议的 PMIC（电源管理芯片）功能及使用方法。

## 模块介绍

**PMIC（Power Management IC，电源管理芯片）**是负责系统电源管理的集成电路，通过 Regulator（电源调节器）子系统控制各路电源的开关和电压调节。

K3 平台使用 **RPMI（RISC-V Platform Management Interface）** 协议实现 PMIC 功能。RPMI PMIC 通过 mailbox 机制与固件通信，实现电源的动态管理。

### 功能介绍

K3 RPMI PMIC 架构：

```
用户空间/驱动
    ↓
Regulator 框架
    ↓
RPMI Regulator 驱动
    ↓
Mailbox 子系统
    ↓
MPXY Mailbox
    ↓
esos（Regulator）
    ↓
硬件 PMIC
```

**Regulator 框架组成：**

1. **Regulator Consumer**：需要电源供电的设备，消耗调节器提供的电力
2. **Regulator Framework**：提供标准的内核接口，控制系统的电压/电流调节器
3. **Regulator Driver**：驱动代码，负责向框架注册设备并与底层硬件通信
4. **Machine**：配置各个 regulator 的属性，如电压范围、初始状态等

### 源码结构介绍

Regulator 模块在内核源码中的路径为 `drivers/regulator/`：

```
drivers/regulator/
├── core.c                  # Regulator 框架核心代码
├── of_regulator.c          # 设备树解析
├── helpers.c               # 辅助函数
├── regulator-rpmi.c        # K3 RPMI Regulator 驱动
└── ...
```

### RPMI Regulator 类型

RPMI 支持两种类型的 regulator：

1. **RPMI_REGULATOR_DISCRETE**：离散电压型（固定电压档位）
2. **RPMI_REGULATOR_LINEAR**：线性电压型（连续可调电压范围）

### K3 电源域说明

K3 平台主要电源域：

- **edcdc1（或 adcdc1）**：为 X100 核心供电
- **edcdc2（或 adcdc2）**：为 A100 核心供电
- **dcdc1-6、aldo1-4、dldo1-7：其他 DCDC 和 LDO 电源
- **pvin**：主电源输入

> **注意**：不同板级 DTS 中，edcdc 可能命名为 adcdc，功能相同。

## 关键特性

- 支持多路 DCDC 和 LDO 电源
- 支持电压动态调节
- 支持电源开关控制（enable/disable）
- 基于 RPMI 协议通信
- 通过 mailbox 与固件交互
- 支持线性电压范围和离散电压档位
- 支持电源依赖关系管理
- 支持 CPU 核心电压动态调节（DVFS）

## 配置介绍

主要包括 **内核 CONFIG 配置** 和 **DTS 配置**

### 内核 CONFIG 配置

#### 启用 Regulator 框架

`CONFIG_REGULATOR` 为内核 Regulator 框架提供支持。

```
Symbol: REGULATOR [=y]
Device Drivers
      -> Voltage and Current Regulator Support (REGULATOR [=y])
```

#### 启用 RPMI Regulator 驱动

需要启用 `CONFIG_REGULATOR_RPMI` 以支持 K3 的 RPMI Regulator 驱动。

```
Symbol: REGULATOR_RPMI [=y]
Device Drivers
      -> Voltage and Current Regulator Support
            -> RISC-V RPMI Regulator (REGULATOR_RPMI [=y])
```

### DTS 配置

#### DTSI 配置示例

在 `dtsi` 文件中定义 RPMI Regulator 的 mailbox 通道。
通常情况下，该部分**无需修改**。

**RPMI Regulator 节点（k3.dtsi）**

```dts
rpmi_regulator: rpmi_regulator@0 {
    compatible = "riscv,rpmi-regulator";
    mboxes = <&mpxy_mbox 0x0007 0x0>;
    status = "okay";
};
```

**属性说明：**

- `compatible`：兼容字符串，标识为 RPMI Regulator 设备
- `mboxes`：mailbox 通道配置，用于与固件通信
  - `&mpxy_mbox`：mailbox 控制器
  - `0x0007`：Regulator 服务 ID
  - `0x0`：通道参数

#### DTS 板级配置示例

在板级 DTS 中配置各路电源的属性（以 k3_evb.dts 为例）：

```dts
&rpmi_regulator {
    status = "okay";

    pvin: pvin {
        regulator-min-microvolt = <5000000>;
        regulator-max-microvolt = <5000000>;
        regulator-always-on;
        regulator-boot-on;
    };

    edcdc2: edcdc2 {
        /* A100 核心供电 */
        regulator-min-microvolt = <800000>;
        regulator-max-microvolt = <800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    edcdc1: edcdc1 {
        /* X100 核心供电 */
        regulator-min-microvolt = <534000>;
        regulator-max-microvolt = <10000000>;
        regulator-always-on;
        regulator-boot-on;
    };

    p3v3: p3v3 {
        regulator-min-microvolt = <3300000>;
        regulator-max-microvolt = <3300000>;
        regulator-always-on;
        regulator-boot-on;
    };

    p1v8: p1v8 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dcdc1: dcdc1 {
        regulator-min-microvolt = <1050000>;
        regulator-max-microvolt = <1050000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dcdc3: dcdc3 {
        regulator-min-microvolt = <800000>;
        regulator-max-microvolt = <800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dcdc4: dcdc4 {
        regulator-min-microvolt = <2100000>;
        regulator-max-microvolt = <2100000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dcdc5: dcdc5 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dcdc6: dcdc6 {
        regulator-min-microvolt = <500000>;
        regulator-max-microvolt = <600000>;
        regulator-always-on;
        regulator-boot-on;
    };

    aldo1: aldo1 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <3300000>;
        regulator-always-on;
        regulator-boot-on;
    };

    aldo2: aldo2 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    aldo3: aldo3 {
        regulator-min-microvolt = <500000>;
        regulator-max-microvolt = <3400000>;
    };

    aldo4: aldo4 {
        regulator-min-microvolt = <3300000>;
        regulator-max-microvolt = <3300000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo1: dldo1 {
        regulator-min-microvolt = <1200000>;
        regulator-max-microvolt = <1200000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo2: dldo2 {
        regulator-min-microvolt = <900000>;
        regulator-max-microvolt = <900000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo3: dldo3 {
        regulator-min-microvolt = <800000>;
        regulator-max-microvolt = <800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo4: dldo4 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo5: dldo5 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo6: dldo6 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };

    dldo7: dldo7 {
        regulator-min-microvolt = <1800000>;
        regulator-max-microvolt = <1800000>;
        regulator-always-on;
        regulator-boot-on;
    };
};
```

**常用属性说明：**

| 属性 | 说明 |
|------|------|
| `regulator-name` | 电源名称（可选，默认使用节点名） |
| `regulator-min-microvolt` | 最小电压（微伏） |
| `regulator-max-microvolt` | 最大电压（微伏） |
| `regulator-always-on` | 始终保持开启 |
| `regulator-boot-on` | 启动时开启 |
| `regulator-ramp-delay` | 电压变化延迟（微秒/伏） |

## 接口说明

### RPMI 服务 ID

RPMI Regulator 支持以下服务：

| 服务名称 | 功能说明 |
|---------|---------|
| GET_NUM_DOMAINS | 获取电源域数量 |
| GET_ATTRIBUTES | 获取电源属性 |
| GET_SUPPORTED_LEVELS | 获取支持的电压档位 |
| SET_CONFIG | 设置电源配置（开关） |
| GET_CONFIG | 获取电源配置 |
| SET_LEVEL | 设置电压 |
| GET_LEVEL | 获取电压 |

### 内核 API

驱动实现了标准的 `regulator_ops` 接口：

```c
static const struct regulator_ops regulator_rpmi_ops = {
    .list_voltage = regulator_list_voltage_linear_range,
    .map_voltage = regulator_map_voltage_linear_range,
    .set_voltage_sel = regulator_rpmi_set_voltage_sel,
    .get_voltage_sel = regulator_rpmi_get_voltage_sel,
    .enable = regulator_rpmi_enable,
    .disable = regulator_rpmi_disable,
    .is_enabled = regulator_rpmi_is_enabled,
};
```

### 用户空间接口

#### sysfs 接口

通过 sysfs 查看和操作 regulator：

```bash
# 查看所有 regulator
ls /sys/class/regulator/

# 查看某个 regulator 的信息
cat /sys/class/regulator/regulator.0/name
cat /sys/class/regulator/regulator.0/type
cat /sys/class/regulator/regulator.0/microvolts
cat /sys/class/regulator/regulator.0/state

# 查看电压范围
cat /sys/class/regulator/regulator.0/min_microvolts
cat /sys/class/regulator/regulator.0/max_microvolts

# 查看使用该 regulator 的设备
cat /sys/class/regulator/regulator.0/num_users
```

# 查看所有 regulator 的信息
cat /sys/kernel/debug/regulator/regulator_summary

#### 内核驱动使用示例

在驱动中使用 regulator：

```c
#include <linux/regulator/consumer.h>

struct regulator *reg;
int ret;

/* 获取 regulator */
reg = regulator_get(dev, "vdd");
if (IS_ERR(reg)) {
    dev_err(dev, "Failed to get regulator\n");
    return PTR_ERR(reg);
}

/* 设置电压 */
ret = regulator_set_voltage(reg, 1800000, 1800000);
if (ret) {
    dev_err(dev, "Failed to set voltage\n");
    goto err;
}

/* 使能 regulator */
ret = regulator_enable(reg);
if (ret) {
    dev_err(dev, "Failed to enable regulator\n");
    goto err;
}

/* 获取当前电压 */
int uV = regulator_get_voltage(reg);
dev_info(dev, "Current voltage: %d uV\n", uV);

/* 禁用 regulator */
regulator_disable(reg);

/* 释放 regulator */
regulator_put(reg);

err:
    regulator_put(reg);
    return ret;
```

#### 设备树中引用 regulator

在设备节点中引用 regulator：

```dts
&i2c0 {
    sensor@48 {
        compatible = "example,sensor";
        reg = <0x48>;
        vdd-supply = <&dcdc1>;      /* 引用 dcdc1 电源 */
        vddio-supply = <&aldo1>;    /* 引用 aldo1 电源 */
    };
};
```

## Debug 说明

### 基本测试

1. 检查 regulator 设备是否注册

   ```bash
   ls /sys/class/regulator/
   ```

2. 查看驱动加载情况

   ```bash
   dmesg | grep -i regulator
   dmesg | grep -i rpmi
   ```

3. 查看所有 regulator 信息

   ```bash
   cat /sys/kernel/debug/regulator/regulator_summary
   ```

### 查看 Regulator 详细信息

```bash
# 遍历所有 regulator
for reg in /sys/class/regulator/regulator.*; do
    echo "=== $(basename $reg) ==="
    echo "Name: $(cat $reg/name 2>/dev/null)"
    echo "Type: $(cat $reg/type 2>/dev/null)"
    echo "State: $(cat $reg/state 2>/dev/null)"
    echo "Voltage: $(cat $reg/microvolts 2>/dev/null) uV"
    echo "Min: $(cat $reg/min_microvolts 2>/dev/null) uV"
    echo "Max: $(cat $reg/max_microvolts 2>/dev/null) uV"
    echo "Users: $(cat $reg/num_users 2>/dev/null)"
    echo ""
done
```

### 测试程序示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dirent.h>

#define REGULATOR_PATH "/sys/class/regulator"

void print_regulator_info(const char *reg_name)
{
    char path[256];
    char buf[256];
    FILE *fp;

    printf("=== %s ===\n", reg_name);

    // 读取 name
    snprintf(path, sizeof(path), "%s/%s/name", REGULATOR_PATH, reg_name);
    fp = fopen(path, "r");
    if (fp) {
        if (fgets(buf, sizeof(buf), fp))
            printf("Name: %s", buf);
        fclose(fp);
    }

    // 读取 state
    snprintf(path, sizeof(path), "%s/%s/state", REGULATOR_PATH, reg_name);
    fp = fopen(path, "r");
    if (fp) {
        if (fgets(buf, sizeof(buf), fp))
            printf("State: %s", buf);
        fclose(fp);
    }

    // 读取 voltage
    snprintf(path, sizeof(path), "%s/%s/microvolts", REGULATOR_PATH, reg_name);
    fp = fopen(path, "r");
    if (fp) {
        if (fgets(buf, sizeof(buf), fp))
            printf("Voltage: %s uV\n", buf);
        fclose(fp);
    }

    printf("\n");
}

int main(void)
{
    DIR *dir;
    struct dirent *entry;

    dir = opendir(REGULATOR_PATH);
    if (!dir) {
        perror("opendir");
        return -1;
    }

    while ((entry = readdir(dir)) != NULL) {
        if (strncmp(entry->d_name, "regulator.", 10) == 0) {
            print_regulator_info(entry->d_name);
        }
    }

    closedir(dir);
    return 0;
}
```

## 测试说明

### 电压设置测试

通过内核驱动接口测试电压设置（需要编写测试驱动）：

```c
struct regulator *reg;

reg = regulator_get(dev, "dcdc1");
if (!IS_ERR(reg)) {
    /* 设置电压为 1.2V */
    regulator_set_voltage(reg, 1200000, 1200000);
    regulator_enable(reg);
    
    /* 读取实际电压 */
    int uV = regulator_get_voltage(reg);
    pr_info("dcdc1 voltage: %d uV\n", uV);
    
    regulator_put(reg);
}
```

### 电源开关测试

测试电源的开关功能：

```bash
# 查看电源状态
cat /sys/class/regulator/regulator.0/state

# 通过驱动代码测试开关
# (需要在驱动中调用 regulator_enable/disable)
```

## FAQ

### Regulator 设备不存在

检查以下几点：

1. 确认内核配置已启用 `CONFIG_REGULATOR` 和 `CONFIG_REGULATOR_RPMI`
2. 确认 DTS 中 rpmi_regulator 节点 status 为 "okay"
3. 检查 mailbox 驱动是否正常加载：`dmesg | grep -i mbox`
4. 查看内核日志：`dmesg | grep -i regulator`

### 电压设置失败

可能的原因：

1. 设置的电压超出了 DTS 中定义的范围
2. RPMI 通信失败
3. 固件不支持该电压值

解决方法：

```bash
# 检查电压范围
cat /sys/class/regulator/regulator.X/min_microvolts
cat /sys/class/regulator/regulator.X/max_microvolts

# 查看错误日志
dmesg | grep -i "regulator\|rpmi"
```

### 如何查看 regulator 的使用者

```bash
# 查看使用该 regulator 的设备数量
cat /sys/class/regulator/regulator.X/num_users

# 查看详细的 regulator 树状结构
cat /sys/kernel/debug/regulator/regulator_summary
```

### RPMI 通信失败

如果出现 RPMI 通信错误：

1. 检查 mailbox 驱动是否正常：`dmesg | grep mpxy`
2. 确认固件版本是否支持 RPMI Regulator
3. 查看详细错误信息：`dmesg | grep -i "rpmi\|regulator"`

### 电源依赖关系配置

某些电源可能依赖其他电源，需要在 DTS 中配置：

```dts
dcdc1: dcdc1 {
    regulator-min-microvolt = <1050000>;
    regulator-max-microvolt = <1050000>;
    vin-supply = <&pvin>;  /* dcdc1 依赖 pvin 供电 */
};
```

### 如何在启动时设置电压

在 DTS 中配置 `regulator-boot-on` 和电压范围：

```dts
dcdc1: dcdc1 {
    regulator-min-microvolt = <1200000>;
    regulator-max-microvolt = <1200000>;
    regulator-boot-on;      /* 启动时自动开启 */
    regulator-always-on;    /* 始终保持开启 */
};
```

## 注意事项

1. **RPMI 依赖**：RPMI Regulator 依赖于固件支持，确保固件版本正确
2. **Mailbox 通道**：Regulator 使用 mailbox 通道 0x0007，不要与其他设备冲突
3. **电压范围**：设置电压时必须在 DTS 定义的 min/max 范围内
4. **电源依赖**：注意电源之间的依赖关系，避免循环依赖
5. **always-on 属性**：标记为 `regulator-always-on` 的电源不能被禁用
6. **电压精度**：实际电压可能与设置值略有差异，取决于硬件支持的电压档位
7. **并发访问**：Regulator 框架已处理并发访问，驱动中无需额外加锁
8. **CPU 核心供电**：
   - edcdc1（或 adcdc1）为 X100 核心供电，调整电压时需谨慎
   - edcdc2（或 adcdc2）为 A100 核心供电，调整电压时需谨慎
   - 不同板级 DTS 中命名可能不同（edcdc 或 adcdc），但功能相同
   - CPU 电压调节通常由 DVFS（动态电压频率调节）机制自动管理

## 参考资料

- Linux Regulator Framework 文档：`Documentation/power/regulator/`
- RPMI 规范：RISC-V Platform Management Interface Specification
- 设备树绑定文档：`Documentation/devicetree/bindings/regulator/`
