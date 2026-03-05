---
sidebar_position: 2
---

# ISP API 开发指南

本文对 SDK API 接口进行说明

## ISP API

本节介绍 ISP 模块 API 的使用方法，这些 API 都是 ISP SDK 面向应用的，主要分为：

- 系统控制 API
- 图像效果设置 API
- Tuning 相关的 API

详细解释了相关的参数数据结构、错误码和返回值。主要面向 ISP 效果相关的 tuning 和算法工程师，以及图像相关功能开发的应用工程师。

### 系统控制 API

系统控制 API 如下：

- **ASR_ISP_Construct：** 构造 ISP pipeline 上下文环境接口。
- **ASR_ISP_Destruct：** 析构 ISP pipeline 上下文环境接口。
- **ASR_ISP_GetFwFrameInfoSize()：** 获取 ISP firmware frameinfo 结构体大小接口。
- **ASR_ISP_RegSensorCallBack：** 向 ISP pipeline 注册 sensor 回调函数接口。
- **ASR_ISP_UnRegSensorCallBack：** 从 ISP pipeline 注销 sensor 回调函数接口。
- **ASR_ISP_RegAfMotorCallBack：** 向 ISP pipeline 注册 sensor 马达回调函数接口。
- **ASR_ISP_EnableOfflineMode：** 使能 ISP pipeline 的 Offline 模式接口。
- **ASR_ISP_SetPubAttr：** 设置 ISP pipeline 公共属性接口。
- **ASR_ISP_SetTuningParams：** 设置 ISP pipeline tuning 相关参数接口。
- **ASR_ISP_SetChHwPipeID：** 设置 ISP pipeline channel 工作的真实硬件 pipe ID 接口。
- **ASR_ISP_Init：** 初始化 ISP pipeline。
- **ASR_ISP_DeInit：** 撤销 ISP pipeline 的初始化。
- **ASR_ISP_EnablePDAF：** 使能 ISP pipeline 的 PDAF。
- **ASR_ISP_SetFps：** 设置该 pipeline 上 sensor/ISP 工作的帧率。
- **ASR_ISP_SetFrameinfoCallback：** 设置 ISP pipeline frameinfo 的回调。
- **ASR_ISP_QueueFrameinfoBuffer：** 设置 ISP pipeline frameinfo buffer 的入队列。
- **ASR_ISP_FlushFrameinfoBuffer：** 清除 ISP pipeline 上的 frameinfo buffer。
- **ASR_ISP_Streamon：** 启动 ISP pipeline。
- **ASR_ISP_Streamoff：** 停止 ISP pipeline。
- **ASR_ISP_TriggerRawCapture：** 触发 ISP pipeline 进行 Raw 拍照流程。
- **ASR_ISP_ReInitPreviewChannel：** 重新初始化预览 channel。
- A**SR_ISP_NotifyOnceHDRRawCapture：** 通知 ISP pipeline 进入一次 HDR Raw 拍照流程。
- **ASR_ISP_UpdateNoneZslStreamAeParams：** 更新 sensor 相关 AE 的参数。

#### ASR_ISP_Construct

【描述】

构造 ISP pipeline 上下文环境。

【语法】

int ASR_ISP_Construct(uint32_t pipelineID);

【参数】

| 参数名称   | 描述               | 输入/输出 |
| ------------------- | --------------------------- | ------------------ |
| pipelineID | ISP pipeline 的 ID | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`
- 库文件：`libisp.so`

【注意】

- 输入参数 `pipelineID` 描述的是个虚拟的 pipeline，对应的是 ISPfirmware，一个 `pipelineID` 对应一个 firmware。

#### ASR_ISP_Destruct

【描述】

析构 ISP pipeline 上下文环境。

【语法】

int ASR_ISP_Destruct (uint32_t pipelineID);

【参数】

| 参数名称   | 描述               | 输入/输出 |
| ------------------- | --------------------------- | ------------------ |
| pipelineID | ISP pipeline 的 ID | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`
- 库文件：`libisp.so`

#### ASR_ISP_GetFwFrameInfoSize

【描述】

获取 ISP Firmware frameinfo 结构体的大小。

【语法】

int ASR_ISP_GetFwFrameInfoSize ();

【参数】

无。

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| &gt;0       | 结构体大小       |
| &lt;0       | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`
- 库文件：`libisp.so`

#### ASR_ISP_RegSensorCallBack

【描述】

向 ISP pipeline 注册 sensor 的回调函数。

【语法】

IntASR_ISP_RegSensorCallBack(uint32_tpipelineID,ISP_SENSOR_ATTR_S *pstSensorInfo, ISP_SENSOR_REGISTER_S *pstRegister);

【参数】

| 参数名称      | 描述                             | 输入/输出 |
| ---------------------- | ----------------------------------------- | ------------------ |
| pipelineID    | ISP pipeline 的 ID               | 输入      |
| pstSensorInfo | 注册 sensor 的属性               | 输入      |
| pstRegister   | 注册 sensor 的回调函数结构体指针 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_sensor_comm.h`
- 库文件：`libisp.so`、`libcam_sensors.so`

#### ASR_ISP_UnRegSensorCallBack

【描述】

向 ISP pipeline 注销 sensor 的回调函数。

【语法】

IntASR_ISP_UnRegSensorCallBack(uint32_tpipelineID,ISP_SENSOR_ATTR_S *pstSensorInfo);

【参数】

| 参数名称      | 描述               | 输入/输出 |
| ---------------------- | --------------------------- | ------------------ |
| pipelineID    | ISP pipeline 的 ID | 输入      |
| pstSensorInfo | 注册 sensor 的属性 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：spm_cam_isp.h、spm_isp_sensor_comm.h
- 库文件：libisp.so

#### ASR_ISP_RegAfMotorCallBack

【描述】

向 ISP pipeline 注册 sensor 马达的回调函数。

【语法】

IntASR_ISP_RegAfMotorCallBack(uint32_tpipelineID,ISP_AF_MOTOR_REGISTER_S *pstAfRegister);

【参数】

| 参数名称      | 描述                                 | 输入/输出 |
| ---------------------- | --------------------------------------------- | ------------------ |
| pipelineID    | ISP pipeline 的 ID                   | 输入      |
| pstAfRegister | 注册 sensor 马达的回调函数结构体指针 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_sensor_comm.h`
- 库文件：`libisp.so`

#### ASR_ISP_EnableOfflineMode

【描述】

使能 ISP pipeline 的 Offline 模式，使 ISP 的输入 stream 来自 DDR 而非 sensor，由 VI 模块从 DDR 读取数据。

【语法】

IntASR_ISP_EnableOfflineMode(uint32_tpipelineID,uint32_tenable,const ISP_OFFLINE_ATTR_S *pstOfflineAttr);

【参数】

| 参数名称       | 描述                              | 输入/输出 |
| ----------------------- | ------------------------------------------ | ------------------ |
| pipelineID     | ISP pipeline 的 ID                | 输入      |
| enable         | 使能/关闭 offline 模式            | 输入      |
| pstOfflineAttr | 指向 Offline 模式属性结构体得指针 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_comm.h`
- 库文件：`libisp.so`

【注意】

- 使能 offline 模式时，sensor 以及马达的 callback 不需要注册，即使注册了，ISP 也不会使用。
- 特殊的拍照场景中 offline pipeline 的处理不需要使用该函数。

#### ASR_ISP_SetPubAttr

【描述】

设置 ISP pipeline 上 channel 的公共属性。

【语法】

int ASR_ISP_SetPubAttr(uint32_t pipelineID, uint32_t channelID, const ISP_PUB_ATTR_S *pstPubAttr);

【参数】

| 参数名称   | 描述                              | 输入/输出 |
| ------------------- | ------------------------------------------ | ------------------ |
| pipelineID | ISP pipeline ID 号                | 输入      |
| channelID  | ISP pipeline channel 的 ID        | 输入      |
| pstPubAttr | ISP pipeline 的公共属性结构体指针 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_comm.h`
- 库文件：`libisp.so`

#### ASR_ISP_SetTuningParams

【描述】

设置 ISP pipeline 上 tuning 相关的属性。为了支持算法开发调试方便，ISP 初始化的 tuning 参数会由两个部分决定：

1. 先从 sensor 那边获取（事先将 tuning 好的文件转换成代码）设置；
2. 如果此接口告诉 ISP 当前有 tuning file 存在，ISP 会以该文件优先，覆盖之前设置的参数；否则不会执行任何操作。

【语法】

intASR_ISP_SetTuningParams(uint32_tpipelineID,ISP_TUNING_ATTRS_S *pstTuningAttr);

【参数】

| 参数名称      | 描述                                  | 输入/输出 |
| ---------------------- | ---------------------------------------------- | ------------------ |
| pipelineID    | ISP pipeline ID 号                    | 输入      |
| pstTuningAttr | ISP pipeline 的 tuning 属性结构体指针 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_comm.h`
- 库文件：`libisp.so`

【注意】

调用该接口之前，`ASR_ISP_RegSensorCallBack` 或者 `ASR_ISP_EnableOfflineMode` 需要先调用。

#### ASR_ISP_SetChHwPipeID

【描述】

设置 ISP pipeline 的 channel 实际工作的硬件 pipe ID。

【语法】

int ASR_ISP_SetChHwPipeID(uint32_t pipelineID, uint32_t channelID, uint32_t hwPipeID);

【参数】

