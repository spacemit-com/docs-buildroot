---
sidebar_position: 2
---

# ISP API Development Guide

This document explains the SDK API interfaces.

## ISP API

This section introduces the usage of ISP module APIs. These APIs are part of the ISP SDK aimed at applications, mainly divided into:

- System Control APIs
- Image Effect Setting APIs
- Tuning-related APIs

It provides detailed explanations of relevant parameter data structures, error codes, and return values. The primary audience includes tuning and algorithm engineers focused on ISP effects, as well as application engineers developing image-related functionalities.

## System Control APIs

These APIs manage the overall operation of the ISP pipeline. They are primarily used by image processing, tuning, and application engineers.

- **ASR_ISP_Construct**: Constructs the ISP pipeline context environment.
- **ASR_ISP_Destruct**: Destroys the ISP pipeline context environment.
- **ASR_ISP_GetFwFrameInfoSize()**: Retrieves the size of the ISP firmware frame info structure.
- **ASR_ISP_RegSensorCallBack**: Registers a sensor callback function to the ISP pipeline.
- **ASR_ISP_UnRegSensorCallBack**: Unregisters a sensor callback from the ISP pipeline.
- **ASR_ISP_RegAfMotorCallBack**: Registers an autofocus motor callback to the ISP pipeline.
- **ASR_ISP_EnableOfflineMode**: Enables the ISP pipeline’s offline mode.
- **ASR_ISP_SetPubAttr**: Sets public attributes for the ISP pipeline.
- **ASR_ISP_SetTuningParams**: Sets tuning-related parameters for the ISP pipeline.
- **ASR_ISP_SetChHwPipeID**: Sets the actual hardware pipe ID for the pipeline channel.
- **ASR_ISP_Init**: Initializes the ISP pipeline.
- **ASR_ISP_DeInit**: Deinitializes the ISP pipeline.
- **ASR_ISP_EnablePDAF**: Enables PDAF (Phase Detection Auto Focus) in the ISP pipeline.
- **ASR_ISP_SetFps**: Sets the frame rate for the sensor/ISP on this pipeline.
- **ASR_ISP_SetFrameinfoCallback**: Sets the callback function for receiving frame info.
- **ASR_ISP_QueueFrameinfoBuffer**: Queues a frame info buffer in the pipeline.
- **ASR_ISP_FlushFrameinfoBuffer**: Flushes the frame info buffer queue.
- **ASR_ISP_Streamon**: Starts the streaming process in the ISP pipeline.
- **ASR_ISP_Streamoff**: Stops the streaming process in the ISP pipeline.
- **ASR_ISP_TriggerRawCapture**: Triggers a raw capture process.
- **ASR_ISP_ReInitPreviewChannel**: Reinitializes the preview channel.
- **ASR_ISP_NotifyOnceHDRRawCapture**: Notifies the pipeline to perform a single HDR raw capture.
- **ASR_ISP_UpdateNoneZslStreamAeParams**: Updates AE (Auto Exposure) parameters for non-ZSL streams.

#### ASR_ISP_Construct

**[Description]**
Constructs the ISP pipeline context.

**[Syntax]** 

```c
int ASR_ISP_Construct(uint32_t pipelineID);
```

**[Parameters]**

| Name        | Description               | In/Out |
|-------------|---------------------------|--------|
| pipelineID  | ID of the ISP pipeline    | Input  |

**[Return Value]**

| Value | Description        |
|-------|--------------------|
| 0     | Success            |
| Non-0 | Failure (error code) |

**[Requirements]**
- Header file: `spm_cam_isp.h`
- Library: `libisp.so`

**[Note]**  
- The `pipelineID` parameter refers to a virtual pipeline, each mapping to one ISP firmware instance.

#### ASR_ISP_Destruct

**[Description]** 
Destructs (destroys) the ISP pipeline context.

**[Syntax]**

```c
int ASR_ISP_Destruct(uint32_t pipelineID);
```

**[Parameters]**

| Name        | Description               | In/Out |
|-------------|---------------------------|--------|
| pipelineID  | ID of the ISP pipeline    | Input  |

**[Return Value]**

| Value | Description        |
|-------|--------------------|
| 0     | Success            |
| Non-0 | Failure (error code) |

**[Requirements]**
- Header file: `spm_cam_isp.h`
- Library: `libisp.so`

#### ASR_ISP_GetFwFrameInfoSize

**[Description]** 

Gets the size of the ISP Firmware `frameinfo` structure.

**[Syntax]**

```c
int ASR_ISP_GetFwFrameInfoSize(void);
```

**[Parameters]**  

None.

**[Return Value]**

| Return Value | Description                |
|--------------|----------------------------|
| &gt;0         | Size of the structure       |
| &lt;0         | Failure (error code value) |

**[Requirements]**

- Header file: `spm_cam_isp.h`  
- Library file: `libisp.so`

#### ASR_ISP_RegSensorCallBack

**[Description]** 

Registers a sensor callback function to the ISP pipeline.

**[Syntax]** 

```c
int ASR_ISP_RegSensorCallBack(
    uint32_t pipelineID,
    ISP_SENSOR_ATTR_S *pstSensorInfo,
    ISP_SENSOR_REGISTER_S *pstRegister
);
```

**[Parameters]**

| Parameter Name   | Description                                      | Input/Output |
|------------------|--------------------------------------------------|--------------|
| pipelineID       | ID of the ISP pipeline                           | Input        |
| pstSensorInfo    | Attributes of the sensor being registered        | Input        |
| pstRegister      | Pointer to the structure of sensor callbacks     | Input        |

**[Return Value]**

| Return Value | Description                |
|--------------|----------------------------|
| 0            | Success                    |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_sensor_comm.h`  
- Library files: `libisp.so`, `libcam_sensors.so`

#### ASR_ISP_UnRegSensorCallBack

**[Description]**
Unregisters the sensor callback function from the ISP pipeline.

**[Syntax]** 
```c
int ASR_ISP_UnRegSensorCallBack(
    uint32_t pipelineID,
    ISP_SENSOR_ATTR_S *pstSensorInfo
);
```

**[Parameters]**

| Parameter Name   | Description                             | Input/Output |
|------------------|-----------------------------------------|--------------|
| pipelineID       | ID of the ISP pipeline                  | Input        |
| pstSensorInfo    | Attributes of the sensor to be unregistered | Input     |

**[Return Value]**

| Return Value | Description                |
|--------------|----------------------------|
| 0            | Success                    |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_sensor_comm.h`  
- Library file: `libisp.so`

#### ASR_ISP_RegAfMotorCallBack

**[Description]** 

Registers the sensor AF (autofocus) motor callback function to the ISP pipeline.

**[Syntax]** 

```c
int ASR_ISP_RegAfMotorCallBack(
    uint32_t pipelineID,
    ISP_AF_MOTOR_REGISTER_S *pstAfRegister
);
```

**[Parameters]**

| Parameter Name  | Description                              | Input/Output |
|-----------------|------------------------------------------|--------------|
| pipelineID      | ID of the ISP pipeline                   | Input        |
| pstAfRegister   | Pointer to the sensor AF motor callback function structure | Input        |

**[Return Value]**

| Return Value | Description                |
|--------------|----------------------------|
| 0            | Success                    |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_sensor_comm.h`  
- Library file: `libisp.so`

#### ASR_ISP_EnableOfflineMode

**[Description]**

Enables the ISP pipeline Offline mode, where the ISP input stream comes from DDR instead of the sensor, and the VI module reads data from DDR.

**[Syntax]** 

```c
Int ASR_ISP_EnableOfflineMode(
    uint32_t pipelineID, 
    uint32_t enable, 
    const ISP_OFFLINE_ATTR_S *pstOfflineAttr
);
```

**[Parameters]**

| Parameter Name  | Description                                   | Input/Output |
|-----------------|-----------------------------------------------|--------------|
| pipelineID      | ID of the ISP pipeline                        | Input        |
| enable          | Enable/disable offline mode                   | Input        |
| pstOfflineAttr  | Pointer to the Offline mode attribute struct | Input        |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_comm.h`
- Library file: `libisp.so`

**[Note]**

- When offline mode is enabled, sensor and motor callbacks do not need to be registered; even if registered, the ISP will not use them.
- In special photo-taking scenarios, the offline pipeline processing does not require using this function.


#### ASR_ISP_SetPubAttr

**[Description]** 

Sets the public attributes of a channel on the ISP pipeline.

**[Syntax]** 

```c
int ASR_ISP_SetPubAttr(
    uint32_t pipelineID, 
    uint32_t channelID, 
    const ISP_PUB_ATTR_S *pstPubAttr
);
```

**[Parameters]**

| Parameter Name | Description                                | Input/Output |
|----------------|--------------------------------------------|--------------|
| pipelineID     | ISP pipeline ID                            | Input        |
| channelID      | ID of the ISP pipeline channel             | Input        |
| pstPubAttr     | Pointer to the ISP pipeline public attribute structure | Input |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_comm.h`
- Library file: `libisp.so`


#### ASR_ISP_SetTuningParams

**[Description]**

Sets tuning-related attributes on the ISP pipeline. To facilitate algorithm development and debugging, the ISP initialization tuning parameters are determined by two parts:

1. Parameters first obtained from the sensor side (by converting pre-tuned files into code);
2. If this interface informs the ISP that a tuning file currently exists, the ISP will prioritize that file, overriding previously set parameters; otherwise, no operation is performed.

**[Syntax]**

```c
int ASR_ISP_SetTuningParams(
    uint32_t pipelineID, 
    ISP_TUNING_ATTRS_S *pstTuningAttr
);
```

**[Parameters]**

| Parameter Name  | Description                                    | Input/Output |
|-----------------|------------------------------------------------|--------------|
| pipelineID      | ISP pipeline ID                                | Input        |
| pstTuningAttr   | Pointer to the ISP pipeline tuning attribute structure | Input        |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_comm.h`
- Library file: `libisp.so`

