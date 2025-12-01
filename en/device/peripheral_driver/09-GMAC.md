# GMAC

GMAC Functionality and Usage Guide.

## Overview

The GMAC (Gigabit Media Access Controller) module is a controller that enables Gigabit Ethernet communication, responsible for sending and receiving data frames and managing network traffic.

### Functional Description

![](static/net.png)

- **Application Layer:** Provides application services to users.
- **Protocol Stack Layer:** Implements network protocols and provides system call interfaces for the application layer.
- **Network Device Abstraction Layer:** Shields the details of driver implementation and provides a unified interface to the protocol stack.
- **Network Device Driver Layer:** Responsible for implementing data transmission and device management.
- **Physical Layer:** Network hardware devices.

### Source Code Structure

The GMAC driver code is located in the `drivers\net\ethernet\spacemit` directory:

```
drivers\net\ethernet\spacemit
|-- emac-ptp.c          #Provides PTP protocol support
|-- k1x-emac.c          #K1 GMAC driver code
|-- k1x-emac.h          #K1 GMAC driver header file
```

## Key Features

| Feature | Description |
| :-----| :----|
| Support for 10/100/1000M Ethernet | Compatible with multi-rate Ethernet connections |
| Support for DMA | Efficient data transfer to reduce CPU load |
| Support for NAPI | Enhance interrupt handling efficiency and reduce CPU overhead |
| Interrupt coalescing mechanism | Merge interrupts to improve performance under high load |
| Support for RGMII/RMII | Adapt to multiple application scenarios |
| Support for PTP | Achieve sub-microsecond-level time synchronization between devices |
| Support for power management | Support for suspend and resume to meet low-power requirements |

### Performance Parameters

|  | Single Card Half-Duplex | Single Card Full-Duplex | Dual Card Half-Duplex | Dual Card Full-Duplex |
| :---: | :---: | :---: | :---: | :---: |
| TX Rate (Mbps) | 942 | 930 | 942 | 797 |
| RX Rate (Mbps) | 941 | 940 | 941 | 881 |  

**Note:** The test bandwidth in full-duplex scenarios has some fluctuation.

### Performance Testing

#### Testing Environment

**Testing Device:** One K1-DEB1 board; one PC (model: HP ProBook 450 G10, operating system: Ubuntu 22.04.4 LTS)

**Network Topology:** The eth0 port of K1-DEB1 is directly connected to the Ethernet port of the PC; the eth1 port of K1-DEB1 is connected to the PC via a 2.5G USB-to-Ethernet adapter.

**IP Configuration:** The directly connected network ports are set in the same subnet, and the PC runs two iperf servers

```
# Set IP for the PC
ifconfig <ethernet-interface> 192.168.1.100 netmask 255.255.255.0
ifconfig <usb-ethernet-interface> 192.168.2.100 netmask 255.255.255.0

# Set IP for the k1-deb1 net device
ifconfig eth0 192.168.1.200 netmask 255.255.255.0
ifconfig eth1 192.168.2.200 netmask 255.255.255.0

# Start iperf3 server on the PC
iperf3 -s -B 192.168.1.100 -A 10 -D
iperf3 -s -B 192.168.2.100 -A 11 -D
```

#### Performance Optimization

To maximize network throughput, before testing, reasonably bind and allocate the interrupts of the network interfaces on the K1-DEB1 board to CPUs to fully utilize multi-core resources.

**Step 1:** View the current interrupt distribution by using the following command to confirm the interrupt numbers corresponding to the two network cards:

 ```
cat /proc/interrupts | grep eth*
 85:      11041    2332003          0          0          0          0          0          0  SiFive PLIC 131 Edge      eth0
 86:        234          0     409744          0          0          0          0          0  SiFive PLIC 133 Edge      eth1
 ```

**Step 2:** Bind the network interface hardware interrupts to different CPU cores. For example, bind eth0 to CPU1 and eth1 to CPU2.

```
echo 02 > /proc/irq/85/smp_affinity
echo 04 > /proc/irq/86/smp_affinity
```

**Step 3:** Enable RPS (Receive Packet Steering) to balance the soft interrupt load on the receiving end. For example, the following commands allow CPU4 to handle received packets on eth0 and CPU5 to handle received packets on eth1, fully leveraging the advantages of multi-core processing.