| 参数名称   | 描述                       | 输入/输出 |
| ------------------- | ----------------------------------- | ------------------ |
| pipelineID | ISP pipeline ID 号         | 输入      |
| channelID  | ISP pipeline channel 的 ID | 输入      |
| hwPipeID   | 实际硬件 pipe ID           | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_comm.h`
- 库文件：`libisp.so`

【注意】

- 该接口要在 `ASR_ISP_Init` 之前调用，否则 ISP channel 以默认的 channel ID（硬件 pipe0） 开始运行。
- 输入参数 `pipelineID` 对应的是 ISPfirmware，`channelID` 对应的是工作模式 preview 或 capture，`hwPipeID` 对应的是 ISP 硬件 pipelineID。对于特殊的拍照场景，可以用一套 firmware 管理 2 个硬件 pipeline 工作在不同的模式下，这样保证 2 个硬件 pipeline 的图像效果保持一致。

#### ASR_ISP_Init

【描述】

以之前设置的参数，初始化 ISPpipeline，经过初始化后，第一帧的所有参数已经 ready，原则上不建议在该接口后再设置初始化的参数，因为不会在第一帧生效，而会在第二帧生效。

【语法】

int ASR_ISP_Init(uint32_t pipelineID);

【参数】

| 参数名称   | 描述               | 输入/输出 |
| ------------------- | --------------------------- | ------------------ |
| pipelineID | ISP pipeline ID 号 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_comm.h`
- 库文件：`libisp.so`

#### ASR_ISP_DeInit

【描述】

撤销 ISP pipeline 的初始化，清除所有设置在该 pipeline 上的效果参数，注册的一些 callback 是不会清除的。

【语法】

int ASR_ISP_ DeInit (uint32_t pipelineID);

【参数】

| 参数名称   | 描述               | 输入/输出 |
| ------------------- | --------------------------- | ------------------ |
| pipelineID | ISP pipeline ID 号 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_comm.h`
- 库文件：`libisp.so`

#### ASR_ISP_EnablePDAF

【描述】

使能/关闭 ISP pipeline 的 PDAF 功能。

【语法】

int ASR_ISP_EnablePDAF(uint32_t pipelineID, uint32_t enable);

【参数】

| 参数名称   | 描述                       | 输入/输出 |
| ------------------- | ----------------------------------- | ------------------ |
| pipelineID | ISP pipeline ID 号         | 输入      |
| enable     | 使能或者关闭 PDAF 的标识位 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_comm.h`
- 库文件：`libisp.so`

【注意】

- 必须在 `ASR_ISP_Init` 之后调用。

#### ASR_ISP_SetFps

【描述】

设置 ISPpipeline 的帧率，ISP 支持动态帧率，如果需要以特定的帧率运行，请将 `fminFps` 和 `fmaxFps` 设置成一样的值。该接口可以在 `ASR_ISP_Init` 之前或者 ISP 运行期间设置，前者设置后，第一帧便以设置的帧率运行。

【语法】

int ASR_ISP_SetFps(uint32_t pipelineID, float fminFps, float fmaxFps);

【参数】

| 参数名称   | 描述               | 输入/输出 |
| ------------------- | --------------------------- | ------------------ |
| pipelineID | ISP pipeline ID 号 | 输入      |
| fminFps    | 最小帧率           | 输入      |
| FmaxFps    | 最大帧率           | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_comm.h`
- 库文件：`libisp.so`

#### ASR_ISP_SetFrameinfoCallback

【描述】

设置 ISP pipeline 的通知获取 frameinfo 回调函数，ISP 在更新每帧 frameinfo 时，通过该 callback 将 frameinfo 送至 user。

【语法】

int ASR_ISP_SetFrameinfoCallback(uint32_t pipelineID, GetFrameInfoCallBack callback);

【参数】

| 参数名称   | 描述                           | 输入/输出 |
| ------------------- | --------------------------------------- | ------------------ |
| pipelineID | ISP pipeline 的 ID             | 输入      |
| callback   | User 获取 frameinfo 的回调函数 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_comm.h`
- 库文件：`libisp.so`

#### ASR_ISP_QueueFrameinfoBuffer

【描述】

ISP pipeline 的 frameinfo buffer 入队列函数，ISP 内部以队列的形式使用 frameinfo buffer，先进先出。

【语法】

intASR_ISP_QueueFrameinfoBuffer(uint32_tpipelineID,IMAGE_BUFFER_S

 *pFrameInfoBuf);

【参数】

| 参数名称      | 描述                | 输入/输出 |
| ---------------------- | ---------------------------- | ------------------ |
| pipelineID    | ISP pipeline 的 ID  | 输入      |
| pFrameInfoBuf | FrameInfo 的 buffer | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`cam_module_interface.h`
- 库文件：`libisp.so`

【注意】

输入参数 `pFrameInfoBuf` 指针会被 SDK 直接使用。所以应用层应当保证该指针所指向的内存空间不能被释放，直到注册的回调函数被调用。

#### ASR_ISP_FlushFrameinfoBuffer

【描述】

清除 ISP pipeline 上的 frameinfo buffer 队列，被清除的 buffer 通过设置的回调返还。

【语法】

int ASR_ISP_FlushFrameinfoBuffer(uint32_t pipelineID);

【参数】

| 参数名称   | 描述                  | 输入/输出 |
| ------------------- | ------------------------------ | ------------------ |
| pipelineID | ISP 的 pipeline ID 号 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_comm.h`
- 库文件：`libisp.so`

【注意】

Flush 出去的 buffer 不要再入队列，容易引发死锁问题。

#### ASR_ISP_Streamon

【描述】

运行 ISP pipeline，sensor 的 streamon 要在此接口后面。

【语法】

int ASR_ISP_Streamon(uint32_t pipelineID);

【参数】

| 参数名称   | 描述               | 输入/输出 |
| ------------------- | --------------------------- | ------------------ |
| pipelineID | ISP pipeline ID 号 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_comm.h`
- 库文件：`libisp.so`

【注意】

- 要在 `ASR_ISP_Init` 之后调用。
- 在此之前不要先打开 sensor。

#### ASR_ISP_Streamoff

【描述】

停止 ISP pipeline 的运行。

【语法】

int ASR_ISP_Streamoff(uint32_t pipelineID);

【参数】

| 参数名称   | 描述               | 输入/输出 |
| ------------------- | --------------------------- | ------------------ |
| pipelineID | ISP pipeline 的 ID | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`spm_isp_comm.h`
- 库文件：`libisp.so`

【注意】

要在 `ASR_ISP_Streamon` 之后调用。

#### ASR_ISP_TriggerRawCapture

【描述】

触发 ISP pipeline 进行一次特殊的拍照场景处理流程，如果当前正在进行拍照，此时会触发失败。

【语法】

int ASR_ISP_TriggerRawCapture(uint32_t pipelineID, IMAGE_BUFFER_S *pFrameInfoBuf, uint32_t hdrCapture);

【参数】

| 参数名称           | 描述                                                                                                                            | 输入/输出 |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| pipelineID         | ISP pipeline 的 ID                                                                                                              | 输入      |
| pFrameInfoBuf  | 当前 RAW 图对应的 frameinfo buffer，该 frameinfo 应该和拍照时预览帧的 framei nfo 一样，否则拍照的图像质量有可能和预览不一样 | 输入      |
| hdrCapture         | 是否是 hdr 拍照的标志位，本系统 HDR 拍照暂时只支持在 RAW 域合成的 HDR 拍照                                                  | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`cam_module_interface.h`
- 库文件：`libisp.so`

【注意】

该接口只能在 ISP 运行期间调用。

#### ASR_ISP_ReInitPreviewChannel

【描述】

重新初始化 ISP pipeline 的预览 channel。在某些场景下，会停止某个 pipeline 的预览，进行复用硬件 pipe，在恢复该预览时，需要调用此接口，用来达到断流前后画面效果一致。

【语法】

int ASR_ISP_ReInitPreviewChannel(uint32_t pipelineID, IMAGE_BUFFER_S *pFrameInfoBuf);

【参数】

| 参数名称      | 描述                              | 输入/输出 |
| ---------------------- | ------------------------------------------ | ------------------ |
| pipelineID    | ISP pipeline 的 ID                | 输入      |
| pFrameInfoBuf | 需要恢复场景时的 frameinfo buffer | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`、`cam_module_interface.h`
- 库文件：`libisp.so`

#### ASR_ISP_NotifyOnceHDRRawCapture

【描述】

通知 ISP pipeline 进行一次 HDR Raw 拍照流程，ISP 返回 HDR 第一帧有效 Raw frame ID，目前 HDR Raw 图顺序是正常曝光，长曝光和短曝光。

【语法】

int ASR_ISP_NotifyOnceHDRRawCapture(uint32_t pipelineID, uint32_t ZSLCapture, int32_t

 *startFrameID);

【参数】

| 参数名称     | 描述                               | 输入/输出 |
| --------------------- | ------------------------------------------- | ------------------ |
| pipelineID   | ISP pipeline 的 ID                 | 输入      |
| ZSLCapture   | 是否是 ZSL HDR 拍照的标志位        | 输入      |
| startFrameID | 存放第一帧有效 HDR Raw 的 frame ID | 输出      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`
- 库文件：`libisp.so`

【注意】

该接口只能在 ISP 运行期间调用。

#### ASR_ISP_UpdateNoneZslStreamAeParams

【描述】

更新 ISP pipeline 上 sensor AE 相关的参数，该接口只有在 pipeline 断流后再重新开流时有用，用来保持 sensor 在断流前后状态一致。

【语法】

int ASR_ISP_UpdateNoneZslStreamAeParams(uint32_tpipelineID,IMAGE_BUFFER_S *pFrameInfoBuf,int backPreview, int updateSnsReg);

【参数】

| 参数名称      | 描述                                                                                          | 输入/输出 |
| ---------------------- | ------------------------------------------------------------------------------------------------------ | ------------------ |
| pipelineID    | ISP pipeline 的 ID                                                                            | 输入      |
| pFrameInfoBuf | 需要更新状态对应的 frameinfo                                                                  | 输入      |
| backPreview   | 是否切换预览标志位                                                                            | 输入      |
| updateSnsReg  | 是否需要更新 sensor 的 AE 寄存器，在 sensor 重新 config 后，需要使能该标志位；否则可以不需要 | 输入  |

【返回值】

| 返回值 | 描述                 |
| --------------- | ----------------------------- |
| 0      | 成功。               |
| 非 0   | 失败，其值为错误码。 |

【需求】

- 头文件：`spm_cam_isp.h`、`cam_module_interface.h`
- 库文件：`libisp.so`

### 效果设置 API

本小节描述的 API 是用来帮助使用者设置一些和图像效果有关的 setting。在本软件中，图像效果的设置通过 command 的形式使用，这些 command 由统一接口对外提供 API：

