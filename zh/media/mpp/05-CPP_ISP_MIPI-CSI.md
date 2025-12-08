# CPP & ISP & MIPI-CSI

K1 的 CPP、ISP 和 MIPI-CSI 模块基于标准 V4L2 接口实现，且提供了完整的测试程序供参考。

## 1 规格介绍

### MIPI CSI (CSI-2 V1.1) 4 lane (x2)

- 支持多种 Lane 组合模式：
  - 4 Lane + 4 Lane
  - 4 Lane + 2 Lane
  - 4 Lane + 4 Lane + 2 Lane（支持三路传感器）
- DPHY V1.1，最高速率可达 1.5 Gbps/lane
- 支持 RAW8/RAW10/RAW12/RAW14 格式，以及传统的 YUV420 8-bit 输入格式

### ISP（图像信号处理器）

ISP 用于处理传感器输出的图像信号，通过一系列数字图像处理算法，实现坏点校正、镜头阴影校正、降噪、相位补偿与矫正、背光补偿、色彩增强等图像优化。
- 支持双路 pipeline（时分复用），可同时处理两路图像流，来源可为传感器或 DDR 读取
- 每路 pipeline 输出最大图像分辨率为 1920x1080
- 两路 pipeline 同时工作时：
  - 不开启 PDAF（相位对焦），最大输入图像尺寸为 4748x8188
  - 开启 PDAF，最大输入图像尺寸为 3264x8188
- 单路 pipeline 工作时：
  - 不开启 PDAF，最大输入尺寸为 9496x8188
  - 开启 PDAF，最大输入尺寸为 6528x8188
- 输入宽度和高度均需为 4 的倍数

### CPP（图像后处理模块）

CPP 主要负责离线处理 ISP 输出的 NV12 数据，采用金字塔式多层分时处理，功能包括镜头畸变校正、空域与时域降噪、频域降噪、边缘增强等。

- 支持输入格式：NV12_DWT
- 支持输出格式：NV12\DWT、FBC_DWT
  （NV12_DWT 格式由标准 NV12 Buffer 和 ASR 私有 DWT Buffer 组成）
- 支持的最大图像尺寸：
  - 宽度：4224
  - 高度：3136
- 支持的最小图像尺寸：
  - 宽度：480
  - 高度：288
- 输入输出尺寸需保持一致

## 2 流程框图

### 2.1 ISP online 整体流程

ISP online 时模块连接如下：

```shell
sensor –> VI_DEV -> ISP_FW –> VI_CHN -> DDR -> CPP
```

编写代码时，需要先配置 sensor, VI, ISP, CPP 各个模块，注册各个模块 buffer 回调，然后依次 streamon ISP 和 sensor。sensor 开始出流后，ISP 图像处理过程中会发生中断，各模块进行中断处理后调用模块回调处理 buffer；当程序退出时，建议先停止 vi，再停止 sensor，再依次执行 CPP, ISP, VI 的反初始化配置，并释放使用的 buffer。软件流程图如下：

**开发流程概述：**

1. **初始化配置**：依次配置 Sensor、VI、ISP、CPP 模块。
2. **注册回调**：为各模块注册 buffer 回调函数。
3. **启动流程**：依次 streamon ISP 与 Sensor。
4. **数据流转**：
   - Sensor 开始出流后，ISP 图像处理会触发中断。
   - 各模块在中断处理后调用对应的 buffer 回调。
   - 用户在回调中获取 buffer 数据，并将处理完成的 buffer 重新 queue 回模块。
5. **退出流程**：建议先停止 VI，再停止 Sensor，最后依次执行 CPP、ISP、VI 的反初始化，并释放所有 buffer。

**ISP online 整体流程图**
![](static/OEBwb8QzxoIrKBxEcoqcsywknFe.png)

**Buffer 轮转：**

- streamon 前：准备输入与输出 buffer，并将输出 buffer queue 到模块 buffer list 中。
- streamon 后：输出 buffer list 就绪时触发模块的 buffer callback。
- 回调函数：由用户实现，负责数据处理并将已完成的 buffer 重新 queue。