**[Note]**

- Before calling this interface, either `ASR_ISP_RegSensorCallBack` or `ASR_ISP_EnableOfflineMode` must be called first.


#### ASR_ISP_SetChHwPipeID

**[Description]** 

Sets the actual hardware pipe ID for the channel in the ISP pipeline.

**[Syntax]**

```c
int ASR_ISP_SetChHwPipeID(
    uint32_t pipelineID, 
    uint32_t channelID, 
    uint32_t hwPipeID
);
```

**[Parameters]**

| Parameter Name | Description                   | Input/Output |
|----------------|-------------------------------|--------------|
| pipelineID     | ISP pipeline ID               | Input        |
| channelID      | ID of the ISP pipeline channel | Input        |
| hwPipeID       | Actual hardware pipe ID       | Input        |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_comm.h`
- Library file: `libisp.so`

**[Note]**

- This interface must be called before `ASR_ISP_Init`; otherwise, the ISP channel will start running with the default channel ID (hardware pipe 0).
- The input parameter `pipelineID` corresponds to the ISP firmware; `channelID` corresponds to the working mode (preview or capture); `hwPipeID` corresponds to the ISP hardware pipeline ID. For special photo-taking scenarios, a single firmware can manage two hardware pipelines working in different modes, ensuring consistent image quality across both hardware pipelines.

#### ASR_ISP_Init

**[Description]** 

Initializes the ISP pipeline with previously set parameters. After initialization, all parameters for the first frame are ready. In principle, it is not recommended to set initialization parameters after this interface call, as changes will not take effect in the first frame but only from the second frame onward.

**[Syntax]**

```c
int ASR_ISP_Init(
    uint32_t pipelineID
);
```

**[Parameters]**

| Parameter Name | Description          | Input/Output |
|----------------|----------------------|--------------|
| pipelineID     | ISP pipeline ID      | Input        |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_comm.h`
- Library file: `libisp.so`


#### ASR_ISP_DeInit

**[Description]**

Deinitializes the ISP pipeline, clearing all effect parameters set on the pipeline. Some registered callbacks will not be cleared.

**[Syntax]**

```c
int ASR_ISP_DeInit(
    uint32_t pipelineID
);
```

**[Parameters]**

| Parameter Name | Description          | Input/Output |
|----------------|----------------------|--------------|
| pipelineID     | ISP pipeline ID      | Input        |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_comm.h`
- Library file: `libisp.so`

#### ASR_ISP_EnablePDAF

**[Description]** 

Enables or disables the PDAF function of the ISP pipeline.

**[Syntax]**

```c
int ASR_ISP_EnablePDAF(
    uint32_t pipelineID, 
    uint32_t enable
);
```

**[Parameters]** 

| Parameter Name | Description                       | Input/Output |
|----------------|---------------------------------|--------------|
| pipelineID     | ISP pipeline ID                  | Input        |
| enable         | Flag to enable or disable PDAF  | Input        |

**[Return Value]** 

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]** 

- Header files: `spm_cam_isp.h`, `spm_isp_comm.h`
- Library file: `libisp.so`

**[Note]**

- Must be called after `ASR_ISP_Init`.


#### ASR_ISP_SetFps

**[Description]**

Sets the frame rate of the ISP pipeline. The ISP supports dynamic frame rates. To run at a specific fixed frame rate, set both `fminFps` and `fmaxFps` to the same value. This interface can be called either before `ASR_ISP_Init` or while the ISP is running; in the former case, the first frame will run at the set frame rate.

**[Syntax]** 

```c
int ASR_ISP_SetFps(
   uint32_t pipelineID, 
   float fminFps, 
   float fmaxFps
);
```
**[Parameters]**

| Parameter Name | Description      | Input/Output |
|----------------|------------------|--------------|
| pipelineID     | ISP pipeline ID  | Input        |
| fminFps        | Minimum frame rate | Input        |
| fmaxFps        | Maximum frame rate | Input        |

**[Return Value]**

| Parameter | Description         |
|-----------|---------------------|
| 0         | Success             |
|  Non-0  | Failure, error code |

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_comm.h`
- Library file: `libisp.so`

#### ASR_ISP_SetFrameinfoCallback

**[Description]** 

Sets the callback function for notifying the user to obtain frameinfo of the ISP pipeline. The ISP sends the frameinfo to the user via this callback each time it updates the frameinfo of a frame.

**[Syntax]** 

```c
int ASR_ISP_SetFrameinfoCallback(
    uint32_t pipelineID, 
    GetFrameInfoCallBack callback
);

```
**[Parameters]**

| Parameter Name | Description                      | Input/Output |
|----------------|---------------------------------|--------------|
| pipelineID     | ID of the ISP pipeline           | Input        |
| callback       | User callback function to obtain frameinfo | Input        |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_comm.h`
- Library file: `libisp.so`

#### ASR_ISP_QueueFrameinfoBuffer

**[Description]** 

Function to enqueue the frameinfo buffer of the ISP pipeline. The ISP internally uses the frameinfo buffer as a queue, operating in a first-in-first-out (FIFO) manner.

**[Syntax]** 

```c
int ASR_ISP_QueueFrameinfoBuffer(
    uint32_t pipelineID, 
    IMAGE_BUFFER_S *pFrameInfoBuf
);
```
**[Parameters]**

| Parameter Name  | Description             | Input/Output |
|-----------------|-------------------------|--------------|
| pipelineID      | ID of the ISP pipeline  | Input        |
| pFrameInfoBuf   | FrameInfo buffer        | Input        |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header files: `spm_cam_isp.h`, `cam_module_interface.h`
- Library file: `libisp.so`

**[Note]**

- The input parameter pointer `pFrameInfoBuf` is used directly by the SDK. Therefore, the application layer must ensure that the memory pointed to by this pointer is not released until the registered callback function is called.


#### ASR_ISP_FlushFrameinfoBuffer

**[Description]** 

Clears the frameinfo buffer queue on the ISP pipeline. Buffers that are cleared are returned via the registered callback.

**[Syntax]** 

```c
int ASR_ISP_FlushFrameinfoBuffer(
    uint32_t pipelineID
);
```
**[Parameters]**

| Parameter Name | Description          | Input/Output |
|----------------|----------------------|--------------|
| pipelineID     | ISP pipeline ID      | Input        |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_comm.h`
- Library file: `libisp.so`

**[Note]**

- Do not enqueue buffers that have been flushed, as this may cause deadlock issues.


#### ASR_ISP_Streamon

**[Description]** 

Runs the ISP pipeline. The sensor's streamon must be called after this function.

**[Syntax]** 

```c
int ASR_ISP_Streamon(
    uint32_t pipelineID
);
```

**[Parameters]**