- **ASR_ISP_SetEffectParams：** 设置 ISP pipeline 相关效果参数接口。

ISP 支持的效果 command 如下：

- ISP_EFFECT_CMD_S_AE_MODE
- ISP_EFFECT_CMD_S_AWB_MODE
- ISP_EFFECT_CMD_S_AF_MODE
- ISP_EFFECT_CMD_S_TRIGGER_AF
- ISP_EFFECT_CMD_S_ANTIFLICKER_MODE
- ISP_EFFECT_CMD_S_LSC_MODE
- ISP_EFFECT_CMD_S_CCM_MODE
- ISP_EFFECT_CMD_S_AECOMPENSATION
- ISP_EFFECT_CMD_S_METERING_MODE
- ISP_EFFECT_CMD_S_ZOOM_RATIO_IN_Q8
- ISP_EFFECT_CMD_S_SENSITIVITY_MODE
- ISP_EFFECT_CMD_S_SENSOR_EXPOSURE_MODE
- ISP_EFFECT_CMD_S_TRIGGER_AE_QUICK_RESPONSE
- ISP_EFFECT_CMD_S_AECOMPENSATION_STEP
- ISP_EFFECT_CMD_S_AE_SCENE_MODE
- ISP_EFFECT_CMD_S_FILTER_MODE
- ISP_EFFECT_CMD_S_YUV_RANGE
- ISP_EFFECT_CMD_S_SOLID_COLOR_MODE
- ISP_EFFECT_CMD_G_AF_MODE
- ISP_EFFECT_CMD_G_ANTIFLICKER_MODE
- ISP_EFFECT_CMD_G_METERING_MODE
- ISP_EFFECT_CMD_G_AF_MOTOR_RANGE
- ISP_EFFECT_CMD_G_AE_MODE
- ISP_EFFECT_CMD_G_AWB_MODE

#### ASR_ISP_SetEffectParams

【描述】

设置 ISP 图像效果接口。

【语法】

int ASR_ISP_SetEffectParams(uint32_t pipelineID, uint32_t effectCmd, void *pData, int dataSize);

【参数】

| 参数名称   | 描述                       | 输入/输出 |
| ------------------- | ----------------------------------- | ------------------ |
| pipelineID | ISP pipeline ID            | 输入      |
| effectCmd  | 效果参数命令               | 输入      |
| pData      | 指向此效果参数结构体的指针 | 输入      |
| dataSize   | 此效果参数结构体的大小     | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`
- 库文件：`libisp.so`

#### 效果 command 介绍

1. **ISP_EFFECT_CMD_S_AE_MODE**
   设置图像曝光模式的命令，对应参数类型是结构体 `asrISP_AE_INFO_S`。

2. **ISP_EFFECT_CMD_S_AWB_MODE**
   设置图像白平衡模式的命令，对应参数类型是结构体 `asrISP_AWB_INFO_S`。

3. **ISP_EFFECT_CMD_S_AF_MODE**
   设置图像对焦模式的命令，对应参数类型是结构体 `asrISP_AF_INFO_S`。

4. **ISP_EFFECT_CMD_S_TRIGGER_AF**
   触发图像对焦的命令，对应参数类型是 `int32_t`，`1` 表示触发，`0` 表示取消。

5. **ISP_EFFECT_CMD_S_ANTIFLICKER_MODE**
   设置图像 flicker 模式的命令，对应参数类型是 `int32_t`，该参数的值由枚举 `asrISP_ANTIFLICKE R_MODE_E` 定义。

6. **ISP_EFFECT_CMD_S_LSC_MODE**
   设置图像 lens shading 模式的命令，对应参数类型是结构体 `asrISP_LSC_INFO_S`。

7. **ISP_EFFECT_CMD_S_CCM_MODE**
   设置图像 CCM 模式的命令，对应参数类型是结构体 `asrISP_CCM_INFO_S`。

8. **ISP_EFFECT_CMD_S_AECOMPENSATION**
   设置图像曝光补偿模式的命令，对应参数类型是 `int32_t`，数值目前支持 [-6,6]。

9. **ISP_EFFECT_CMD_S_METERING_MODE**
   设置图像测光模式的命令，对应参数类型是结构体 `asrISP_METERING_INFO_S`。

10. **ISP_EFFECT_CMD_S_ZOOM_RATIO_IN_Q8**
   设置图像缩放比例的命令，对应参数类型是 `int32_t`，单位是 Q8。

11. **ISP_EFFECT_CMD_S_SENSITIVITY_MODE**
   设置图像 ISO 模式的命令，对应参数类型是结构体 `asrISP_SENSITIVITY_INFO_S`，支持 auto 和 manual 模式。

12. **ISP_EFFECT_CMD_S_SENSOR_EXPOSURE_MODE**

   - 设置图像 Sensor 曝光模式的命令，这个命令和 `ISP_EFFECT_CMD_S_AE_MODE` 的区别是： 该命令只能设置曝光信息，而后者可以设置曝光和增益信息；对应参数类型是结构体 `asr ISP_SENSOR_EXPOSURE_INFO_S`，支持 auto 和 manual 模式。
   - 这两个 command 以及 `ISP_EFFECT_CMD_S_SENSITIVITY_MODE` 的关系如下表所示。  

| AE MODE                | SENSITIVITY AUTO | SENSITIVITY MANUAL |
| ------------------------------- | ------------------------- | --------------------------- |
| SENSOR EXPOSURE AUTO   | Auto             | Auto               |
| SENSOR EXPOSURE MANUAL | Auto             | Manual             |

- 如表中所示，`SENSITIVITY_MODE_MANUAL` 是在 Auto Exposure 时，固定增益，曝光由算法根据场景自动计算；而 `SENSOR_EXPOSURE_MODE_MANUAL` 是在 Auto Exposure 时，固定曝光时间，增益由算法根据场景自动计算。

13. **ISP_EFFECT_CMD_S_TRIGGER_AE_QUICK_RESPONSE**
   触发算法 AE 快速收敛一次的命令，对应参数类型是 `int32_t`，`1` 表示触发，`0` 表示取消。该命令只会触发一次，算法等 AE 快速收敛后会清除此状态，一般用于 touch 场景。

14. **ISP_EFFECT_CMD_S_AECOMPENSATION_STEP**
   设置 ISP 曝光补偿 step 的值，对应参数类型是 `float`。设置后曝光补偿每次按照这个 step 进行调整，默认值为 1/3。

15. **ISP_EFFECT_CMD_S_AE_SCENE_MODE**
   设置 ISPAE 的场景模式，对应参数类型是 `int32_t`，值由枚举 `asrISP_AE_SCENE_MODE_E` 定义。ISP 支持的模式有普通和人脸模式，在人脸模式下，AE 的收敛速度比普通模式下快。

16. **ISP_EFFECT_CMD_S_FILTER_MODE**
   设置 ISP 颜色滤镜模式，对应参数类型是 `int32_t`，值由枚举 `asrISP_COLOR_FILTER_MODE_E` 定义。目前支持普通和黑白两种模式，默认工作再普通模式下。

17. **ISP_EFFECT_CMD_S_YUV_RANGE**
   设置 ISP 的 yuv range，对应参数类型是 `int32_t`，值由枚举 `asrISP_YUV_RANGE_E` 定义。目前支持 full（y:0~255, uv:0~255）和 compress（y:16~235, uv:16~240）两种模式。

18. **ISP_EFFECT_CMD_S_SOLID_COLOR_MODE**
   设置 ISP 纯色模式，对应参数类型是 `int32_t`，值由枚举 `asrISP_SOLID_COLOR_MODE_E` 定义。目前仅支持纯黑色模式，默认关闭此模式。

19. **ISP_EFFECT_CMD_G_AF_MODE**
   获取 ISP 当前 AF 模式的命令，对应参数类型是结构体 `asrISP_AF_INFO_S`。

20. **ISP_EFFECT_CMD_G_ANTIFLICKER_MODE**
   获取 ISP 当前 antiflicker 模式的命令，对应参数类型是 `int32_t`，该参数的值由枚举 `asrISP_ANTIF LICKER_MODE_E` 定义。

21. **ISP_EFFECT_CMD_G_METERING_MODE**
   获取 ISP 当前测光模式的命令，对应参数类型是结构体 `asrISP_METERING_INFO_S`。

22. **ISP_EFFECT_CMD_G_AF_MOTOR_RANGE**
   获取对焦马达移动的范围（最小值和最大值），对应参数类型是结构体 `asrISP_RANGE_S`。

23. **ISP_EFFECT_CMD_G_AE_MODE**
   获取当前 ISP 的 AE 模式，对应参数类型是结构体 `ISP_AE_INFO_S`。

24. **ISP_EFFECT_CMD_G_AWB_MODE**
   获取当前 ISP 的 AWB 模式，对应参数类型是结构体 `ISP_AWB_INFO_S`。

#### Tuning 相关 API

本小节描述的 API 用来帮助 tuning 工程师调节更具体、更细节的参数，ASR 发布的 tuning 工具也是使用该小节相关的 API。

tuning 相关 API 如下：

- **ASR_ISP_SetFwPara：** 设置 ISP pipeline 上 Firmware 的参数。
- **ASR_ISP_GetFWPara：** 获取 ISP pipeline 上 Firmware 的参数。
- **ASR_ISP_SetRegister：** 设置指定寄存器值。
- **ASR_ISP_GetRegister：** 获取指定寄存器值。
- **ASR_ISP_LoadSettingFile：** 加载 ISP pipeline 上 Firmware 的 setting file。
- **ASR_ISP_SaveSettingFile：** 保存 ISP pipeline 上 Firmware 的 setting file。

#### ASR_ISP_SetFwPara

【描述】

设置 ISP Firmware 参数。

【语法】

int ASR_ISP_SetFwPara(uint32_t pipelineID, const char *paramter, const char *name, uint32_t row, uint32_t column, int value);

【参数】

| 参数名称   | 描述               | 输入/输出 |
| ------------------- | --------------------------- | ------------------ |
| pipelineID | ISP pipeline 的 ID | 输入      |
| paramter   | ISP 子模块名称    | 输入      |
| name       | 参数名称          | 输入      |
| row        | 行号               | 输入      |
| column     | 列号               | 输入      |
| value      | 值                 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`
- 库文件：`libisp.so`

