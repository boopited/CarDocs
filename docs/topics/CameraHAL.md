---
layout: post
title:  "相机HAL"
parent: "专题"
date:   2020-01-10
nav_order: 3
---

# 车载相机 HAL

Android 8.0 包含一个汽车 HIDL 硬件抽象层 (HAL)，可用于在 Android 启动过程的初期提供图像捕获和显示，并在系统启动后继续运行直到系统终止。HAL 包含外部视景系统 (EVS) 堆栈，通常用于在具有车载信息娱乐 (IVI) 系统（基于 Android）的车辆中支持后视摄像头和环绕视图显示。EVS 还支持在用户应用中实现高级功能。

Android 8.0 还包含一个特定于 EVS 的捕获和显示驱动程序接口（在 /hardware/interfaces/automotive/evs/1.0 中）。尽管可以基于现有的 Android 相机和显示服务编译后视摄像头应用，但这类应用可能会在 Android 启动过程中太迟运行。使用专用 HAL 可实现简化的接口，并清楚地说明原始设备制造商 (OEM) 需要实现什么才能支持 EVS 堆栈。

## 系统组件

EVS 包括以下系统组件：

![](/assets/images/vhal_evs_components.png)

**图 1** EVS 系统组件概览。

### EVS 应用

可作为参考实现的 C++ EVS 示例应用 (`/packages/services/Car/evs/app`)。该应用负责从 EVS 管理器请求视频帧，并将用于显示的已完成的帧发送回 EVS 管理器。 EVS 和汽车服务可供使用后，它便立即由 init 启动（设置目标为在开机两 (2) 秒内启动）。原始设备制造商 (OEM) 可视需要修改或替换 EVS 应用。

### EVS 管理器

EVS 管理器 (`/packages/services/Car/evs/manager`) 可提供 EVS 应用所需的构建块，以实现从简单的后视摄像头显示到 6DOF 多摄像头渲染的任何功能。它的接口通过 HIDL 呈现，并且能够接受多个并发客户端。 其他应用和服务（特别是汽车服务）可以查询 EVS 管理器状态，以了解 EVS 系统何时处于活动状态。

### EVS HIDL 接口

在 EVS 系统中，相机和显示元素均在 `android.hardware.automotive.evs` 程序包中进行定义。用于实践接口的示例实现（生成合成测试图像并验证图像进行往返的过程）在 `/hardware/interfaces/automotive/evs/1.0/default` 中进行提供。

原始设备制造商 (OEM) 负责实现由 `/hardware/interfaces/automotive/evs` 中的 .hal 文件表示的 API。这种实现负责从物理相机配置和收集数据，并通过 Gralloc 可识别的共享内存缓冲区传送这些数据。实现的显示端负责提供可由应用填充（通常通过 EGL 渲染的方式）的共享内存缓冲区，并优先呈现已完成的帧（在任何可能会显示在物理显示设备上的其他内容之前）。EVS 接口的供应商实现可以存储在 `/vendor/… /device/…` 或 `hardware/…`（例如 `/hardware/[vendor]/[platform]/evs`）下。

### 内核驱动程序

支持 EVS 堆栈的设备需要使用内核驱动程序。原始设备制造商 (OEM) 无需创建新驱动程序，他们可以选择通过现有相机和/或显示硬件驱动程序来支持 EVS 所需的功能。重复使用驱动程序可能会有好处，对于图像呈现可能需要与其他活动线程协调的显示驱动程序来说尤其如此。Android 8.0 包含一个基于 v4l2 的示例驱动程序（在 `packages/services/Car/evs/sampleDriver` 中），该驱动程序依靠内核实现 v4l2 支持，并依靠 SurfaceFlinger 呈现输出图像。

>  **注意**：对于实际的供应商实现来说，依靠 SurfaceFlinger 是不合适的，因为 EVS 必须能够在开机后的几秒钟内运行，这个时间要远早于 SurfaceFlinger 本身启动的时间。 不过，示例驱动程序实现一般与硬件无关，而且支持 EVS 应用开发和测试与 EVS 驱动程序开发同时进行。

