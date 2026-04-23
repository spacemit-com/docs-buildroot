# PCIe

This document describes PCIe functionality and usage.

## Module Overview

PCIe (Peripheral Component Interconnect Express) is a high-speed serial computer expansion bus standard. It uses high-speed serial point-to-point, dual-channel, high-bandwidth transmission, and each connected device has exclusive access to its channel bandwidth.

The K3 platform provides 5 PCIe controllers (Port A ~ Port E) and 6 independent PHYs, supporting multiple PCIe peripherals, including NVMe SSDs, SATA controllers, and Wi-Fi modules.

**Note:** Port D and USB3 controller share the same PHY hardware resource, so they cannot be enabled simultaneously. You must enable either PCIe or USB3 in the device tree according to actual requirements.

### Functional Overview

![](static/linux_pcie.png)

The Linux PCIe subsystem framework consists of the following three components:

1. **PCIe Core**

   - Performs PCIe bus enumeration, resource allocation, and interrupt management.
   - Handles PCIe device addition and removal.
   - Manages PCIe device driver registration and deregistration.

2. **PCIe Controller Driver**

   - Operates the PCIe host controller.

3. **PCIe Device Driver**

    - Device drivers for specific PCIe devices, such as GPUs, NICs, and NVMe devices

### Source Code Structure

The controller driver code is located in the `drivers/pci/controller/dwc` directory:

```
|-- pcie-designware.c           #dwc pcie common driver code
|-- pcie-designware-host.c      #dwc pcie host driver code
|-- pcie-spacemit-k1.c          #k3 pcie controller driver
|-- spacemit_pcie_phy.c         #k3 pcie phy driver
```

## Key Features

### Features

| Feature | Description |
| :-----| :----|
| Supported Modes | Supports RC (Root Complex) mode |
| Supported Protocols and Lane Counts | Supports Gen3x1, Gen3x2, Gen3x4, and Gen3x8 |
| Supported Devices | Supports NVMe SSDs, PCIe-to-SATA devices, PCIe network cards, and PCIe graphics cards |

### Mapping Between Controllers and PHYs

The mapping between the 5 PCIe controllers and 6 PHYs on the K3 platform is as follows:

| Controller | Port | Maximum Lane Number | Available PHY | Description |
| :-----| :----| :----| :----| :----|
| pcie0_rc | Port A | x8 | phy0~phy5 | Supports x8/x4/x2 modes and can aggregate multiple PHYs |
| pcie1_rc | Port B | x2 | phy1 | Available only when Port A is used in x2 mode |
| pcie2_rc | Port C | x2 | phy2, phy3 | Supports x2 (dual-PHY) or x1 mode |
| pcie3_rc | Port D | x1 | phy4 | Shared with USB3; only one can be used |
| pcie4_rc | Port E | x1 | phy5 | Independent |

### Performance Parameters

| SSD Model (Capacity) | Sequential Read (MB/s) | Sequential Write (MB/s)| Random Read (4K MB/s) | Random Write(4K MB/s) |
| :-----| :----: | :----: | :----: | :----: |
| Colorful CN600 128GB | 1280 | 1156 | 358 | 352 |
| HS-SSD-C2000Pro 256G | 2150 | 258 | 321 | 209 |
| MAXIO MAP1202 512G | 3470 | 3138 | 282 | 325 |

> The data is based on `deb1` (K3) + a PCIe Gen3 x4 slot, tested on the Buildroot 2024.12 + Linux-6.18 SDK using the `fio` method described below. All random workloads use a 4K block size.

**Test Method**

```
# Sequential Read
fio --name=nvme_seq_read --filename=/mnt/nvme_vfat/fiotest.bin --size=1G --bs=1M --rw=read --ioengine=libaio --direct=1 --iodepth=32 --numjobs=1 --time_based --runtime=30 --group_reporting

# Sequential Write
fio --name=nvme_seq_write --filename=/mnt/nvme_vfat/fiotest.bin --size=1G --bs=1M --rw=write --ioengine=libaio --direct=1 --iodepth=32 --numjobs=1 --time_based --runtime=30 --group_reporting

# Random Read (4K)
fio --name=nvme_rand_read --filename=/mnt/nvme_vfat/fiotest.bin --size=1G --bs=4k --rw=randread --ioengine=libaio --direct=1 --iodepth=32 --numjobs=1 --time_based --runtime=30 --group_reporting

# Random Write (4K)
fio --name=nvme_rand_write --filename=/mnt/nvme_vfat/fiotest.bin --size=1G --bs=4k --rw=randwrite --ioengine=libaio --direct=1 --iodepth=32 --numjobs=1 --time_based --runtime=30 --group_reporting
```