【注意】无

#### ASR_ISP_GetFWPara

【描述】

获取 ISP Firmware 参数。

【语法】

int ASR_ISP_GetFWPara(uint32_t pipelineID, const char *paramter, const char *name, uint32_t row, uint32_t column, int *pValue);

【参数】

| 参数名称   | 描述               | 输入/输出 |
| ------------------- | --------------------------- | ------------------ |
| pipelineID | ISP pipeline 的 ID | 输入      |
| paramter   | ISP 子模块名称    | 输入      |
| name       | 参数名称         | 输入      |
| row        | 行号               | 输入      |
| column     | 列号               | 输入      |
| value      | 值                 | 输出      |

【返回值】

| 参数名称 | 描述                      |
| ----------------- | ---------------------------------- |
| -1       | 输入检查错误          |
| 0        | 不能找到指定参数          |
| &gt;0       | 成功，返回获取的值数量 |

【需求】

- 头文件：`spm_cam_isp.h`
- 库文件：`libisp.so`

【注意】无

#### ASR_ISP_SetRegister

【描述】

设置指定寄存器值。

【语法】

int ASR_ISP_SetRegister(uint32_t addr, uint32_t value, uint32_t mask);

【参数】

| 参数名称 | 描述        | 输入/输出 |
| ----------------- | -------------------- | ------------------ |
| addr     | 寄存器地址  | 输入      |
| value    | 寄存器值    | 输入      |
| mask     | 寄存器 mask | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`
- 库文件：`libisp.so`

【注意】无

#### ASR_ISP_GetRegister

【描述】

获取指定寄存器值。

【语法】

int ASR_ISP_GetRegister(uint32_t addr, int *pValue);

【参数】

| 参数名称 | 描述       | 输入/输出 |
| ----------------- | ------------------- | ------------------ |
| addr     | 寄存器地址 | 输入      |
| pValue   | 寄存器值   | 输出      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`
- 库文件：`libisp.so`

【注意】无

#### ASR_ISP_LoadSettingFile

【描述】

加载 ISP Firmware setting file。

【语法】

int ASR_ISP_LoadSettingFile(uint32_t pipelineID, const char *pFileName);

【参数】