| Parameter Name | Description          | Input/Output |
|----------------|----------------------|--------------|
| pipelineID     | ISP pipeline ID      | Input        |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_comm.h`
- Library file: `libisp.so`

**[Note]**

- Must be called after `ASR_ISP_Init`.
- Do not start the sensor before this call.

#### ASR_ISP_Streamoff

**[Description]**

Stop the ISP pipeline operation.

**[Syntax]**

```c
int ASR_ISP_Streamoff(
    uint32_t pipelineID
);
```
**[Parameters]**

| Parameter Name | Description          | Input/Output |
|----------------|----------------------|--------------|
| pipelineID     | ISP pipeline ID      | Input        |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header files: `spm_cam_isp.h`, `spm_isp_comm.h`
- Library file: `libisp.so`

**[Note]**

- Must be called after `ASR_ISP_Streamon`.

#### ASR_ISP_TriggerRawCapture

**[Description]** 

Triggers the ISP pipeline to perform a special capture process for specific photo scenarios. If a capture is already in progress, this call will fail.

**[Syntax]** 

```c
int ASR_ISP_TriggerRawCapture(
    uint32_t pipelineID, 
    IMAGE_BUFFER_S *pFrameInfoBuf, 
    uint32_t hdrCapture
);
```

**[Parameters]**

| Parameter Name   | Description                                                                                                                        | Input/Output |
|------------------|------------------------------------------------------------------------------------------------------------------------------------|--------------|
| pipelineID       | ID of the ISP pipeline                                                                                                             | Input        |
| pFrameInfoBuf    | The frameinfo buffer corresponding to the current RAW image. This frameinfo should be the same as the preview frame’s frameinfo during capture; otherwise, the image quality of the capture may differ from the preview. | Input        |
| hdrCapture       | Flag indicating whether this is an HDR capture. Currently, HDR capture in this system only supports HDR synthesis in the RAW domain. | Input        |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header files: `spm_cam_isp.h`, `cam_module_interface.h`
- Library file: `libisp.so`

**[Note]**
- This function is valid only when the ISP pipeline is running.

#### ASR_ISP_ReInitPreviewChannel

**[Description]** 

Reinitialize the preview channel of the ISP pipeline. In some scenarios, the preview of a pipeline may be stopped to reuse the hardware pipe. When restoring the preview, this function needs to be called to ensure the image quality before and after the stream interruption remains consistent.

**[Syntax]** 

```c
int ASR_ISP_ReInitPreviewChannel(
    uint32_t pipelineID, 
    IMAGE_BUFFER_S *pFrameInfoBuf
);
```
**[Parameters]**

| Parameter Name   | Description                          | Input/Output |
|------------------|------------------------------------|--------------|
| pipelineID       | ID of the ISP pipeline              | Input        |
| pFrameInfoBuf    | The frameinfo buffer for the scene to be restored | Input        |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header files: `spm_cam_isp.h`, `cam_module_interface.h`
- Library file: `libisp.so`

#### ASR_ISP_NotifyOnceHDRRawCapture

**[Description]** 

Notify the ISP pipeline to perform an HDR Raw capture process. The ISP returns the valid Raw frame ID of the first HDR frame. Currently, the HDR Raw frame order is normal exposure, long exposure, and short exposure.

**[Syntax]** 

```c
int ASR_ISP_NotifyOnceHDRRawCapture(
    uint32_t pipelineID, 
    uint32_t ZSLCapture, 
    int32_t *startFrameID
);
```

**[Parameters]**

| Parameter Name  | Description                                   | Input/Output |
|-----------------|-----------------------------------------------|--------------|
| pipelineID      | ID of the ISP pipeline                         | Input        |
| ZSLCapture      | Flag indicating whether this is a ZSL HDR capture | Input        |
| startFrameID    | Stores the frame ID of the first valid HDR Raw frame | Output       |

**[Return Value]**

| Parameter | Description                |
|-----------|----------------------------|
| 0         | Success                    |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header file: `spm_cam_isp.h`
- Library file: `libisp.so`

**[Note]**

- This interface can only be called while the ISP is running.

#### ASR_ISP_UpdateNoneZslStreamAeParams

**[Description]** 

Updates the sensor AE-related parameters on the ISP pipeline. This interface is only useful when restarting the pipeline after a stream interruption, to keep the sensor state consistent before and after the interruption.

**[Syntax]** 

```c
int ASR_ISP_UpdateNoneZslStreamAeParams(
    uint32_t pipelineID, 
    IMAGE_BUFFER_S *pFrameInfoBuf, 
    int backPreview, 
    int updateSnsReg
);
```

**[Parameters]**

| Parameter Name  | Description                                                                                             | Input/Output |
|-----------------|---------------------------------------------------------------------------------------------------------|--------------|
| pipelineID      | ID of the ISP pipeline                                                                                   | Input        |
| pFrameInfoBuf   | Frameinfo corresponding to the state that needs to be updated                                            | Input        |
| backPreview     | Flag indicating whether to switch preview                                                               | Input        |
| updateSnsReg    | Flag indicating whether the sensor’s AE registers need to be updated. This should be enabled after the sensor is reconfigured; otherwise, it is not necessary. | Input        |

**[Return Value]**

| Return Value | Description                       |
|--------------|---------------------------------|
| 0            | Success                         |
|  Non-0     | Failure (error code value)    |

**[Requirements]**

- Header files: `spm_cam_isp.h`, `cam_module_interface.h`
- Library file: `libisp.so`

### Effect Setting API

This section describes APIs designed to help users configure settings related to image effects. In this software, image effect settings are implemented via commands. These commands are exposed through a unified API interface:

- **ASR_ISP_SetEffectParams:** Interface to set effect parameters related to the ISP pipeline.

The supported effect commands by the ISP are as follows:

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

**[Description]** 

Interface to set ISP image effect parameters.

**[Syntax]** 

```c
int ASR_ISP_SetEffectParams(
    uint32_t pipelineID, 
    uint32_t effectCmd, 
    void *pData, 
    int dataSize
);
```

**[Parameters]**

| Parameter Name | Description                        | Input/Output |
|----------------|----------------------------------|--------------|
| pipelineID     | ISP pipeline ID                  | Input        |
| effectCmd      | Effect parameter command         | Input        |
| pData          | Pointer to the effect parameter structure | Input        |
| dataSize       | Size of the effect parameter structure | Input        |

**[Return Value]**

| Parameter | Description                 |
|-----------|-----------------------------|
| 0         | Success                     |
|  Non-0  | Failure (error code value)|

**[Requirements]**

- Header file: `spm_cam_isp.h`
- Library file: `libisp.so`

#### Effect Command Descriptions

1. **ISP_EFFECT_CMD_S_AE_MODE**  
   Command to set image exposure mode. The corresponding parameter type is the structure `asrISP_AE_INFO_S`.

2. **ISP_EFFECT_CMD_S_AWB_MODE**  
   Command to set image white balance mode. The corresponding parameter type is the structure `asrISP_AWB_INFO_S`.

3. **ISP_EFFECT_CMD_S_AF_MODE**  
   Command to set image autofocus mode. The corresponding parameter type is the structure `asrISP_AF_INFO_S`.

4. **ISP_EFFECT_CMD_S_TRIGGER_AF**  
   Command to trigger autofocus. The corresponding parameter type is `int32_t`. A value of `1` triggers autofocus, and `0` cancels it.

5. **ISP_EFFECT_CMD_S_ANTIFLICKER_MODE**  
   Command to set the image anti-flicker mode. The corresponding parameter type is `int32_t`, and the value is defined by the enum `asrISP_ANTIFLICKER_MODE_E`.

6. **ISP_EFFECT_CMD_S_LSC_MODE**  
   Command to set the lens shading correction mode. The corresponding parameter type is the structure `asrISP_LSC_INFO_S`.

7. **ISP_EFFECT_CMD_S_CCM_MODE**  
   Command to set the color correction matrix (CCM) mode. The corresponding parameter type is the structure `asrISP_CCM_INFO_S`.

8. **ISP_EFFECT_CMD_S_AECOMPENSATION**  
   Command to set the exposure compensation. The corresponding parameter type is `int32_t`, with a currently supported range of [-6, 6].

9. **ISP_EFFECT_CMD_S_METERING_MODE**  
   Command to set the metering mode. The corresponding parameter type is the structure `asrISP_METERING_INFO_S`.

10. **ISP_EFFECT_CMD_S_ZOOM_RATIO_IN_Q8**  
    Command to set the zoom ratio. The corresponding parameter type is `int32_t`, with the unit in Q8 format.

11. **ISP_EFFECT_CMD_S_SENSITIVITY_MODE**  
    Command to set the ISO sensitivity mode. The corresponding parameter type is the structure `asrISP_SENSITIVITY_INFO_S`, which supports both auto and manual modes.

12. **ISP_EFFECT_CMD_S_SENSOR_EXPOSURE_MODE**  
   - Command to set the sensor exposure mode. The difference between this and `ISP_EFFECT_CMD_S_AE_MODE` is that this command only sets exposure information, while the other also sets gain information.  
   - The corresponding parameter type is the structure `asrISP_SENSOR_EXPOSURE_INFO_S`, supporting both auto and manual modes.
   - The relationship among these three commands (`ISP_EFFECT_CMD_S_SENSOR_EXPOSURE_MODE`, `ISP_EFFECT_CMD_S_AE_MODE`, and `ISP_EFFECT_CMD_S_SENSITIVITY_MODE`) is shown in the table below (not included here).

| AE MODE                | SENSITIVITY AUTO | SENSITIVITY MANUAL |
| ------------------------------- | ------------------------- | --------------------------- |
| SENSOR EXPOSURE AUTO   | Auto             | Auto               |
| SENSOR EXPOSURE MANUAL | Auto             | Manual             |

- As shown in the table above, `SENSITIVITY_MODE_MANUAL` means that under Auto Exposure, the gain is fixed, and exposure is automatically calculated by the algorithm based on the scene; while `SENSOR_EXPOSURE_MODE_MANUAL` means that under Auto Exposure, the exposure time is fixed, and gain is automatically calculated by the algorithm.

13. **ISP_EFFECT_CMD_S_TRIGGER_AE_QUICK_RESPONSE**  
    Command to trigger a quick AE convergence once. The corresponding parameter type is `int32_t`, where `1` triggers and `0` cancels. This command only takes effect once. After quick AE convergence, the algorithm clears this state. Typically used in touch scenarios.

14. **ISP_EFFECT_CMD_S_AECOMPENSATION_STEP**  
    Sets the exposure compensation step value in ISP. Parameter type: `float`. After setting, exposure compensation will be adjusted in steps according to this value. The default is 1/3.

15. **ISP_EFFECT_CMD_S_AE_SCENE_MODE**  
    Sets the AE scene mode of the ISP. Parameter type: `int32_t`, with values defined by the enum `asrISP_AE_SCENE_MODE_E`. Supported modes include Normal and Face Detection. In Face Detection mode, AE converges faster than in Normal mode.

16. **ISP_EFFECT_CMD_S_FILTER_MODE**  
    Sets the ISP color filter mode. Parameter type: `int32_t`, defined by enum `asrISP_COLOR_FILTER_MODE_E`. Currently supports Normal and Black & White modes. Default is Normal.

17. **ISP_EFFECT_CMD_S_YUV_RANGE**  
    Sets the YUV range of the ISP. Parameter type: `int32_t`, defined by enum `asrISP_YUV_RANGE_E`. Supported modes:
    - Full: y:0~255, uv:0~255
    - Compressed: y:16~235, uv:16~240

18. **ISP_EFFECT_CMD_S_SOLID_COLOR_MODE**  
    Sets the ISP solid color mode. Parameter type: `int32_t`, defined by enum `asrISP_SOLID_COLOR_MODE_E`. Currently only supports solid black mode. This mode is disabled by default.

19. **ISP_EFFECT_CMD_G_AF_MODE**  
    Gets the current AF mode of the ISP. Parameter type: struct `asrISP_AF_INFO_S`.

20. **ISP_EFFECT_CMD_G_ANTIFLICKER_MODE**  
    Gets the current anti-flicker mode of the ISP. Parameter type: `int32_t`, defined by enum `asrISP_ANTIFLICKER_MODE_E`.

21. **ISP_EFFECT_CMD_G_METERING_MODE**  
    Gets the current metering mode of the ISP. Parameter type: struct `asrISP_METERING_INFO_S`.

22. **ISP_EFFECT_CMD_G_AF_MOTOR_RANGE**  
    Gets the range of movement of the autofocus motor (min and max values). Parameter type: struct `asrISP_RANGE_S`.

23. **ISP_EFFECT_CMD_G_AE_MODE**  
    Gets the current AE mode of the ISP. Parameter type: struct `ISP_AE_INFO_S`.

24. **ISP_EFFECT_CMD_G_AWB_MODE**  
    Gets the current AWB mode of the ISP. Parameter type: struct `ISP_AWB_INFO_S`.

#### Tuning-Related API

This section describes APIs intended to help tuning engineers adjust more specific and detailed parameters. The tuning tool released by ASR also uses the APIs described in this section.

The tuning-related APIs are as follows:

- **ASR_ISP_SetFwPara:**  
  Set firmware parameters on the ISP pipeline.

- **ASR_ISP_GetFwPara:**  
  Get firmware parameters from the ISP pipeline.

- **ASR_ISP_SetRegister:**  
  Set the value of a specified register.

- **ASR_ISP_GetRegister:**  
  Get the value of a specified register.

- **ASR_ISP_LoadSettingFile:**  
  Load a setting file into the firmware on the ISP pipeline.

- **ASR_ISP_SaveSettingFile:**  
  Save the current firmware setting file from the ISP pipeline.

#### ASR_ISP_SetFwPara

**[Description]**

Sets a parameter in the ISP firmware.

**[Syntax]**

```c
int ASR_ISP_SetFwPara(
    uint32_t pipelineID,
    const char *parameter,
    const char *name,
    uint32_t row,
    uint32_t column,
    int value
);
```

**[Parameters]**

| Parameter Name | Description               | Input/Output |
| -------------- | ------------------------- | ------------ |
| pipelineID     | ID of the ISP pipeline    | Input        |
| paramter       | Name of the ISP submodule | Input        |
| name           | Parameter name            | Input        |
| row            | Row number                | Input        |
| column         | Column number             | Input        |
| value          | Value                     | Input        |

**[Return Value]**

| Parameter Name | Description               |
| -------------- | ------------------------- |
| 0              | Success                   |
|  Non-0       | Failure, the (error code value) |

**[Requirements]**

- Header file: `spm_cam_isp.h`
- Library file: `libisp.so`

**[Note]** 
None

#### ASR_ISP_GetFWPara

**[Description]**

Gets ISP Firmware parameters.

**[Syntax]**

```c
int ASR_ISP_GetFWPara(
    uint32_t pipelineID,
    const char *paramter,
    const char *name,
    uint32_t row,
    uint32_t column,
    int *pValue
);
```
**[Parameters]**

| Parameter Name | Description               | Input/Output |
| -------------- | ------------------------- | ------------ |
| pipelineID     | ID of the ISP pipeline    | Input        |
| paramter       | Name of the ISP submodule | Input        |
| name           | Parameter name            | Input        |
| row            | Row number                | Input        |
| column         | Column number             | Input        |
| value          | Value                     | Output       |

**[Return Value]**

| Parameter Name | Description                        |
| -------------- | -------------------------------- |
| -1             | Input check error                 |
| 0              | Specified parameter not found    |
| >0             | Success, returns the number of values obtained |

**[Requirements]**

- Header file: `spm_cam_isp.h`
- Library file: `libisp.so`

**[Note]** 
None

#### ASR_ISP_SetRegister

#### ASR_ISP_SetRegister

**[Description]**

Sets the specified register value.

**[Syntax]**

```c
int ASR_ISP_SetRegister(
    uint32_t addr,
    uint32_t value,
    uint32_t mask
);
```

**[Parameters]**

| Parameter Name | Description      | Input/Output |
| -------------- | ---------------- | ------------ |
| addr           | Register address | Input        |
| value          | Register value   | Input        |
| mask           | Register mask    | Input        |

**[Return Value]**

| Parameter Name | Description                  |
| -------------- | ---------------------------- |
| 0              | Success                      |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `spm_cam_isp.h`
- Library file: `libisp.so`

**[Note]** 
None

#### ASR_ISP_GetRegister

**[Description]**

Gets the value of the specified register.

**[Syntax]**

```c
int ASR_ISP_GetRegister(
    uint32_t addr,
    int *pValue
);
```

**[Parameters]**

| Parameter Name | Description       | Input/Output |
| -------------- | ----------------- | ------------ |
| addr           | Register address  | Input        |
| pValue         | Register value    | Output       |

**[Return Value]**

| Parameter Name | Description                  |
| -------------- | ---------------------------- |
| 0              | Success                      |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `spm_cam_isp.h`
- Library file: `libisp.so`

**[Note]** 
None

#### ASR_ISP_LoadSettingFile

**[Description]**

Loads the ISP Firmware setting file.

**[Syntax]**

```c
int ASR_ISP_LoadSettingFile(
    uint32_t pipelineID,
    const char *pFileName
);
```

**[Parameters]**

| Parameter Name | Description                        | Input/Output |
| -------------- | -------------------------------- | ------------ |
| pipelineID     | ID of the ISP pipeline            | Input        |
| pFileName      | Parameter file name, including absolute path | Input |

**[Return Value]**

| Parameter Name | Description                  |
| -------------- | ---------------------------- |
| 0              | Success                      |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `spm_cam_isp.h`
- Library file: `libisp.so`

**[Note]** 

None

#### ASR_ISP_SaveSettingFile

**[Description]**

Saves the ISP Firmware setting file.

**[Syntax]**

```c
int ASR_ISP_SaveSettingFile(
    uint32_t pipelineID,
    const char *pFileName
);
```
**[Parameters]**

| Parameter Name | Description                        | Input/Output |
| -------------- | -------------------------------- | ------------ |
| pipelineID     | ID of the ISP pipeline            | Input        |
| pFileName      | Parameter file name, including absolute path | Input |

**[Return Value]**

| Parameter Name | Description                  |
| -------------- | ---------------------------- |
| 0              | Success                      |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `spm_cam_isp.h`
- Library file: `libisp.so`

**[Note]** 

None

### Main API Usage Flow

This section mainly describes the usage flow of the system control API, which is divided into the following stages: construction, registration, setting, initialization, streamon, streamoff, de-initialization, and destruction.

1. **Construct the context environment of the ISP pipeline.**

2. **Registration**: Register the sensor callback function, autofocus motor callback function, and frameinfo callback function to the ISP pipeline.

3. **Setting**: Configure the ISP pipeline's common image properties, tuning-related properties, the actual hardware pipe ID used by the channel, frame rate, whether to enable offline mode, and enqueue the frameinfo buffer.

4. **Initialization**: Initialize the ISP pipeline based on the information set in the previous steps.

5. **Streamon**: Start the ISP pipeline and enter the working state.

6. **Streamoff**: Stop the ISP pipeline operation and exit the working state.

7. **De-initialization**: Clear the set parameters and other information (except the previously registered callbacks).

8. **Destruction**: Destroy the context environment of the ISP pipeline.

The above process is pipeline-oriented. The ISP software can support two pipelines working simultaneously. If another pipeline needs to be started, the process is similar, but attention should be paid to the actual hardware pipe ID in use, as sometimes resources are shared.

### Data Types

#### Error Codes

| Error Code | Macro Definition           | Description            |
| ---------- | -------------------------- | ---------------------- |
| -22        | ASR_ERR_ISP_ILLEGAL_PARAM  | Illegal parameter      |
| -17        | ASR_ERR_ISP_EXIST          | Resource already exists|
| -19        | ASR_ERR_ISP_NOTEXIST       | Resource does not exist|
| -22        | ASR_ERR_ISP_NULL_PTR       | Null pointer           |
| -1         | ASR_ERR_ISP_NOT_SUPPORT    | Operation not supported|
| -1         | ASR_ERR_ISP_NOT_PERM       | Operation not permitted|
| -12        | ASR_ERR_ISP_NOMEM          | Insufficient memory    |
| -14        | ASR_ERR_ISP_BADADDR        | Invalid address        |
| -16        | ASR_ERR_ISP_BUSY           | System busy            |

## VI API

This section introduces the usage of the ASR MARS11-ISP VI module APIs. The described APIs are part of the VI SDK designed for applications, with detailed explanations of related parameter data structures and return values.

### APIs

The VI module provides the following APIs:

- **ASR_VI_SetDevAttr**: Sets the attributes of the VI Device.  
- **ASR_VI_GetDevAttr**: Gets the attributes of the VI Device.  
- **ASR_VI_EnableDev**: Enables the VI Device.  
- **ASR_VI_DisableDev**: Disables the VI Device.  
- **ASR_VI_SetChnAttr**: Sets the attributes of the VI Channel.  
- **ASR_VI_GetChnAttr**: Gets the attributes of the VI Channel.  
- **ASR_VI_SetCallback**: Sets the upper-layer Callback function to the VI Device.  
- **ASR_VI_EnableChn**: Enables the VI Channel.  
- **ASR_VI_DisableChn**: Disables the VI Channel.  
- **ASR_VI_Init**: Initializes VI module resources.  
- **ASR_VI_Deinit**: Releases VI module resources.  
- **ASR_VI_SetBayerReadAttr**: Sets the attributes for offline Bayer raw reading.  
- **ASR_VI_GetBayerReadAttr**: Gets the attributes for offline Bayer raw reading.  
- **ASR_VI_EnableBayerRead**: Enables offline Bayer raw reading.  
- **ASR_VI_DisableBayerRead**: Disables offline Bayer raw reading.  
- **ASR_VI_EnableBayerDump**: Enables Bayer Dump.  
- **ASR_VI_DisableBayerDump**: Disables Bayer Dump.  
- **ASR_VI_ChnQueueBuffer**: Queues a buffer into the channel.

#### ASR_VI_SetDevAttr

**[Description]**

Sets the VI device attributes. Basic device attributes have default chip configurations to meet the requirements of most sensor integrations.

**[Syntax]**

```c
int32_t ASR_VI_SetDevAttr(
    uint32_t nDev,
    VI_DEV_ATTR_S *pstDevAttr
);
```

**[Parameters]**

| Parameter Name | Description                                               | Input/Output |
| -------------- | --------------------------------------------------------- | ------------ |
| nDev           | VI device number, valid range: [0, VIU_MAX_DEV_NUM)       | Input        |
| pstDevAttr     | Pointer to VI device attributes. Static attributes.       | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

**[Note]**

- Changing the binding relationship between device and channel is not supported.  
- Before calling, ensure the VI device is disabled. If the VI device is enabled, you can disable it using `ASR_CAM_VI_DisableDev`.

#### ASR_VI_GetDevAttr

**[Description]**

Gets the VI device attributes.

**[Syntax]**

```c
int32_t ASR_VI_GetDevAttr(
    uint32_t nDev,
    VI_DEV_ATTR_S *pstDevAttr
);
```

**[Parameters]**

| Parameter Name | Description                                 | Input/Output |
| -------------- | ------------------------------------------- | ------------ |
| nDev           | VI device number. Valid range: [0, VIU_MAX_DEV_NUM) | Input        |
| pstDevAttr     | Pointer to VI device attributes.            | Output       |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

#### ASR_VI_EnableDev

**[Description]**

Enables the VI device.

**[Syntax]**

```c
int32_t ASR_VI_EnableDev(
    uint32_t nDev
);
```

**[Parameters]**

| Parameter Name | Description                                       | Input/Output |
| -------------- | ------------------------------------------------ | ------------ |
| nDev           | VI device number. Valid range: [0, VIU_MAX_DEV_NUM) | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

**[Note]**

- Device attributes must be set before enabling; otherwise, it returns failure.  
- Can be enabled repeatedly without returning failure.

#### ASR_VI_DisableDev

**[Description]**

Disables the VI device.

**[Syntax]**

```c
int32_t ASR_VI_DisableDev(
    uint32_t nDev
);
```

**[Parameters]**

| Parameter Name | Description                                       | Input/Output |
| -------------- | ------------------------------------------------ | ------------ |
| nDev           | VI device number. Valid range: [0, VIU_MAX_DEV_NUM) | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

**[Note]**

- All VI channels bound to the VI device must be disabled before disabling the VI device.  
- Can be disabled repeatedly without returning failure.

#### ASR_VI_FlushDev

**[Description]**

Flushes the buffers that have been queued into the VI device.

**[Syntax]**

```c
int32_t ASR_VI_FlushDev(
    uint32_t nDev
);
```

**[Parameters]**

| Parameter Name | Description                                       | Input/Output |
| -------------- | ------------------------------------------------ | ------------ |
| nDev           | VI device number. Valid range: [0, VIU_MAX_DEV_NUM) | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

#### ASR_VI_SetChnAttr

**[Description]**

Sets the VI channel attributes.

**[Syntax]**

```c
int32_t ASR_VI_SetChnAttr(
    uint32_t nChn,
    VI_CHN_ATTR_S *pstAttr
);
```

**[Parameters]**

| Parameter Name | Description                                      | Input/Output |
| -------------- | ------------------------------------------------ | ------------ |
| nChn           | VI channel number. Valid range: [0, VIU_MAX_CHN_NUM) | Input        |
| pstAttr        | Pointer to VI channel attributes. Static attributes. | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

**[Note]**

- Device attributes must be set before setting channel attributes; otherwise, it will return failure.  
- The channel must be in the Disabled state to set channel attributes.

#### ASR_VI_GetChnAttr

**[Description]**

Gets the VI channel attributes.

**[Syntax]**

```c
int32_t ASR_VI_GetChnAttr(
    uint32_t nChn,
    VI_CHN_ATTR_S *pstAttr
);
```
**[Parameters]**

| Parameter Name | Description                                     | Input/Output |
| -------------- | ----------------------------------------------- | ------------ |
| nChn           | VI channel number. Valid range: [0, VIU_MAX_CHN_NUM) | Input        |
| pstAttr        | Pointer to VI channel attributes. Static attributes. | Output       |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

**[Note]**

- Channel attributes must be set before getting them; otherwise, it will return failure.

#### ASR_VI_EnableChn

**[Description]**

Enables the VI channel.

**[Syntax]**

```c
int32_t ASR_VI_EnableChn(
    uint32_t nChn
);
```

**[Parameters]**

| Parameter Name | Description                                     | Input/Output |
| -------------- | ----------------------------------------------- | ------------ |
| nChn           | VI channel number. Valid range: [0, VIU_MAX_CHN_NUM) | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

**[Note]**

- Channel attributes must be set first, and the VI device bound to the channel must be enabled.  
- VI channel can be enabled repeatedly without returning failure.

#### ASR_VI_DisableChn

**[Description]**

Disables the VI channel.

**[Syntax]**

```c
int32_t ASR_VI_DisableChn(
    uint32_t nChn
);
```

**[Parameters]**

| Parameter Name | Description                                     | Input/Output |
| -------------- | ----------------------------------------------- | ------------ |
| nChn           | VI channel number. Valid range: [0, VIU_MAX_CHN_NUM) | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

**[Note]**

- VI channel can be disabled repeatedly without returning failure.

#### ASR_VI_SetCallback

**[Description]**

Sets the callback function used for frame buffer rotation.

**[Syntax]**

```c
int32_t ASR_VI_SetCallback(
    uint32_t nChn,
    int32_t (*callback)(uint32_t nChn, VI_IMAGE_BUFFER_S *vi_buffer)
);
```

**[Parameters]**

| Parameter Name | Description                                     | Input/Output |
| -------------- | ----------------------------------------------- | ------------ |
| nChn           | VI channel number. Valid range: [0, VIU_MAX_CHN_NUM) | Input        |
| callback       | Pointer to the callback function                 | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

**[Note]**

- Must be called before `ASR_VI_EnableChn`.

#### ASR_VI_SetBayerReadAttr

**[Description]**

Sets the attributes for offline raw Bayer data reading.

**[Syntax]**

```c
int32_t ASR_VI_SetBayerReadAttr(
    uint32_t nDev,
    const VI_BAYER_READ_ATTR_S *pstBayerReadAttr
);
```

**[Parameters]**

| Parameter Name     | Description                                         | Input/Output |
| ------------------ | --------------------------------------------------- | ------------ |
| nDev               | VI device number. Valid range: [0, VIU_MAX_DEV_NUM) | Input        |
| pstBayerReadAttr   | Pointer to the Bayer read attribute structure       | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

#### ASR_VI_GetBayerReadAttr

**[Description]**

Gets the attributes for offline raw Bayer data reading.

**[Syntax]**

```c
int32_t ASR_VI_GetBayerReadAttr(
    uint32_t nDev,
    VI_BAYER_READ_ATTR_S *pstBayerReadAttr
);
```

**[Parameters]**

| Parameter Name     | Description                                         | Input/Output |
| ------------------ | --------------------------------------------------- | ------------ |
| nDev               | VI device number. Valid range: [0, VIU_MAX_DEV_NUM) | Input        |
| pstBayerReadAttr   | Pointer to the Bayer read attribute structure       | Output       |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

#### ASR_VI_EnableBayerRead

**[Description]**

Enables offline processing.

**[Syntax]**

```c
int32_t ASR_VI_EnableBayerRead(
    uint32_t nDev
);
```

**[Parameters]**

| Parameter Name | Description                                     | Input/Output |
| -------------- | ----------------------------------------------- | ------------ |
| nDev           | VI device number. Valid range: [0, VIU_MAX_DEV_NUM) | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

#### ASR_VI_DisableBayerRead

**[Description]**

Disables offline processing.

**[Syntax]**

```c
int32_t ASR_VI_DisableBayerRead(
    uint32_t nDev
);
```

**[Parameters]**

| Parameter Name | Description                                     | Input/Output |
| -------------- | ----------------------------------------------- | ------------ |
| nDev           | VI device number. Valid range: [0, VIU_MAX_DEV_NUM) | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

#### ASR_VI_EnableBayerDump

**[Description]**

Enables RAW DATA capture.

**[Syntax]**

```c
int32_t ASR_VI_EnableBayerDump(
    uint32_t nDev
);
```

**[Parameters]**

| Parameter Name | Description                                     | Input/Output |
| -------------- | ----------------------------------------------- | ------------ |
| nDev           | VI device number. Valid range: [0, VIU_MAX_DEV_NUM) | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

**[Note]**

- The width and height of the saved RAW data are consistent with the width and height set for the device.  
- The starting channel number for RAW Dump channels can be obtained via `VIU_GET_RAW_CHN`.

#### ASR_VI_DisableBayerDump

**[Description]**

Disables RAW DATA capture.

**[Syntax]**

```c
int32_t ASR_VI_DisableBayerDump(
    uint32_t nDev
);
```

**[Parameters]**

| Parameter Name | Description                                     | Input/Output |
| -------------- | ----------------------------------------------- | ------------ |
| nDev           | VI device number. Valid range: [0, VIU_MAX_DEV_NUM) | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

#### ASR_VI_ChnQueueBuffer

**[Description]**

Queues a buffer to the channel.

**[Syntax]**

```c
int32_t ASR_VI_ChnQueueBuffer(
    uint32_t nChn,
    IMAGE_BUFFER_S *camBuf
);
```

**[Parameters]**

| Parameter Name | Description                                    | Input/Output |
| -------------- | ---------------------------------------------- | ------------ |
| nChn           | VI channel number. Valid range: [0, VI_CHN_CNT) | Input        |
| camBuf         | Pointer to the buffer                           | Input        |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

**[Note]**

- The input parameter `camBuf` pointer will be used directly by the SDK. Therefore, the application layer must ensure that the memory pointed to by this pointer is not freed until the registered callback function is called.

#### ASR_VI_Init

**[Description]**

Initializes the VI module.

**[Syntax]**

```c
int32_t ASR_VI_Init(void);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
| -------------- | ----------- | ------------ |
| None           |             |              |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

