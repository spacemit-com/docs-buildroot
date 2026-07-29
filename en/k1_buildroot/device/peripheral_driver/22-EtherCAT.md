# EtherCAT

This document describes EtherCAT Master functionality and usage.

## Overview

The IGH EtherCAT Master is a kernel module for high-performance real-time communication. It supports slave scanning, configuration management, and distributed clock synchronization. It can efficiently schedule and manage multiple slave devices and is widely used in industrial automation applications with stringent real-time performance and reliability requirements.

### Functional Overview

![](static/EtherCAT.png)  

As shown in the preceding figure, the EtherCAT Master architecture consists of four layers:

- **Application Layer:** User applications implement industrial control logic and interact with the EtherCAT Master driver through its interfaces.

- **EtherCAT Master Driver Layer:** Implements the core protocol, monitors the bus topology, automatically configures slaves, and synchronizes distributed clocks.

- **EtherCAT Device Driver Layer:** Consists of real-time network interface drivers that transmit and receive EtherCAT frames.

- **Physical Layer:** Network hardware devices.

### Source Code Structure

The EtherCAT Master driver code is located in the `drivers/net/ethercat` directory:  

```c
# Many xxx.h and xxx.c file pairs are present in the source tree. The .h files define data structures and interfaces, while the .c files provide their implementations.

├── device                     # EtherCAT device driver
│   ├── ecdev.h                
│   ├── ec_generic.c           # Generic network device driver
│   ├── ec_k1x_emac.c          # Real-time network interface driver for the K1 Ethernet controller
│   ├── ec_k1x_emac.h          
│   ├── Kconfig               
│   └── Makefile
├── include
│   ├── config.h               # Global configuration items and macro definitions
│   ├── ecrt.h                 # User application interface
│   ├── ectty.h
│   └── globals.h              # Global variables
├── Kconfig                    # Kernel configuration file
├── Makefile                   # Main Makefile for building the project
└── master                     # EtherCAT Master module
    ├── cdev.c                 # Provides the EtherCAT character device initialization interface
    ├── cdev.h                 
    ├── coe_emerg_ring.c       # Interface for handling CoE emergency messages
    ├── coe_emerg_ring.h       
    ├── datagram.c             # Interface for constructing ECAT datagrams
    ├── datagram.h             
    ├── datagram_pair.c        # Interface for constructing ECAT datagram pairs
    ├── datagram_pair.h        
    ├── debug.c                # Debugging interface
    ├── debug.h                
    ├── device.c               # Interface for network card device abstraction and management
    ├── device.h               
    ├── domain.c               # Interface for EtherCAT domain-related functions
    ├── domain.h               
    ├── doxygen.c              # Doxygen documentation source file
    ├── eoe_request.c          
    ├── eoe_request.h          
    ├── ethernet.c             # Core implementation of EoE functionality
    ├── ethernet.h             
    ├── flag.c                 
    ├── flag.h                 
    ├── fmmu_config.c          # Interface for constructing FMMU configuration messages
    ├── fmmu_config.h          
    ├── foe.h                  
    ├── foe_request.c          # FoE request handling interface
    ├── foe_request.h          
    ├── fsm_change.c           # State transition state machine implementation
    ├── fsm_change.h           
    ├── fsm_coe.c              # CoE protocol state machine implementation
    ├── fsm_coe.h              
    ├── fsm_eoe.c              # EoE protocol state machine implementation
    ├── fsm_eoe.h              
    ├── fsm_foe.c              # FoE protocol state machine implementation
    ├── fsm_foe.h              
    ├── fsm_master.c           # Main state machine implementation
    ├── fsm_master.h           
    ├── fsm_pdo.c              # PDO read/write state machine implementation
    ├── fsm_pdo_entry.c        # PDO entry read/write state machine implementation
    ├── fsm_pdo_entry.h        
    ├── fsm_pdo.h              
    ├── fsm_sii.c              # Slave information interface read/write state machine implementation
    ├── fsm_sii.h              
    ├── fsm_slave.c            # Slave state machine implementation
    ├── fsm_slave_config.c     # Slave configuration state machine implementation
    ├── fsm_slave_config.h     
    ├── fsm_slave.h            
    ├── fsm_slave_scan.c       # Slave scanning state machine implementation
    ├── fsm_slave_scan.h       
    ├── fsm_soe.c              # SoE (Servo over EtherCAT) state machine implementation
    ├── fsm_soe.h              
    ├── globals.h              
    ├── ioctl.c                # IOCTL interface for user-space interaction
    ├── ioctl.h               
    ├── Kconfig                
    ├── mailbox.c              # ECAT mailbox message interface
    ├── mailbox.h              
    ├── Makefile               
    ├── master.c               # Core logic of the EtherCAT master module
    ├── master.h               
    ├── module.c               # Initialization and cleanup of the master module
    ├── pdo.c                  # PDO management interface
    ├── pdo_entry.c            # PDO entry management interface
    ├── pdo_entry.h            
    ├── pdo.h                  
    ├── pdo_list.c             # PDO list management interface
    ├── pdo_list.h             
    ├── reg_request.c          # Interface for slave register read/write requests
    ├── reg_request.h          
    ├── rtdm.c                 # RTDM (Real-Time Driver Model) support
    ├── rtdm_details.h         
    ├── rtdm.h                 
    ├── rtdm-ioctl.c           # RTDM IOCTL interface implementation
    ├── rtdm_xenomai_v3.c      # Interface for Xenomai v3 real-time framework support
    ├── rt_locks.h             # Real-time lock implementation
    ├── sdo.c                  # SDO (Service Data Object) management
    ├── sdo_entry.c            # SDO entry management
    ├── sdo_entry.h            
    ├── sdo.h                  
    ├── sdo_request.c          # SDO request handling
    ├── sdo_request.h          
    ├── slave.c                # Slave state management logic
    ├── slave_config.c         # Interface for slave configuration
    ├── slave_config.h         
    ├── slave.h                
    ├── soe_errors.c           # Definitions for SoE protocol error codes
    ├── soe_request.c          # SoE request-related interface
    ├── soe_request.h          
    ├── sync.c                 # Interface for synchronization manager
    ├── sync_config.c          # Interface for configuring the synchronization manager
    ├── sync_config.h          
    ├── sync.h                 
    ├── voe_handler.c          # VOE (Vendor-specific over EtherCAT) request interface
    └── voe_handler.h  
```