### 2.2 ISP offline 整体流程

ISP offline 时模块连接如下：

```shell
DDR -> VI_DEV -> ISP_FW –> VI_CHN -> DDR -> CPP
```

**与 ISP online 模式的主要区别：**
- 数据源来自 DDR（非 Sensor 实时采集）。
- 数据输入、buffer 回调的配置方式不同。

其他流程与 Online 模式基本一致。

**Offline Capture Mode 流程图：**

![](static/PEhLbGTjconnrpx2ZvBcBkUenoh.png)

## 3 测试程序使用说明

`k1x-cam` 是一套用于测试和验证 **K1 芯片的 MIPI CSI + ASR ISP/CPP** 功能的工具集。
除了用于功能验证，也可作为客户在开发自定义应用（需要直接对接 ISP/CPP API）时的参考实现。

### 3.1 安装说明

#### 3.1.1 Bianbu 桌面系统

软件源已集成 `k1x-cam`，直接使用 `apt` 命令来安装即可。

```shell
sudo apt update
sudo apt install k1x-cam
```

#### 3.1.2 Buildroot 系统

TODO

### 3.2 使用说明

`k1x-cam` 的测试程序集中主要包含以下测试程序：

- **`cam-test`**：用于单路 pipeline，双路 pipeline，单 raw pipeline，单 CPP 处理等测试验证
- **`cam_sensors_test`**：简单验证 Sensor 的检测、初始化与 stream on 流程。

#### 3.2.1 `cam-test`

基本用法示例：

```shell
使用实例：cam-test <file.json>

//单pipeline在线测试：imx135(4208x3120@30fps raw10 4lane) –> ISP -> DDR(1080p@30fps) -> CPP
命令：cam-test demo/cfgs/0/camtest_sensor0_mode0.json

//单pipeline在线测试：imx135(4208x2416@30fps raw10 4lane) –> ISP -> DDR(1080p@30fps) -> CPP
命令：cam-test demo/cfgs/0/camtest_sensor0_mode1.json

//单pipeline在线测试：imx135(2104x1560@30fps raw10 4lane) –> ISP -> DDR(1080p@30fps) -> CPP
命令：cam-test demo/cfgs/0/camtest_sensor0_mode2.json

//单pipeline在线测试：imx135(2104x1560@30fps raw10 4lane) –> ISP -> DDR(1080p@30fps) -> CPP
命令：cam-test demo/cfgs/0/camtest_sensor0_mode2.json

//双pipeline capture测试：imx135(2104x1560@30fps raw10 4lane) –> ISP -> DDR(1080p@30fps) -> CPP
命令：cam-test demo/cfgs/2/camtest_sensor0_mode0.json

//only raw dump pipeline测试：imx135(4208x3120@30fps raw10 4lane) –> ISP（VI） -> DDR
命令：cam-test demo/cfgs/3/camtest_sensor0_mode0.json

//only isp online pipeline测试：imx135 –> ISP -> DDR（NV12）
命令：cam-test demo/cfgs/4/camtest_sensor0_mode0_nv12.json

//only isp online pipeline测试：imx135 –> ISP -> DDR（p010）
命令：cam-test demo/cfgs/4/camtest_sensor0_mode0_p010.json

//only isp online pipeline测试：imx135 –> ISP -> DDR（p210）
命令：cam-test demo/cfgs/4/camtest_sensor0_mode0_p210.json

//only isp online pipeline测试：imx135 –> ISP -> DDR（rgb565）
命令：cam-test demo/cfgs/4/camtest_sensor0_mode0_rgb565.json

//only isp online pipeline测试：imx135 –> ISP -> DDR（rgb888）
命令：cam-test demo/cfgs/4/camtest_sensor0_mode0_rgb888.json

//only isp online pipeline测试：imx135 –> ISP -> DDR（y210）
命令：cam-test demo/cfgs/4/camtest_sensor0_mode0_y210.json

//双pipeline online测试：imx135+gc2375h –> ISP -> DDR -> CPP
命令：cam-test demo/cfgs/1/camtest_main_aux.json

```