## Configuration

This mainly includes **driver enablement configuration** and **DTS configuration**.

### CONFIG Configuration

`CONFIG_PCI` provides support for the PCI and PCIe bus protocols. This option defaults to `Y`.

```
Device Drivers
    PCI support (PCI [=y])
```

`PCIE_SPACEMIT_K1` provides support for the K3 PCIe controller driver. This option defaults to `Y`.

```
Device Drivers
    PCI support (PCI [=y])
        PCI controller drivers
            DesignWare-based PCIe controllers
                SpacemiT K1 PCIe controller - Host Mode (PCIE_SPACEMIT_K1 [=y])
```

### DTS Configuration

#### Configuration Space Allocation

Each PCIe Root Complex on K3 declares four types of address windows in the DTS: `config` configuration space, `I/O` space, `MEM non-prefetchable` space, and `MEM prefetchable` space. The allocation for each controller in `linux-6.18/arch/riscv/boot/dts/spacemit/k3.dtsi` is as follows:

**Base Address of Each Controller Address Space:**

| Controller | config | I/O | Non-prefetchable MEM | Prefetchable MEM |
| :--- | :--- | :--- | :--- | :--- |
| Port A (`pcie0_rc`) | `0x11_0000_0000` | `0x11_0001_0000` | `0x11_0011_0000` | `0x18_0000_0000` |
| Port B (`pcie1_rc`) | `0x11_8000_0000` | `0x11_8001_0000` | `0x11_8011_0000` | `0x16_0000_0000` |
| Port C (`pcie2_rc`) | `0x12_0000_0000` | `0x12_0001_0000` | `0x12_0011_0000` | `0x15_0000_0000` |
| Port D (`pcie3_rc`) | `0x12_8000_0000` | `0x12_8001_0000` | `0x12_8011_0000` | `0x14_0000_0000` |
| Port E (`pcie4_rc`) | `0x12_c000_0000` | `0x12_c001_0000` | `0x12_c011_0000` | `0x13_0000_0000` |

**Window Sizes:**

| Window Type | Port A / B / C | Port D / E |
| :--- | :---: | :---: |
| config | 64 KiB | 64 KiB |
| I/O | 1 MiB | 1 MiB |
| Non-prefetchable MEM | ~2046.94 MiB (`0x7fef0000`) | ~1022.94 MiB (`0x3fef0000`) |
| Prefetchable MEM | 4 GiB | 4 GiB |

> The non-prefetchable MEM windows for Port D/E are smaller because their overall address-space windows are smaller than those of Port A/B/C. After subtracting the config and I/O regions, less space remains.

##### Configuration Space Description

Take Port A (`pcie0_rc`) as an example to illustrate how address space is configured in the DTS:

```c
pcie0_rc: pcie@80000000 {
        ...
        reg = <0x0 0x80000000 0x0 0x00001000>, /* dbi */
              <0x0 0x80100000 0x0 0x00001000>, /* dbi2 */
              <0x0 0x80300000 0x0 0x00003f20>, /* atu registers */
              <0x11 0x00000000 0x0 0x00010000>, /* config space */
              <0x0 0x82900000 0x0 0x00001000>; /* phy ahb (link) */
        reg-names = "dbi", "dbi2", "atu", "config", "link";
        ranges = <0x01000000 0x00 0x00010000 0x11 0x00010000 0x0 0x00100000>,
                 <0x02000000 0x0 0x00110000 0x11 0x00110000 0x0 0x7fef0000>,
                 <0x43000000 0x18 0x00000000 0x18 0x00000000 0x1 0x00000000>;
        ...
};
```

Each entry in the `ranges` property follows the format `<flags child-bus-address cpu-physical-address size>`. The meaning of the flags field is as follows:

| flags | Type |
| :--- | :--- |
| `0x01000000` | I/O space |
| `0x02000000` | 32-bit non-prefetchable memory space |
| `0x43000000` | 64-bit prefetchable memory space |

The three `ranges` entries for Port A correspond to the following windows:

| Window | CPU Physical Address | Size |
| :--- | :--- | :--- |
| I/O | `0x11_0001_0000` | `0x0010_0000` (1 MiB) |
| Non-prefetchable MEM | `0x11_0011_0000` | `0x7fef_0000` (~2046.94 MiB) |
| Prefetchable MEM | `0x18_0000_0000` | `0x1_0000_0000` (4 GiB) |

`config` space is allocated via the `reg` property, with base address `0x11_0000_0000` and size 64 KiB.

