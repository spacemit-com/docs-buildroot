# UFS

This document describes the structure, configuration methods, and debugging points of the K3 UFS host controller driver. Based on `linux-6.18/drivers/ufs/host/ufs-spacemit.c`, it is intended for UFS driver development, board-level adaptation, and issue troubleshooting.

## Overview

UFS (Universal Flash Storage) is a serial interface for high-speed embedded storage. The current implementation on K3 adopts a layered architecture consisting of 
- The Linux generic UFS Core (`ufshcd`)
- The platform layer (`ufshcd-pltfrm`)
- The SpacemiT K3 variant driver (`ufs-spacemit`)

UFS devices are ultimately exposed to the system through the SCSI subsystem, usually as `/dev/sdX` and its partition nodes.

### Functionality

The K3 UFS software stack can be divided into the following layers:

- **UFS Core**: Responsible for generic logic such as UTP transport, UIC commands, SCSI passthrough, error recovery, and sysfs/debugfs interfaces.
- **Platform Glue**: Responsible for parsing generic DT properties, clock lists, lane counts, etc.
- **K3 Variant Driver**: Responsible for K3 controller proprietary registers, M-PHY initialization, UniPro parameter tuning, Hibern8 entry/exit sequences, and K3-specific quirks.
- **SCSI / Block Layer**: Exposes the UFS device as a block device and partitions, enabling file system and raw block access.

### Source Code Structure

K3 UFS-related code and descriptor files are mainly located in the following paths:

```text
linux-6.18/
|-- drivers/ufs/Kconfig
|-- drivers/ufs/core/
|   |-- ufshcd.c
|   |-- ufs-sysfs.c
|   |-- ufs-debugfs.c
|-- drivers/ufs/host/
|   |-- ufshcd-pltfrm.c
|   |-- ufs-spacemit.c
|   `-- ufs-spacemit.h
|-- arch/riscv/configs/k3_defconfig
`-- arch/riscv/boot/dts/spacemit/
    |-- k3.dtsi
    |-- k3_deb1.dts
    |-- k3_evb.dts
    `-- ...
```

- `drivers/ufs/host/ufs-spacemit.c` is the main implementation of the K3 UFS variant driver;
- `drivers/ufs/host/ufs-spacemit.h` defines K3 proprietary registers, link capabilities and host proprietary data structures;
- `drivers/ufs/host/ufshcd-pltfrm.c` is responsible for parsing generic properties such as `freq-table-hz` and `lanes-per-direction`;
- `drivers/ufs/core/ufshcd.c` is the UFS Core and handles probe, SCSI enumeration, power management, error recovery, etc;
- `drivers/ufs/core/ufs-sysfs.c` and `ufs-debugfs.c` provide sysfs and debugfs debugging interfaces respectively;  
- `arch/riscv/boot/dts/spacemit/k3.dtsi` defines the K3 UFS controller node;
- Board-level DTS (such as `k3_deb1.dts` and `k3_evb.dts`) typically only enable the `&ufshc` node as needed.

## Key Features

| Item | Current Implementation |
| :----- | :----- |
| Driver Model | `ufshcd` Core + `ufshcd-pltfrm` + `ufs-spacemit` |
| Device Tree Compatible | `spacemit,k3-ufshcd` |
| Lane capability | Up to 2 lanes per direction |
| Max HS Gear | HS Gear 3 |
| Max PWM Gear | PWM Gear 4 |
| HS Rate | `PA_HS_MODE_B` |
| UFS ACLK | Supports configuration to `409600000`, `491520000`, `500000000`, `600000000` Hz; DTS currently configures `491520000` Hz |
| Ref Clock | Reference DTS currently configures `19200000` Hz |
| Hibern8 Strategy | Auto-Hibern8 is disabled; custom variant timing is only used when Hibern8 entry/exit is explicitly initiated by the UFS Core. The default PM policy does not enter Hibern8 |
| Default PM Level | `rpm_lvl = UFS_PM_LVL_2`, `spm_lvl = UFS_PM_LVL_2` |
| Standby Special Handling | Temporarily raises to `UFS_PM_LVL_5` during `standby` |
| Error Observation | Supports vendor register dump, PA/DL/ABORT event logging |
| Current Concurrency Limit | SCSI queue depth per LU is currently fixed at `1` |

## Configuration

This mainly includes **Kconfig configuration** and **DTS configuration**.

### Kconfig Configuration

The basic configuration required for K3 UFS includes at least:

- `CONFIG_SCSI_UFSHCD`
- `CONFIG_SCSI_UFSHCD_PLATFORM`
- `CONFIG_SCSI_UFS_SPACEMIT_K3`

`arch/riscv/configs/k3_defconfig` current configuration is as follows:

```text
CONFIG_SCSI_UFSHCD=y
CONFIG_SCSI_UFSHCD_PLATFORM=y
CONFIG_SCSI_UFS_SPACEMIT_K3=y
```

Relevant Kconfig paths:

```text
Device Drivers
        SCSI device support
        Universal Flash Storage Controller (SCSI_UFSHCD)
                Platform bus based UFS Controller support (SCSI_UFSHCD_PLATFORM)
                Spacemit K3 specific hooks to UFS controller platform driver (SCSI_UFS_SPACEMIT_K3)