#### 3.2.2 JSON 参数说明

以 `sdktest_main_aux.json` 为例进行说明：

```shell
{
    "tuning_server_enable":1, //用于isp tunning服务使能，在only isp online、单pipeline online、双pipeline online测试有效
    "show_fps":1,    //统计从0~120帧的平均帧率
    "auto_run": 1,    //自动测试，没有用户交互过程

    "cpp_node": [    //CPP模块
            {
                    "name": "cpp0",    //cpp group0
                    "enable": 1,
                    "format":"NV12",
                    "src_from_file": 0,    //如果ISP和cpp都enable了，cpp的输入就来自ISP输出

                    "src_path":"/tmp/cpp_case_in_data/1920x1080/",
                    "size_width":1920,
                    "size_height":1080,
            },
            {
                    "name": "cpp1",    //cpp group1
                    "enable": 1,
                    "format":"NV12",
                    "src_from_file": 0,    //

                    "src_path":"/vendor/etc/camera/",
                    "size_width":1920,
                    "size_height":1080,
            },
        ],

    "isp_node":[    //ISP模块，1个ISP可以接入两路video stream input
            {
                    "name": "isp0",    //isp0在线模式工作，输出1080p@30fps NV12
                    "enable": 1,
                    "work_mode":"online",
                    "format":"NV12",
                    "out_width":1920,
                    "out_height":1080,

                    "sensor_name":"imx135_asr",    //imx135对应/dev/cam_sensor0，工作在模式0
                    "sensor_id" : 0,
                    "sensor_work_mode":0,
                    "fps":30,

                    "src_file":"/tmp/1920x1080_raw12_long_packed.vrf",    //不生效(使用在其他模式)
                    "bit_depth": 12,    //不生效
                    "in_width":1920,    //不生效
                    "in_height":1080,    //不生效

            },
            {
                    "name": "isp1",    //isp1在线模式工作，输出1600x1200@30fps NV12
                    "enable": 1,
                    "work_mode":"online",
                    "format":"NV12",
                    "out_width":1600,
                    "out_height":1200,

                    "src_file":"/tmp/1920x1080_raw12_long_packed.vrf",    //不生效
                    "bit_depth": 12,    //不生效
                    "in_width":1920,    //不生效
                    "in_height":1080,    //不生效

                    "sensor_name":"gc2375h_asr",    //gc2375h对应/dev/cam_sensor1，工作在模式0
                    "sensor_id" : 1,
                    "sensor_work_mode":0,
                    "fps":30,
            },
    ]
}

```

关于 json 参数更详细的作用，可分析 `config.c` 和 `online_pipeline_test.c/main.c` 的具体应用场景。

#### 3.2.3 `cam_sensors_test`

基本用法示例：

```shell
使用示例：cam_sensors_test [devId] [sensors_name]

```

输入执行命令后，在交互终端输入 `s` 字符后进行开流动作。
如果程序没有报错，说明以下流程基本正常 sensor detect -> init -> stream on。

## 4 SENSOR 调试

详见 [相机开发指南](https://sdk.spacemit.com/camera/camera_development_guide)

## 5 ISP 效果调试

ISP 效果调试可能需要使用到的工具包括：调试工具（Tuning Tool）、定标插件（Calibration Plugins）、图像分析工具（VRF viewer），平台调试辅助等。

详见 [ISP PQ 工具用户指南](https://sdk.spacemit.com/camera/isp_pq_tools_user_guide)

## 6 API 使用说明

这些 API 的详细参数、数据结构、错误码和返回值已在 [ISP API 开发指南](https://sdk.spacemit.com/camera/isp_api_development_guide) 中进行说明，主要面向以下两类开发者：
- ISP 效果相关的 tuning 和算法工程师
- 图像相关功能开发的应用工程师
