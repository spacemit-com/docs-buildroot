# CPUFreq

## Overview

### Functionality

CPUFreq is the Linux kernel subsystem that adjusts CPU operating frequency according to system load, user settings, and power and temperature limits.

K3's device-tree-based frequency scaling uses the generic `cpufreq-dt` driver together with SpacemiT K3 platform code:

- **CPUFreq core:** Manages policies, frequency limits, transition notifications, and sysfs interfaces.
- **Governor:** Selects a target frequency within the permitted range based on load or user requests.
- **cpufreq-dt:** Converts the target frequency into an OPP request and performs the adjustment through the clock and regulator frameworks.
- **K3 platform code:** Selects OPP tables, configures shared frequency domains and multiple clocks, and handles PLL switching.
- **OPP framework:** Manages frequencies and their associated voltage settings.
- **CPUFreq stats:** Provides statistics such as time spent at each frequency and transition counts; it must be enabled separately.

This guide describes K3's device-tree-based frequency-scaling implementation using `cpufreq-dt`. For systems using another CPUFreq driver, check its configuration and interface behavior.

### K3 Frequency-Scaling Flow

The main call path on K3 is:

```text
Governor / user-space sysfs
       |
    CPUFreq core
       |------------------ frequency transition notification
       |                         |
       |                  K3 platform code
       |                  handles PLL and other clock switches
       |
   cpufreq-dt: set_target()
       |
   dev_pm_opp_set_rate()
       |
       +-- Clock framework
       |      |
       |   K3 CPU clock driver
       |
       +-- Regulator framework (X100)
            |
         RPMI regulator
            |
         OpenSBI / RCPU
            |
           PMIC
```

CPUFreq uses **kHz** for frequencies; `cpufreq-dt` converts them to **Hz** when calling the OPP interface.

Voltage adjustments for X100 are handled through the RPMI regulator. CPU clocks are managed by the Linux clock framework and the K3 clock driver.

## Key Features

### Frequency Domains

The current device tree defines two shared frequency domains:

| Domain | CPUs | Normal frequencies | Boost frequencies | Voltage management |
|---|---|---|---|---|
| X100 | CPU0–CPU7 | 614.4 MHz–2.15 GHz | 2.2, 2.3, 2.4 GHz | Voltage configured through `edcdc1` |
| A100 | CPU8–CPU15 | 614.4 MHz–1.8 GHz | 1.85, 1.9, 2.0 GHz | No voltage configured in the current OPP table |

These ranges contain discrete frequency points; not every frequency within the ranges is supported.

CPUs in the same frequency domain share one CPUFreq policy. Changes to its governor, frequency, or limits affect every CPU in that domain.

In a typical configuration where all the CPUs above are enabled, these directories are usually present:

```text
/sys/devices/system/cpu/cpufreq/policy0
/sys/devices/system/cpu/cpufreq/policy8
```

`policy0` normally corresponds to X100, and `policy8` to A100. Confirm the actual mapping using `related_cpus`.

Each domain has two cluster core clocks, `cls0` and `cls1`. The two frequency domains describe shared frequency-scaling relationships; they do not mean that K3 has only two physical clusters.

### X100 OPP Table Selection

The current implementation provides two X100 OPP tables with identical frequency points but different target voltages for normal frequencies.

The driver selects a table from the `pmic-compatible` property of `/cpus` in the final device tree:

| Property | OPP index | Table | Normal target voltage |
|---|---:|---|---:|
| `spacemit,au4562` | 1 | `clst_core_opp_table1_x100` | 0.91 V |
| `spacemit,tda38740` | 0 | `clst_core_opp_table0_x100` | 0.95 V |
| Other value or missing property | 0 | `clst_core_opp_table0_x100` | 0.95 V |

U-Boot reads `TLV_CODE_PMIC_TYPE` from the board EEPROM and attempts to write the PMIC type to `/cpus/pmic-compatible`. Linux uses the final property it receives.