#### ASR_VI_Deinit

**[Description]**

Deinitializes the VI module.

**[Syntax]**

```c
int32_t ASR_VI_Deinit(void);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
| -------------- | ----------- | ------------ |
| None           |             |              |

**[Return Value]**

| Return Value | Description                   |
| ------------ | ----------------------------- |
| 0            | Success                      |
|  Non-0     | Failure (error code value) |

**[Requirements]**

- Header files: `spm_comm_vi.h`, `spm_cam_vi.h`
- Library file: `libvi.so`

### Data Types

#### VI_DEV_ATTR_S

**[Description]**

Defines the attributes of the video input device.

**[Definition]**

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

**[Members]**

| Member Name       | Description                                                                                              |
| ----------------- | -------------------------------------------------------------------------------------------------------- |
| enWorkMode        | CAM_VI_WORK_MODE_ONLINE, CAM_VI_WORK_MODE_RAWDUMP, CAM_VI_WORK_MODE_OFFLINE                              |
| enRawType         | CAM_SENSOR_RAWTYPE_RAW8, CAM_SENSOR_RAWTYPE_RAW10, CAM_SENSOR_RAWTYPE_RAW12, CAM_SENSOR_RAWTYPE_RAW14, CAM_SENSOR_RAWTYPE_INVALID |
| Width             | The width of the image to capture settable by the VI device. The minimum and maximum capture width range: [VIU_DEV_MIN_WIDTH, VIU_DEV_MAX_WIDTH] |
| Height            | The height of the image to capture settable by the VI device. The minimum and maximum capture height range: [VIU_DEV_MIN_HEIGHT, VIU_DEV_MAX_HEIGHT] |
| bindSensorIdx     | Sensor ID bound to the VI device. Range: [0, 2]                                                          |
| bOfflineSlice     | Whether offline mode is for photo capture                                                                |
| mipi_lane_num     | Number of lanes of the sensor MIPI interface                                                             |
| bCapture2Preview  | Whether it is photo capture back to preview                                                              |

#### VI_CHN_ATTR_S

**[Description]**

Defines the attributes of a VI channel.

**[Definition]**

```java
typedef struct asrVI_CHN_ATTR_S { 
    CAM_VI_PIXEL_FORMAT_E enPixFormat; 
    uint32_t width;
    uint32_t height;
} VI_CHN_ATTR_S;
```

**[Members]**

| Member Name  | Description          |
| ------------ | -------------------- |
| enPixFormat  | Channel output format |
| Width        | Output image width    |
| Height       | Output image height   |

#### VI_BAYER_READ_ATTR_S

**[Description]**

Defines the attributes for offline raw data reading.

**[Definition]**

```java
typedef struct asrVI_BAYER_READ_ATTR_S {
    bool bGenTiming;
    int32_t s32FrmRate;
} VI_BAYER_READ_ATTR_S;
```

**[Members]**

| Member Name | Description                                             |
| ----------- | ------------------------------------------------------- |
| bGenTiming  | Whether VI automatically generates fixed frame rate reading timing |
| s32FrmRate  | If bGenTiming is true, indicates the frame rate value  |

#### VI_IMAGE_BUFFER_S

**[Description]**

Defines the structure of the buffer in the VI channel callback function.

**[Definition]**

```java
Typedef struct asrVI_IMAGE_BUFFER_S { 
    IMAGE_BUFFER_S *buffer;
    Bool bValid;
    Bool bCloseDown;
    Uint64_t timestamp; 
    Uint32_t frameId;
} VI_IMAGE_BUFFER_S;
```

**[Members]**

| Member Name | Description                                                                                      |
| ----------- | ------------------------------------------------------------------------------------------------ |
| Buffer      | Pointer to the buffer memory structure                                                           |
| bValid      | Indicates whether the current frame content is valid; if invalid, the upper-level application should discard the current frame |
| bCloseDown  | Dedicated for RAW DATA channel; before disabling the RAW DATA channel, a bCloseDown signal must be received in the buffer callback |
| timeStamp   | Indicates the timestamp of the current frame                                                     |
| frameId     | Indicates the frame number of the current frame, starting from 0                                 |

#### VIU_GET_RAW_CHN

**[Description]**

Defines the channel number for obtaining RAW DATA.

**[Definition]**

```java
#define VIU_GET_RAW_CHN(ViDev, RawChn) do{
        RawChn = VIU_MAX_CHN_NUM + ViDev;
    }while(0)