| 参数名称   | 描述                     | 输入/输出 |
| ------------------- | --------------------------------- | ------------------ |
| pipelineID | ISP pipeline 的 ID       | 输入      |
| pFileName  | 参数文件名，包含绝对路径 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`
- 库文件：`libisp.so`

【注意】无

#### ASR_ISP_SaveSettingFile

【描述】

保存 ISP Firmware setting file。

【语法】

int ASR_ISP_SaveSettingFile(uint32_t pipelineID, const char *pFileName);

【参数】

| 参数名称   | 描述                     | 输入/输出 |
| ------------------- | --------------------------------- | ------------------ |
| pipelineID | ISP pipeline 的 ID       | 输入      |
| pFileName  | 参数文件名，包含绝对路径 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`spm_cam_isp.h`
- 库文件：`libisp.so`

【注意】无

### 主要 API 使用流程

本小节主要描述系统控制 API 的使用流程，总共分为一下几个阶段：构造、注册、设置、初始化、streamon、streamoff、反初始化、析构。

1. **构造 ISP pipeline 的上下文环境**。

2. **注册**：向 ISP pipeline 注册 sensor 的回调函数、对焦马达的回调函数以及 frameinfo 的回调函数。

3. **设置**：设置 ISP pipeline 的图像公共属性、tuning 相关的属性、channel 工作的实际硬件 pipe ID、帧率、是否使能 offline 模式、frameinfo buffer 入队列。

4. **初始化**：根据以上步骤设置的信息，初始化 ISP pipeline。

5. **streamon**：启动 ISP pipeline，进入工作状态。

6. **streamoff**：停止 ISP pipeline 的运行，退出工作状态。

7. **反初始化**：清除设置的参数等信息 (除之前注册的回调外)。

8. **析构**：析构 ISP pipeline 的上下文环境。

以上流程面向 pipeline，ISP 软件能同时支持两个 pipeline 一起工作，如果需要启动另个 pipeline，流程类似，但是需要注意实际工作的硬件 pipe ID，因为有时分复用。

### 数据类型

### 错误码

| 错误代码 | 宏定义                    | 描述           |
| ----------------- | ---------------------------------- | ----------------------- |
| -22      | ASR_ERR_ISP_ILLEGAL_PARAM | 参数非法       |
| -17      | ASR_ERR_ISP_EXIST         | 资源已经存在   |
| -19      | ASR_ERR_ISP_NOTEXIST      | 资源不存在     |
| -22      | ASR_ERR_ISP_NULL_PTR      | 空指针         |
| -1       | ASR_ERR_ISP_NOT_SUPPORT   | 操作当前不支持 |
| -1       | ASR_ERR_ISP_NOT_PERM      | 操作不允许     |
| -12      | ASR_ERR_ISP_NOMEM         | 内存不足       |
| -14      | ASR_ERR_ISP_BADADDR       | 无效的地址     |
| -16      | ASR_ERR_ISP_BUSY          | 系统正忙       |

## VI API

本节介绍 ASR MARS11-ISP VI 模块 API 使用方法，描述的 API 都是 VI SDK 面向应用的，并详细解释了相关的参数数据结构和返回值。

### API

VI 模块提供以下 API：

- **ASR_VI_SetDevAttr**：设置 VI Device 的属性。
- **ASR_VI_GetDevAttr**：获取 VI Device 的属性。
- **ASR_VI_EnableDev**：使能 VI Device。
- **ASR_VI_DisableDev**：关闭 VI Device。
- **ASR_VI_SetChnAttr**：设置 VI Channel 的属性。
- **ASR_VI_GetChnAttr**：获取 VI Channel 的属性。
- **ASR_VI_SetCallback**：设置上层的 Callback 函数到 VI Device。
- **ASR_VI_EnableChn**：使能 VI channel。
- **ASR_VI_DisableChn**：关闭 VI channel。
- **ASR_VI_Init**：初始化 VI 模块资源。
- **ASR_VI_Deinit**：释放 VI 模块资源。
- **ASR_VI_SetBayerReadAttr**：设置 offline 读取 Bayer raw 的属性。
- **ASR_VI_GetBayerReadAttr**：获取 offline 读取 Bayer raw 的属性。
- **ASR_VI_EnableBayerRead**：使能 offline Bayer raw 读取。
- **ASR_VI_DisableBayerRead**：关闭 offline Bayer raw 读取。
- **ASR_VI_EnableBayerDump**：使能 Bayer Dump。
- **ASR_VI_DisableBayerDump**：关闭 Bayer Dump。
- **ASR_VI_ChnQueueBuffer**：向 channel 压入 buffer。

#### ASR_VI_SetDevAttr

【描述】

设置 VI 设备属性。基本设备属性默认了部分芯片配置,满足绝大部分的 sensor 对接要求。

【语法】

int32_t ASR_VI_SetDevAttr(uint32_t nDev, VI_DEV_ATTR_S *pstDevAttr)

【参数】

| 参数名称   | 描述                                           | 输入/输出 |
| ------------------- | ------------------------------------------------------- | ------------------ |
| nDev       | VI 设备号 取值范围：[0, VIU_MAX_DEV_NUM)。 | 输入      |
| pstDevAttr | VI 设备属性指针。静态属性。                    | 输入      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

【注意】

- 不支持更改设备与通道的绑定关系。
- 在调用前要保证 VI 设备处于禁用状态。如果 VI 设备已处于使能状态,可以使用 `ASR_CAM_VI_DisableDev` 来禁用设备。

#### ASR_VI_GetDevAttr

【描述】

获取 VI 设备属性。

【语法】

int32_t ASR_VI_GetDevAttr(uint32_t nDev, VI_DEV_ATTR_S *pstDevAttr)

【参数】

| 参数名称   | 描述                                          | 输入/输出 |
| ------------------- | ------------------------------------------------------ | ------------------ |
| nDev       | VI 设备号。 取值范围:[0, VIU_MAX_DEV_NUM) | 输入      |
| pstDevAttr | VI 设备属性指针。                             | 输出      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

#### ASR_VI_EnableDev

【描述】

启用 VI 设备。

【语法】

int32_t ASR_VI_EnableDev(uint32_t nDev);

【参数】

| 参数名称 | 描述                                            | 输入/输出 |
| ----------------- | -------------------------------------------------------- | ------------------ |
| nDev     | VI 设备号。 取值范围:[0, VIU_MAX_DEV_NUM)。 | 输入      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

【注意】

- 启用前必须已经设置设备属性，否则返回失败。
- 可重复启用，不返回失败。

#### ASR_VI_DisableDev

【描述】

禁用 VI 设备。

【语法】

int32_t ASR_VI_DisableDev(uint32_t nDev);

【参数】

| 参数名称 | 描述                                            | 输入/输出 |
| ----------------- | -------------------------------------------------------- | ------------------ |
| nDev     | VI 设备号。 取值范围:[0, VIU_MAX_DEV_NUM)。 | 输入      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

【注意】

- 必须先禁用所有与该 VI 设备绑定的 VI 通道后，才能禁用 VI 设备。
- 可重复禁用，不返回失败。

#### ASR_VI_FlushDev

【描述】

将已压入 VI 设备中的 Buffer flush 出去。

【语法】

int32_t ASR_VI_FlushDev(uint32_t nDev);

【参数】

| 参数名称 | 描述                                            | 输入/输出 |
| ----------------- | -------------------------------------------------------- | ------------------ |
| nDev     | VI 设备号。 取值范围:[0, VIU_MAX_DEV_NUM)。 | 输入      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

#### ASR_VI_SetChnAttr

【描述】

设置 VI 通道属性。

【语法】

int32_t ASR_VI_SetChnAttr(uint32_t nChn, VI_CHN_ATTR_S *pstAttr);

【参数】

| 参数名称 | 描述                                          | 输入/输出 |
| ----------------- | ------------------------------------------------------ | ------------------ |
| nChn     | VI 通道号。 取值范围:[0, VIU_MAX_CHN_NUM) | 输入      |
| pstAttr  | VI 通道属性指针。静态属性。                   | 输入      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

【注意】

- 必须先设置设备属性才能设置通道属性，否则会返回失败。
- 通道必须处于 Disable 状态才能设置通道属性。

#### ASR_VI_GetChnAttr

【描述】

获取 VI 通道属性。

【语法】

int32_t ASR_VI_GetChnAttr(uint32_t nChn, VI_CHN_ATTR_S *pstAttr);

【参数】

| 参数名称 | 描述                                          | 输入/输出 |
| ----------------- | ------------------------------------------------------ | ------------------ |
| nChn     | VI 通道号。 取值范围:[0, VIU_MAX_CHN_NUM) | 输入      |
| pstAttr  | VI 通道属性指针。静态属性。                   | 输出      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

【注意】

- 必须先设置通道属性再获取属性，否则将返回失败。

#### ASR_VI_EnableChn

【描述】

启用 VI 通道。

【语法】

int32_t ASR_VI_EnableChn(uint32_t nChn);

【参数】

| 参数名称 | 描述                                          | 输入/输出 |
| ----------------- | ------------------------------------------------------ | ------------------ |
| nChn     | VI 通道号。 取值范围:[0, VIU_MAX_CHN_NUM) | 输入      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

【注意】

- 必须先设置通道属性，且通道所绑定的 VI 设备必须使能。
- 可重复启用 VI 通道，不返回失败。

#### ASR_VI_DisableChn

【描述】

禁用 VI 通道。

【语法】

int32_t ASR_VI_DisableChn(uint32_t nChn);

【参数】

| 参数名称 | 描述                                          | 输入/输出 |
| ----------------- | ------------------------------------------------------ | ------------------ |
| nChn     | VI 通道号。 取值范围:[0, VIU_MAX_CHN_NUM) | 输入      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

【注意】

- 可重复禁用 VI 通道，不返回失败。

#### ASR_VI_SetCallback

【描述】

设置 frame buffer 轮转用的 callback。

【语法】

int32_tASR_VI_SetCallback(uint32_tnChn,int32_t( *callback)(uint32_tnChn, VI_IMAGE_BUFFER_S *vi_buffer));

【参数】

| 参数名称 | 描述                                          | 输入/输出 |
| ----------------- | ------------------------------------------------------ | ------------------ |
| nChn     | VI 通道号。 取值范围:[0, VIU_MAX_CHN_NUM) | 输入      |
| callback | 回调函数指针                                  | 输入      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

【注意】

- 在 `ASR_VI_EnalbeChn` 之前调用。

#### ASR_VI_SetBayerReadAttr

【描述】

设置 offline 处理 raw 读入属性。

【语法】

int32_tASR_VI_SetBayerReadAttr(uint32_tnDev,constVI_BAYER_READ_ATTR_S *pstBayerReadAttr);

【参数】

| 参数名称         | 描述                                          | 输入/输出 |
| ------------------------- | ------------------------------------------------------ | ------------------ |
| nDev             | VI 设备号。 取值范围:[0, VIU_MAX_DEV_NUM) | 输入      |
| pstBayerReadAttr | Bayer 读取属性结构体指针。                    | 输入      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

#### ASR_VI_GetBayerReadAttr

【描述】

获取 offline 处理 raw 读入属性。

【语法】

int32_TASR_VI_GetBayerReadAttr(uint32_tnDev,constVI_BAYER_READ_ATTR_S *pstBayerReadAttr);

【参数】

| 参数名称         | 描述                                          | 输入/输出 |
| ------------------------- | ------------------------------------------------------ | ------------------ |
| nDev             | VI 设备号。 取值范围:[0, VIU_MAX_DEV_NUM) | 输入      |
| pstBayerReadAttr | Bayer 读取属性结构体指针。                    | 输出      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

#### ASR_VI_EnableBayerRead

【描述】

使能 offline 处理。

【语法】

int32_t ASR_VI_EnableBayerRead (uint32_t nDev);

【参数】

| 参数名称 | 描述                                          | 输入/输出 |
| ----------------- | ------------------------------------------------------ | ------------------ |
| nDev     | VI 设备号。 取值范围:[0, VIU_MAX_DEV_NUM) | 输入      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

#### ASR_VI_DisableBayerRead

【描述】

关闭 offline 处理。

【语法】

int32_t ASR_VI_DisableBayerRead (uint32_t nDev);

【参数】

| 参数名称 | 描述 | 输入/输出 |
| ---------| ------| -------|
| nDev     | VI 设备号。 取值范围:[0, VIU_MAX_DEV_NUM) | 输入      |

【返回值】

| 返回值 | 描述   |
| --------------- | --------|
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

#### ASR_VI_EnableBayerDump

【描述】

使能获取 RAW DATA。

【语法】

int32_t ASR_VI_EnableBayerDump(uint32_t nDev);

【参数】

| 参数名称 | 描述 | 输入/输出 |
| -------- | -------| ------|
| nDev     | VI 设备号。 取值范围:[0, VIU_MAX_DEV_NUM) | 输入      |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

【注意】

- 保存 RAW 数据的宽高与 DEV 的 设置宽高一致。
- 可通过 `VIU_GET_RAW_CHN` 获取 RAW Dump 通道的起始通道号。

#### ASR_VI_DisableBayerDump

【描述】

关闭获取 RAW DATA。

【语法】

int32_t ASR_VI_DisableBayerDump(uint32_t nDev);

【参数】

| 参数名称 | 描述    | 输入/输出 |
| ---------| -------| ----------|
| nDev     | VI 设备号。 取值范围:[0, VIU_MAX_DEV_NUM) | 输入      |

【返回值】

| 返回值 | 描述                |
| -------| --------------------|
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

#### ASR_VI_ChnQueueBuffer

【描述】

向 channel 压入 buffer。

【语法】

int32_t ASR_VI_ChnQueueBuffer (uint32_t nChn, IMAGE_BUFFER_S *camBuf);

【参数】

| 参数名称 | 描述     | 输入/输出 |
| ---------| ---------- | --------|
| nChn     | VI 通道号。 取值范围:[0, VI_CHN_CNT) | 输入      |
| camBuf   | Buffer 指针       | 输入      |

【返回值】

| 返回值 | 描述                |
| -------- | ----------|
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

【注意】

- 输入参数 camBuf 指针会被 SDK 直接使用。所以应用层应当保证该指针所指向的内存空间不能被释放，直到注册的回调函数被调用。

#### ASR_VI_Init

【描述】

VI 模块初始化。

【语法】

int32_t ASR_VI_Init (void);

【参数】

| 参数名称 | 描述 | 输入/输出 |
| ----------------- | ------------- | ------------------ |
| 无       |               |                    |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

头文件：`spm_comm_vi.h`、`spm_cam_vi.h`

库文件：`libvi.so`

#### ASR_VI_Deinit

【描述】

VI 模块反初始化。

【语法】

int32_t ASR_VI_Deinit (void);

【参数】

| 参数名称 | 描述 | 输入/输出 |
| ----------------- | ------------- | ------------------ |
| 无       |               |                    |

【返回值】

| 返回值 | 描述                |
| --------------- | ---------------------------- |
| 0      | 成功。              |
| 非 0   | 失败,其值为错误码。 |

【需求】

- 头文件：`spm_comm_vi.h`、`spm_cam_vi.h`
- 库文件：`libvi.so`

### 数据类型

#### VI_DEV_ATTR_S

【说明】

定义视频输入设备的属性。

【定义】

```java
typedef struct asrVI_DEV_ATTR_S { 
    CAM_VI_WORK_MODE_E enWorkMode; 
    CAM_SENSOR_RAWTYPE_E enRawType; 
    uint32_t width;
    uint32_t height; 
    uint32_t bindSensorIdx;
    uint32_t mipi_lane_num; 
    bool bCapture2Preview;
} VI_DEV_ATTR_S;
```

【成员】

| 成员名称  | 描述   |
| ----------| ------ |
| enWorkMode       | CAM_VI_WORK_MODE_ONLINE, CAM_VI_WORK_MODE_RAWDUMP, CAM_VI_WORK_MODE_OFFLINE,   |
| enRawType        | CAM_SENSOR_RAWTYPE_RAW8, CAM_SENSOR_RAWTYPE_RAW10, CAM_SENSOR_RAWTYPE_RAW12, CAM_SENSOR_RAWTYPE_RAW14, CAM_SENSOR_RAWTYPE_INVALID, |
| Width            | VI 设备可设置要捕获图像的宽。捕获图像的最小宽与最大宽度：[VIU_DEV_MIN_WIDTH, VIU_DEV_MAX_WIDTH] |
| Height           | VI 设备可设置要捕获图像的高。捕获图像的最小高与最大高度：[VIU_DEV_MIN_HEIGHT, VIU_DEV_MAX_HEIGHT] |
| bindSensorIdx    | VI 设备绑定的 sensor id。[0, 2] |
| bOfflineSlice    | Offline 是否是拍照模式 |
| mipi_lane_num    | Sensor MIPI 接口的 lane 个数 |
| bCapture2Preview | 是否是拍照回预览  |

#### VI_CHN_ATTR_S

【说明】

定义 VI 通道属性。

【定义】

```java
typedef struct asrVI_CHN_ATTR_S { 
    CAM_VI_PIXEL_FORMAT_E enPixFormat; 
    uint32_t width;
    uint32_t height;
} VI_CHN_ATTR_S;
```

【成员】

| 成员名称    | 描述             |
| -------------------- | ------------------------- |
| enPixFormat | Channel 输出格式 |
| Width       | 输出图像宽       |
| Height      | 输出图像高       |

#### VI_BAYER_READ_ATTR_S

【说明】

定义 offline raw 读入属性。

【定义】

```java
typedef struct asrVI_BAYER_READ_ATTR_S {
    bool bGenTiming;
    int32_t s32FrmRate;
} VI_BAYER_READ_ATTR_S;
```

【成员】

| 成员名称   | 描述                                  |
| ------------------- | --------------------------- |
| bGenTiming | 是否由 VI 自动产生固定帧率的读入时序  |
| s32FrmRate | 如果 bGenTiming 为 true，表示帧率大小 |

#### VI_IMAGE_BUFFER_S

【说明】

定义 VI channel 的 buffer 回调函数中 buffer 的结构体。

【定义】

```java
Typedef struct asrVI_IMAGE_BUFFER_S { 
    IMAGE_BUFFER_S *buffer;
    Bool bValid;
    Bool bCloseDown;
    Uint64_t timestamp; 
    Uint32_t frameId;
} VI_IMAGE_BUFFER_S;
```

【成员】

| 成员名称    | 描述     |
| -------------------- | ------------------------ |
| Buffer      | 指向 buffer 内存的结构体指针    |
| bValid  | 表示当前帧内容是否有效，无效时，上层应用应该丢弃当前帧     |
| bCloseDown  | RAW DATA channel 专用，RAW DATA channel 做 disable 操作前需要先收到 buffer 回调中的 bCloseDown 信号 |
| timeStamp   | 表示当前帧的时间戳   |
| frameId     | 表示当前帧的帧号，从 0 开始计 |

#### VIU_GET_RAW_CHN

【说明】

定义获取 RAW DATA 的通道号。

【定义】

```java
#define VIU_GET_RAW_CHN(ViDev, RawChn) do{
        RawChn = VIU_MAX_CHN_NUM + ViDev;
    }while(0)