## Key Features

| Feature | Description |
| :-----| :----|
| Automatic Slave Configuration | Supports automatic scanning and configuration of connected slave devices, simplifying network configuration |
| Distributed Clock Synchronization | Provides distributed clock (DC) synchronization with an accuracy of less than 1 µs |
| Multi-Protocol Support | Supports CoE, SoE, FoE, and other protocols |
| High Real-Time Performance | Supports a 1 ms DC cycle, meeting the real-time requirements of most industrial applications |
| Multi-Master Configuration | Supports multiple masters. Each master can manage two network devices: a primary device and a backup device |

## Configuration

It mainly includes driver enablement configuration and DTS configuration.

### CONFIG Configuration

- `ETHERCAT`: Set this option to `Y` to enable EtherCAT services.

```c
menuconfig ETHERCAT
        bool "Ethercat native network driver support"
        depends on NET
        default y
        help
          This section contains all the Ethercat drivers.
```

- `EC_MASTER`: Enables the Master driver.

```c
config EC_MASTER
        tristate "Ethercat Master driver support"
        depends on ETHERCAT
        default n
        help
          Ethercat Master driver support.

```

- `EC_GENERIC`: Enables the generic network interface driver.
- `EC_K1X_EMAC`: Enables the real-time network interface driver.

```c
config EC_GENERIC
        tristate "Ethercat generic device driver support"
        depends on ETHERCAT
        default n
        help
          generic native ethercat device driver support.

config EC_K1X_EMAC
        tristate "k1x native thercat device driver support"
        depends on ETHERCAT
        default n
        help
          Ethercat generic device driver support.

```