## EVS 硬件接口说明

本部分介绍 HAL。供应商应该会提供该 API 的实现（已针对其硬件进行调整）。

### IEvsEnumerator

该对象负责枚举系统中可用的 EVS 硬件（一个或多个相机和单个显示设备）。

```
getCameraList() generates (vec<CameraDesc> cameras);
```

返回包含系统中所有相机的说明的矢量。假设相机组是固定的且在启动时是可知的。要详细了解相机说明，请参阅 `CameraDesc`。

```
openCamera(string camera_id) generates (IEvsCamera camera);
```

获取用于与唯一 camera_id 字符串所标识的特定相机进行交互的接口对象。失败时返回 NULL。 尝试重新打开已打开的相机不会失败。为了避免出现与应用启动和关闭相关的争用状况，重新打开相机应该会关闭上一个实例，以满足新的请求。以这种方式被优先占用的相机实例必须置于非活动状态，等待最终销毁并响应任何请求以利用返回代码 `OWNERSHIP_LOST` 影响相机状态。

```
closeCamera(IEvsCamera camera);
```

释放 IEvsCamera 接口（与 `openCamera()` 调用相反）。必须通过依次调用 `stopVideoStream()` 和 `closeCamera` 来停止相机视频流。

```
openDisplay() generates (IEvsDisplay display);
```

获取用于专门与系统的 EVS 显示进行交互的接口对象。目前只有一个客户端可以持有 IEvsDisplay 的函数实例。与 `openCamera` 中描述的激进式打开行为相似，您可以随时创建一个新的 IEvsDisplay 对象，并停用任何先前的实例。无效的实例会继续存在并响应来自其所有者的函数调用，但不得在终止后执行任何变相的操作。最终，客户端应用应注意到 `OWNERSHIP_LOST` 错误返回代码，并关闭和释放终止的接口。

```
closeDisplay(IEvsDisplay display);
```

释放 IEvsDisplay 接口（与 `openDisplay()` 调用相反）。通过 `getTargetBuffer()` 调用接收的未完成缓冲区必须在显示关闭之前返回到显示。

```
getDisplayState() generates (DisplayState state);
```