```

#### VIU_MAX_CHN_NUM

【说明】

定义 VI 通道的最大个数。

【定义】

```java
#define VIU_MAX_CHN_NUM (VIU_MAX_PHYCHN_NUM)
```

#### VIU_MAX_PHYCHN_NUM

【说明】

定义 VI 物理通道的最大个数。

【定义】

```java
#define VIU_MAX_PHYCHN_NUM 2
```

#### VIU_MAX_RAWCHN_NUM

【说明】

定义获取 RAW DATA 通道的最大个数。

【定义】

```java
#define VIU_MAX_RAWCHN_NUM 2
```

#### VIU_MAX_DEV_NUM

【说明】

定义视频输入设备的最大个数。

【定义】

```java
#define VIU_MAX_DEV_NUM 2
```

#### VIU_DEV_MIN_WIDTH

【说明】

VI 设备捕获图像的最小宽度。

【定义】

```java
#define VIU_DEV_MIN_WIDTH 256
```

#### VIU_DEV_MIN_HEIGHT

【说明】

VI 设备捕获图像的最小高度。

【定义】

```java
#define VIU_DEV_MIN_HEIGHT 144
```

#### VIU_DEV_MAX_WIDTH

【说明】

VI 设备捕获图像的最大宽度。

【定义】

```java
#define VIU_DEV_MAX_WIDTH 2688
```

#### VIU_DEV_MAX_HEIGHT

【说明】

VI 设备捕获图像的最大高度。

【定义】

```java
#define VIU_DEV_MAX_HEIGHT 1944
```

#### VIU_CHN_MIN_WIDTH

【说明】

VI 物理通道支持的最小宽度。

【定义】

```java
#define VIU_CHN_MIN_WIDTH VIU_DEV_MIN_WIDTH
```

#### VIU_CHN_MIN_HEIGHT

【说明】

VI 物理通道支持的最小高度。

【定义】

```java
#define VIU_CHN_MIN_HEIGHT VIU_DEV_MIN_HEIGHT
```

#### VIU_CHN_MAX_WIDTH

【说明】

VI 物理通道支持的最大宽度。

【定义】

```java
#define VIU_CHN_MAX_WIDTH VIU_DEV_MAX_WIDTH
```

#### VIU_CHN_MAX_HEIGHT

【说明】

VI 物理通道支持的最大高度。

【定义】

```java
#define VIU_CHN_MAX_HEIGHT VIU_DEV_MAX_HEIGHT
```

## CPP API

本节介绍 ASR MARS11-ISP CPP 模块 API 使用方法，描述的 API 都是 CPP SDK 面向应用的，并详细解释了相关的参数数据结构、错误码和返回值。

### API

CPP 模块为用户提供以下 API：

- **cam_cpp_create_grp**：创建一个 CPP 模块 group。
- **cam_cpp_destroy_grp**：销毁一个 CPP 模块 group。
- **cam_cpp_start_grp**：启用 cam cpp 模块。
- **cam_cpp_stop_grp**：禁用 cam cpp 模块。
- **cam_cpp_post_buffer**：用户向 CPP 发送数据。
- **cam_cpp_set_callback**：用户设置回调函数。
- **cam_cpp_get_grp_attr**：获取 CPP Group 属性。
- **cam_cpp_set_grp_attr**：设置 CPP Group 属性。
- **cam_cpp_get_tuning_param**：获取 CPP Group 的 tuning 参数。
- **cam_cpp_set_tuning_param**：设置 CPP Group 的 tuning 参数。
- **cam_cpp_load_settingfile**：tuning 调试接口，加载 firmware setting file。
- **cam_cpp_save_settingfile**：tuning 调试接口，保存 frimware setting file。
- **cam_cpp_read_fw**：调试接口，获取 CPP Group Firmware 模块的参数。
- **cam_cpp_write_fw**：tuning 调试接口，设置 CPP Group Firmware 模块的参数。
- **cam_cpp_read_reg**：tuning 调试接口，获取 CPP Hardware 的寄存器值。
- **cam_cpp_write_reg**：tuning 调试接口，设置 CPP Hardware 的寄存器值。
- **cam_cpp_dump_frame**：保存 group 的输出和相应输入图像到指定目录。

#### cam_cpp_create_grp

【描述】

创建一个 cam CPP 模块 group。

【语法】

int32_t cam_cpp_create_grp(uint32_t grpId);

【参数】

| 参数名称 | 描述    | 输入/输出 |
| ----------------- | --------------------- | ------------------ |
| grpId    | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

- 不支持重复创建。

【关联操作】

`cam_cpp_destroy_grp`

#### cam_cpp_destroy_grp

【描述】

销毁一个 cam CPP 模块 group。

【语法】

int32_t cam_cpp_destroy_grp(uint32_t grpId);

【参数】

| 参数名称 | 描述    | 输入/输出 |
| ----------------- | -------------------- | ------------------ |
| grpId    | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | -------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

- CPP group 必须已创建。
- 调用此接口之前，必须先调用 `cam_cpp_stop_grp` 禁用此 group。
- 调用此接口时，会一直等待此 group 当前任务处理结束才会真正销毁。

【关联操作】

`cam_cpp_create_grp`

#### cam_cpp_start_grp

【描述】

启动 cam CPP 模块。

【语法】

int32_t cam_cpp_start_grp(uint32_t grpId);

【参数】

| 参数名称 | 描述               | 输入/输出 |
| ----------------- | ---------------------| ------------------ |
| grpId    | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

- 不支持重复创建。
- 重复调用该函数设置同一个 group 时返回成功。

【关联操作】

`cam_cpp_stop_grp`

#### cam_cpp_stop_grp

【描述】

禁用 cam CPP 模块 group。

【语法】

int32_t cam_cpp_stop_grp(uint32_t grpId);

【参数】

| 参数名称 | 描述                                            | 输入/输出 |
| ----------------- | -------------------------------------------------------- | ------------------ |
| grpId    | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

- 不支持重复创建。
- 重复调用该函数设置同一个 group 时返回成功。
- 通过 `cam_cpp_post_buffer` 接口发送给 cpp group 的 buffer, 会在 stop 的过程中 return。

【关联操作】

cam_cpp_start_grp

#### cam_cpp_post_buffer

【描述】

用户向 CPP 模块发送一帧图像数据。

【语法】

int32_t cam_cpp_post_buffer(uint32_t grpId, const IMAGE_BUFFER_S *inputBuf, const IMAGE_BUFFER_S *outputBuf, int32_t frameId, FRAME_INFO_S *frameInfo);

【参数】

| 参数名称  | 描述        | 输入/输出 |
| ------------------ | ----------------------| ------------------ |
| grpId     | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |
| inputBuf  | 输入图像的信息 具体描述请参见 IMAGE_BUFFER_S | 输入      |
| outputBuf | 输出图像的信息 具体描述请参见 IMAGE_BUFFER_S | 输出      |
| frameInfo | 具体描述请参见 FRAME_INFO_S                 | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`、`cam_module_interface.h`
- 库文件：`libcpp.so`

【注意】

- group 必须已创建。

【关联操作】

`cam_cpp_ReturnBuffer`

#### cam_cpp_set_callback

【描述】

用户设置回调函数。

【语法】

int32_t cam_cpp_set_callback(uint32_t grpId, CppCallback callback);

【参数】

| 参数名称 | 描述                                            | 输入/输出 |
| ----------------- | ---------------------| ------------------ |
| grpId    | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |
| callback | 当模块处理完毕，输出图像准备好了之后调用此接口  | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

Group 必须已创建。

【关联操作】

`cam_cpp_ReturnBuffer`

#### cam_cpp_get_grp_attr

【描述】

获取 CPP group 属性。

【语法】

int32_t cam_cpp_get_grp_attr(uint32_t grpId, CPP_GRP_ATTR_S *attr);

【参数】

| 参数名称 | 描述                                            | 输入/输出 |
| ----------------- | -------------------------------------------------------- | ------------------ |
| grpId    | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |
| attr     | CPP group 属性指针                              | 输出      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

- Group 必须已经创建。
- Group 属性必须合法，其中静态属性不可动态设置，具体请参见 `CPP_GRP_ATTR_S`。

【关联操作】

cam_cpp_set_grp_attr

#### cam_cpp_set_grp_attr

【描述】

设置 CPP group 属性。

【语法】

int32_t cam_cpp_set_grp_attr(uint32_t grpId, const CPP_GRP_ATTR_S *attr);