> **Note:** Enabling the `EC_K1X_EMAC` real-time network interface driver is recommended for better performance.

### DTS Configuration

The following EtherCAT parameters can be configured in DTS:

1. `run-on-cpu`: Specifies the CPU core on which EtherCAT Master real-time tasks run.
2. `debug-level`: Configures the EtherCAT Master debug log level. Supported values are `0`, `1`, and `2`. Higher values produce more detailed debug information. `0` is recommended for normal operation.
3. `master-count`: Specifies the number of EtherCAT Master instances. A maximum of 32 Masters is supported.
4. `ec-devices`: Specifies the list of network devices bound to the EtherCAT Master.
5. `master-indexes`: Specifies the Master index to which each network device is bound. Valid values range from `0` to `master-count-1` and must correspond to `ec-devices`.
6. `modes`: Specifies the operating mode of each network device and must correspond to `ec-devices` item by item. `ec_main` is the primary communication interface and is typically used. `ec_backup` is a redundant interface that maintains communication if the primary interface fails.

The default configuration is located in `linux-6.6/arch/riscv/boot/dts/spacemit/k1-x.dtsi`.

```c
ec_master: ethercat_master {
        compatible = "igh,k1x-ec-master";
        run-on-cpu = <1>;
        debug-level = <0>;
        master-count = <1>;
        ec-devices = <&eth0>, <&eth1>;
        master-indexes = <0>, <0>;
        modes = "ec_main", "ec_backup";
        status = "disabled";
};
```

The default configuration creates one EtherCAT Master:

- `eth0` is the EtherCAT primary communication device.
- `eth1` is the EtherCAT backup communication device.

To use the default configuration, enable the EtherCAT Master in the board DTS and configure the corresponding Ethernet nodes. For the DEB1 board, modify the following file:

```
linux-6.6/arch/riscv/boot/dts/spacemit/k1-x_deb1.dts
```

Configure the nodes as follows:

```c
&ec_master {
        status = "okay";
};

&eth0 {
        compatible = "spacemit,k1x-ec-emac";
        /* Keep other configurations unchanged */
        ...
        status = "okay";
};

&eth1 {
        compatible = "spacemit,k1x-ec-emac";
        /* Keep other configurations unchanged */
        ...
        status = "okay";
};
```

To customize the Master configuration, override the `ec_master` configuration in the board DTS.

Example 1: Configure one EtherCAT Master, with `eth0` used for EtherCAT and `eth1` used for standard Ethernet communication.

```c
&ec_master {
        master-count = <1>;
        ec-devices = <&eth0>;
        master-indexes = <0>;
        modes = "ec_main";
        status = "okay";
};

&eth0 {
        compatible = "spacemit,k1x-ec-emac";
        /* Keep other configurations unchanged */
        ...
        status = "okay";
};
```

Example 2: Configure two EtherCAT Masters, binding `eth0` and `eth1` to Master 0 and Master 1, respectively.

```c
&ec_master {
        master-count = <2>;
        ec-devices = <&eth0>, <&eth1>;
        master-indexes = <0>, <1>;
        modes = "ec_main","ec_main";
        status = "okay";
};

&eth0 {
        compatible = "spacemit,k1x-ec-emac";
        /* Keep other configurations unchanged */
        ...
        status = "okay";
};

&eth1 {
        compatible = "spacemit,k1x-ec-emac";
        /* Keep other configurations unchanged */
        ...
        status = "okay";
};
```

## Interface

### API

- Request a Master instance.

```c
ec_master_t *ecrt_request_master(unsigned int master_id);
```

- Create a process data domain.

```c
ec_domain_t *ecrt_master_create_domain(ec_master_t *master);
```

- Activate the Master.

```c
int ecrt_master_activate(ec_master_t *master);
```

- Synchronize the Master reference clock.

```c
int ecrt_master_sync_reference_clock_to(ec_master_t *master, uint64_t ref_time);
```

- Synchronize all slave clocks.