```

Common configuration options for developing and debugging:

- `CONFIG_DEBUG_FS`  
  Used to enable the `debugfs` node provided by `drivers/ufs/core/ufs-debugfs.c`;
- `CONFIG_SCSI_UFS_BSG`  
  Enable this option when sending UPIU / Query requests through BSG device node.

**Note:**

- If the root file is located on the UFS device, it is not recommended to compile `CONFIG_SCSI_UFSHCD` as a module. 

### DTS Configuration 

#### 1. SoC Basic Node

The base nodes of the K3 UFS controller are defined in `arch/riscv/boot/dts/spacemit/k3.dtsi`:

```dts
ufshc: ufshc@0xc0e00000 {
    #address-cells = <2>;
    #size-cells = <2>;
    compatible = "spacemit,k3-ufshcd";
    reg = <0x0 0xc0e00000 0x0 0x40000>;
    interrupt-parent = <&saplic>;
    interrupts = <135 0x4>;
    clocks = <&syscon_apmu CLK_APMU_UFS_ACLK>;
    clock-names = "ufs-aclk";
    resets = <&syscon_apmu RESET_APMU_UFS_ACLK>;
    reset-names = "ufs-aclk-rst";
    clock-freq = <491520000>;
    freq-table-hz = <491520000 491520000>;
    lanes-per-direction = <2>;
    ref-clk-freq = <19200000>;
    status = "disabled";
};
```

The meanings of several key properties are as follows:

- `clock-names = "ufs-aclk"`: The K3 variant retrieves the main clock through this name;
- `reset-names = "ufs-aclk-rst"`: Probe asserts/deasserts this reset line early;
- `clock-freq`: A K3 variant proprietary property. `ufs_spacemit_platform_init()` uses it with higher priority to set the `ufs-aclk` frequency. The currently supported `ufs-aclk` values are `409600000`, `491520000`, `500000000`, and `600000000` Hz;
- `freq-table-hz`: A generic frequency table property for the UFS platform layer;
- `lanes-per-direction`: The platform layer parses this property and stores it in `hba->lanes_per_direction`. The default value is `2`;
- `ref-clk-freq`: Used by the UFS Core to enumerate the reference clock frequency. The current configuration is 19.2 MHz. 

#### 2. Board-Level Enablement

Most current board-level DTS for K3 reuse the base SoC node, so typically only enabling the node at the board level is required:

```dts
&ufshc {
    status = "okay";
};
```

Board-level files in the repository that already use this pattern include:

- `k3_deb1.dts`
- `k3_evb.dts`
- `k3_evb2-1.dts`
- `k3_evb2-2.dts`
- `k3_com260.dts`
- `k3_com260_kit_v02.dts`
- `k3_gemini_c0.dts`
- `k3_gemini_c1.dts`

#### 3. Power Supply

On the current K3 platform, UFS power supply is handled by the **Efficiency core (ESOS)**. The `ufshc` node should not configure the following supply properties:

- `vdd-hba-supply`
- `vcc-supply`
- `vccq-supply`
- `vccq2-supply`

Refer to [ESOS Power Management](../../esos/esos_power.md) for the overall power supply control mechanism.

It is necessary to distinguish between the following two meanings:

- **Generic UFS Core capability**: `ufshcd-pltfrm.c` / `ufshcd.c` support parsing `*-supply` properties and obtaining, enabling, and disabling these power supplies through the regulator framework.
- **Constraints of the current K3 Platform**: The efficiency core handles UFS supply centrally, so the `ufshc` node cannot provide these supply phandles. The UFS driver must not take over power supply control.

#### 4. Suggestions for New Board Development

It is recommended to follow the check sequence below when porting UFS to a new board:

1. First, inherit the existing `ufshc` base node from `k3.dtsi`, and only change the `status` property to `"okay"` in the board-level DTS file.
2. Ensure the actual input frequency of `ufs-aclk` matches `clock-freq`, or parameters such as `UFS_SYS1CLK_1US` and link startup timing may be incorrectly programmed.
    It is recommended to select `clock-freq` from `409600000`, `491520000`, `500000000`, and `600000000`.
3. Do not add `*-supply` properties to the `ufshc` on the current K3 platform. UFS power supply is still handled by the efficiency core (ESOS). See [ESOS Power Management](../../esos/esos_power.md) for detailed information.

## Interface

### Key Variant Ops Callbacks
 
The K3 UFS variant connects to the UFS Core through `struct ufs_hba_variant_ops`. The main callbacks currently implemented are as follows:

| Callback | Role | Focus |
| :----- | :----- | :----- |
| `init` | Initialize host private data, set the default PM level, and register quirks | Currently sets both `rpm_lvl` and `spm_lvl` to `UFS_PM_LVL_2` |
| `link_startup_notify` | Pre/post link startup callbacks | Performs M-PHY initialization and UniPro parameter writes in the PRE phase; handles version compatibility in the POST phase |
| `pwr_change_notify` | Power mode negotiation | Currently capped at 2-lane, HS-G3, Rate-B  |
| `setup_clocks` | Restore `ufs-aclk` frequency after clocks are enabled | Currently only effective on the `POST_CHANGE && on` path, depends on `clock-freq` / `freq-table-hz` |
| `setup_xfer_req` | Barrier before request submission | Inserts `wmb()` before ringing the doorbell to mitigate command loss under high-concurrency I/O |
| `config_scsi_dev` | Configure each SCSI device | Currently enforces queue depth to 1 to avoid stability issues |
| `device_reset` | Device reset sequence | Skipped on first initialization; subsequent resets clear M-PHY related registers |
| `hibern8_notify` | Hibern8 entry/exit sequence | Currently only handles explicit Hibern8 enter/exit initiated by the UFS Core; does not rely on Auto-Hibern8, nor is it equivalent to all suspend/resume paths |
| `event_notify` | Error event observation | Focus on three types of events: PA / DL / ABORT |
| `apply_dev_quirks` | Adjust link parameters based on device quirks | Includes LCC disable, `TX_MIN_ACTIVATETIME` adjustment, and WDC-specific handling |
| `hce_enable_notify` | Host Controller Enable sequence control | Different handling paths for the first and subsequent HCE operations |
| `dbg_register_dump` | Vendor register dump | Used to assist in locating private register states when errors occur |

### Key Timing Sequence

#### 1. Probe and Platform Initialization

The key path of `probe()` is as follows:  

1. Match the device tree node by `compatible = "spacemit,k3-ufshcd"`.
2. Enter `ufs_spacemit_platform_init()`:
    - Obtain `ufs-aclk`
    - Obtain `ufs-aclk-rst`
    - Perform reset assert/deassert
    - Set the `ufs-aclk` frequency according to `clock-freq` or `freq-table-hz`
3. Call `ufshcd_pltfrm_init()` to enter the generic UFS Core initialization flow.

If the probe fails early, check the following first:

- Whether `ufs-aclk` can be obtained 
- Whether `ufs-aclk-rst` exists
- Whether `clock-freq` is valid
- Whether `status = "okay"` is enabled in the final effective board-level DTS

#### 2. Pre-Link Startup Handling

In `link_startup_notify(PRE_CHANGE)`, the K3 variant performs three tasks:

- Calls `ufs_spacemit_mphy_init()` to perform basic sequences such as M-PHY reset, power-up, device reset deassert, and PLL lock.
- Calls `ufs_spacemit_uniprov1p6_init()` to write a set of UniPro / M-PHY tuning registers.
- Programs `UFS_SYS1CLK_1US`, `UFS_TX_SYMBOL_CLK_NS_US` and `UFS_PA_LINK_STARTUP_TIMER` according to the current `ufs-aclk` frequency.

This is the core stage of K3 UFS bring-up. Problems such as incorrect clock configuration, PLL unlock, or abnormal M-PHY sequencing are usually exposed here.

#### 3. Post-Link Startup Handling

In `link_startup_notify(POST_CHANGE)`, the current implementation will:

- Read the local/peer UniPro version
- Send a dummy frame for compatibility handling in scenarios with some older peer versions
- Set `DL_AFC0REQTIMEOUTVAL` to its maximum value

If the link comes up unreliably, check whether the compatibility handling at this stage is actually taking effect.

#### 4. Power Mode Negotiation

`pwr_change_notify(PRE_CHANGE)` will call `ufshcd_negotiate_pwr_params()`, and limit K3 default capabilities to:

- RX / TX lane: 2
- HS Gear: G3
- PWM Gear: G4
- PWM power mode: `SLOW_MODE`
- HS power mode: `FAST_MODE`
- HS rate: `PA_HS_MODE_B`

In the `POST_CHANGE` stage, the negotiation results are cached and the M-PHY PLL lock is checked again.

#### 5. Hibern8 and System Suspend

On K3, link-level Hibern8 and system-level suspend must be considered separately. In the current implementation, not all suspend/resume paths will enter Hibern8:

- `hibern8_notify()` is only executed when the UFS Core issues `UIC_CMD_DME_HIBER_ENTER` / `UIC_CMD_DME_HIBER_EXIT`. 

- The K3 variant explicitly sets `UFSHCD_QUIRK_BROKEN_AUTO_HIBERN8`. For stability, HCI Auto-Hibern8 is currently disabled; reading or writing `$UFS_DEV/auto_hibern8` on this platform returns `Operation not supported`, which is expected.

- `rpm_lvl = UFS_PM_LVL_2` and `spm_lvl = UFS_PM_LVL_2` correspond to `UFS_SLEEP_PWR_MODE + UIC_LINK_ACTIVE_STATE` by default. Default runtime suspend and general system suspend path do not automatically put the link into Hibern8.

- The K3-specific custom Hibern8 sequence is only triggered by the UFS Core when the PM level is set to a profile whose link target is `HIBERN8`. The current target state can be verified directly via `$UFS_DEV/rpm_target_link_state` and `$UFS_DEV/spm_target_link_state`.

- The K3 Hibern8 exit path re-enables the reference clock, restores the internal M-PHY power state, and waits for PLL lock; the entry path adjusts several RX/TX MIBs and powers down the M-PHY in the POST phase.

- `standby` uses a separate path. When `ufs_spacemit_suspend_prepare()` runs with `pm_suspend_target_state == PM_SUSPEND_STANDBY`, `spm_lvl` is temporarily raised to `UFS_PM_LVL_5`, and `resume_complete()` restores the original value. `UFS_PM_LVL_5` corresponds to `UFS_POWERDOWN_PWR_MODE + UIC_LINK_OFF_STATE`, which verifies power-down/link-off recovery rather than Hibern8 entry/exit.

Therefore, if the UFS device is lost, re-enumerated, or hangs after `standby`, first debug power-down/link-off recovery. If failures occur after explicitly using a Hibern8-level profile, focus on the PLL lock and MIB restore timing in `hibern8_notify()`.

#### 6. Current Implementation Limitations

There are several limitations in the current implementation that should be explicitly documented during development:

- `config_scsi_dev()` limits the queue depth for each LU to `1` to avoid stability issues under high-concurrency random I/O.
- Auto-Hibern8 is currently disabled to avoid Hibern8-related stability issues. Therefore, `auto_hibern8` cannot be considered an available tuning option.
- The default PM policy intentionally avoids Hibern8: `rpm_lvl` and `spm_lvl` both default to `2`, while `standby` temporarily forces the level to `5`.

## Debugging

### Kernel Logs

The most direct way to debug is to check the kernel logs:

```bash
dmesg | grep -iE 'ufs|ufshcd|scsi'
```

It is recommended to focus on the following key information:

- `M-PHY PLL lock timeout`
- `Runtime suspend failed`
- `Runtime resume failed`
- `queue depth limited to 1`
- Error prints related to PA / DL / ABORT

If the UFS Core triggers a register dump, `dbg_register_dump` of K3 variant will additionally print vendor-specific registers, M-PHY registers and ATOP related registers to the log.

### sysfs

The UFS Core automatically creates a set of sysfs nodes. You can first locate the UFS platform device:

```bash
UFS_DEV=/sys/bus/platform/devices/c0e00000.ufshc
echo "$UFS_DEV"
```

Then focus on:

```bash
cat $UFS_DEV/rpm_lvl
cat $UFS_DEV/rpm_target_dev_state
cat $UFS_DEV/rpm_target_link_state