```
echo 10 > /sys/devices/platform/soc/cac80000.ethernet/net/eth0/queues/rx-0/rps_cpus
echo 4096 > /sys/devices/platform/soc/cac80000.ethernet/net/eth0/queues/rx-0/rps_flow_cnt

echo 20 > /sys/devices/platform/soc/cac81000.ethernet/net/eth1/queues/rx-0/rps_cpus
echo 4096 > /sys/devices/platform/soc/cac81000.ethernet/net/eth1/queues/rx-0/rps_flow_cnt
 ```

#### Single Network Card Test

##### Half-Duplex/TX

Use the eth0 port for single network card testing and bind the current iperf3 process to CPU6.

##### Half-Duplex/TX

```
iperf3 -c 192.168.1.100 -B 192.168.1.200 -t 100 -A 6
```

##### Half-Duplex/RX

```
iperf3 -c 192.168.1.100 -B 192.168.1.200 -t 100 -A 6 -R
```

##### Full-Duplex

```
iperf3 -c 192.168.1.100 -B 192.168.1.200 -t 100 -A 6 --bidir
```

#### Dual Network Card Test

In the dual network card test, bind the two iperf3 processes to CPU6 and CPU7 respectively.

##### Half-Duplex/TX

```
# Bind CPU6 for the first test, bind CPU7 for the second test
iperf3 -c 192.168.2.100 -B 192.168.2.200 -t 100 -A 6 > 1.txt &
iperf3 -c 192.168.1.100 -B 192.168.1.200 -t 100 -A 7 > 2.txt &

# View the test results
cat 1.txt
cat 2.txt
```

##### Half-Duplex/RX

```
iperf3 -c 192.168.2.100 -B 192.168.2.200 -t 100 -A 6 -R > 1.txt &
iperf3 -c 192.168.1.100 -B 192.168.1.200 -t 100 -A 7 -R > 2.txt &

cat 1.txt
cat 2.txt
```

##### Full-Duplex

```
iperf3 -c 192.168.2.100 -B 192.168.2.200 -t 100 -A 6 --bidir > 1.txt &
iperf3 -c 192.168.1.100 -B 192.168.1.200 -t 100 -A 7 --bidir > 2.txt &

cat 1.txt
cat 2.txt
```

## Configuration

It mainly includes **driver enablement configuration** and **DTS configuration**.

### CONFIG Configuration

`NET_VENDOR_SPACEMIT`：If you are using an Ethernet chip of the SpacemiT type, set this option to `Y`

```
config NET_VENDOR_SPACEMIT
        bool "Spacemit devices"
        default y
        depends on SOC_SPACEMIT
        help
          If you have a network (Ethernet) chipset belonging to this class,
          say Y.

          Note that the answer to this question does not directly affect
          the kernel: saying N will just cause the configurator to skip all
          the questions regarding Spacemit chipsets. If you say Y, you will
          be asked for your specific chipset/driver in the following questions.
     
```

`K1X_EMAC`：Enable the GMAC driver for SpacemiT.

```
config K1X_EMAC
        bool "k1-x Emac Driver"
        depends on SOC_SPACEMIT_K1X
        select PHYLIB
        help
          This Driver support Spacemit k1-x Ethernet MAC
          Say Y to enable support for the Spacemit Ethernet.

```

### DTS Configuration

For GMAC DTS configuration, it is necessary to determine the pin group used by the Ethernet, the PHY reset GPIO, the PHY model, and the address. The TX phase and RX phase generally use the default values.

#### pinctrl

Refer to the board schematic for PHY reset GPIO and pin assignments to locate the pin group used by GMAC.

Assuming the pins used by eth0 are GPIO00 to GPIO14 and GPIO45, the corresponding pinctrl configuration can use p`pinctrl_gmac0` from `k1-x_pinctrl.dtsi`.

The pinctrl configuration for eth0 using gmac0 in the solution DTS is as follows:

```c
&eth0 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_gmac0>;
};
```

#### GPIO

Check the development board schematic to locate the Ethernet PHY reset signal GPIO. Assuming the PHY reset GPIO for eth0 is GPIO 110.

The configuration for eth0 using GPIO 110 in the solution DTS is as follows:

```c
&eth0 {
    emac,reset-gpio = <&gpio 110 0>;
}
```

#### PHY Configuration

##### PHY Identification

Check the development board schematic to confirm the Ethernet PHY model and PHY ID.
For example, if the Ethernet PHY is RTL8821F-CG, its PHY ID is 001c.c916.
The PHY ID information can be found in the PHY specification or provided by contacting the PHY manufacturer.

##### PHY Address

Check the development board schematic to confirm the address of the Ethernet PHY, which is assumed to be 1.

##### PHY Configuration

The DTS configuration for eth0 is as follows:

```c
&eth0 {
    ...
    mdio-bus {
                #address-cells = <0x1>;
                #size-cells = <0x0>;
                rgmii0: phy@0 {
                        compatible = "ethernet-phy-id001c.c916";
                        device_type = "ethernet-phy";
                        reg = <0x1>;
                        phy-mode = "rgmii";
                };
    };
};
```

#### TX phase and RX phase

The default value for tx-phase is `90`, and for rx-phase it is `73`.

Different boards may require adjustments to tx-phase and rx-phase. If the eth0 port can be brought up but fails to obtain an IP address, it is necessary to contact the relevant support team to adjust the tx-phase and rx-phase.

```c
&eth0 {
    tx-phase = <90>;
    rx-phase = <73>;
};
```

#### DTS Configuration

Integrating the Ethernet hardware information of the development board, the configuration is as follows:

```c
&eth0 {
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_gmac0>;

        emac,reset-gpio = <&gpio 110 0>;
        emac,reset-active-low;
        emac,reset-delays-us = <0 10000 100000>;

        /* store forward mode */
        tx-threshold = <1518>;
        rx-threshold = <12>;
        tx-ring-num = <128>;
        rx-ring-num = <128>;
        dma-burst-len = <5>;

        ref-clock-from-phy;

        clk-tuning-enable;
        clk-tuning-by-delayline;
        tx-phase = <90>;
        rx-phase = <73>;

        phy-handle = <&rgmii0>;

        status = "okay";

        mdio-bus {
                #address-cells = <0x1>;
                #size-cells = <0x0>;
                rgmii0: phy@0 {
                        compatible = "ethernet-phy-id001c.c916";
                        device_type = "ethernet-phy";
                        reg = <0x1>;
                        phy-mode = "rgmii";
                };
        };
};
```

## Interface

### API

#### emac_ioctl

`emac_ioctl` is used to access the `PHY` (Physical Layer) device registers and configure the hardware timestamp feature. The meanings of the commands are as follows:

- `SIOCGMIIPHY`: Get the address of the `PHY` device.
- `SIOCGMIIREG`: Read the specified `PHY` register.
- `SIOCSMIIREG`: Write to the specified `PHY` register.
- `SIOCSHWTSTAMP`: Configure the device's hardware timestamp.

```c
static int emac_ioctl(struct net_device *ndev, struct ifreq *rq, int cmd)
{
 int ret = -EOPNOTSUPP;

 if (!netif_running(ndev))
  return -EINVAL;

 switch (cmd) {
 case SIOCGMIIPHY:
 case SIOCGMIIREG:
 case SIOCSMIIREG:
  if (!ndev->phydev)
   return -EINVAL;
  ret = phy_mii_ioctl(ndev->phydev, rq, cmd);
  break;
 case SIOCSHWTSTAMP:
  ret = emac_hwtstamp_ioctl(ndev, rq);
  break;
 default:
  break;
 }

 return ret;
}
```

#### emac_get_link_ksettings

The `emac_get_link_ksettings` function is used to retrieve link information for a network device, such as speed, duplex mode, and autonegotiation status. This function is called when the user executes the `ethtool <INTERFACE>` command:

```
# ethtool eth0
Settings for eth0:
        Supported ports: [ TP MII ]
        Supported link modes:   10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
        Supported pause frame use: Symmetric Receive-only
        Supports auto-negotiation: Yes
        Supported FEC modes: Not reported
        Advertised link modes:  10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
        Advertised pause frame use: No
        Advertised auto-negotiation: Yes
        Advertised FEC modes: Not reported
        Speed: Unknown!
        Duplex: Unknown! (255)
        Port: Twisted Pair
        PHYAD: 1
        Transceiver: external
        Auto-negotiation: on
        MDI-X: Unknown
        Link detected: no
```