```c
void ecrt_master_sync_slave_clocks(ec_master_t *master);
```

- Configure a slave.

```c
ec_slave_config_t *ecrt_master_slave_config(ec_master_t *master, uint16_t alias, uint16_t position, uint32_t vendor_id, uint32_t product_code);

```

- Configure slave PDO mapping.

```c
int ecrt_slave_config_pdos(ec_slave_config_t *sc, uint16_t sync_index, const ec_sync_info_t *syncs);
```

- Register a PDO entry with the specified data domain.

```c
int ecrt_slave_config_reg_pdo_entry(ec_slave_config_t *sc, uint16_t index, uint8_t subindex， ec_domain_t *domain, unsigned int *offset);

```

- Configure the distributed clock for a slave.

```c
int ecrt_slave_config_dc(ec_slave_config_t *sc, uint16_t assign_activate, uint32_t sync0_cycle_time, int32_t sync0_shift, uint32_t sync1_cycle_time, int32_t sync1_shift);
```

## Debugging

### sysfs

Master information can be viewed through `/sys/class/EtherCAT/EtherCAT0`:

```c
/sys/class/EtherCAT/EtherCAT0
.
|-- dev
|-- power
|   |-- autosuspend_delay_ms
|   |-- control
|   |-- runtime_active_time
|   |-- runtime_status
|   `-- runtime_suspended_time
|-- subsystem -> ../../../../class/EtherCAT
`-- uevent

```
- `dev`: Provides Master device number information.
- `power`: Manages the device power state.
- `subsystem`: A subsystem link indicating that the device belongs to the EtherCAT subsystem.
- `uevent`: Contains the Master device number and device name.

## Testing

The EtherCAT Master test procedure is as follows:

1. Connect the slave devices to the Master network interface.
2. Power on the system. The kernel automatically loads the EtherCAT Master driver.
3. The Master automatically scans the slaves and outputs logs after successful identification.
4. The Master enters the `PREOP` state and waits for the user application to issue control commands.

An example boot log is shown below:

```c
[  966.525910] k1x_ec_emac cac80000.ethernet ecm0 (uninitialized): Link is Up - 100Mbps/Full - flow control off
[  966.535906] EtherCAT 0: Link state of ecm0 changed to UP.
[  966.552545] EtherCAT 0: 1 slave(s) responding on main device.
[  966.558389] EtherCAT 0: Slave states on main device: INIT.
[  966.564036] EtherCAT 0: Scanning bus.
[  966.739197] EtherCAT 0: Bus scanning completed in 176 ms.
[  966.745275] EtherCAT 0: Using slave 0 as DC reference clock.
[  966.756564] EtherCAT 0: Slave states on main device: PRE-OP.

```

Testing can be performed with `examples/dc_user/main.c` from the [official EtherLab examples](https://gitlab.com/etherlab.org/ethercat). Run the test with two connected slaves and a 1 ms DC communication cycle.

Example output:

```c
period         995099 ...    1004890
exec            14500 ...     106835
latency          7227 ...      13169
period         994556 ...    1005557
exec            14625 ...     105543
latency          7409 ...      13805
period         995306 ...    1004974
exec            14458 ...     105127
latency          7269 ...      13205
period         995390 ...    1004807
exec            14583 ...     137586
latency          7284 ...      13893
period         995265 ...    1005516
exec            14792 ...     108710
latency          7460 ...      13658
period         995598 ...    1004557
exec            14458 ...     112502
latency          7299 ...      12821
period         994807 ...    1005056
exec            14459 ...     105085
latency          7428 ...      13340
period         995390 ...    1005016
exec            14792 ...     110502
latency          7230 ...      13237
period         994432 ...    1007265
exec            14959 ...     110668
latency          7199 ...      15479
period         994848 ...    1004682
exec            14709 ...     113544
latency          7630 ...      13930
```

> **Note:**
>
> - `period`: Range of communication cycle variation within one second.
> - `exec`: Range of Master periodic task execution time variation within one second.
> - `latency`: Range of Master wake-up error variation within one second.

## FAQ