```

#### VIU_MAX_CHN_NUM

**[Description]**

Defines the maximum number of VI channels.

**[Definition]**

```java
#define VIU_MAX_CHN_NUM (VIU_MAX_PHYCHN_NUM)
```

#### VIU_MAX_PHYCHN_NUM

**[Description]**

Defines the maximum number of VI physical channels.

**[Definition]**

```java
#define VIU_MAX_PHYCHN_NUM 2
```

#### VIU_MAX_RAWCHN_NUM

**[Description]**

Define the maximum number of RAW DATA channels that can be obtained.

**[Syntax]**

```java
#define VIU_MAX_RAWCHN_NUM 2
```

#### VIU_MAX_DEV_NUM

**[Description]**

Define the maximum number of video input devices.

**[Syntax]**

```java
#define VIU_MAX_DEV_NUM 2
```

#### VIU_DEV_MIN_WIDTH

**[Description]**

Define the minimum width of the image captured by the VI device.

**[Syntax]**

```java
#define VIU_DEV_MIN_WIDTH 256
```

#### VIU_DEV_MIN_HEIGHT

**[Description]**

Define the minimum height of the image captured by the VI device.

**[Syntax]**

```java
#define VIU_DEV_MIN_HEIGHT 144
```

#### VIU_DEV_MAX_WIDTH

**[Description]**

The maximum width of the image captured by the VI device.

**[Syntax]**

```java
#define VIU_DEV_MAX_WIDTH 2688
```

#### VIU_DEV_MAX_HEIGHT

**[Description]**

The maximum height of the image captured by the VI device

**[Syntax]**

```java
#define VIU_DEV_MAX_HEIGHT 1944
```

#### VIU_CHN_MIN_WIDTH

**[Description]**

The minimum width supported by the VI physical channel

**[Syntax]**

```java
#define VIU_CHN_MIN_WIDTH VIU_DEV_MIN_WIDTH
```

#### VIU_CHN_MIN_HEIGHT

**[Description]**

The minimum height supported by the VI physical channel.

**[Syntax]**

```java
#define VIU_CHN_MIN_HEIGHT VIU_DEV_MIN_HEIGHT
```

#### VIU_CHN_MAX_WIDTH

**[Description]**

The maximum width supported by the VI physical channel.

**[Syntax]**

```java
#define VIU_CHN_MAX_WIDTH VIU_DEV_MAX_WIDTH
```

#### VIU_CHN_MAX_HEIGHT

**[Description]**

The maximum height supported by the VI physical channel.

**[Syntax]**

```java
#define VIU_CHN_MAX_HEIGHT VIU_DEV_MAX_HEIGHT
```

## CPP API

This section introduces the usage of the ASR MARS11-ISP CPP module API. The described APIs are all application-oriented in the CPP SDK, and the related parameter data structures, error codes, and return values are explained in detail

### API

The CPP module provides the following APIs to users:

- **cam_cpp_create_grp**: Create a CPP module group.
- **cam_cpp_destroy_grp**: Destroy a CPP module group.
- **cam_cpp_start_grp**: Enable the cam cpp module.
- **cam_cpp_stop_grp**: Disable the cam cpp module.
- **cam_cpp_post_buffer**: Users send data to CPP.
- **cam_cpp_set_callback**: Users set a callback function.
- **cam_cpp_get_grp_attr**: Get CPP Group attributes.
- **cam_cpp_set_grp_attr**: Set CPP Group attributes.
- **cam_cpp_get_tuning_param**: Get tuning parameters of the CPP Group.
- **cam_cpp_set_tuning_param**: Set tuning parameters of the CPP Group.
- **cam_cpp_load_settingfile**: Tuning debugging interface, load firmware setting file.
- **cam_cpp_save_settingfile**: Tuning debugging interface, save firmware setting file.
- **cam_cpp_read_fw**: Debugging interface, get parameters of the CPP Group Firmware module.
- **cam_cpp_write_fw**: Tuning debugging interface, set parameters of the CPP Group Firmware module.
- **cam_cpp_read_reg**: Tuning debugging interface, get register values of the CPP Hardware.
- **cam_cpp_write_reg**: Tuning debugging interface, set register values of the CPP Hardware.
- **cam_cpp_dump_frame**: Save the output and corresponding input images of the group to a specified directory.

#### cam_cpp_create_grp

**[Description]**

Create a cam CPP module group.

**[Syntax]**
```c
int32_t cam_cpp_create_grp(uint32_t grpId);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