cat $UFS_DEV/spm_lvl
cat $UFS_DEV/spm_target_dev_state
cat $UFS_DEV/spm_target_link_state

cat $UFS_DEV/auto_hibern8 2>/dev/null || echo "auto_hibern8: Operation not supported (expected on K3)"

cat $UFS_DEV/power_info/lane
cat $UFS_DEV/power_info/mode
cat $UFS_DEV/power_info/rate
cat $UFS_DEV/power_info/gear
cat $UFS_DEV/power_info/dev_pm
cat $UFS_DEV/power_info/link_state
```

The meanings of these nodes:

| Node | Meaning |
| :----- | :----- |
| `$UFS_DEV/rpm_lvl` | Runtime PM level, indicating the target power level used during runtime suspend. Writing to this node affects the device state and link state entered during runtime suspend. |
| `$UFS_DEV/rpm_target_dev_state` | The target device state corresponding to the current `rpm_lvl`. When debugging why Hibern8 / powerdown is not entered, check this first rather than just the level number.|
| `$UFS_DEV/rpm_target_link_state` | The target link state corresponding to the current `rpm_lvl`. If this is not `HIBERN8`, the runtime suspend path will not enter Hibern8. |
| `$UFS_DEV/spm_lvl` | System PM level, indicating the target power level used during system suspend. Writing to this node affects the device state and link state that will be entered during system suspend. |
| `$UFS_DEV/spm_target_dev_state` | The target device state corresponding to the current `spm_lvl`. Useful for distinguishing between normal suspend, explicit Hibern8 suspend, and the `standby` power-down path. |
| `$UFS_DEV/spm_target_link_state` | The target link state corresponding to the current `spm_lvl`. With the default `spm_lvl = 2`, this will show `ACTIVE`; if changed to a Hibern8 level before suspend, it will show `HIBERN8`. The temporary switch to `OFF` inside `standby` only occurs during the suspend prepare phase, and the node will revert to its original configuration after resume. |
| `$UFS_DEV/auto_hibern8` | Generic UFS Core Auto-Hibern8 idle timer interface. If supported by the platform, it shows how long the link must be idle before automatically entering Hibern8; on K3, this is expected to return `Operation not supported`. |
| `$UFS_DEV/power_info/lane` | The negotiated lane count of the current link. The actual field displayed in code is `lane_rx`. It can be used to confirm whether the link is running in 1-lane or 2-lane mode. |
| `$UFS_DEV/power_info/mode` | The current link power mode. Common values include `FAST_MODE`, `SLOW_MODE`, `FASTAUTO_MODE` and `SLOWAUTO_MODE`. |
| `$UFS_DEV/power_info/rate` | The current HS rate gear. Common values include `HS_RATE_A` or `HS_RATE_B`. |
| `$UFS_DEV/power_info/gear` | The current actual gear. If the link is in HS mode, it displays `HS_GEARx`; if it is in PWM mode, it displays `PWM_GEARx`. |
| `$UFS_DEV/power_info/dev_pm` | The current UFS device power state. Common values include `ACTIVE`, `SLEEP`, `POWERDOWN`, and `DEEPSLEEP`. |
| `$UFS_DEV/power_info/link_state` | The current UIC link state. Common values include `ACTIVE`, `HIBERN8`, `OFF`, and `BROKEN`. |

Check the following for descriptors and attributes:

```bash
ls $UFS_DEV/device_descriptor
ls $UFS_DEV/interconnect_descriptor
ls $UFS_DEV/geometry_descriptor
ls $UFS_DEV/health_descriptor
ls $UFS_DEV/flags
ls $UFS_DEV/attributes
```

These directories themselves are groups of kernel nodes. Their meanings are:

| Directory | Meaning |
| :----- | :----- |
| `$UFS_DEV/device_descriptor` | Mirror of the UFS Device Descriptor, including static capability information such as device type, LUN count, vendor ID, queue depth, and FFU timeout. |
| `$UFS_DEV/interconnect_descriptor` | Mirror of the UFS Interconnect Descriptor. Focus on the UniPro version and M-PHY version. |
| `$UFS_DEV/geometry_descriptor` | Mirror of the UFS Geometry Descriptor. Focus on raw capacity, block size, allocation unit, and Write Booster-related geometry parameters. |
| `$UFS_DEV/health_descriptor` | Mirror of the UFS Health Descriptor. Focus on lifetime estimation and EOL information. |
| `$UFS_DEV/flags` | A collection of UFS Query Flag nodes, suitable for viewing boolean states such as `device_init`, `wb_enable`, and `busy_rtc`. |
| `$UFS_DEV/attributes` | A collection of UFS Query Attribute nodes, suitable for viewing runtime attributes such as `current_power_mode`, `reference_clock_frequency`, and `exception_event_status`. |

For basic performance observation, you can use the monitor group:

```bash
echo 1 > $UFS_DEV/monitor/monitor_enable
cat $UFS_DEV/monitor/read_total_sectors
cat $UFS_DEV/monitor/write_total_sectors
cat $UFS_DEV/monitor/read_req_latency_avg
cat $UFS_DEV/monitor/write_req_latency_avg
```

The meaning of these nodes:

| Node | Meaning |
| :----- | :----- |
| `$UFS_DEV/monitor/monitor_enable` | Monitor switch. Write `1` to start statistics; write `0` to stop and clear current statistics. |
| `$UFS_DEV/monitor/read_total_sectors` | Total number of sectors read since monitor was enabled.|
| `$UFS_DEV/monitor/write_total_sectors` | Total number of sectors written since monitor was enabled. |
| `$UFS_DEV/monitor/read_req_latency_avg` | Average read request latency, in microseconds. |
| `$UFS_DEV/monitor/write_req_latency_avg` | Average write request latency, in microseconds. |

The `monitor` directory also contains nodes such as `read_total_busy`, `read_nr_requests`, `read_req_latency_max`, `read_req_latency_min`, `write_total_busy` and `write_nr_requests`. These are used to view cumulative busy time, request counts, and maximum/minimum request latencies, respectively. 

**Note:**

- The `auto_hibern8` sysfs entry returning `Operation not supported` on K3 is expected and does not indicate a driver probe failure.
- To verify whether a PM level will use Hibern8, first check `rpm_target_link_state` / `spm_target_link_state`, rather than relying solely on the numerical values of `rpm_lvl` and `spm_lvl`.
- Whether capabilities such as `clock_scaling` and `write_booster` show `1` in sysfs depends on whether the corresponding capability is actually enabled in the variant driver.

### debugfs

If the kernel is built with `CONFIG_DEBUG_FS`, the UFS Core creates directories under `debugfs`:

```bash
if grep -wq debugfs /proc/mounts; then
    :