【参数】

| 参数名称 | 描述                       | 输入/输出 |
| ----------------- | ---------------------| ------------------ |
| grpId    | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |
| attr     | CPP group 属性指针                           | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

- Group 必须已经创建。
- Group 属性必须合法，其中静态属性不可动态设置，具体请参见 `CPP_GRP_ATTR_S`。

【关联操作】

`cam_cpp_get_grp_attr`

#### cam_cpp_get_tuning_param

【描述】

获取 CPP Group 的 tuning 参数。

【语法】

int32_t cam_cpp_get_tuning_param(uint32_t grpId, cpp_tuning_params_t *tuningParam);

【参数】

| 参数名称    | 描述                                            | 输入/输出 |
| -------------------- | ------------------------| ------------------ |
| grpId       | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |
| tuningParam | 模块 tuning 参数指针                            | 输出      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`、`CPPGlobalDefine.h`
- 库文件：`libcpp.so`

【注意】

- Group 必须已经创建。

【关联操作】

`cam_cpp_set_tuning_param`

#### cam_cpp_set_tuning_param

【描述】

设置 CPP Group 的 tuning 参数。

【语法】

int32_t cam_cpp_set_tuning_param(uint32_t grpId, cpp_tuning_params_t *tuningParam);

【参数】

| 参数名称    | 描述                                            | 输入/输出 |
| -------------------- | -------------------------| ------------------ |
| grpId       | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |
| tuningParam | 模块 tuning 参数指针                            | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`、`CPPGlobalDefine.h`
- 库文件：`libcpp.so`

【注意】

- Group 必须已经创建。

【关联操作】

`cam_cpp_get_tuning_param`

#### cam_cpp_load_settingfile

【描述】

Tuning 调试接口，加载 firmware setting file。

【语法】

int32_t cam_cpp_load_settingfile(uint32_t grpId, const char *fileName);

【参数】

| 参数名称 | 描述                                            | 输入/输出 |
| ----------------- | -------------------------------------------------------- | ------------------ |
| grpId    | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |
| fileName | cpp firmware 配置文件路径指针                   | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

- Group 必须已经创建。

【关联操作】

`cam_cpp_save_settingfile`

#### cam_cpp_save_settingfile

【描述】

Tuning 调试接口，保存 firmware setting file。

【语法】

int32_t cam_cpp_save_settingfile(uint32_t grpId, const char *fileName);

【参数】

| 参数名称 | 描述                                            | 输入/输出 |
| ----------------- | -------------------------------------------------------- | ------------------ |
| grpId    | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |
| fileName | cpp firmware 配置文件路径指针                   | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

- Group 必须已经创建。

【关联操作】

`cam_cpp_load_settingfile`

#### cam_cpp_read_fw

【描述】

Tuning 调试接口，获取 CPP Group Firmware 模块的参数。

【语法】

int32_t cam_cpp_read_fw(uint32_t grpId, const char *filter, const char *param, uint32_t row, uint32_t column, int32_t *pVal);

【参数】

| 参数名称 | 描述                                            | 输入/输出 |
| ----------------- | --------------------- | ------------------ |
| grpId    | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |
| filter   | Firmware Filter 名指针                          | 输入      |
| param    | Firmware Paramter 名指针                        | 输入      |
| row      | Firmware Paramter 元素的行偏移                  | 输入      |
| column   | Firmware Paramter 元素的列偏移                  | 输入      |
| pVal     | Firmware Paramter 元素的列表指针                | 输出      |

【返回值】

| 参数名称        | 描述                    |
| ------------------------ | ----------------------------- |
| -1              | 失败. 检查输入参数名。   |
| 0               | 失败. 没有获取到参数数值。 |
| Positive values | 成功.表示获取参数的元素个数. 1 表示单个元素, &gt;1 表示多个元素 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

- Group 必须已经创建。

【关联操作】

`cam_cpp_write_fw`

#### cam_cpp_write_fw

【描述】

Tuning 调试接口，设置 CPP Group Firmware 模块的参数。

【语法】

int32_tcam_cpp_write_fw(uint32_tgrpId,constchar *pFilterName,constchar

 *pParamName, uint32_t row, uint32_t column, int32_t val);

【参数】

| 参数名称 | 描述     | 输入/输出 |
| ----------------- | -------------------- | ------------------ |
| grpId    | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |
| filter   | Firmware Filter 名指针                          | 输入      |
| param    | Firmware Paramter 名指针                        | 输入      |
| row      | Firmware Paramter 元素的行偏移                  | 输入      |
| column   | Firmware Paramter 元素的列偏移                  | 输入      |
| s32Val   | Firmware Paramter 元素值                        | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

- Group 必须已经创建。

【关联操作】

`cam_cpp_read_fw`

#### cam_cpp_read_reg

【描述】

Tuning 调试接口，获取 CPP Hardware 寄存器值。

【语法】

int32_t cam_cpp_read_reg(uint32_t addr, uint32_t *val);

【参数】

| 参数名称 | 描述           | 输入/输出 |
| ----------------- | ----------------------- | ------------------ |
| addr     | 硬件寄存器地址 | 输入      |
| val      | 硬件寄存器值   | 输出      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

- Group 必须已经创建。
- 只有一套硬件寄存器资源，并且低 16-bit 指定地址偏移。

【关联操作】

`cam_cpp_write_reg`

#### cam_cpp_write_reg

【描述】

Tuning 调试接口，设置 CPP Hardware 寄存器值。

【语法】

int32_t cam_cpp_write_reg(uint32_t addr, uint32_t val, uint32_t mask);

【参数】

| 参数名称 | 描述            | 输入/输出 |
| ----------------- | ------------------------ | ------------------ |
| addr     | 硬件寄存器地址  | 输入      |
| val      | 硬件寄存器值    | 输入      |
| mask     | 硬件寄存器 mask | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

- Group 必须已经创建。
- 只有一套硬件寄存器资源，并且低 16-bit 指定地址偏移。

【关联操作】

`cam_cpp_read_reg`

#### cam_cpp_dump_frame

【描述】

保存 group 指定帧数的输入输出图像到指定目录。

【语法】

int32_t cam_cpp_dump_frame(uint32_t grpId, const char *path, uint32_t count);

【参数】

| 参数名称 | 描述                                            | 输入/输出 |
| ----------------- | -------------------------------------------------------- | ------------------ |
| grpId    | CPP group ID 取值范围：[0, CPP_GRP_MAX_NUM) | 输入      |
| path     | 指定 dump 目录，不能为 NULL                     | 输入      |
| count    | 指定需要连续 dump 的帧数。                      | 输入      |

【返回值】

| 参数名称 | 描述             |
| ----------------- | ------------------------- |
| 0        | 成功             |
| 非 0     | 失败，值为错误码 |

【需求】

- 头文件：`cam_cpp.h`
- 库文件：`libcpp.so`

【注意】

- Group 必须已经创建。
- 该 API 为异步操作，返回成功后连续 dump 该时刻起的连续 count 帧数，当指定帧数未 dump 结束时，再次调用该接口会中断上一次的操作并开始当前操作。

【关联操作】无

### 数据类型

CPP 模块相关数据类型定义如下：

- **CPP_GRP_MAX_NUM：** 定义 CPP GROUP 的最大个数。
- **CPP_MIN_IMAGE_WIDTH：** 定义 CPP 图像的最小宽度。
- **CPP_MIN_IMAGE_HEIGHT：** 定义 CPP 图像的最小高度。
- **CPP_MAX_IMAGE_WIDTH：** 定义 CPP 图像的最大宽度。
- **CPP_MAX_IMAGE_HEIGHT：** 定义 CPP 图像的最大高度。
- **CPP_GRP：** 定义 CPP 组号。
- **CPP_MOD_ID_E：** 定义 CPP 子模块 ID。
- **CPP_GRP_ATTR_S：** 定义 CPP GROUP 属性。
- **CPP_GRP_NR_ATTR_S：** 定义 CPP GROUP 的 NR 属性。
- **CPP_GRP_EE_ATTR_S：** 定义 CPP GROUP 的 EE 属性。
- **CPP_GRP_TNR_ATTR_S：** 定义 CPP GROUP 的 TNR 属性。

#### CPP_GRP_MAX_NUM

【说明】

定义 CPP Group 的最大个数。

【定义】
```
#define CPP_GRP_MAX_NUM (16)
```
【成员变量】无

【注意】无

【相关类型及数据结构】无

#### CPP_MIN_IMAGE_WIDTH

【说明】

定义 CPP 模块支持图像的最小宽度。

【定义】
```
#define CPP_MIN_IMAGE_WIDTH (480)
```
【成员变量】无

【注意】无

【相关类型及数据结构】无

#### CPP_MIN_IMAGE_HEIGHT

【说明】

定义 CPP 图像支持的最小高度。

【定义】
```
#define CPP_MIN_IMAGE_HEIGHT (288)
```
【成员变量】无

【注意】无

【相关类型及数据结构】无

#### CPP_MAX_IMAGE_WIDTH

【说明】

定义 CPP 图像支持的最大宽度。

【定义】
```
#define CPP_MAX_IMAGE_WIDTH (4224)
```
【成员变量】无

【注意】无

【相关类型及数据结构】无

#### CPP_MAX_IMAGE_HEIGHT

【说明】

定义 CPP 图像的最大高度。

【定义】
```
#define CPP_MAX_IMAGE_HEIGHT (3136)
```
【成员变量】无

【注意】无

【相关类型及数据结构】无

#### CPP_MOD_ID_E

【说明】

定义 CPP 子模块 ID。

【定义】

```java
typedef enum { 
    CPP_ID_3DNR, 
    CPP_ID_MAX,
} CPP_MOD_ID_E
```

【成员变量】无

【注意】无

【相关类型及数据结构】无

#### CPP_GRP_ATTR_S

【说明】

定义 CPP Group 的属性。

【定义】

```java
typedef struct asrCPP_GRP_ATTR_S { 
    uint32_t width;
    uint32_t height; 
    PIXEL_FORMAT_E format; 
    CPP_GRP_WORKMODE_E mode;
} CPP_GRP_ATTR_S;
```