#### PHY Configuration

K3 PCIe uses independent PHY nodes. Both the PHY and the controller must be enabled in the board-level DTS.

The PHY node is defined in `k3.dtsi`:

```c
phy0: phy-pcie@81d00000 {
        compatible = "spacemit,k3-pcie-phy";
        reg = <0x0 0x81d00000 0x0 0x1000>;
        spacemit,syscon-apb-spare = <&pll>;
        num-lanes = <2>;
        spacemit,phy-id = <0>;
        #phy-cells = <0>;
        status = "disabled";
};
```

#### pinctrl Configuration

The PCIe `pinctrl` handles only sideband signals, not the high-speed differential lanes. The actual data link is bound to the PCIe PHY node through `phys = <&phyX ...>`. `pinctrl` mainly handles the following signal types:

- `PERST#`: Device reset signal
- `WAKE#`: Device wake signal
- `CLKREQ#`: Reference clock request signal

The configuration steps are recommended to be performed in the following order:

1. First, use the schematic to confirm which sideband signals are actually routed on the board and which PAD groups they connect to.
2. Then locate the corresponding `pcie*_cfg` group in `k3-pinctrl.dtsi`.
3. If the common `pinctrl` group does not exactly match your board-level wiring, redefine the group in the board-level DTS and keep only the PADs that are actually used.

In `K3_PADCONF(pin, func)`, `pin` is the PAD number and `func` is the selected mux function for that PAD. Properties such as `bias-disable`, `bias-pull-up`, `drive-strength`, and `power-source` must match the board-level voltage domain and pull-up configuration.

Taking `deb1` as an example, the board-level DTS trims the common pinctrl group. For example, `pcie0-1-cfg` retains only `PERST#` and `CLKREQ#` and does not configure `WAKE#`:

```c
pcie0-1-cfg {
        pcie0-0-pins {
                pinmux = <K3_PADCONF(79, 5)>,
                         <K3_PADCONF(81, 5)>;
                bias-pull-up;
                drive-strength = <33>;
                power-source = <1800>;
        };
};
```

The unit of the `power-source` field is mV, and `<1800>` corresponds to 1.8 V. If the board-level design connects the sideband signals to 3.3 V, change this value to `<3300>` according to the schematic so that the PAD voltage domain matches the peripheral.

If your board does not route out the `WAKE#` signal, you can override the default pin group in the board-level DTS by referring to this method, instead of rigidly using the 3-wire configuration shared in `k3-pinctrl.dtsi`.

#### Board DTS Configuration Example (deb1)

The `deb1` board uses the `k3_deb1.dts` file. It enables `phy0/1/2/3/5` and `pcie0/1/2/4` by default, while `phy4` and `pcie3_rc` remain disabled.

Important note on the relationship between Port A and Port B:
Although the `k3_deb1.dts` example sets `pcie0_rc { num-lanes = <4>; }`, the driver will dynamically downgrade Port A to x2 at runtime after detecting that the bifurcation GPIO is asserted, freeing the other two lanes for Port B. In other words, whether `pcie1_rc` can actually be enabled depends on the actual level of the bifurcation GPIO, not just the static `num-lanes` value in DTS.

First, enable the relevant PHYs:

```c
&phy0{
        status = "okay";
};

&phy1{
        status = "okay";
};

&phy2{
        status = "okay";
};

&phy3{
        status = "okay";
};

&phy5{
        status = "okay";
};
```

Then configure each PCIe controller:

```c
&pcie0_rc {
        pinctrl-names = "default";
        pinctrl-0 = <&pcie0_1_cfg>;
        phys = <&phy0>, <&phy1>;
        phy-names = "phy0", "phy1";
        num-lanes = <4>;
        spacemit,bifurcation-gpios = <&gpio 2 25 GPIO_ACTIVE_HIGH>;
        status = "okay";
};

&pcie1_rc {
        pinctrl-names = "default";
        pinctrl-0 = <&pcie1_1_cfg>;
        phys = <&phy1>;
        phy-names = "phy1";
        num-lanes = <2>;
        spacemit,device-detect-gpios = <&gpio 2 25 GPIO_ACTIVE_HIGH>;
        status = "okay";
};

&pcie2_rc {
        pinctrl-names = "default";
        pinctrl-0 = <&pcie2_1_cfg>;
        phys = <&phy2>, <&phy3>;
        phy-names = "phy2", "phy3";
        num-lanes = <2>;
        status = "okay";
};

&pcie4_rc {
        pinctrl-names = "default";
        pinctrl-0 = <&pcie4_0_cfg>;
        phys = <&phy5>;
        phy-names = "phy5";
        num-lanes = <1>;
        status = "okay";
};
```
Here:
The `spacemit,bifurcation-gpios` property of `pcie0_rc` is used to control the bifurcation mode of Port A.
The `spacemit,device-detect-gpios` property of `pcie1_rc` is a board-level device-detection GPIO.
During board-level porting, in addition to `status` and `phys`, these two GPIOs must also be verified against the schematic.