elif [ -d /sys/kernel/debug ]; then
    mount -t debugfs debugfs /sys/kernel/debug
fi

if [ -d /sys/kernel/debug/ufshcd ]; then
    ls /sys/kernel/debug/ufshcd
else
    echo "ufshcd debugfs unavailable"
fi
```

If `mount` reports `Device or resource busy`, it usually indicates debugfs is mounted and you can proceed directly. 

After entering the specific controller directory, check:

```bash
cat /sys/kernel/debug/ufshcd/c0e00000.ufshc/stats
cat /sys/kernel/debug/ufshcd/c0e00000.ufshc/saved_err
cat /sys/kernel/debug/ufshcd/c0e00000.ufshc/saved_uic_err
cat /sys/kernel/debug/ufshcd/c0e00000.ufshc/exception_event_mask
cat /sys/kernel/debug/ufshcd/c0e00000.ufshc/exception_event_rate_limit_ms
```

The meanings of these nodes:

| Node | Meaning |
| :----- | :----- |
| `/sys/kernel/debug/ufshcd/<dev_name>/stats` | Aggregated UFS error event statistics. Includes counts of PA/DL/NL/TL/DME errors, Auto-Hibern8 errors, Link Startup failures, PM suspend/resume failures, LU resets, host resets, and aborts. |
| `/sys/kernel/debug/ufshcd/<dev_name>/saved_err` | Controller error bitmap saved by the UFS Core, referenced by the error recovery path. Suitable for viewing the most recent or cumulative HCI/interrupt-level errors. |
| `/sys/kernel/debug/ufshcd/<dev_name>/saved_uic_err` | UIC error bitmap saved by the UFS Core, suitable for locating link-layer anomalies related to PA/DL/NL/TL. |
| `/sys/kernel/debug/ufshcd/<dev_name>/exception_event_mask` | User-side exception event mask, controlling which exception events need to be temporarily masked or re-enabled. Written as a bitmask. |
| `/sys/kernel/debug/ufshcd/<dev_name>/exception_event_rate_limit_ms` | Rate limiting time for exception events, in milliseconds. After an exception occurs, the Core delays restoring EE control by this time window to avoid exception storms. |

Both `saved_err` and `saved_uic_err` are "error bitmaps", not stringified results. They are typically used in conjunction with `dmesg`, `stats` and the UFS Core error handling code.

## Testing

### Enumeration and Identification Test

After enabling the node, first verify whether the controller is enumerated correctly:

```bash
dmesg | grep -iE 'ufs|ufshcd|scsi'
cat /proc/scsi/scsi
cat /proc/partitions
ls /dev/sd* 2>/dev/null
```

If the system has successfully identified the UFS disk, the following will appear:

- UFS Host Controller initialization success logs
- Corresponding SCSI disk device
- `/dev/sdX` or `/dev/sdXn` node

If the system provides `lsblk`, you can also run `lsblk -o NAME,SIZE,MODEL` additionally.

### Basic Read & Write Test

If the system boots from UFS, it is recommended to perform filesystem-level tests on the rootfs instead of performing raw block writes to `/dev/sda`.

Taking the current system as an example, the UFS disk is enumerated as `/dev/sda`:

- `/dev/sda` is the entire UFS disk
- `/dev/sda1` and `/dev/sda2` are the preceding 256 MiB partitions
- `/dev/sda3` is the main data partition (about `118.8G`), which is also typically the rootfs partition

First, confirm that the current root filesystem is located on UFS:

```bash
mount | grep ' / '
fdisk -l /dev/sda
df -h /
```

After confirming that `/` is mounted on the UFS partition, non-destructive tests can be performed directly on the rootfs. The following example places test files under `/root`:

```bash
dd if=/dev/zero of=/root/ufs_write_test.bin bs=1M count=256 conv=fsync
md5sum /root/ufs_write_test.bin > /root/ufs_write_test.bin.md5
sync
md5sum -c /root/ufs_write_test.bin.md5
dd if=/root/ufs_write_test.bin of=/dev/null bs=1M
rm -f /root/ufs_write_test.bin /root/ufs_write_test.bin.md5
```
Some directories, such as `/tmp`, may be mounted as `tmpfs` in some configurations and therefore do not represent the actual UFS path. To verify the UFS partition where the rootfs resides, test files should be placed in `/root`, `/var`, or other directories confirmed to be on `/dev/sda3`.

For random I/O stress testing, it is recommended to limit `iodepth` to `1` first, since the current driver limits the queue depth to 1:

```bash
if command -v fio >/dev/null; then
    fio --name=ufs_randrw \
        --filename=/root/fio.bin \
        --rw=randrw \
        --bs=128k \
        --iodepth=1 \
        --size=1G \
        --direct=1 \
        --runtime=60 \
        --time_based
    rm -f /root/fio.bin
else
    echo "fio is not installed in current rootfs"
fi
```

If the available space in the root filesystem is insufficient, `count` or `--size` can be reduced accordingly. Since this is an online root filesystem stress test, confirm that there are no critical background write tasks running before execution.

## FAQ

### 1. Why does the current driver appear to have low concurrency performance?

This is because `config_scsi_dev()` limits the queue depth of each LU to `1`. This is a conservative strategy adopted by the current K3 UFS driver to avoid command loss under high-concurrency random I/O. It is not recommended to remove this limit for performance reasons until the root cause is fully identified.

### 2. Why is there no independent pinctrl configuration in the document?

The pins used by the K3 UFS controller are fixed. Unlike UART, SPI, and other peripherals, there are no pinctrl groups that require separate board-level switching or tuning. Therefore, they are not documented separately.
