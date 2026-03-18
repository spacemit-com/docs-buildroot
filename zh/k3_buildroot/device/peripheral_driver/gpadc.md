# GPADC

当前先给一个直接结论：

**按这轮对 K3 SDK 的源码、binding 和 DTS 回查，暂时没有找到一条可落地的 K3 GPADC 主线。**

也就是说，`zh/k3_buildroot/device/peripheral_driver/gpadc.md` 这个文件在当前 K3 文档目录里，更像是一个**预留占位项**，而不是已经有明确 K3 实现可对应的外设文档。

## 结论先行

这轮我按老规矩回查了三条线：

1. Linux IIO/ADC 驱动
2. DT binding
3. K3 DTS / DTSI 节点

回查结果是：

- **没有找到 SpacemiT K3 专用 ADC / GPADC 驱动**；
- **没有找到 SpacemiT K3 对应的 ADC / GPADC binding**；
- **没有在 K3 DTS / DTSI 里找到 `adc@...` / `gpadc` / `io-channels` 这类实际使用节点**。

所以这篇文档目前不能像 I2C / SPI / CAN / V2D / EtherCAT 那样，直接按“已有 K3 驱动 + DTS 节点 + 用户态接口”去写。

## 这轮实际核实了什么

### 1. 查了 K1 参考文档

K1 的 `gpadc.md` 写法是：

- 以 **IIO 子系统** 为基础；
- GPADC 来自 **PMIC 内嵌 ADC**；
- 配置项是：
  - `SPACEMIT_P1_ADC`
- 典型路径是：
  - `drivers/iio/adc/k1x_adc.c`

也就是说，K1 那篇本质上更接近：

- **P1 PMIC ADC 文档**

而不是一个 SoC 内建 SAR ADC 文档。

### 2. 查了 K3 Linux `drivers/iio/adc/`

这轮专门扫了：

- `linux-6.18/drivers/iio/adc/`
- `linux-6.18/drivers/iio/`

结果是：

- 目录里有大量通用 ADC 驱动；
- 也能看到一些 PMIC ADC / GPADC 驱动，比如：
  - `88pm886-gpadc.c`
  - `ab8500-gpadc.c`
  - `da9150-gpadc.c`
  - `palmas_gpadc.c`
  - `sun20i-gpadc-iio.c`
  - `twl6030-gpadc.c`
- **但没有看到 SpacemiT K3 专用 ADC / GPADC 驱动文件**；
- **也没有看到类似 K1 文档里提到的 `k1x_adc.c` 这类 K3 可直接对应的文件**。

### 3. 查了 K3 IIO binding

这轮也扫了：

- `Documentation/devicetree/bindings/iio/`
- `Documentation/devicetree/bindings/iio/adc/`

结果同样是：

- 有大量通用 / 其他厂商 ADC binding；
- **没有看到 SpacemiT K3 自己的 ADC / GPADC binding**。

### 4. 查了 K3 DTS / DTSI

这轮重点 grep 了：

- `adc@`
- `gpadc`
- `#io-channel-cells`
- `io-channels`
- `spacemit.*adc`
- `spacemit.*gpadc`

目标范围是：

- `arch/riscv/boot/dts/spacemit/k3*.dts`
- `arch/riscv/boot/dts/spacemit/k3.dtsi`

结果是：

- **没有发现 K3 ADC / GPADC 设备节点**；
- **没有发现其他外设把 K3 ADC 当作 `io-channels` consumer 来使用**。

这点基本已经很能说明问题了：

- 如果 K3 真有一条成熟的 GPADC 主线，通常至少应该有：
  - driver
  - binding
  - DTS node
  - 或 consumer usage
- 但这轮三条线都没对上。

## 当前最稳的判断

### 判断 1：`gpadc.md` 大概率是占位文件

从当前 K3 文档目录看，`gpadc.md` 更像是：

- 文档框架阶段先占了一个名字
- 但当前 SDK 还没有一条完整 K3 对应实现

### 判断 2：K1 那篇不能直接迁移到 K3

K1 这篇的实质是：

- P1 PMIC 内嵌 ADC
- IIO 驱动 + pinctrl + PMIC 节点

但当前 K3 这轮核实下来，并没有直接看到：

- K3 文档可稳定落地的同名驱动
- K3 DTS 中正在实际启用的 ADC 节点

所以不能把 K1 的内容硬搬过来，不然就会写成“文档有，代码没对上”。

## 现在这篇为什么先这样写

因为当前最负责任的做法不是“凑一篇差不多的 ADC 文档”，而是把现状说清楚：

- 目前**没有足够证据**证明 K3 在当前 SDK 中存在一条可独立成文的 GPADC 支持主线；
- 因此这篇文档先保留为**核查结论页**，而不是伪装成一篇已经完整可用的开发指南。

## 后续如果要把这篇真正补起来，需要补齐什么

后续如果你想把 `gpadc.md` 变成一篇真正的 K3 文档，至少需要先再确认下面任意一条是否存在：

1. **SoC 内建 ADC 驱动**
   - `drivers/iio/adc/` 下新增 SpacemiT/K3 相关驱动
2. **PMIC ADC 驱动**
   - 并且 K3 板级 DTS 中实际启用
3. **K3 DTS 节点**
   - 出现明确 `adc@...` / `gpadc` / `io-channels`
4. **用户态可验证路径**
   - `/sys/bus/iio/devices/iio:deviceX/...`
   - 并且能和 K3 板级原理图、引脚、通道对应上

只要后续源码里这几条有一条补齐，这篇就能继续往正式文档方向写。

## FAQ

### 1. K3 现在到底有没有 GPADC？

**按当前这轮 SDK 回查，不能确认有。**

更准确地说：

- 没找到 K3 专用驱动
- 没找到 K3 binding
- 没找到 K3 DTS 节点

所以现在不能把它当成“已有明确支持的 K3 外设”来写。

### 2. K1 那篇为什么不能直接抄过来？

因为 K1 那篇本质更像：

- **P1 PMIC ADC 文档**

而当前 K3 这边没有查到能一一对上的：

- 驱动
- DTS 节点
- binding
- 用户态接口

### 3. 这篇现在最该记住哪句话？

**当前 K3 的 `gpadc.md` 先当“未确认实现的占位项”处理，不要按已支持外设去宣传或指导用户配置。**