- Duplicate creation is not supported.

**[Related Operations]**

`cam_cpp_destroy_grp`

#### cam_cpp_destroy_grp

**[Description]**

Destroy a cam CPP module group.

**[Syntax]**
```c
int32_t cam_cpp_destroy_grp(uint32_t grpId);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

- The CPP group must have been created.
- Before calling this interface, `cam_cpp_stop_grp` must be called to disable this group.
- When calling this interface, it will wait for the current task of this group to finish processing before truly destroying it.

**[Related Operations]**

`cam_cpp_create_grp`

#### cam_cpp_start_grp

**[Description]**

Start the cam CPP module.

**[Syntax]**
```c
int32_t cam_cpp_start_grp(uint32_t grpId);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

- Duplicate creation is not supported.
- Repeatedly calling this function to set the same group returns success.

**[Related Operations]**

`cam_cpp_stop_grp`

#### cam_cpp_stop_grp

**[Description]**

Disable the cam CPP module group.

**[Syntax]**
```c
int32_t cam_cpp_stop_grp(uint32_t grpId);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

- Duplicate creation is not supported.
- Repeatedly calling this function to set the same group returns success.
- Buffers sent to the cpp group via the `cam_cpp_post_buffer` interface will be returned during the stop process.

**[Related Operations]**

`cam_cpp_start_grp`

#### cam_cpp_post_buffer

**[Description]**

The user sends a frame of image data to the CPP module.

**[Syntax]**
```c
int32_t cam_cpp_post_buffer(
    uint32_t grpId, 
    const IMAGE_BUFFER_S *inputBuf, 
    const IMAGE_BUFFER_S *outputBuf, 
    int32_t frameId, 
    FRAME_INFO_S *frameInfo
);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |
| inputBuf       | Information of the input image. For details, see `IMAGE_BUFFER_S`. | Input       |
| outputBuf      | Information of the output image. For details, see `IMAGE_BUFFER_S`. | Output      |
| frameInfo      | For details, see `FRAME_INFO_S`. | Input       |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header files: `cam_cpp.h`, `cam_module_interface.h`
- Library file: `libcpp.so`