【成员变量】

| 变量         | 说明    
| --------------------- | ------------------------ |
| width height | 输入图像的尺寸 宽度的有效范围：[CPP_MIN_IMAGE_WIDTH, CPP_MAX_IMAGE_WIDTH] 高度的有效范围：[CPP_MIN_IMAGE_HEIGHT, CPP_MAX_IMAGE_HEIGHT] 这是静态属性。也就是说，group 创建之后设置，不能动态修改。 |
| format       | 输出图像的格式 这是静态属性。也就是说，group 创建之后设置，不能动态修改。         |
| mode         | 工作模式   |

【注意】

- 输入图像尺寸必须根据 CPP 工作的实际宽度和高度配置。根据输入图像尺寸分配 TNR KGain buffer 和输出图像 buffer。输入图像格式只支持 YUV 数据。

【相关类型及数据结构】

- `PIXEL_FORMAT_E` 
- `CPP_GRP_WORKMODE_E`

#### CPP_GRP_WORKMODE_E

【说明】

定义 CPP Group 的工作模式。

【定义】

```java
typedef enum { 
    CPP_GRP_FRAME_MODE, 
    CPP_GRP_SLICE_MODE, 
    CPP_GRP_MODE_MAX,
} CPP_GRP_WORKMODE_E;
```

【成员变量】

| 成员名称           | 描述                                    |
| --------------------------- | ------------------------------|
| CPP_GRP_FRAME_MODE | 数据按照整帧处理，硬件处理优先级高      |
| CPP_GRP_SLICE_MODE | 数据划分成 slice 处理，硬件处理优先级低 |

【注意】无

【相关类型及数据结构】无

### 错误码

参考 Standard C library 头文件 `errno.h`

## tuningtools API

本节介绍 ASR MARS11-ISP tuningtools 模块 API 使用方法，并详细解释了相关的参数数据结构和返回值。

### API

tuningtools 为用户提供以下 API：

- **ASR_TuningAssistant_Create：** 创建一个 tuning Assistant。
- **ASR_TuningAssistant_Destroy：** 销毁一个 tuning Assistant。

#### ASR_TuningAssistant_Create

【描述】

创建一个 tuningtool Assistant。

【语法】

```
ASR_TUNING_ASSISTANT_HANDLE

ASR_TuningAssistant_Create(const ASR_TUNING_ASSISTANT_TRIGGER_S *trigger);
```

【参数】

| 参数名称 | 描述                      | 输入/输出 |
| ----------------- | ---------------- | ------------------ |
| trigger  | tuning assistant 配置参数 | 输入      |

【返回值】

| 参数名称 | 描述                             |
| ----------------- | ----------------------------------------- |
| 非 NULL  | 成功，返回 assistant handle 指针 |
| NULL     | 失败                             |

【需求】

- 头文件：`asr_cam_tuning_assistant.h`
- 库文件：`libtuningtools.so`

【注意】

- 不支持重复创建。
- `ASR_TUNING_ASSISTANT_TRIGGER_S` 结构参见下一节数据类型描述。

【关联操作】

`ASR_TuningAssistant_Destroy`

#### ASR_TuningAssistant_Destroy

【描述】

销毁一个 tuningtool Assistant。

【语法】
```
bool ASR_TuningAssistant_Destroy(ASR_TUNING_ASSISTANT_HANDLE handle);
```
【参数】

| 参数名称 | 描述                                                                            | 输入/输出 |
| ----------------- | -------------------- | ------------------ |
| handle   | 待销毁 tuningassistant 结构指针，必须是 `ASR_TuningAssistant_Create` 的返回值 | 输入      |

【返回值】

| 参数名称 | 描述 |
| ----------------- | ------------- |
| true     | 成功 |
| false    | 失败 |

【需求】

- 头文件：`asr_cam_tuning_assistant.h`
- 库文件：`libtuningtools.so`

【注意】

- 不支持重复销毁。

【关联操作】

`ASR_TuningAssistant_Create`

### 数据类型

#### ASR_TUNING_ASSISTANT_HANDLE

【说明】

定义 tuning assistant handler 指针。

【定义】
```
typedef void *ASR_TUNING_ASSISTANT_HANDLE;
```
【成员】无

【注意】无

【相关类型及数据结构】无

#### TUNING_MODULE_TYPE_E

【说明】

定义可支持的 tuning 模块类型。

【定义】

```java
typedef enum TUNING_MODULE_TYPE { 
    TUNING_MODULE_TYPE_ISP, 
    TUNING_MODULE_TYPE_CPP, 
    TUNING_MODULE_TYPE_ALGO, 
    TUNING_MODULE_TYPE_MAX
} TUNING_MODULE_TYPE_E;
```

【成员】

| 成员名称                | 描述                         |
| -------------------------------- | ------------------------------------- |
| TUNING_MODULE_TYPE_ISP  | ISP 模块类型                 |
| TUNING_MODULE_TYPE_CPP  | CPP 模块类型                 |
| TUNING_MODULE_TYPE_ALGO | 算法模块类型（用户无需关心） |
| TUNING_MODULE_TYPE_MAX  | 可 tuning 的模块类型数最大值 |

【注意】无

【相关类型及数据结构】无

#### TUNING_BUFFER_S

【说明】

定义 tuning 模块打开 dump raw 图功能时用到的 buffer 结构。

【定义】

```java
typedef struct TUNING_BUFFER { 
    uint32_t frameId;
    uint32_t length; 
    void *virAddr;
} TUNING_BUFFER_S, *TUNING_BUFFER_S_PTR;
```

【成员】

| 成员名称 | 描述                 |
| ----------------- | ----------------------------- |
| frameId  | 图像帧号             |
| length   | 图像 buffer 长度     |
| virAddr  | 图像 buffer 起始地址 |

【注意】

跟 `ASR_TUNING_ASSISTANT_TRIGGER_S` 配合使用，用于 `StartDumpRaw` 的入参之一

【相关类型及数据结构】无

#### TUNING_MODULE_OBJECT_S

【说明】

定义待 tuning 的单个模块实体的基本信息。

【定义】

```java
typedef struct TUNING_MODULE_OBJECT { 
    TUNING_MODULE_TYPE_E type;
    void *moduleHandle; 
    uint32_t groupId; 
    uint8_t dumpRaw; 
    char name[32];
    int (*loadSettingFile)(void *handle, const char *filename); 
    int (*saveSettingFile)(void *handle, const char *filename);
    int (*saveFilterSettingFile)(void *handle, const char *filtername, const char *filename);
    int (*readFirmware) (void *handle, const char *filter, const char *param, int32_t row, int32_t colum, int32_t *pVal);
    int (*writeFirmware) (void *handle, const char *filter, const char *param, int32_t row, int32_t colum, int32_t val);
} TUNING_MODULE_OBJECT_S, *TUNING_MODULE_OBJECT_S_PTR;
```

【成员】

| 成员名称              | 描述                      |
| ------------------------------ | --------------------------------- |
| type                  | 待 tuning 模块的类型          |
| moduleHandle          | 待 tuning 模块的 handle 指针，算法模块使用 对于 ISP 和 CPP 类型无需关注，tuningtools 内部实现              |
| groupId               | 待 tuning 模块的组号 对于 ISP 类型， 取值范围[0, 1] 对于 CPP 类型，取值范围[0, 15]                     |
| dumpRaw               | 是否支持 dump RAW 图像数据      |
| name                  | 待 tuning 模块的名称            |
| loadSettingFile       | 对于算法模块，为 load setting file 的回调 对于 ISP 和 CPP， 无需关注，tuningtools 内部实现                 |
| saveSettingFile       | 对于算法模块，为 save setting file 的回调 对于 ISP 和 CPP， 无需关注，tuningtools 内部实现                 |
| saveFilterSettingFile | 对于算法模块，为 save 某个特定 filter setting file 的回调 对于 ISP 和 CPP， 无需关注，tuningtools 内部实现 |
| readFirmware          | 对于算法模块，为读取某个特定 filter 值的回调 对于 ISP 和 CPP， 无需关注，tuningtools 内部实现              |
| writeFirmware         | 对于算法模块，为设置某个特定 filter 值的回调 对于 ISP 和 CPP， 无需关注，tuningtools 内部实现              |

【注意】

tuning ISP 和 CPP 时，重点关注 `type`、`groupId`、`name` 即可。`dumpRaw` 一般设置为 `0`。

【相关类型及数据结构】无

#### ASR_TUNING_ASSISTANT_TRIGGER

【说明】

定义待 tuning 的所有模块合集的基础回调。

【定义】

```java
typedef struct ASR_TUNING_ASSISTANT_TRIGGER { 
    uint32_t (*GetModuleCount)();
    int32_t (*GetModules)(uint32_t moduleCount, TUNING_MODULE_OBJECT_S *modules); 
    int32_t(*StartDumpRaw)(TUNING_MODULE_TYPE_Etype,uint32_tgroupId,uint32_t frameCount, TUNING_BUFFER_S *frames);
    int32_t (*EndDumpRaw)(TUNING_MODULE_TYPE_E type, uint32_t groupId);
} ASR_TUNING_ASSISTANT_TRIGGER_S, *ASR_TUNING_ASSISTANT_TRIGGER_S_PTR;
```

【成员】

| 成员名称       | 描述                                                                                              |
| ----------------------- | ------------------------------ |
| GetModuleCount | 返回需要 tuning 的模块总数               |
| GetModules     | 获取 `moduleCount` 个待 tuning 的模块列表（放在入参 `modules` 数组中）， 并返回实际上获取到的模块个数 |
| StartDumpRaw   | 开始 rawdump 的回调，不推荐使用   |
| EndDumpRaw     | 结束 rawdump 的回调，不推荐使用   |

【注意】

tuningISP 和 CPP 时，`TUNING_MODULE_OBJECT_S` 的成员应依据真实使用的 ISP 和 CPP `groupId` 而定，可以参考 `demo/online_pipeline_test.c` 中 `tuning_server_init()` 的用法。

【相关类型及数据结构】无
