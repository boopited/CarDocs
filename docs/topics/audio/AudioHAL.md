---
layout: post
title:  "音频HAL"
parent: "音频"
grand_parent: "专题"
date:   2020-01-06
nav_order: 1
---

# 车载音频 HAL

车载音频实现依赖于标准 [Android 音频 HAL](https://source.android.google.cn/devices/audio/implement)，其中包括以下内容：

- IDevice (`hardware/interfaces/audio/2.0/IDevice.hal`)。负责创建输入流和输出流、处理主音量和静音操作，以及使用：
  - `createAudioPatch` 在设备之间创建 external-external 补丁程序。
  - `IDevice.setAudioPortConfig()` 为各个物理音频流提供音量。
- IStream (`hardware/interfaces/audio/2.0/IStream.hal`)。连同它的输入和输出变体一起管理进出硬件的样本音频流。

## 车载设备类型

以下设备类型与车载平台相关：

| 设备类型                         | 说明                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| `AUDIO_DEVICE_OUT_BUS`           | Android 的主要输出（Android 的所有音频均通过这种方式提供给车辆）。用作消除各个上下文的信息流歧义的地址。 |
| `AUDIO_DEVICE_OUT_TELEPHONY_TX`  | 用于传输路由到手机无线装置的音频。                           |
| `AUDIO_DEVICE_IN_BUS`            | 用于尚未进行分类的输入。                                     |
| `AUDIO_DEVICE_IN_FM_TUNER`       | 仅用于广播无线装置输入。                                     |
| `AUDIO_DEVICE_IN_TV_TUNER`       | 用于电视设备（如果存在）。                                   |
| `AUDIO_DEVICE_IN_LINE`           | 用于 AUX 输入耳机插孔。                                      |
| `AUDIO_DEVICE_IN_BLUETOOTH_A2DP` | 通过蓝牙接收到的音乐。                                       |
| `AUDIO_DEVICE_IN_TELEPHONY_RX`   | 用于从手机无线装置接收到的与通话相关联的音频。               |

## 路由音频源

您应使用 AudioRecord 或相关 Android 机制来捕获大多数音频源。接下来，可以[通过 AudioAttributes 类为数据分配音频属性](https://source.android.google.cn/devices/audio/attributes)并通过 AudioTrack 播放数据，只需依赖默认的 Android 路由逻辑或通过对 AudioRecord 或 AudioTrack 对象显式调用 `setPreferredDevice()` 即可。

对于与外部混音器之间有专用硬件连接的来源或具有极为严苛的延迟要求的来源而言，您可以使用 `createAudioPatch()` 和 `releaseAudioPatch()` 来启用和停用外部设备之间的路由（在样本传输过程中无需使用 AudioFlinger）。

## 配置音频设备

Android 可见的音频设备必须在 `/audio_policy_configuration.xml` 中进行定义，其中包括以下组件：

- module name。 支持“primary”（用于车载使用场景）、“A2DP”、“remote_submix”和“USB”。模块名称和相应音频驱动程序应编译到 `audio.primary.$(variant).so` 中。
- devicePorts。 包含可从此模块访问的所有输入和输出设备（包括永久连接的设备和可移除设备）的设备描述符列表。
  - 对于每种输出设备，您可以定义增益控制（包含以 millibel 为单位的 `min/value/step/default` 值，其中 1 millibel = 1/100 分贝 = 1/1000 贝尔）。
  - devicePort 实例上的地址属性可用于查找设备（即使多个设备的设备类型均为 `AUDIO_DEVICE_OUT_BUS`）。
- mixPorts。 包含由音频 HAL 提供的所有输出流和输入流的列表。每个 `mixPort` 实例都可被视为传输到 Android AudioService 的物理音频流。
- routes。 定义输入和输出设备之间或音频流和设备之间可能存在的连接的列表。

以下示例定义了输出设备 `bus0_phone_out`，其中所有 Android 音频流都通过 `mixer_bus0_phone_out` 完成混音。路由将 `mixer_bus0_phone_out` 的输出流传递到 `device bus0_phone_out`。

```
<audioPolicyConfiguration version="1.0" xmlns:xi="http://www.w3.org/2001/XInclude">
    <modules>
        <module name="primary" halVersion="3.0">
            <attachedDevices>
                <item>bus0_phone_out</item>
<defaultOutputDevice>bus0_phone_out</defaultOutputDevice>
            <mixPorts>
                <mixPort name="mixport_bus0_phone_out"
                         role="source"
                         flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000"
                            channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
            </mixPorts>
            <devicePorts>
                <devicePort tagName="bus0_phone_out"
                            role="sink"
                            type="AUDIO_DEVICE_OUT_BUS"
                            address="BUS00_PHONE">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000"
                            channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-8400"
                                maxValueMB="4000"
                                defaultValueMB="0"
                                stepValueMB="100"/>
                    </gains>
                </devicePort>
            </devicePorts>
            <routes>
                <route type="mix" sink="bus0_phone_out"
                       sources="mixport_bus0_phone_out"/>
            </routes>
        </module>
    </modules>
</audioPolicyConfiguration>
```

## 指定 devicePorts

车载平台应该为每个输入到 Android 和从 Android 输出的物理音频流指定 devicePort 实例。对于输出音频流，每个 devicePort 实例的类型都应该为 `AUDIO_DEVICE_OUT_BUS`，并采用整数（即总线 0、总线 1 等）编址。mixPort 实例与 devicePort 实例的数量之比应为 1:1，并应允许指定可以路由到每个总线的数据格式。

车载实现可以使用多个输入设备类型，包括 `FM_TUNER`（保留以用于广播无线装置输入）、MIC 设备（用于处理麦克风输入）和 `TYPE_AUX_LINE`（用于表示模拟线输入）。所有其他输入流都会分配到 `AUDIO_DEVICE_IN_BUS` 并在通过 `AudioManager.getDeviceList()` 调用枚举设备时被发现。各个来源可根据其 `AudioDeviceInfo.getProductName()` 进行区分。

您还可以将外部设备定义为端口，然后通过音频 HAL 的 `IDevice::createAudioPatch` 方法（通过新的 `CarAudioManager` 入口点提供）使用这些端口与外部硬件互动。

如果存在基于总线的音频驱动程序，则必须将 `audioUseDynamicRouting` 标记设置为 true：

```
<resources>
    <bool name="audioUseDynamicRouting">true</bool>
</resources>
```
如需了解详情，请参阅 `device/generic/car/emulator/audio/overlay/packages/services/Car/service/res/values/config.xml`。