**[Note]**

- The group must have been created.

**[Related Operations]**

`cam_cpp_ReturnBuffer`

#### cam_cpp_set_callback

**[Description]**

The user sets the callback function.

**[Syntax]**

```c
int32_t cam_cpp_set_callback(
    uint32_t grpId, 
    CppCallback callback
);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |
| callback       | This interface is called when the module has finished processing and the output image is ready. | Input       |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

The group must have been created.

**[Related Operations]**

`cam_cpp_ReturnBuffer`

#### cam_cpp_get_grp_attr

**[Description]**

Get the attributes of the CPP group.

**[Syntax]**

```c
int32_t cam_cpp_get_grp_attr(
    uint32_t grpId,
    CPP_GRP_ATTR_S *attr
);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |
| attr           | Pointer to the CPP group attributes | Output      |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

- The group must have been created.
- The group attributes must be valid. Static attributes cannot be dynamically set. For details, see `CPP_GRP_ATTR_S`.

**[Related Operations]**

`cam_cpp_set_grp_attr`

#### cam_cpp_set_grp_attr

**[Description]**

Set the attributes of the CPP group.

**[Syntax]**

```c
int32_t cam_cpp_set_grp_attr(
    uint32_t grpId,
    const CPP_GRP_ATTR_S *attr
);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |
| attr           | Pointer to the CPP group attributes | Input       |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

- The group must have been created.
- The group attributes must be valid. Static attributes cannot be dynamically set. For details, see `CPP_GRP_ATTR_S`.

**[Related Operations]**

`cam_cpp_get_grp_attr`

#### cam_cpp_get_tuning_param

**[Description]**

Get the tuning parameters of the CPP Group.

**[Syntax]**

```c
int32_t cam_cpp_get_tuning_param(
    uint32_t grpId,
    cpp_tuning_params_t *tuningParam
);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |
| tuningParam    | Pointer to the module tuning parameters | Output      |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header files: `cam_cpp.h`, `CPPGlobalDefine.h`
- Library file: `libcpp.so`

**[Note]**

- The group must have been created.

**[Related Operations]**

`cam_cpp_set_tuning_param`

#### cam_cpp_set_tuning_param

**[Description]**

Set the tuning parameters of the CPP Group.

**[Syntax]**

```c
int32_t cam_cpp_set_tuning_param(
    uint32_t grpId,
    cpp_tuning_params_t *tuningParam
);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |
| tuningParam    | Pointer to the module tuning parameters | Input       |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header files: `cam_cpp.h`, `CPPGlobalDefine.h`
- Library file: `libcpp.so`

**[Note]**

- The group must have been created.

**[Related Operations]**

`cam_cpp_get_tuning_param`

#### cam_cpp_load_settingfile

**[Description]**

Tuning debugging interface, load firmware setting file.

**[Syntax]**

```c
int32_t cam_cpp_load_settingfile(
    uint32_t grpId,
    const char *fileName
);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |
| fileName       | Pointer to the path of the cpp firmware configuration file | Input       |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

- The group must have been created.

**[Related Operations]**

`cam_cpp_save_settingfile`

#### cam_cpp_save_settingfile

**[Description]**

Tuning debugging interface, save firmware setting file.

**[Syntax]**

```c
int32_t cam_cpp_save_settingfile(
    uint32_t grpId,
    const char *fileName
);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |
| fileName       | Pointer to the path of the cpp firmware configuration file | Input       |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

- The group must have been created.

**[Related Operations]**

`cam_cpp_load_settingfile`

#### cam_cpp_read_fw

**[Description]**

Tuning debugging interface, get parameters of the CPP Group Firmware module.

**[Syntax]**

```c
int32_t cam_cpp_read_fw(
    uint32_t grpId,
    const char *filter,
    const char *param,
    uint32_t row,
    uint32_t column,
    int32_t *pVal
);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |
| filter         | Pointer to the name of the Firmware Filter | Input       |
| param          | Pointer to the name of the Firmware Parameter | Input       |
| row            | Row offset of the Firmware Parameter element | Input       |
| column         | Column offset of the Firmware Parameter element | Input       |
| pVal           | Pointer to the list of Firmware Parameter elements | Output      |

**[Return Values]**

| Parameter Name | Description |
|----------------|-------------|
| -1             | Failure. Check the input parameter names. |
| 0              | Failure. No parameter value was obtained. |
| Positive values| Success. Indicates the number of elements obtained for the parameter. 1 for a single element, >1 for multiple elements |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

- The group must have been created.

**[Related Operations]**

`cam_cpp_write_fw`

#### cam_cpp_write_fw

**[Description]**

Tuning debugging interface, set parameters of the CPP Group Firmware module.

**[Syntax]**

```c
int32_t cam_cpp_write_fw(
    uint32_t grpId,
    const char *pFilterName,
    const char *pParamName,
    uint32_t row,
    uint32_t column,
    int32_t val
);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |
| filter         | Pointer to the name of the Firmware Filter | Input       |
| param          | Pointer to the name of the Firmware Parameter | Input       |
| row            | Row offset of the Firmware Parameter element | Input       |
| column         | Column offset of the Firmware Parameter element | Input       |
| s32Val         | Value of the Firmware Parameter element | Input       |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

- The group must have been created.

**[Related Operations]**

`cam_cpp_read_fw`

#### cam_cpp_read_reg

**[Description]**

Tuning debugging interface, get the value of the CPP Hardware register.

**[Syntax]**

```c
int32_t cam_cpp_read_reg(
    uint32_t addr,
    uint32_t *val
);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| addr           | Hardware register address | Input       |
| val            | Hardware register value | Output      |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

- The group must have been created.
- There is only one set of hardware register resources, and the lower 16-bit specifies the address offset.

**[Related Operations]**

`cam_cpp_write_reg`

#### cam_cpp_write_reg

**[Description]**

Tuning debugging interface, set the value of the CPP Hardware register.

**[Syntax]**

```c
int32_t cam_cpp_write_reg(
    uint32_t addr,
    uint32_t val,
    uint32_t mask
);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| addr           | Hardware register address | Input       |
| val            | Hardware register value | Input       |
| mask           | Hardware register mask | Input       |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

- The group must have been created.
- There is only one set of hardware register resources, and the lower 16-bit specifies the address offset.

**[Related Operations]**

`cam_cpp_read_reg`

#### cam_cpp_dump_frame

**[Description]**

Save the input and output images of the specified number of frames for the group to the specified directory.

**[Syntax]**

```c
int32_t cam_cpp_dump_frame(
    uint32_t grpId,
    const char *path,
    uint32_t count
);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| grpId          | CPP group ID. Range: [0, CPP_GRP_MAX_NUM) | Input       |
| path           | Specified dump directory, cannot be NULL | Input       |
| count          | Number of consecutive frames to be dumped | Input       |

**[Return Values]**

| Parameter Name | Description             |
|----------------|-------------------------|
| 0              | Success                 |
|  Non-0       | Failure (error code value) |

**[Requirements]**

- Header file: `cam_cpp.h`
- Library file: `libcpp.so`

**[Note]**

- The group must have been created.
- This API is an asynchronous operation. After returning success, it will continuously dump the specified number of frames starting from that moment. If the specified number of frames has not been dumped yet, calling this interface again will interrupt the previous operation and start the current operation.

**[Related Operations]**

None

### Data Types

The data types related to the CPP module are defined as follows:

- **CPP_GRP_MAX_NUM:** Defines the maximum number of CPP GROUPs.
- **CPP_MIN_IMAGE_WIDTH:** Defines the minimum width of the CPP image.
- **CPP_MIN_IMAGE_HEIGHT:** Defines the minimum height of the CPP image.
- **CPP_MAX_IMAGE_WIDTH:** Defines the maximum width of the CPP image.
- **CPP_MAX_IMAGE_HEIGHT:** Defines the maximum height of the CPP image.
- **CPP_GRP:** Defines the CPP group number.
- **CPP_MOD_ID_E:** Defines the CPP submodule ID.
- **CPP_GRP_ATTR_S:** Defines the attributes of the CPP GROUP.
- **CPP_GRP_NR_ATTR_S:** Defines the NR attributes of the CPP GROUP.
- **CPP_GRP_EE_ATTR_S:** Defines the EE attributes of the CPP GROUP.
- **CPP_GRP_TNR_ATTR_S:** Defines the TNR attributes of the CPP GROUP.

#### CPP_GRP_MAX_NUM

**[Description]**

Defines the maximum number of CPP Groups.

**[Definition]**

```
#define CPP_GRP_MAX_NUM (16)
```

**[Member Variables]**

None

**[Note]**

None

**[Related Types and Data Structures]**

None

#### CPP_MIN_IMAGE_WIDTH

**[Description]**

Defines the minimum width of the image supported by the CPP module.

**[Definition]**

```
#define CPP_MIN_IMAGE_WIDTH (480)
```

**[Member Variables]**

None

**[Note]**

None

**[Related Types and Data Structures]**

None

#### CPP_MIN_IMAGE_HEIGHT

**[Description]**

Defines the minimum height supported by the CPP image.

**[Definition]**

```
#define CPP_MIN_IMAGE_HEIGHT (288)

```

**[Member Variables]**

None

**[Note]**

None

**[Related Types and Data Structures]**

None

#### CPP_MAX_IMAGE_WIDTH

**[Description]**

Defines the maximum width supported by the CPP image.

**[Definition]**

```
#define CPP_MAX_IMAGE_WIDTH (4224)
```

**[Member Variables]**

None

**[Note]**

None

**[Related Types and Data Structures]**

None

#### CPP_MAX_IMAGE_HEIGHT

**[Description]**

Defines the maximum height of the CPP image.