The implementation of the `emac_get_link_ksettings` function is as follows:

```c
static int emac_get_link_ksettings(struct net_device *ndev,
     struct ethtool_link_ksettings *cmd)
{
 if (!ndev->phydev)
                return -ENODEV;
        /* Get physical layer link info via PHY driver interface */
 phy_ethtool_ksettings_get(ndev->phydev, cmd); 
 return 0;
}
```

#### emac_set_link_ksettings

The `emac_set_link_ksettings` function is used to configure the link settings of a network device, such as speed, duplex mode, and autonegotiation status. This function is invoked when the user runs the following command:

```
#Set link speed to 1000M, full-duplex, and enable autonegotiation
ethtool -s eth0 speed 1000 duplex full autoneg on
```

The implementation of the `emac_set_link_ksettings` function is as follows:

```c
static int emac_set_link_ksettings(struct net_device *ndev,
     const struct ethtool_link_ksettings *cmd)
{
 if (!ndev->phydev)
                return -ENODEV;
        /* Call the phy driver interface to configure the physical layer link settings. */
 return phy_ethtool_ksettings_set(ndev->phydev, cmd);
}
```

#### emac_get_ethtool_stats

`emac_get_ethtool_stats` is used to obtain GMAC statistics, and all the statistical items are as follows:

```c
struct emac_hw_stats {
    u32 tx_ok_pkts;                // Number of successfully transmitted data packets
    u32 tx_total_pkts;             // Total number of packets attempted to be transmitted, including successful and failed ones
    u32 tx_ok_bytes;               // Total number of bytes successfully transmitted
    u32 tx_err_pkts;               // Number of packets that encountered errors during transmission
    u32 tx_singleclsn_pkts;        // Number of packets successfully transmitted after a single collision
    u32 tx_multiclsn_pkts;         // Number of packets successfully transmitted after multiple collisions
    u32 tx_lateclsn_pkts;          // Number of packets discarded due to late collisions (detected during transmission)
    u32 tx_excessclsn_pkts;        // Number of packets that failed transmission due to excessive collisions
    u32 tx_unicast_pkts;           // Number of unicast packets successfully transmitted
    u32 tx_multicast_pkts;         // Number of multicast packets successfully transmitted
    u32 tx_broadcast_pkts;         // Number of broadcast packets successfully transmitted
    u32 tx_pause_pkts;             // Number of control packets transmitted (e.g., pause frames for flow control)
    u32 rx_ok_pkts;                // Number of successfully received valid data packets
    u32 rx_total_pkts;             // Total number of packets received, including successful and failed ones
    u32 rx_crc_err_pkts;           // Number of packets received with CRC errors detected
    u32 rx_align_err_pkts;         // Number of packets received with alignment errors detected
    u32 rx_err_total_pkts;         // Total number of received packets that encountered errors
    u32 rx_ok_bytes;               // Total number of bytes successfully received
    u32 rx_total_bytes;            // Total number of bytes received, including successful and failed packets
    u32 rx_unicast_pkts;           // Number of unicast packets successfully received
    u32 rx_multicast_pkts;         // Number of multicast packets successfully received
    u32 rx_broadcast_pkts;         // Number of broadcast packets successfully received
    u32 rx_pause_pkts;             // Number of control packets received (e.g., pause frames for flow control)
    u32 rx_len_err_pkts;           // Number of packets that failed reception due to length errors
    u32 rx_len_undersize_pkts;     // Number of packets received that are too short (less than the minimum Ethernet frame length)
    u32 rx_len_oversize_pkts;      // Number of packets received that are too long (exceeding the maximum Ethernet frame length)
    u32 rx_len_fragment_pkts;      // Number of fragmented packets received (incomplete packets)
    u32 rx_len_jabber_pkts;        // Number of jabber packets received (overlong and containing errors)
    u32 rx_64_pkts;                // Number of packets received with a length of 64 bytes
    u32 rx_65_127_pkts;            // Number of packets received with a length between 65 and 127 bytes
    u32 rx_128_255_pkts;           // Number of packets received with a length between 128 and 255 bytes
    u32 rx_256_511_pkts;           // Number of packets received with a length between 256 and 511 bytes
    u32 rx_512_1023_pkts;          // Number of packets received with a length between 512 and 1023 bytes
    u32 rx_1024_1518_pkts;         // Number of packets received with a length between 1024 and 1518 bytes
    u32 rx_1519_plus_pkts;         // Number of packets received with a length greater than 1518 bytes
    u32 rx_drp_fifo_full_pkts;     // Number of packets discarded due to the receive FIFO being full
    u32 rx_truncate_fifo_full_pkts;// Number of packets truncated due to the receive FIFO being full
    // Must be placed at the end
    spinlock_t stats_lock;         // Spinlock used to protect access to the above statistics to prevent data races
};
```