获取当前的显示状态。HAL 实现应报告实际的当前状态，该状态可能与最近请求的状态不同。负责更改显示状态的逻辑应存在于设备层之上，从而使 HAL 实现无需自发更改显示状态。如果显示目前未被任何客户端持有（通过调用 openDisplay），则此函数会返回 `NOT_OPEN`。否则，它会报告 EVS 显示的当前状态（请参阅 [IEvsDisplay API](https://source.android.com/devices/automotive/camera-hal#ievsdisplay-hw)）。

```
struct CameraDesc {
    string      camera_id;
    int32       vendor_flags;       // Opaque value
}
```

- `camera_id`。用于唯一标识指定相机的字符串。 该字符串可以是设备的内核设备名称，也可以是设备的名称，例如 rearview。该字符串的值由 HAL 实现选择，并由上面的堆栈以不透明的方式使用。
- `vendor_flags`。用于以不透明的方式将特殊相机信息从驱动程序传送到自定义 EVS 应用的一种方法。这些信息未经解释即从驱动程序向上直通到 EVS 应用，可直接将其忽略。

### IEvsCamera

该对象表示单个相机，是用于拍摄图像的主要接口。

```
getCameraInfo() generates (CameraDesc info);
```

返回此相机的 `CameraDesc`。

```
setMaxFramesInFlight(int32 bufferCount) generates (EvsResult result);
```

指定要求相机支持的缓冲区链的深度。IEvsCamera 的客户端可以同时持有数量如此多的帧。如果数量如此多的帧已传送到接收器但 `doneWithFrame` 未将其返回，则数据流会跳过帧，直到返回缓冲区以供重新使用。该调用在任何时候返回都是合法的，即使数据流已在运行（这种情况下，应相应地在链中添加或移除缓冲区）。如果未对此入口点进行任何调用，则 IEvsCamera 默认支持至少一个帧（更多的帧也可以接受）。

如果无法容纳请求的 bufferCount，该函数会返回 `BUFFER_NOT_AVAILABLE` 或其他相关的错误代码。在这种情况下，系统将继续使用先前设置的值运行。

```
startVideoStream(IEvsCameraStream receiver) generates (EvsResult result);
```

请求从此相机传送 EVS 相机帧。IEvsCameraStream 开始接收带有新图像帧的定期调用，直到调用 `stopVideoStream()` 为止。必须在调用 `startVideoStream` 后的 500 毫秒内开始传送帧，而且开始后，必须至少以 10 FPS 的速率生成帧。实际上，启动视频流所需的时间需要根据任何后视摄像头启动时间要求进行计算。如果未启动视频流，则必须返回错误代码；否则，则返回 OK。

```
oneway doneWithFrame(BufferDesc buffer);
```

返回传送到 IEvsCameraStream 的帧。传送到 IEvsCameraStream 接口的帧用完以后，必须被返回到 IEvsCamera 以供重新使用。只有极少的数量有限（最少可能只有一个）的缓冲区可供使用；如果供应的缓冲区耗尽，则在返回缓冲区之前不再传送任何帧，这可能会导致产生跳过的帧（具有空句柄的缓冲区表示数据流结束，无需通过此函数返回）。成功时返回 OK，否则返回相应错误代码（可能包括 `INVALID_ARG` 或 `BUFFER_NOT_AVAILABLE`）。

```
stopVideoStream();
```

停止传送 EVS 相机帧。由于传送是异步的，在此调用返回后的一段时间内，系统可能会继续传送帧。在将数据流关闭信号发送到 IEvsCameraStream 之前，所有帧均必须返回。对已停止或从未启动的数据流调用 `stopVideoStream` 是合法的，在这种情况下，系统会忽略该调用。

```
getExtendedInfo(int32 opaqueIdentifier) generates (int32 value);
```

从 HAL 实现中请求特定于驱动程序的信息。允许用于 `opaqueIdentifier` 的值是特定于驱动程序的，但传送任何值都不会使驱动程序崩溃。对于任何无法识别的 `opaqueIdentifier`，驱动程序应返回 0。

```
setExtendedInfo(int32 opaqueIdentifier, int32 opaqueValue) generates (EvsResult result);
```

将特定于驱动程序的值发送到 HAL 实现。提供这项扩展的目的只是为了便于实现特定于车辆的扩展，而且任何 HAL 实现均不得要求该调用在默认状态下运行。如果驱动程序识别出并接受值，则应返回 OK；否则应返回 `INVALID_ARG` 或其他有代表性的错误代码。

```
struct BufferDesc {
    uint32  width;      // Units of pixels
    uint32  height;     // Units of pixels
    uint32  stride;     // Units of pixels
    uint32  pixelSize;  // Size of single pixel in bytes
    uint32  format;     // May contain values from android_pixel_format_t
    uint32  usage;      // May contain values from Gralloc.h
    uint32  bufferId;   // Opaque value
    handle  memHandle;  // gralloc memory buffer handle
}
```

描述通过 API 传送的图像。HAL 驱动程序负责填写此结构以描述图像缓冲区，而且 HAL 客户端应将此结构视为只读。这些字段包含足够的信息供客户端重建 `ANativeWindowBuffer` 对象，因为可能需要通过 `eglCreateImageKHR()` 扩展才能将图像与 EGL 结合使用。

- `width`。所呈现图像的宽度（以像素为单位）。
- `height`。所呈现图像的高度（以像素为单位）。
- `stride`。每行在存储设备中实际占用的像素数；用来计算行对齐的任何内边距。以像素表示，用于匹配 gralloc 为其缓冲区说明采用的惯例。
- `pixelSize`。由每个单独的像素占据的字节数；用于计算在图像中分隔开各个行所需的字节大小（以字节为单位的 `stride` = 以像素为单位的 `stride` × `pixelSize`）。
- `format`。图像使用的像素格式。提供的格式必须与平台的 OpenGL 实现相兼容。要通过兼容性测试，相机使用应首选 `HAL_PIXEL_FORMAT_YCRCB_420_SP`；显示应首选 `RGBA` 或 `BGRA`。
- `usage`。由 HAL 实现设置的使用标记。HAL 客户端应原封不动地传送这些标记（有关详细信息，请参阅 `Gralloc.h` 相关标记）。
- `bufferId`。由 HAL 实现指定的唯一值，用于在通过 HAL API 进行往返之后识别缓冲区。存储在该字段中的值可由 HAL 实现任意选择。
- `memHandle`。包含图像数据的底层内存缓冲区的句柄。HAL 实现可能会选择在此处存储 Gralloc 缓冲区句柄。

### IEvsCameraStream

客户端实现此接口以接收异步传送的视频帧。

```
deliverFrame(BufferDesc buffer);
```

每次视频帧准备好进行检查时，都从 HAL 接收调用。 通过此方法接收的缓冲区句柄必须通过调用 `IEvsCamera::doneWithFrame()` 来返回。当视频流通过调用 `IEvsCamera::stopVideoStream()` 停止时，此回调可能会像下水道排水一样持续一段时间。每个帧仍必须返回；当传送完流数据流中的最后一帧后，系统将传送一个 NULL bufferHandle，这表示数据流结束，不会再发生进一步的帧传递。NULL bufferHandle 本身不需要通过 `doneWithFrame()` 发回，但是必须返回所有其他句柄

尽管专用缓冲区格式在技术上是可行的，但兼容性测试要求缓冲区采用以下五种受支持的格式之一：NV21 (YCrCb 4:2:0 Semi-Planar)、YV12 (YCrCb 4:2:0 Planar)、YUYV (YCrCb 4:2:2 Interleaved)、RGBA (32 位 R:G:B:x)、BGRA (32 位 B:G:R:x)。所选格式必须是平台的 GLES 实现上的有效 GL 纹理源。

应用**不**得依赖于 `bufferId` 字段与 `BufferDesc` 结构中的 `memHandle` 之间的任何对应关系。一般来说，`bufferId` 值对于 HAL 驱动程序实现是私有的，该实现可以在适当的情况下使用（和重新使用）这些值。

### IEvsDisplay

该对象可表示 EVS 显示，控制显示状态，并处理图像的实际呈现效果。

```
getDisplayInfo() generates (DisplayDesc info);
```

返回关于系统提供的 EVS 显示的基本信息（请参阅 [DisplayDesc](https://source.android.com/devices/automotive/camera-hal#displaydesc)）。

```
setDisplayState(DisplayState state) generates (EvsResult result);
```

设置显示状态。客户端可以将显示状态设置为所需的状态，而且不论目前处于何种状态，HAL 实现都必须无条件接受更改为另一种状态的请求，尽管响应可能会忽略该请求。

在初始化后，显示会被定义为以 `NOT_VISIBLE` 状态开始，之后，客户端应请求 `VISIBLE_ON_NEXT_FRAME` 状态并开始提供视频。当不再需要显示时，客户端应在传送最后一个视频帧后请求 `NOT_VISIBLE` 状态。

任何时候都可以请求任何状态。如果显示已处于可见状态，那么设置为 `VISIBLE_ON_NEXT_FRAME` 后，显示应保持可见状态。系统将一律返回 OK，除非请求的状态是无法识别的枚举值（在这种情况下会返回 `INVALID_ARG`）。

```
getDisplayState() generates (DisplayState state);
```

获取显示状态。HAL 实现应报告实际的当前状态，这可能与最近请求的状态不同。负责更改显示状态的逻辑应该存在于设备层之上，从而使 HAL 实现无需自发更改显示状态。

```
getTargetBuffer() generates (handle bufferHandle);
```

将句柄返回到与显示相关联的帧缓冲区。该缓冲区可能会被软件和/或 GL 锁定和写入。此缓冲区必须通过调用 `returnTargetBufferForDisplay()` 来返回（即使显示已不再处于可见状态）。

尽管专用缓冲区格式在技术上是可行的，但兼容性测试要求缓冲区采用以下五种受支持的格式之一：NV21 (YCrCb 4:2:0 Semi-Planar)、YV12 (YCrCb 4:2:0 Planar)、YUYV (YCrCb 4:2:2 Interleaved)、RGBA (32 位 R:G:B:x)、BGRA (32 位 B:G:R:x)。所选格式必须是平台的 GLES 实现上的有效 GL 渲染目标。

在出现错误时，系统会返回一个带有空句柄的缓冲区，但这种缓冲区不需要传送回 `returnTargetBufferForDisplay`。

```
returnTargetBufferForDisplay(handle bufferHandle) generates (EvsResult result);
```

向显示指明缓冲区已可供显示。只有通过调用 `getTargetBuffer()` 而检索的缓冲区才能用于此调用，而且 `BufferDesc` 的内容不能由客户端应用进行修改。在此调用后，缓冲区不再可供客户端使用。系统会在成功时返回 OK，否则会返回相应错误代码（可能包括 `INVALID_ARG` 或 `BUFFER_NOT_AVAILABLE`）。

```
struct DisplayDesc {
     string  display_id;
     int32   vendor_flags;  // Opaque value
}
```

该函数描述 EVS 显示的基本属性，并且 EVS 实现需要该函数。HAL 负责填写此结构以描述 EVS 显示。这种显示可以是物理显示，也可以是与其他演示设备重叠或混合的虚拟显示。

- `display_id`。唯一标识显示的字符串。 该字符串可以是设备的内核设备名称，也可以是设备的名称，例如“rearview”。该字符串的值由 HAL 实现进行选择，并由上面的堆栈以不透明的方式使用。
- `vendor_flags`。用于以不透明的方式将特殊相机信息从驱动程序传送到自定义 EVS 应用的一种方法。这些信息未经解释即从驱动程序向上直通到 EVS 应用，可直接将其忽略。

```
enum DisplayState : uint32 {
    NOT_OPEN,               // Display has not been “opened” yet
    NOT_VISIBLE,            // Display is inhibited
    VISIBLE_ON_NEXT_FRAME,  // Will become visible with next frame
    VISIBLE,                // Display is currently active
    DEAD,                   // Display is not available. Interface should be closed
}
```

用于描述 EVS 显示的状态，这种状态可以是“已停用”（对驱动程序不可见），也可以是“已启用”（向驱动程序显示图像）。 它包括一个过渡状态；在该状态下，显示尽管还处于不可见状态，但已准备好通过 `returnTargetBufferForDisplay()` 调用传送下一帧图像，从而变为可见状态。

## EVS 管理器

EVS 管理器提供 EVS 系统的公共接口，用于收集和呈现外部相机视图。在硬件驱动程序针对每个资源（相机或显示设备）仅允许支持一个活动接口的情况下，EVS 管理器可为对相机进行共享访问提供便利。单个主要 EVS 应用是 EVS 管理器的第一个客户端，并且是有权写入显示数据的唯一客户端（其他客户端可获得对相机图像的只读访问权限）。

EVS 管理器实现与底层 HAL 驱动程序相同的 API，并通过支持多个并发客户端来提供扩展服务（多个客户端可以通过 EVS 管理器打开相机并接收视频流）。

![](/assets/images/vhal_evs_manager.png)

**图 2.** EVS 管理器镜像底层 EVS 硬件 API

除非 EVS 管理器 API 允许对相机数据流进行并发访问，否则通过 EVS 硬件 HAL 实现操作还是通过 EVS 管理器 API 操作，对应用而言将不会有任何差异。EVS 管理器本身就是 EVS 硬件 HAL 层允许的客户端，并且充当 EVS 硬件 HAL 的代理。

以下部分仅介绍在 EVS 管理器实现中具有不同（扩展）行为的调用，其余调用与 EVS HAL 说明中介绍的相同。

### IEvsEnumerator

```
openCamera(string camera_id) generates (IEvsCamera camera);
```

获取用于与唯一 camera_id 字符串所标识的特定相机进行交互的接口对象。失败时返回 NULL。 在 EVS 管理器层，只要有足够的系统资源可用，已经打开的相机便可由另一进程再次打开，从而允许将视频流传送到多个消费者应用。EVS 管理器层上的 `camera_id` 字符串与向 EVS 硬件层报告的字符串相同。

### IEvsCamera

EVS 管理器提供的 IEvsCamera 实现在内部虚拟化，以便一个客户端在相机上进行的操作不会影响其他客户端，从而使客户端保持对其相机的独立访问。

```
startVideoStream(IEvsCameraStream receiver) generates (EvsResult result);
```

启动视频流。多个客户端可以相互独立地在同一底层相机上启动和停止视频流。底层相机在第一个客户端启动时启动。

```
doneWithFrame(uint32 frameId, handle bufferHandle) generates (EvsResult result);
```

返回帧。每个客户端必须在帧完成后返回帧，但是可以根据需要将帧持有任意长的时间。当客户端持有的帧数达到其配置的上限时，将不会再接收到任何帧，直到它返回一个帧。此次跳帧不会影响其他客户端，这些客户端可以继续接收预期要接收的所有帧。

```
stopVideoStream();
```

停止视频流。每个客户端都可以随时停止其视频流，而不会影响其他客户端。当给定相机的最后一个客户端停止其数据流时，硬件层的底层相机数据流将停止。

```
setExtendedInfo(int32 opaqueIdentifier, int32 opaqueValue) generates (EvsResult result);
```

发送特定于驱动程序的值，这可能会使一个客户端影响另一个客户端。由于 EVS 管理器无法理解供应商定义的控制字的含义，因此控制字不会被虚拟化，而且任何副作用都会殃及给定相机的所有客户端。例如，如果某个供应商使用此调用来更改帧速率，则受影响的硬件层相机的所有客户端都将以新的速率接收帧。

### IEvsDisplay

显示仅允许有一个所有者，即使在 EVS 管理器层面也是如此。管理器不添加任何功能，只是将 IEvsDisplay 接口直接传送给底层 HAL 实现。

## EVS 应用

Android 8.0 包含与 EVS 管理器和车载 HAL 通信的 EVS 应用的原生 C++ 参考实现，以提供基本的后视摄像头功能。该应用应在系统启动过程的早期启动，根据可用的相机和汽车状态（车轮和转向信号状态）显示合适的视频。 原始设备制造商 (OEM) 可以使用自己的特定于车辆的逻辑和呈现来修改或替换 EVS 应用。

![](/assets/images/vhal_evs_get_camera.png)

**图 3.** EVS 应用逻辑样本，获取相机列表。

![](/assets/images/vhal_evs_receive_frame.png)

**图 4.** EVS 应用逻辑样本，接收帧回调。

由于图像数据通过标准图形缓冲区呈现给应用，因此应用负责将图像从源缓冲区移动到输出缓冲区。尽管这会产生复制数据的成本，但也为应用提供了机会以任何所需的方式将图像呈现到显示缓冲区。

例如，应用可以选择移动像素数据本身，从而可以进行内嵌缩放或旋转操作。应用还可以选择使用源图像作为 OpenGL 纹理，并将复杂场景呈现给输出缓冲区，包括虚拟元素（如图标、准则和动画）。更复杂的应用还可以选择多个并发输入相机并将它们合并到单个输出帧中（例如用于从上到下地显示车辆周围环境的虚拟视图）。

## Android 8.0 之外

Android 8.0 支持基本后视摄像头应用，但 Google 发现了几种可能成为 EVS 堆栈未来扩展基础的用例。

> **注意**：Google 保留随时更改版本功能的权利。

- **环绕视图**。为了解决车辆的车身周围有多个相机时的用例，Google 正在考虑增强 EVS 应用，将来自多个并发相机的视频转换成适合狭小空间操作（例如在狭窄的空间中停车）的 3D 演示。
- **用户输入**。在 Android 8.0 中，应用负责从内核设备获取和解析输入事件。通过与 dev/event# 内核设备进行互动，可以在启动周期早期读取用户输入，并且从这些流读取事件不会影响 Android InputFlinger 监测这些相同输入流的能力。为了解决输入事件用例的相关问题，Google 正在考虑采用支持单轻触事件的简单 EVS 接口。