**[Definition]**

```
#define CPP_MAX_IMAGE_HEIGHT (3136)
```

**[Member Variables]**

None

**[Note]**

None

**[Related Types and Data Structures]**

None

#### CPP_MOD_ID_E

**[Description]**

Defines the CPP submodule ID.

**[Definition]**

```java
typedef enum { 
    CPP_ID_3DNR, 
    CPP_ID_MAX,
} CPP_MOD_ID_E
```

**[Member Variables]**

None

**[Note]**

None

**[Related Types and Data Structures]**

None

#### CPP_GRP_ATTR_S

**[Description]**

Defines the attributes of the CPP Group.

**[Definition]**

```java
typedef struct asrCPP_GRP_ATTR_S { 
    uint32_t width;
    uint32_t height; 
    PIXEL_FORMAT_E format; 
    CPP_GRP_WORKMODE_E mode;
} CPP_GRP_ATTR_S;
```

**[Member Variables]**

| Variable | Description |
|----------|-------------|
| width, height | Dimensions of the input image. Valid range for width: [CPP_MIN_IMAGE_WIDTH, CPP_MAX_IMAGE_WIDTH]. Valid range for height: [CPP_MIN_IMAGE_HEIGHT, CPP_MAX_IMAGE_HEIGHT]. These are static attributes, meaning they cannot be dynamically modified after the group is created. |
| format        | Format of the output image. This is a static attribute, meaning it cannot be dynamically modified after the group is created. |
| mode          | Operating mode   |

**[Note]**

- The input image dimensions must be configured according to the actual width and height of the CPP operation. The TNR KGain buffer and output image buffer are allocated based on the input image dimensions. The input image format only supports YUV data.

**[Related Types and Data Structures]**

- `PIXEL_FORMAT_E` 
- `CPP_GRP_WORKMODE_E`

#### CPP_GRP_WORKMODE_E

**[Description]**

Defines the working mode of the CPP Group.

**[Definition]**

```java
typedef enum { 
    CPP_GRP_FRAME_MODE, 
    CPP_GRP_SLICE_MODE, 
    CPP_GRP_MODE_MAX,
} CPP_GRP_WORKMODE_E;
```

**[Member Variables]**

| Member Name | Description |
|---------------------------|-------------------------------|
| CPP_GRP_FRAME_MODE | Data is processed on a per-frame basis, with high hardware processing priority |
| CPP_GRP_SLICE_MODE | Data is divided and processed in slices, with low hardware processing priority |

**[Note]**

None

**[Related Types and Data Structures]**

None

### Error Codes

Refer to the Standard C library header file `errno.h`

## tuningtools API

This section introduces the usage of the ASR MARS11-ISP tuningtools module API, and provides detailed explanations of the related parameter data structures and return values.

### API

The tuningtools module provides the following APIs to users:

- **ASR_TuningAssistant_Create:** Create a tuning Assistant.
- **ASR_TuningAssistant_Destroy:** Destroy a tuning Assistant.

#### ASR_TuningAssistant_Create

**[Description]**

Create a tuningtool Assistant.

**[Syntax]**

```
ASR_TUNING_ASSISTANT_HANDLE

ASR_TuningAssistant_Create(const ASR_TUNING_ASSISTANT_TRIGGER_S *trigger);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| trigger        | Configuration parameters for the tuning assistant | Input       |

**[Return Values]**

| Parameter Name | Description                             |
|----------------|-----------------------------------------|
| Non-NULL       | Success, returns a pointer to the assistant handle |
| NULL           | Failure                                 |

**[Requirements]**

- Header file: `asr_cam_tuning_assistant.h`
- Library file: `libtuningtools.so`

**[Note]**

- Duplicate creation is not supported.
- The structure of `ASR_TUNING_ASSISTANT_TRIGGER_S` is described in the next section on data types.

**[Related Operations]**

`ASR_TuningAssistant_Destroy`

#### ASR_TuningAssistant_Destroy

**[Description]**

Destroy a tuningtool Assistant.

**[Syntax]**

```
bool ASR_TuningAssistant_Destroy(ASR_TUNING_ASSISTANT_HANDLE handle);
```

**[Parameters]**

| Parameter Name | Description | Input/Output |
|----------------|-------------|--------------|
| handle         | Pointer to the tuningassistant structure to be destroyed, must be the return value of `ASR_TuningAssistant_Create` | Input   |

**[Return Values]**

| Parameter Name | Description |
|----------------|-------------|
| true           | Success     |
| false          | Failure     |

**[Requirements]**

- Header file: `asr_cam_tuning_assistant.h`
- Library file: `libtuningtools.so`

**[Note]**

- Duplicate destruction is not supported.

**[Related Operations]**

`ASR_TuningAssistant_Create`

### Data Types

#### ASR_TUNING_ASSISTANT_HANDLE

**[Description]**

Defines the tuning assistant handler pointer.

**[Definition]**

```
typedef void *ASR_TUNING_ASSISTANT_HANDLE;
```

**[Members]**

None

**[Note]**

None

**[Related Types and Data Structures]**

None

#### TUNING_MODULE_TYPE_E

**[Description]**

Defines the types of supported tuning modules.

**[Definition]**

```java
typedef enum TUNING_MODULE_TYPE { 
    TUNING_MODULE_TYPE_ISP, 
    TUNING_MODULE_TYPE_CPP, 
    TUNING_MODULE_TYPE_ALGO, 
    TUNING_MODULE_TYPE_MAX
} TUNING_MODULE_TYPE_E;
```

**[Members]**

| Member Name | Description |
|--------------------------------|-------------------------------------|
| TUNING_MODULE_TYPE_ISP  | ISP module type |
| TUNING_MODULE_TYPE_CPP  | CPP module type |
| TUNING_MODULE_TYPE_ALGO | Algorithm module type (users do not need to concern themselves with this) |
| TUNING_MODULE_TYPE_MAX  | Maximum number of tunable module types |

**[Note]**

None

**[Related Types and Data Structures]**

None

#### TUNING_BUFFER_S

**[Description]**

Defines the buffer structure used when the tuning module enables the dump raw image feature.

**[Definition]**

```java
typedef struct TUNING_BUFFER { 
    uint32_t frameId;
    uint32_t length; 
    void *virAddr;
} TUNING_BUFFER_S, *TUNING_BUFFER_S_PTR;
```

**[Members]**

| Member Name | Description                 |
|-----------------|-----------------------------|
| frameId         | Image frame ID             |
| length          | Length of the image buffer |
| virAddr         | Starting address of the image buffer |

**[Note]**

- Used in conjunction with `ASR_TUNING_ASSISTANT_TRIGGER_S` as one of the input parameters for `StartDumpRaw`.

**[Related Types and Data Structures]**

None

#### TUNING_MODULE_OBJECT_S

**[Description]**

Defines the basic information of a single module entity to be tuned.

**[Definition]**

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

**[Members]**

| Member Name | Description |
|---------------------------|----------------------------|
| type                  | Type of the module to be tuned |
| moduleHandle          | Handle pointer of the module to be tuned. For algorithm modules, this is used. For ISP and CPP types, it is implemented internally by tuningtools and does not need to be concerned with. |
| groupId               | Group ID of the module to be tuned. For ISP type, the range is [0, 1]. For CPP type, the range is [0, 15]. |
| dumpRaw               | Whether to support dumping RAW image data |
| name                  | Name of the module to be tuned |
| loadSettingFile       | For algorithm modules, this is the callback for loading setting files. For ISP and CPP, it is implemented internally by tuningtools and does not need to be concerned with. |
| saveSettingFile       | For algorithm modules, this is the callback for saving setting files. For ISP and CPP, it is implemented internally by tuningtools and does not need to be concerned with. |
| saveFilterSettingFile | For algorithm modules, this is the callback for saving a specific filter setting file. For ISP and CPP, it is implemented internally by tuningtools and does not need to be concerned with. |
| readFirmware          | For algorithm modules, this is the callback for reading a specific filter value. For ISP and CPP, it is implemented internally by tuningtools and does not need to be concerned with. |
| writeFirmware         | For algorithm modules, this is the callback for setting a specific filter value. For ISP and CPP, it is implemented internally by tuningtools and does not need to be concerned with. |

**[Note]**

- When tuning ISP and CPP, focus on `type`, `groupId`, and `name`. `dumpRaw` is generally set to `0`.

**[Related Types and Data Structures]**

None

#### ASR_TUNING_ASSISTANT_TRIGGER

**[Description]**

Defines the basic callback for the collection of all modules to be tuned.

**[Definition]**

```java
typedef struct ASR_TUNING_ASSISTANT_TRIGGER { 
    uint32_t (*GetModuleCount)();
    int32_t (*GetModules)(uint32_t moduleCount, TUNING_MODULE_OBJECT_S *modules); 
    int32_t(*StartDumpRaw)(TUNING_MODULE_TYPE_Etype,uint32_tgroupId,uint32_t frameCount, TUNING_BUFFER_S *frames);
    int32_t (*EndDumpRaw)(TUNING_MODULE_TYPE_E type, uint32_t groupId);
} ASR_TUNING_ASSISTANT_TRIGGER_S, *ASR_TUNING_ASSISTANT_TRIGGER_S_PTR;
```

**[Members]**

| Member Name | Description |
|-------------|-------------|
| GetModuleCount | Returns the total number of modules to be tuned |
| GetModules     | Obtains a list of `moduleCount` modules to be tuned (placed in the `modules` array), and returns the actual number of modules obtained |
| StartDumpRaw   | Callback to start rawdump, not recommended for use |
| EndDumpRaw     | Callback to end rawdump, not recommended for use |

**[Note]**

- When tuning ISP and CPP, the members of `TUNING_MODULE_OBJECT_S` should be determined based on the actual ISP and CPP `groupId` used. You can refer to the usage in `demo/online_pipeline_test.c` under the `tuning_server_init()` function.

**[Related Types and Data Structures]**

None