The function is called when the user executes the following command:

```
# ethtool -S eth0
NIC statistics:
     tx_ok_pkts: 219
     tx_total_pkts: 219
     tx_ok_bytes: 20102
     tx_err_pkts: 0
     tx_singleclsn_pkts: 0
     tx_multiclsn_pkts: 0
     tx_lateclsn_pkts: 0
     tx_excessclsn_pkts: 0
     tx_unicast_pkts: 4
     tx_multicast_pkts: 187
     tx_broadcast_pkts: 28
     tx_pause_pkts: 0
     rx_ok_pkts: 209
     rx_total_pkts: 209
     rx_crc_err_pkts: 0
     rx_align_err_pkts: 0
     rx_err_total_pkts: 0
     rx_ok_bytes: 18368
     rx_total_bytes: 18368
     rx_unicast_pkts: 3
     rx_multicast_pkts: 175
     rx_broadcast_pkts: 31
     rx_pause_pkts: 0
     rx_len_err_pkts: 0
     rx_len_undersize_pkts: 0
     rx_len_oversize_pkts: 0
     rx_len_fragment_pkts: 0
     rx_len_jabber_pkts: 0
     rx_64_pkts: 17
     rx_65_127_pkts: 177
     rx_128_255_pkts: 0
     rx_256_511_pkts: 15
     rx_512_1023_pkts: 0
     rx_1024_1518_pkts: 0
     rx_1519_plus_pkts: 0
     rx_drp_fifo_full_pkts: 0
     rx_truncate_fifo_full_pkts: 0
```

The implementation of the function is as follows:

```c
static void emac_get_ethtool_stats(struct net_device *dev,
                                   struct ethtool_stats *stats, u64 *data)
{
        struct emac_priv *priv = netdev_priv(dev);
        struct emac_hw_stats *hwstats = priv->hw_stats;
        u32 *data_src;
        u64 *data_dst;
        int i;

        /* Ensure that the network device exists and is operational. */
        if (netif_running(dev) && netif_device_present(dev)) {
                if (spin_trylock_bh(&hwstats->stats_lock)) {
                        emac_stats_update(priv);  // Update the statistics.
                        spin_unlock_bh(&hwstats->stats_lock);
                }
        }

        data_dst = data; 

        /* Iterate through the ethtool statistics array and copy the hardware statistics to the target data array. */
        for (i = 0; i < ARRAY_SIZE(emac_ethtool_stats); i++) {
                data_src = (u32 *)hwstats + emac_ethtool_stats[i].offset;
                *data_dst++ = (u64)(*data_src);
        }
}
```

#### emac_get_ts_info

The `emac_get_ts_info` function is used to provide timestamping information for the network device. This function is called when the user executes the following command:

```
# ethtool --show-time-stamping eth0
Time stamping parameters for eth0:
Capabilities:
        hardware-transmit     (SOF_TIMESTAMPING_TX_HARDWARE)
        software-transmit     (SOF_TIMESTAMPING_TX_SOFTWARE)
        hardware-receive      (SOF_TIMESTAMPING_RX_HARDWARE)
        software-receive      (SOF_TIMESTAMPING_RX_SOFTWARE)
        software-system-clock (SOF_TIMESTAMPING_SOFTWARE)
        hardware-raw-clock    (SOF_TIMESTAMPING_RAW_HARDWARE)
PTP Hardware Clock: 0
Hardware Transmit Timestamp Modes:
        off                   (HWTSTAMP_TX_OFF)
        on                    (HWTSTAMP_TX_ON)
Hardware Receive Filter Modes:
        none                  (HWTSTAMP_FILTER_NONE)
        all                   (HWTSTAMP_FILTER_ALL)
        ptpv1-l4-event        (HWTSTAMP_FILTER_PTP_V1_L4_EVENT)
        ptpv1-l4-sync         (HWTSTAMP_FILTER_PTP_V1_L4_SYNC)
        ptpv1-l4-delay-req    (HWTSTAMP_FILTER_PTP_V1_L4_DELAY_REQ)
        ptpv2-l4-event        (HWTSTAMP_FILTER_PTP_V2_L4_EVENT)
        ptpv2-l4-sync         (HWTSTAMP_FILTER_PTP_V2_L4_SYNC)
        ptpv2-l4-delay-req    (HWTSTAMP_FILTER_PTP_V2_L4_DELAY_REQ)
        ptpv2-event           (HWTSTAMP_FILTER_PTP_V2_EVENT)
        ptpv2-sync            (HWTSTAMP_FILTER_PTP_V2_SYNC)
        ptpv2-delay-req       (HWTSTAMP_FILTER_PTP_V2_DELAY_REQ)
```

The implementation of the `emac_get_ts_info` function is as follows:

```c
static int emac_get_ts_info(struct net_device *dev,
                              struct ethtool_ts_info *info)
{
        struct emac_priv *priv = netdev_priv(dev); 
        if (priv->ptp_support) {

                /* Set the supported timestamping options, including both hardware and software timestamps. */
                info->so_timestamping = SOF_TIMESTAMPING_TX_SOFTWARE |
                                        SOF_TIMESTAMPING_TX_HARDWARE |
                                        SOF_TIMESTAMPING_RX_SOFTWARE |
                                        SOF_TIMESTAMPING_RX_HARDWARE |
                                        SOF_TIMESTAMPING_SOFTWARE |
                                        SOF_TIMESTAMPING_RAW_HARDWARE;

                if (priv->ptp_clock)
                        info->phc_index = ptp_clock_index(priv->ptp_clock);

                /* Set the supported transmit timestamp types. */
                info->tx_types = (1 << HWTSTAMP_TX_OFF) | (1 << HWTSTAMP_TX_ON);

                /* Set the supported timestamp filters */
                info->rx_filters = ((1 << HWTSTAMP_FILTER_NONE) |
                                    (1 << HWTSTAMP_FILTER_PTP_V1_L4_EVENT) |
                                    (1 << HWTSTAMP_FILTER_PTP_V1_L4_SYNC) |
                                    (1 << HWTSTAMP_FILTER_PTP_V1_L4_DELAY_REQ) |
                                    (1 << HWTSTAMP_FILTER_PTP_V2_L4_EVENT) |
                                    (1 << HWTSTAMP_FILTER_PTP_V2_L4_SYNC) |
                                    (1 << HWTSTAMP_FILTER_PTP_V2_L4_DELAY_REQ) |
                                    (1 << HWTSTAMP_FILTER_PTP_V2_EVENT) |
                                    (1 << HWTSTAMP_FILTER_PTP_V2_SYNC) |
                                    (1 << HWTSTAMP_FILTER_PTP_V2_DELAY_REQ) |
                                    (1 << HWTSTAMP_FILTER_ALL));
                return 0;
        } else
                /* If PTP (Precision Time Protocol) is not supported, call the default handler function provided by ethtool */
                return ethtool_op_get_ts_info(dev, info);
}

```

## Debugging

### debugfs

For conveniently querying or modifying the GMAC interface, TX phase, and RX phase configuration

```
/sys/kernel/debug/cac80000.ethernet # cat clk_tuning
Emac MII Interface : RGMII
Current rx phase : 73
Current tx phase : 60
```

## Testing

Check the network interface information.

```c
ifconfig -a
```

Open the network device.

```c
ifconfig <INTERFACE> up
```

Close the network device.

```c
ifconfig <INTERFACE> down
```

Test the connectivity with another host, assuming its IP address is `192.168.0.1`

```c
ping 192.168.0.1 
```

Obtain an IP address using the DHCP protocol.

```c
udhcpc
```

## FAQ