The current driver no longer uses the former four X100 tables or selects them using `svt-dro`.

> The current device-tree target for normal frequencies in the au4562 table is **910000 µV, or 0.91 V**.

### X100 Frequencies and Voltages

Both X100 tables contain these 19 frequencies, in descending order:

| Frequency (MHz) | CPUFreq (kHz) | table0 target (V) | table1 target (V) | Boost |
|---:|---:|---:|---:|---|
| 2400 | 2400000 | 1.00 | 1.00 | Yes |
| 2300 | 2300000 | 1.00 | 1.00 | Yes |
| 2200 | 2200000 | 0.95 | 0.95 | Yes |
| 2150 | 2150000 | 0.95 | 0.91 | No |
| 2100 | 2100000 | 0.95 | 0.91 | No |
| 2000 | 2000000 | 0.95 | 0.91 | No |
| 1900 | 1900000 | 0.95 | 0.91 | No |
| 1850 | 1850000 | 0.95 | 0.91 | No |
| 1800 | 1800000 | 0.95 | 0.91 | No |
| 1700 | 1700000 | 0.95 | 0.91 | No |
| 1600 | 1600000 | 0.95 | 0.91 | No |
| 1500 | 1500000 | 0.95 | 0.91 | No |
| 1400 | 1400000 | 0.95 | 0.91 | No |
| 1300 | 1300000 | 0.95 | 0.91 | No |
| 1200 | 1200000 | 0.95 | 0.91 | No |
| 1100 | 1100000 | 0.95 | 0.91 | No |
| 1000 | 1000000 | 0.95 | 0.91 | No |
| 819.2 | 819200 | 0.95 | 0.91 | No |
| 614.4 | 614400 | 0.95 | 0.91 | No |

These values are the target voltages specified by `opp-microvolt`. The configuration also specifies minimum and maximum permitted voltages; the final setting depends on regulator capabilities and board constraints.

**2.2 GHz is also marked `turbo-mode` in the current device tree and is a Boost frequency.**

### A100 Frequency Configuration

The current A100 table contains 14 frequencies, in descending order:

| Frequency (MHz) | CPUFreq (kHz) | Boost |
|---:|---:|---|
| 2000 | 2000000 | Yes |
| 1900 | 1900000 | Yes |
| 1850 | 1850000 | Yes |
| 1800 | 1800000 | No |
| 1700 | 1700000 | No |
| 1600 | 1600000 | No |
| 1500 | 1500000 | No |
| 1400 | 1400000 | No |
| 1300 | 1300000 | No |
| 1200 | 1200000 | No |
| 1100 | 1100000 | No |
| 1000 | 1000000 | No |
| 819.2 | 819200 | No |
| 614.4 | 614400 | No |

The A100 OPP table has no `opp-microvolt`, and K3 platform code does not configure an OPP regulator for CPU8 and above.

The A100 CPUFreq path described here therefore changes frequency without adjusting voltage through that path.

### High-Frequency PLL Switching

K3 platform code handles high-frequency PLL switching in response to `CPUFREQ_PRECHANGE` notifications:

1. For a target above **1 GHz**, configure the two PLLs for the domain.
2. If the old frequency is also above **1 GHz**, first switch the two OPP-managed cluster core clocks to **819.2 MHz**, then adjust the PLLs.
3. The frequency transition then proceeds to the target OPP.

`TURBO0_FREQUENCY` in the source is 1 GHz, the threshold for this PLL-switching logic.

CPUFreq Boost frequencies, by contrast, are identified by the `turbo-mode` property in the device tree. The two serve different purposes: for example, both 1.6 GHz and 2.15 GHz on X100 use the high-frequency PLL switching path, but they are normal frequencies, not Boost frequencies.

## Configuration

### Kernel Configuration

The device-tree-based scaling path requires:

```text
CONFIG_CPU_FREQ=y
CONFIG_SPACEMIT_K3_CPUFREQ=y
CONFIG_CPUFREQ_DT=y
CONFIG_CPUFREQ_DT_PLATDEV=y

CONFIG_PM_OPP=y
CONFIG_COMMON_CLK=y

CONFIG_REGULATOR=y
CONFIG_REGULATOR_RPMI=y
CONFIG_MAILBOX=y
CONFIG_RISCV_SBI_MPXY_MBOX=y
```

Enable the appropriate K3 clock driver as well; the device tree must provide the CPU clocks and power supply configuration.

The governors enabled in the current workspace are:

```text
CONFIG_CPU_FREQ_GOV_PERFORMANCE=y
CONFIG_CPU_FREQ_GOV_POWERSAVE=y
CONFIG_CPU_FREQ_GOV_USERSPACE=y
CONFIG_CPU_FREQ_GOV_ONDEMAND=y
CONFIG_CPU_FREQ_GOV_CONSERVATIVE=y
```

The default governor in the current workspace is:

```text
CONFIG_CPU_FREQ_DEFAULT_GOV_USERSPACE=y
```

The following are currently disabled:

```text
# CONFIG_CPU_FREQ_GOV_SCHEDUTIL is not set
# CONFIG_CPU_FREQ_STAT is not set
```

These are the settings in the current workspace; other K3 images may use different defaults. At runtime, check `scaling_governor` and `scaling_available_governors`.

Common governors behave as follows:

| Governor | Behavior |
|---|---|
| `performance` | Requests the highest frequency currently permitted by the policy |
| `powersave` | Requests the lowest frequency currently permitted by the policy |
| `userspace` | Allows user space to request a frequency through `scaling_setspeed` |
| `ondemand` | Adjusts frequency dynamically based on CPU load |
| `conservative` | Adjusts frequency gradually based on CPU load |
| `schedutil` | Selects frequency using scheduler load information; must be enabled separately |

`performance` remains subject to Boost state, frequency limits, and thermal constraints; it does not guarantee operation at the highest device-tree frequency.

To enable frequency statistics:

```text
CONFIG_CPU_FREQ_STAT=y
```

CPUFreq thermal throttling also requires the appropriate thermal configuration and board-level cooling bindings. The current workspace enables:

```text
CONFIG_THERMAL=y
CONFIG_CPU_FREQ_THERMAL=y
```

### DTS Configuration

The OPP tables are defined in [k3_opp_table.dtsi](/data/home/liulixin/workspace/k3/linux-6.18/arch/riscv/boot/dts/spacemit/k3_opp_table.dtsi)。.

#### CPU and OPP Table Bindings

X100 CPU nodes reference two OPP tables; K3 platform code selects one. CPU0 is shown below:

```dts
&cpu_0 {
    clst-supply = <&edcdc1>;

    clocks = <&syscon_apmu CLK_APMU_CPU_C0_CORE>,
         <&syscon_apmu CLK_APMU_CPU_C1_CORE>;
    clock-names = "cls0", "cls1";

    operating-points-v2 = <&clst_core_opp_table0_x100>,
                  <&clst_core_opp_table1_x100>;
};
```

CPU1–CPU7 use the same supply, clock, and OPP table bindings.

A100 CPU nodes specify one OPP table. CPU8 is shown below:

```dts
&cpu_8 {
    clocks = <&syscon_apmu CLK_APMU_CPU_C2_CORE>,
         <&syscon_apmu CLK_APMU_CPU_C3_CORE>;
    clock-names = "cls0", "cls1";

    operating-points-v2 = <&clst_core_opp_table0_a100>;
};
```

CPU9–CPU15 use the same clock and OPP table bindings.

#### OPP Properties

The following example shows the normal 2.15 GHz OPP in the au4562 table:

```dts
opp2150000000 {
    opp-hz = /bits/ 64 <2150000000>,
         /bits/ 64 <2150000000>;
    opp-microvolt = <910000 910000 914000>;
    clock-latency-ns = <200000>;
};
```

The properties have the following meanings:

| Property | Meaning |
|---|---|
| `opp-hz` | Target frequency in Hz; the two values correspond to the two OPP clocks |
| `opp-microvolt` | Target, minimum, and maximum voltage, in µV, in that order |
| `clock-latency-ns` | Clock-switching latency specified in the device tree, in ns |
| `turbo-mode` | Marks the OPP as a Boost frequency |
| `opp-shared` | In the OPP table node, indicates that devices using the table share OPP state |

Example Boost OPP:

```dts
opp2200000000 {
    opp-hz = /bits/ 64 <2200000000>,
         /bits/ 64 <2200000000>;
    opp-microvolt = <950000 950000 954000>;
    clock-latency-ns = <200000>;
    turbo-mode;
};
```

`clock-latency-ns = <200000>` specifies a latency of 200 µs. This does not mean that a complete frequency transition, including PLL changes and voltage communication, will necessarily take 200 µs when measured.

Board-level device trees may override common settings, so runtime behavior depends on the device tree actually used at boot.

## Debugging

### Confirm Frequency Domains and Driver

Run as root:

```sh
ls /sys/devices/system/cpu/cpufreq/
```

Inspect the CPUs associated with each policy, including shared and online CPUs, along with the driver and governor:

```sh
for p in /sys/devices/system/cpu/cpufreq/policy*; do
    [ -d "$p" ] || continue

    printf '\nPolicy: %s\n' "$p"

    printf 'related_cpus: '
    cat "$p/related_cpus"

    printf 'affected_cpus: '
    cat "$p/affected_cpus"

    printf 'scaling_driver: '
    cat "$p/scaling_driver"

    printf 'scaling_governor: '
    cat "$p/scaling_governor"
done
```

For the current device-tree scaling path, `scaling_driver` is expected to be:

```text
cpufreq-dt
```

`related_cpus` identifies CPUs sharing the policy; `affected_cpus` identifies online CPUs currently managed by it.

### Common sysfs Interfaces

These nodes are under each `policyX` directory. Frequency values are in **kHz**.

| Node | Purpose |
|---|---|
| `related_cpus` | CPUs sharing the policy's frequency domain |
| `affected_cpus` | Online CPUs currently managed by the policy |
| `scaling_driver` | Current CPUFreq driver |
| `scaling_available_governors` | Available governors |
| `scaling_governor` | Current governor |
| `scaling_available_frequencies` | Normal frequencies in the table |
| `scaling_boost_frequencies` | Boost-marked frequencies in the table |
| `scaling_min_freq` | Effective policy lower limit |
| `scaling_max_freq` | Effective policy upper limit |
| `scaling_cur_freq` | Current frequency reported by CPUFreq |
| `cpuinfo_min_freq` | Lower frequency bound reported by the driver |
| `cpuinfo_max_freq` | Upper frequency bound reported by the driver |
| `cpuinfo_cur_freq` | Current frequency read through the driver |
| `scaling_setspeed` | Target-frequency request under `userspace` |
| `boost` | Per-policy Boost switch |

The global Boost switch is located at:

```text
/sys/devices/system/cpu/cpufreq/boost
```

When reading the current frequency, distinguish the sources:

- In the current K3 `cpufreq-dt` path, `scaling_cur_freq` normally returns `policy->cur` maintained by CPUFreq.
- `cpuinfo_cur_freq` uses the driver's `.get` callback. This driver uses `cpufreq_generic_get()`, which ultimately calls `clk_get_rate(policy->clk)`.

These interfaces show the frequency state reported by CPUFreq and the clock framework; they are not independent hardware-counter measurements of real-time execution frequency.

### Confirm PMIC and OPP Table Selection

Read the PMIC property from the running device tree:

```sh
PMIC_PROP=/sys/firmware/devicetree/base/cpus/pmic-compatible

if [ -r "$PMIC_PROP" ]; then
    tr -d '\000' < "$PMIC_PROP"
    printf '\n'
else
    printf 'pmic-compatible is absent; the K3 driver defaults to X100 table0\n'
fi
```

Check the kernel log:

```sh
dmesg | grep -iE 'Spacemit K3|cpufreq|OPP|rpmi|regulator'
```

The current driver prints messages similar to:

```text
Spacemit K3: pmic-compatible="spacemit,au4562", using X100 OPP table1
```

This example shows the log format used in the source code; actual output depends on the board.

### Frequency Limits and Thermal Control

The frequencies currently available are constrained by all of the following:

- OPPs provided by the device tree.
- The Boost switch state.
- User-configured minimum and maximum frequency limits.
- Thermal limits and other frequency QoS requests.
- Actual clock and regulator capabilities.

When investigating a frequency that does not reach the expected value, first inspect:

```sh
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_min_freq
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_max_freq
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
```

If CPUFreq cooling is enabled in the board configuration, thermal control can restrict the effective upper limit; `userspace` and `performance` must also obey it.

Read thermal zone types and temperatures to aid diagnosis:

```sh
for z in /sys/class/thermal/thermal_zone*; do
    [ -d "$z" ] || continue

    printf '\nThermal zone: %s\n' "$z"
    cat "$z/type"
    cat "$z/temp"
done
```

Thermal zone `temp` is usually in millidegrees Celsius; for example, `65000` means 65 °C.

### Frequency Statistics

When `CONFIG_CPU_FREQ_STAT` is enabled, inspect:

```sh
cat /sys/devices/system/cpu/cpufreq/policy0/stats/time_in_state
cat /sys/devices/system/cpu/cpufreq/policy0/stats/total_trans
```

This option is disabled in the current workspace, so the absence of the `stats` directory alone does not indicate a CPUFreq driver failure.

## Testing

### Pre-Test Checks

The following examples assume that you have confirmed:

- `policy0` corresponds to X100.
- `policy8` corresponds to A100.
- `userspace` is among the available governors.

If the actual policy names differ, substitute the corresponding paths:

```sh
X100=/sys/devices/system/cpu/cpufreq/policy0
A100=/sys/devices/system/cpu/cpufreq/policy8

cat "$X100/related_cpus"
cat "$A100/related_cpus"

cat "$X100/scaling_available_governors"
cat "$A100/scaling_available_governors"
```

Record the original governors, frequency limits, and Boost state before testing so they can be restored afterward.

### Normal Frequency Transitions

#### Set X100 to 1.6 GHz

Check the available normal frequencies and effective limits:

```sh
cat "$X100/scaling_available_frequencies"
cat "$X100/scaling_min_freq"
cat "$X100/scaling_max_freq"
```

After confirming that `1600000` is within the permitted range:

```sh
echo userspace > "$X100/scaling_governor"
echo 1600000 > "$X100/scaling_setspeed"

cat "$X100/scaling_cur_freq"
cat "$X100/cpuinfo_cur_freq"
```

This affects the X100 CPUs sharing the policy.

#### Set A100 to 1.2 GHz

```sh
cat "$A100/scaling_available_frequencies"
cat "$A100/scaling_min_freq"
cat "$A100/scaling_max_freq"
```

After confirming that `1200000` is within the permitted range:

```sh
echo userspace > "$A100/scaling_governor"
echo 1200000 > "$A100/scaling_setspeed"

cat "$A100/scaling_cur_freq"
cat "$A100/cpuinfo_cur_freq"
```

`scaling_setspeed` requests a target; CPUFreq resolves it against the current limits and discrete frequency table. Request a frequency present in the table and verify the result using readbacks and logs.

### Using Boost

#### Boost and Frequency Lists

The current implementation lists normal and Boost frequencies separately:

```sh
cat "$X100/scaling_available_frequencies"
cat "$X100/scaling_boost_frequencies"
```

The two lists classify frequencies by their Boost marker:

- `scaling_available_frequencies` lists normal frequencies.
- `scaling_boost_frequencies` lists Boost frequencies.

**Boost need not be enabled to read the Boost list. Enabling Boost does not merge Boost frequencies into the normal list.**

These files describe entries in the frequency table; they do not mean that every listed frequency is currently permitted. Also check the Boost state and `scaling_min_freq` and `scaling_max_freq`.

#### Global and Per-Policy Switches

Check the current state:

```sh
cat /sys/devices/system/cpu/cpufreq/boost
cat "$X100/boost"
cat "$A100/boost"
```

The switches have the following meanings:

| Global Boost | Policy Boost | Policy Boost state |
|---:|---:|---|
| 0 | 0 | Disabled |
| 1 | 0 | Disabled |
| 1 | 1 | Boost frequencies permitted, subject to other limits |

The current implementation has the following additional behaviors:

- Writing the global switch synchronizes all active policies that support Boost.
- Writing a per-policy switch fails when global Boost is disabled.
- To control the two CPU groups separately, enable global Boost first, then adjust each policy.
- Writing the global switch again resynchronizes the policy states.

Boost is initially disabled in this `cpufreq-dt` implementation, but startup scripts may change it; read the actual runtime state.

#### Request 2.4 GHz on X100

Enable global Boost:

```sh
echo 1 > /sys/devices/system/cpu/cpufreq/boost
```

This also enables Boost for other active policies that support it. To test only X100, disable A100 Boost afterward:

```sh
echo 0 > "$A100/boost"
```

Check X100 Boost state, frequencies, and effective upper limit:

```sh
cat "$X100/boost"
cat "$X100/scaling_boost_frequencies"
cat "$X100/scaling_max_freq"
```

When `2400000` is within the effective range, request 2.4 GHz:

```sh
echo userspace > "$X100/scaling_governor"
echo 2400000 > "$X100/scaling_setspeed"

cat "$X100/scaling_cur_freq"
cat "$X100/cpuinfo_cur_freq"
```

If the frequency is not reached, check thermal limits, regulator configuration, and transition logs.

#### Request 2.0 GHz on A100

With global Boost enabled:

```sh
echo 1 > "$A100/boost"

cat "$A100/scaling_max_freq"

echo userspace > "$A100/scaling_governor"
echo 2000000 > "$A100/scaling_setspeed"

cat "$A100/scaling_cur_freq"
cat "$A100/cpuinfo_cur_freq"
```

Also confirm that `2000000` is within the effective range.

#### Disable Boost

Disable X100 Boost without changing other policies:

```sh
echo 0 > "$X100/boost"
```

Disable global Boost:

```sh
echo 0 > /sys/devices/system/cpu/cpufreq/boost
```

The driver recalculates the allowed range. The highest normal frequencies are:

- X100: 2.15 GHz.
- A100: 1.8 GHz.

Other constraints may lower the effective limit. Disabling Boost does not remove entries from `scaling_boost_frequencies`.

### Evaluate Results and Restore Settings

Do not judge a frequency test solely by whether `echo` succeeds.

Current code constrains the target to the policy range and maps it to a frequency-table entry; for example, a Boost request may be limited to the permitted maximum when Boost is disabled.

The current `scaling_setspeed` sysfs write implementation also does not return the governor callback's result directly to user space. A successful write alone therefore does not prove that the underlying transition succeeded.

After testing, check:

```sh
cat "$X100/scaling_governor"
cat "$X100/scaling_min_freq"
cat "$X100/scaling_max_freq"
cat "$X100/scaling_cur_freq"
cat "$X100/cpuinfo_cur_freq"

dmesg | grep -iE 'cpufreq|OPP|rpmi|regulator'
```