**Configuration Description:**

- `pinctrl-0`: Selects the PADs to which the sideband signals are physically connected. Verify whether PERST#/WAKE#/CLKREQ# are all actually routed on the board. If a signal is not routed out, trim the pin group as shown in the earlier example to avoid enabling floating PADs by mistake.
- `phys` / `phy-names`: The list of PHYs bound to the controller. The order must match the hardware lane wiring. For example, Port A on `deb1` requires `phy0` + `phy1` to operate at x4.
- `num-lanes`: Describes the expected number of lanes. It must satisfy two conditions at the same time: (1) whether the board actually routes that many lanes to the slot; (2) whether it matches the bifurcation/retimer DIP-switch settings. It is recommended to include the hardware wiring table directly in the documentation for cross-checking during debugging.
- `spacemit,bifurcation-gpios`: Whether this is required depends on the board design. If this GPIO is detected high, it indicates that a device is connected to the other PCIe interface sharing the same PHY. On `deb1`, this determines whether lanes are allocated to Port B.
- `spacemit,device-detect-gpios`: Whether this is required depends on the board design. If this GPIO is detected low, it indicates that no device is connected to the PCIe interface, allowing the driver to disable unused controllers.
- `status`: Enables or disables the controller.

#### Complete PCIe DTS

Take the `pcie0_rc` (Port A) controller in DTS as an example:

```
pcie0_rc: pcie@80000000 {
        compatible = "spacemit,k1-pcie";
        reg = <0x0 0x80000000 0x0 0x00001000>, /* dbi */
              <0x0 0x80100000 0x0 0x00001000>, /* dbi2 */
              <0x0 0x80300000 0x0 0x00003f20>, /* atu registers */
              <0x11 0x00000000 0x0 0x00010000>, /* config space */
              <0x0 0x82900000 0x0 0x00001000>; /* phy ahb (link) */
        reg-names = "dbi", "dbi2", "atu", "config", "link";

        bus-range = <0x00 0xff>;
        max-link-speed = <3>;
        num-lanes = <8>;
        device_type = "pci";
        #address-cells = <3>;
        #size-cells = <2>;
        ranges = <0x01000000 0x00 0x00010000 0x11 0x00010000 0x0 0x00100000>,
                 <0x02000000 0x0 0x00110000 0x11 0x00110000 0x0 0x7fef0000>,
                 <0x43000000 0x18 0x00000000 0x18 0x00000000 0x1 0x00000000>;

        interrupt-parent = <&saplic>;
        interrupts = <141 IRQ_TYPE_LEVEL_HIGH>;
        interrupt-names = "pcie_irq";

        clocks = <&syscon_apmu CLK_APMU_PCIE_PORTA_BUS>,
                 <&syscon_apmu CLK_APMU_PCIE_PORTA_MSTR>,
                 <&syscon_apmu CLK_APMU_PCIE_PORTA_SLV>;
        clock-names = "dbi", "mstr", "slv";
        resets = <&syscon_apmu RESET_APMU_PCIE_PORTA_DBI>,
                 <&syscon_apmu RESET_APMU_PCIE_PORTA_MSTR>,
                 <&syscon_apmu RESET_APMU_PCIE_PORTA_SLV>;
        reset-names = "dbi", "mstr", "slv";

        linux,pci-domain = <0>;
        msi-parent = <&simsic>;
        spacemit,apmu = <&syscon_apmu 0x1f0>;
        iommu-map = <0x0 &iommu 0x00000 0x10000>;

        status = "disabled";
};
```

## Interface

### API

- **Register a PCI device driver:**

```
/* Proper probing supporting hot-pluggable devices */
int __must_check __pci_register_driver(struct pci_driver *, struct module *,
                       const char *mod_name);

/* pci_register_driver() must be a macro so KBUILD_MODNAME can be expanded */
#define pci_register_driver(driver)     \
    __pci_register_driver(driver, THIS_MODULE, KBUILD_MODNAME)
```

- **Unregister a PCI device driver:**

```
void pci_unregister_driver(struct pci_driver *dev);
```

## Debugging

