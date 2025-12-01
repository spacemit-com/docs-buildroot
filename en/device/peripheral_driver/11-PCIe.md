# PCIe

PCIe Functionality and Usage Guide.

## Overview

PCIe (Peripheral Component Interconnect Express) is a high-speed serial computer expansion bus standard. It uses high-speed serial point-to-point dual-channel high-bandwidth transmission, where the connected devices have exclusive access to the channel bandwidth. The K1 platform features three PCIe controllers, supporting various PCIe interface devices, including NVMe SSDs, SATA, and Wi-Fi.

PCIe0 shares PHY hardware with the USB3 controller and cannot be used concurrently. Since USB3 is typically preferred in applications, PCIe0 is rarely used.

### Features

![](static/linux_pcie.png)

The Linux PCIe subsystem framework consists of three main components: PCIe core, PCIe controller driver, and PCIe device driver. 
The Linux PCIe subsystem framework comprises three components: the PCIe core, the PCIe controller driver, and the PCIe device driver.
The primary functions of each component are as follows: 

1. PCIe Core  
   - Performs PCIe bus enumeration, resource allocation, and interrupt management.  
   - Handles PCIe device addition and removal.  
   - Manages PCIe device driver registration and deregistration.  

2. PCIe Controller Driver
   - Operates the PCIe host controller.

3. PCIe Device Driver
   - Supports PCIe devices such as GPUs, NICs, and NVMe SSDs.

### Source Code Structure

The controller driver code is located in the drivers/pci/controller/dwc directory:

```
|-- pcie-designware.c           # DWC PCIe common driver code
|-- pcie-designware-host.c      # DWC PCIe host driver code
|-- pcie-k1x.c                  # K1x PCIe controller driver
```

## Key Features

### Features

| Feature | Description |
| :-----| :----|
| Supported Modes | Supports RC (Root Complex) mode |
| Supported Protocols and Lane Counts | Supports gen2x1, gen2x2 |
| Supported Devices | Supports NVMe SSDs, PCIe-to-SATA, PCIe network cards, and PCIe graphics cards |

### Performance Parameters

| SSD Model | Sequential Read (MB/s) | Sequential Write (MB/s) |
| :-----| :----| :----: |
| LRC20Z500GCB | 829 | 812 |  

Testing Method

```
# Testing Read
fio --name read --eta-newline=5s --filename=/dev/nvme0n1 --rw=read --size=2g --io_size=10g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --direct=1 --numjobs=1 --runtime=60 --group_reporting  

# Testing Write
fio --name write --eta-newline=5s --filename=/dev/nvme0n1 --rw=write --size=2g
 --io_size=60g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --
direct=1 --numjobs=1 --runtime=60 --group_reporting  
```

## Configuration

This mainly includes driver enablement configuration and DTS configuration.

### CONFIG Configuration

`CONFIG_PCI` provides support for the PCI and PCIe bus protocols and defaults to `Y`.

```
Device Drivers
    PCI support (PCI [=y])
```

`PCI_K1X_HOST` provides support for the K1 PCIe controller driver and defaults to `Y`.

```
Device Drivers
    PCI support (PCI [=y])
        PCI controller drivers
            DesignWare-based PCIe controllers
                Spacemit K1X PCIe Controller - Host Mode (PCI_K1X_HOST [=y])
```

### DTS Configuration

#### Configuration Space Allocation

The configuration space allocation for each controller is as follows:

1. PCIe0
   - Memory space size: 240MB
   - I/O space size: 1MB
2. PCIe1
   - Memory space size: 240MB
   - I/O space size: 1MB
3. PCIe2
   - Memory prefetchable space size: 256MB
   - Memory non-prefetchable space size: 112MB
   - I/O space size: 1MB

##### Detailed Configuration Space Explanation

Taking the PCIe2 controller as an example, the address space allocation for the PCIe controller is explained. The address space allocated to PCIe2 ranges from 0xa00000000 to 0xb80000000, with a size of 0x18000000 (384MB). You can modify the memory, I/O, and configuration spaces according to your needs, as long as they remain within the range of 0xa00000000 to 0xb80000000 and do not overlap.

The current configuration in `k1-x.dts` is as follows:

```c
pcie2_rc: pcie@ca800000 {
        ...
        reg = <0x0 0xca800000 0x0 0x00001000>, /* dbi */
              ...
              <0x0 0xb7000000 0x0 0x00002000>, /* config space */
              ...
        #address-cells = <3>;
 #size-cells = <2>;
 ranges = <0x01000000 0x0 0xb7002000 0 0xb7002000 0x0 0x100000>,    
             <0x42000000 0x0 0xa0000000 0 0xa0000000 0x0 0x10000000>, 
             <0x02000000 0x0 0xb0000000 0 0xb0000000 0x0 0x7000000>;
        ...
}
```

The meanings are as follows:

- Memory Space:  

```
  <0x42000000 0x0 0xa0000000 0 0xa0000000 0x0 0x10000000>
  Memory prefetchable space, size 256MB
  <0x02000000 0x0 0xb0000000 0 0xb0000000 0x0 0x7000000>
  Memory non-prefetchable space, size 112MB 
```

- I/O Space

```
  <0x01000000 0x0 0xb8000000 0 0xb8000000 0x0 0x10000>
  I/O space, size 1MB 
```

- Configuration Space:

```
  <0x01000000 0x0 0xb8010000 0 0xb8010000 0x0 0x10000>
  Configuration space, size 64KB 
```

#### pinctrl

Check the development board schematic to locate the pin group used by PCIe. Refer to the [PINCTRL](01-PINCTRL.md) to determine the pin group used by PCIe.

Assuming that PCIe2 can directly use the `pinctrl_pcie2_4` defined in `k1-x_pinctrl.dtsi`.

**DTS configuration**

The PCIe root complex (pcie_rc) description in the DTS for this configuration is as follows:

```c
&pcie2_rc {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_pcie2_4>;
        status = "okay";
};
```

#### Complete PCIe DTS

The DTS for the PCIe2 root complex (pcie_rc) mode controller is as follows:

```
pcie2_rc: pcie@ca800000 {
   compatible = "k1x,dwc-pcie";
   reg = <0x0 0xca800000 0x0 0x00001000>, /* dbi */
         <0x0 0xcab00000 0x0 0x0001ff24>, /* atu registers */
         <0x0 0xb7000000 0x0 0x00002000>, /* config space */
         <0x0 0xd4282bdc 0x0 0x00000008>, /* k1x soc config addr */
         <0x0 0xc0d20000 0x0 0x00001000>, /* phy ahb */
         <0x0 0xc0d10000 0x0 0x00001000>, /* phy addr */
         <0x0 0xd4282bcc 0x0 0x00000008>, /* conf0 addr */
         <0x0 0xc0b10000 0x0 0x00001000>; /* phy0 addr */
   reg-names = "dbi", "atu", "config", "k1x_conf", "phy_ahb", "phy_addr", "conf0_addr", "phy0_addr";

   k1x,pcie-port = <2>;
   clocks = <&ccu CLK_PCIE2>;
   clock-names = "pcie-clk";
   resets = <&reset RESET_PCIE2>;
   reset-names = "pcie-reset";
   power-domains = <&power K1X_PMU_BUS_PWR_DOMAIN>;

   bus-range = <0x00 0xff>;
   max-link-speed = <2>;
   num-lanes = <2>;
   num-viewport = <8>;
   device_type = "pci";
   #address-cells = <3>;
   #size-cells = <2>;
   ranges = <0x01000000 0x0 0xb7002000 0 0xb7002000 0x0 0x100000>,
     <0x42000000 0x0 0xa0000000 0 0xa0000000 0x0 0x10000000>,
     <0x02000000 0x0 0xb1000000 0 0xb1000000 0x0 0x07000000>;
   interconnects = <&dram_range2>;
   interconnect-names = "dma-mem";

   interrupts = <143>, <147>;
   interrupt-parent = <&intc>;
   #interrupt-cells = <1>;
   interrupt-map-mask = <0 0 0 0x7>;
   interrupt-map = <0000 0 0 1 &pcie2_intc 1>, /* int_a */
     <0000 0 0 2 &pcie2_intc 2>, /* int_b */
     <0000 0 0 3 &pcie2_intc 3>, /* int_c */
     <0000 0 0 4 &pcie2_intc 4>; /* int_d */
   linux,pci-domain = <2>;

   pcie2_intc: interrupt-controller@0 {
    interrupt-controller;
    reg = <0 0 0 0 0>;
    #address-cells = <0>;
    #interrupt-cells = <1>;
   };

      pinctrl-names = "default";
     pinctrl-0 = <&pinctrl_pcie2_4>;
 
      status = "okay";
};
```

## Interface

### API

Register a PCI device driver:

```
/* Proper probing supporting hot-pluggable devices */
int __must_check __pci_register_driver(struct pci_driver *, struct module *,
                       const char *mod_name);

/* pci_register_driver() must be a macro so KBUILD_MODNAME can be expanded */
#define pci_register_driver(driver)     \
    __pci_register_driver(driver, THIS_MODULE, KBUILD_MODNAME)
```

Unregister a PCI device driver:

```
void pci_unregister_driver(struct pci_driver *dev);
```

## Debugging

### sysfs

The `/sys/bus/pci` directory provides information about PCI bus devices and drivers in the system:

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

1. Checking PCI Bus Topology Information.

```
# lspci
0001:00:00.0 PCI bridge: Device 201f:0001 (rev 01)
0001:01:00.0 Network controller: Realtek Semiconductor Co., Ltd. RTL8852BE PCIe 802.11ax Wireless Network Controller
0002:00:00.0 PCI bridge: Device 201f:0001 (rev 01)
0002:01:00.0 Non-Volatile memory controller: Silicon Motion, Inc. SM2263EN/SM2263XT (DRAM-less) NVMe SSD Controllers (rev 03) 
```
  
2. Checking Detailed Information of a PCI Device.  
To display detailed information for the device 0001:01:00.0:

```
lspci -vvvs 0001:01:00.0
```

3. NVMe SSD Read Test.

```
fio --name read --eta-newline=5s --filename=/dev/nvme0n1 --rw=read --size=2g --io_size=10g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --direct=1 --numjobs=1 --runtime=60 --group_reporting
```

4. NVMe SSD Write Test

```
fio --name write --eta-newline=5s --filename=/dev/nvme0n1 --rw=write --size=2g --io_size=60g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --direct=1 --numjobs=1 --runtime=60 --group_reporting
```

## FAQ