Restore the recorded Boost state, frequency limits, and governors. When restoring per-policy Boost states, handle the global switch first, then the individual policies.

## FAQ

### Why are there no policy directories?

Check in order:

1. Whether `CONFIG_CPU_FREQ`, `CONFIG_SPACEMIT_K3_CPUFREQ`, and `CONFIG_CPUFREQ_DT` are enabled.
2. Whether the active device tree provides CPU `operating-points-v2` properties.
3. Whether CPU clocks, the X100 regulator, and their dependent services initialized successfully.
4. Whether the kernel log reports OPP initialization or driver probe failures.

Start with:

```sh
dmesg | grep -iE 'cpufreq|Spacemit K3|OPP|rpmi|regulator'
```

### Why does changing one CPU's frequency affect others?

CPUs in a frequency domain share a policy. The current device tree places CPU0–CPU7 in X100 and CPU8–CPU15 in A100.

Check `related_cpus` for the sharing relationship.

### Why can't I write to scaling_setspeed?

First check the governor:

```sh
cat "$X100/scaling_governor"
```

The interface requires a corresponding governor callback. To request a frequency manually, switch to `userspace` and confirm it appears in `scaling_available_governors`.

### Why can't the CPU reach 2.4 GHz even though it is in the Boost list?

The Boost list only indicates that the frequency exists in the frequency table. All of the following must also be true:

- Global Boost is enabled.
- Boost is enabled for the current policy.
- The effective frequency upper limit permits that frequency.
- The clock and regulator can apply the requested setting.

Check `boost`, `scaling_max_freq`, frequency readbacks, and logs together.

### Why does the requested frequency differ from the readback?

Common causes include:

- The requested value is not in the discrete frequency table.
- The request exceeds the current policy range.
- Boost is disabled.
- Thermal control or another QoS request limits the upper bound.
- A dynamic governor changed the frequency again before the readback.
- The underlying clock or voltage adjustment failed.

First use `userspace` to request a normal frequency explicitly present in the table and within the permitted range, then check the result.

### Why doesn't X100 voltage change between normal frequencies?

The current table0 sets the target to 0.95 V for every normal frequency; table1 sets it to 0.91 V.

This is a property of the current OPP tables. DVFS does not require every pair of adjacent frequencies to use different voltages.

### Why are schedutil or frequency statistics unavailable?

Both depend on kernel configuration:

- `schedutil` requires `CONFIG_CPU_FREQ_GOV_SCHEDUTIL`.
- Statistics require `CONFIG_CPU_FREQ_STAT`.

Both are disabled in the current workspace; check the actual image configuration.

### Why is performance still below the highest Boost frequency?

`performance` requests the highest frequency currently permitted by the policy.

Disabled Boost, thermal throttling, or other upper-limit requests may lower that limit below the device-tree maximum. Check `scaling_max_freq` rather than inferring the result from the governor name.

## Source Code References

| File | Main role |
|---|---|
| `drivers/cpufreq/cpufreq.c` | CPUFreq core, policies, sysfs, and Boost control |
| `drivers/cpufreq/cpufreq-dt.c` | Device-tree CPUFreq driver; sets target frequencies through OPP |
| `drivers/cpufreq/spacemit-k3-cpufreq.c` | K3 OPP selection, multiple clocks and regulator configuration, and PLL notifications |
| `drivers/cpufreq/freq_table.c` | Frequency-table handling and normal/Boost lists |
| `drivers/cpufreq/cpufreq_userspace.c` | userspace governor |
| `arch/riscv/boot/dts/spacemit/k3_opp_table.dtsi` | X100 and A100 OPP tables and CPU bindings |
| `drivers/clk/spacemit/ccu-k3.c` | K3 clock controller |
| `drivers/regulator/regulator-rpmi.c` | Linux RPMI regulator client |
| `board/spacemit/k3/k3.c` (U-Boot) | Board-level device-tree fixups, including the PMIC property |