### sysfs

`/sys/bus/pci`: provides information about PCI bus devices and drivers in the system:

```
|-- devices                 // Devices on the PCI bus
|-- drivers                 // Drivers registered on the PCI bus
|-- drivers_autoprobe
|-- drivers_probe
|-- rescan
|-- resource_alignment
|-- slots
`-- uevent
```

## Testing

1. Check PCI bus topology information.

```
#lspci
```

2. Check detailed information for a PCI device.

```
lspci -vvvs <BDF>
```

3. NVMe SSD read test

```
fio --name read --eta-newline=5s --filename=/dev/nvme0n1 --rw=read --size=2g --io_size=10g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --direct=1 --numjobs=1 --runtime=60 --group_reporting
```

4. NVMe SSD write test

```
fio --name write --eta-newline=5s --filename=/dev/nvme0n1 --rw=write --size=2g --io_size=60g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --direct=1 --numjobs=1 --runtime=60 --group_reporting
```

## Compatibility with 32-bit PCIe EP Devices

### Background

On the K3 platform, all DDR physical addresses are located above 4 GiB, starting at `0x1_0000_0000`. Some PCIe endpoint devices, such as the MT7921E Wi-Fi card, support only 32-bit DMA addressing and therefore cannot directly access system memory above 4 GiB, resulting in DMA transfer failures.

### Solution

This issue can be resolved using **restricted DMA pool + dma-ranges address mapping**:

1. Reserve a dedicated DMA region in memory
2. Use the PCIe Address Translation Unit (ATU) to map the 32-bit bus addresses visible to the endpoint device to this physical memory region
3. The endpoint device performs DMA operations using 32-bit bus addresses, and the PCIe controller automatically handles address translation

### CONFIG Configuration

The following kernel configurations should be enabled:

```
CONFIG_SWIOTLB=y                  # Software I/O TLB, provides bounce buffer support
CONFIG_DMA_CMA=y                  # DMA contiguous memory allocation
CONFIG_DMA_RESTRICTED_POOL=y      # Support restricted DMA pool
```

### DTS Configuration

#### 1. Reserve DMA Memory Pool

Add a restricted DMA pool under the `reserved-memory` node. The following example reserves 32 MiB starting at `0x1_f000_0000`:

```c
reserved-memory {
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;

        pcie_dma_pool: pcie-dma-pool@1f0000000 {
                compatible = "restricted-dma-pool";
                reg = <0x1 0xf0000000 0x0 0x02000000>;  /* 32 MiB */
        };
};
```

#### 2. Configure PCIe Controller Node

Add `dma-ranges` and `memory-region` to the corresponding PCIe controller node:

```c
&pcie4_rc {
        /* ... other configuration ... */
        dma-ranges = <0x02000000 0x0 0x80000000 0x1 0xf0000000 0x0 0x02000000>;
        memory-region = <&pcie_dma_pool>;
        status = "okay";
};
```

Meaning of the `dma-ranges` entry:

| Field | Value | Description |
| :--- | :--- | :--- |
| flags | `0x02000000` | 32-bit non-prefetchable memory space |
| PCIe bus address | `0x0_8000_0000` | DMA address visible to the EP device (32-bit addressable) |
| CPU physical address | `0x1_f000_0000` | Actual physical memory address |
| Size | `0x0_0200_0000` | 32 MiB |

Workflow: The EP device initiates DMA reads or writes to bus address `0x8000_0000` → the PCIe ATU translates it to CPU physical address `0x1_f000_0000` → the access reaches the reserved DMA memory pool.

## FAQ

### 1. SSD Not Detected by the System: How to Diagnose the Issue?

1. **First, check whether it is enumerated on the PCIe bus.** Run `lspci | grep -i -E "nvme|non-volatile"`. If there is no output, the host has not detected the NVMe device at the PCIe bus level.
2. **Check the pin configuration and voltage domains.** In DTS, `pinctrl`, `power-source`, and `spacemit,*-gpios` must exactly match the board schematic. Make sure the pin number, mux function, PAD power domain, voltage level, and physically connected sideband signals all match.
3. **Rule out SSD hardware failure.** Replace the SSD with a known-good NVMe drive, or test the suspected faulty SSD on another working platform to confirm whether the issue is caused by device damage.
4. **Check initialization timing and signal quality.** If the link drops randomly or fails during link training, use an oscilloscope or protocol analyzer to inspect the timing, voltage swing, and signal integrity (SI) of signals such as PERST#, CLKREQ#, and REFCLK. If necessary, work with the vendor engineering team for joint debugging.

