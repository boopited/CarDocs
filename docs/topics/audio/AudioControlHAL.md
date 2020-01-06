---
layout: post
title:  "AudioControl HAL"
parent: "音频"
grand_parent: "专题"
date:   2020-01-06
nav_order: 2
---

# 实现 AudioControl HAL

Android 9 弃用了之前各版车载 HAL 中的 `AUDIO_*` 属性，并改为使用包含显式函数调用和已输入参数列表的专用音频控制 HAL。

这个新的 HAL 提供了 `IAudioControl` 作为主接口对象，该对象负责提供出于配置和音量控制目的与车辆的音频引擎进行交互所需的入口点。系统可以包含该对象的一个实例，即 `CarAudioService` 启动时创建的实例。该对象是传统 Android 音频 HAL 的汽车扩展；在大多数实现中，发布音频 HAL 接口的进程还应发布 `IAudioControl interfaces`。

## 支持的接口

AudioControl HAL 支持以下接口：

- `getBusforContext`。在启动时针对每个上下文调用一次，以获取从 `ContextNumber` 到 `busAddress` 的映射。用法示例：
```
getBusForContext(ContextNumber contextNumber)
    generates (uint32_t busNumber);
```

使车辆能够针对每个上下文告诉框架将物理输出音频流路由到哪里。对于每个上下文，必须返回有效的总线编号（0 - 总线数-1）。如果遇到无法识别的 `contextNumber`，则应返回 -1。对于任何上下文，如果返回无效的 `busNumber`，都会路由到总线 0。

任何通过此机制与同一个 `busNumber` 相关联的并发声音都会先通过 Android `AudioFlinger` 进行混音，然后才会作为单个音频流提供给音频 HAL。这会取代车载 HAL 属性 `AUDIO_HW_VARIANT` 和 `AUDIO_ROUTING_POLICY`。

- `setBalanceTowardRight`。用于控制车载音响设备的右/左平衡设置。用法示例：
```
setBalanceTowardRight(float value);
```

将音响设备音量调向汽车右侧 (+) 或左侧 (-)。0.0 表示在中间，+1.0 表示完全调到右侧，-1.0 表示完全调到左侧，如果值未在 -1 到 1 之间，则是错误的。

- `setFadeTowardFront`。用于控制车载音响设备的前/后淡化设置。用法示例：
```
setFadeTowardFront(float value);
```

将音响设备音量调向汽车前面 (+) 或后面 (-)。0.0 表示在中间，+1.0 表示完全调到前面，-1.0 表示完全调到后面，如果值未在 -1 到 1 之间，则是错误的。

## 配置音量

Android Automotive 实现应使用硬件放大器（而非软件混音器）来控制音量。为避免产生副作用，请在 `device/generic/car/emulator/audio/overlay/frameworks/base/core/res/res/values/config.xml` 中将 `config_useFixedVolume` 标记设为 `true`（根据需要叠加）：
```
<resources>
    <!-- Car uses hardware amplifier for volume. -->
    <bool name="config_useFixedVolume">true</bool>
</resources>
```
`config_useFixedVolume` 标记未设置或设为 false 时，应用可以调用 `AudioManager.setStreamVolume()`，并在软件混音器中按音频流类型更改音量。用户可能不希望出现这种情况，因为这会对其他应用带来潜在影响，而且使用硬件放大器接收信号时，软件混音器中的音量衰减会导致信号中的可用有效位减少。

## 配置音量组

`CarAudioService` 使用 `packages/services/Car/service/res/xml/car_volume_group.xml` 中定义的音量组。您可以替换此文件，以便根据需要重新定义音量组。系统在运行时会按 XML 文件中的定义顺序来识别音量组。ID 介于 0 到 N-1 之间，其中 N 是音量组的数量。示例：

```
<volumeGroups xmlns:car="http://schemas.android.com/apk/res-auto">
    <group>
        <context car:context="music"/>
        <context car:context="call_ring"/>
        <context car:context="notification"/>
        <context car:context="system_sound"/>
    </group>
    <group>
        <context car:context="navigation"/>
        <context car:context="voice_command"/>
    </group>
    <group>
        <context car:context="call"/>
    </group>
    <group>
        <context car:context="alarm"/>
    </group>
</volumeGroups>
```
此配置中使用的属性在 `packages/services/Car/service/res/values/attrs.xml` 中定义。

## 处理音量键事件

Android 定义了一些用于控制音量的键码，包括 `KEYCODE_VOLUME_UP`、`KEYCODE_VOLUME_DOWN` 和 `KEYCODE_VOLUME_MUTE`。默认情况下，Android 会将音量键事件路由到应用。Automotive 实现应强制将这些键事件路由到 `CarAudioService`，以便该服务可以根据情况适当地调用 `setGroupVolume` 或 `setMasterMute`。

要强制实现此行为，请在 `device/generic/car/emulator/car/overlay/frameworks/base/core/res/res/values/config.xml` 中将 `config_handleVolumeKeysInWindowManager` 标记设为 `true`：
```
<resources>
    <bool name="config_handleVolumeKeysInWindowManager">true</bool>
</resources>
```

## CarAudioManager API

`CarAudioManager` 使用 `CarAudioService` 来配置和控制车载音频系统。该管理器对于系统中的大多数应用来说都是不可见的，但车辆专用组件（如音量控制器）可以使用 `CarAudioManager` API 与系统交互。

下文介绍了 Android 9 对 `CarAudioManager` API 所做的更改。

### 弃用的 API

Android 9 通过现有的 `AudioManager` `getDeviceList` API 处理设备枚举，因此弃用并移除了以下车辆专用函数：

- `String[] getSupportedExternalSourceTypes()`
- `String[] getSupportedRadioTypes()`

Android 9 使用 `AudioAttributes.AttributeUsage` 或基于音量组的入口点处理音量，因此移除了以下依赖于 `streamType` 的 API：

- `void setStreamVolume(int streamType, int index, int flags)`
- `int getStreamMaxVolume(int streamType)`
- `int getStreamMinVolume(int streamType)`
- `void setVolumeController(IVolumeController controller)`

### 新增的 API

Android 9 添加了以下用于控制放大器硬件的新 API（明确基于音量组）：

- `int getVolumeGroupIdForUsage(@AudioAttributes.AttributeUsage int usage)`
- `int getVolumeGroupCount()`
- `int getGroupVolume(int groupId)`
- `int getGroupMaxVolume(int groupId)`
- `int getGroupMinVolume(int groupId)`

Android 9 还提供了以下全新的系统 API 供系统 GUI 使用：

- `void setGroupVolume(int groupId, int index, int flags)`
- `void registerVolumeChangeObserver(@NonNull ContentObserver observer)`
- `void unregisterVolumeChangeObserver(@NonNull ContentObserver observer)`
- `void registerVolumeCallback(@NonNull IBinder binder)`
- `void unregisterVolumeCallback(@NonNull IBinder binder)`
- `void setFadeToFront(float value)`
- `void setBalanceToRight(float value)`

此外，Android 9 还添加了用于管理外部来源的新 API。这些 API 主要用于支持根据媒体类型将音频从外部来源路由到输出总线。借助这些 API，第三方应用还能够访问外部设备。

- `String[] getExternalSources()`：返回一个地址数组，这些地址用于识别系统中 `AUX_LINE`、`FM_TUNER`、`TV_TUNER` 和 `BUS_INPUT` 类型的可用音频端口。
- `CarPatchHandle createAudioPatch(String sourceAddress, int carUsage)`：将来源地址路由到与所提供的 `carUsage` 关联的输出 BUS。
- `int releaseAudioPatch(CarPatchHandle patch)`：移除所提供的音频通路。如果 `CarPatchHandle` 的创建者意外终止，则由 `AudioPolicyService::removeNotificationClient()` 自动处理。

## 创建音频通路

您可以在两个音频端口（混音端口或设备端口）之间创建音频通路。通常，从混音端口到设备端口的音频通路用于播放音频，而从设备端口到混音端口的音频通路则用于捕获音频。

例如，将音频样本直接从 `FM_TUNER` 来源路由到媒体接收器的音频通路会绕过软件混音器。因此，您必须使用硬件混音器为接收器将来自 Android 和 `FM_TUNER` 的音频样本进行混音。创建直接从 `FM_TUNER` 来源到媒体接收器的音频通路时：

- 音量控制会应用于媒体接收器，并且应该会影响 Android 和 `FM_TUNER` 音频。
- 用户应该能够通过简单的应用切换在 Android 和 `FM_TUNER` 音频之间进行切换（应该不需要必须选择显式媒体来源）。

Automotive 实现可能还需要在两个设备端口之间创建音频通路。为此，您必须先在 `audio_policy_configuration.xml` 中声明设备端口和可能的路由，并将混音端口与这些设备端口相关联。

### 配置示例

另请参阅 `device/generic/car/emulator/audio/audio_policy_configuration.xml`。
```
<audioPolicyConfiguration>
    <modules>
        <module name="primary" halVersion="3.0">
            <attachedDevices>
                <item>bus0_media_out</item>
                <item>bus1_audio_patch_test_in</item>
            </attachedDevices>
            <mixPorts>
                <mixPort name="mixport_bus0_media_out" role="source"
                        flags="AUDIO_OUTPUT_FLAG_PRIMARY">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000"
                            channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                </mixPort>
                <mixPort name="mixport_audio_patch_in" role="sink">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                           samplingRates="48000"
                           channelMasks="AUDIO_CHANNEL_IN_STEREO"/>
                </mixPort>
            </mixPorts>
            <devicePorts>
                <devicePort tagName="bus0_media_out" role="sink" type="AUDIO_DEVICE_OUT_BUS"
                        address="bus0_media_out">
                    <profile balance="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_OUT_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-8400" maxValueMB="4000" defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
                <devicePort tagName="bus1_audio_patch_test_in" type="AUDIO_DEVICE_IN_BUS" role="source"
                        address="bus1_audio_patch_test_in">
                    <profile name="" format="AUDIO_FORMAT_PCM_16_BIT"
                            samplingRates="48000" channelMasks="AUDIO_CHANNEL_IN_STEREO"/>
                    <gains>
                        <gain name="" mode="AUDIO_GAIN_MODE_JOINT"
                                minValueMB="-8400" maxValueMB="4000" defaultValueMB="0" stepValueMB="100"/>
                    </gains>
                </devicePort>
            </devicePorts>
            <routes>
                <route type="mix" sink="bus0_media_out" sources="mixport_bus0_media_out,bus1_audio_patch_test_in"/>
                <route type="mix" sink="mixport_audio_patch_in" sources="bus1_audio_patch_test_in"/>
            </routes>
        </module>
    </modules>
</audioPolicyConfiguration>
```

### 音频驱动程序 API

您可以使用 `getExternalSources()` 获取可用来源列表（按地址进行标识），然后按音频用法在这些来源和接收器端口之间创建音频通路。音频 HAL 上相应的入口点显示在 `IDevice.hal` 中：

```Java
Interface IDevice {
...
/
* Creates an audio patch between several source and sink ports.  The handle
* is allocated by the HAL and must be unique for this audio HAL module.
*
* @param sources patch sources.
* @param sinks patch sinks.
* @return retval operation completion status.
* @return patch created patch handle.
*/
createAudioPatch(vec<AudioPortConfig> sources, vec<AudioPortConfig> sinks)
       generates (Result retval, AudioPatchHandle patch);

/
* Release an audio patch.
*
* @param patch patch handle.
* @return retval operation completion status.
*/
releaseAudioPatch(AudioPatchHandle patch) generates (Result retval);
...
}
```

> **注意**：这些 API 钩子从 AUDIO_DEVICE_API_VERSION_3_0 便已开始提供。如需了解更多详情，请参阅** `device/generic/car/emulator/audio/driver/audio_hw.c`。

## 配置音量设置界面

Android 9 将音量设置界面从音量组配置（该配置可叠加，如“配置音量组”中所述）中分离出来。这种分离可确保将来音量组配置发生更改时，无需对音量设置界面进行任何更改。

在汽车设置界面中，`packages/apps/Car/Settings/res/xml/car_volume_items.xml` 文件包含与每个已定义的 `AudioAttributes.USAGE` 相关联的界面元素（标题和图标资源）。此文件通过使用与每个 VolumeGroup 中包含的首个识别出的用法相关联的资源，合理呈现已定义的 VolumeGroup。

例如，以下示例将 VolumeGroup 定义为同时包含 `voice_communication` 和 `voice_communication_signalling`。汽车设置界面的默认实现使用与 `voice_communication` 相关联的资源呈现 VolumeGroup，因为它是文件中的首个匹配项。

```
<carVolumeItems xmlns:car="http://schemas.android.com/apk/res-auto">
    <item car:usage="voice_communication"
          car:title="@*android:string/volume_call"
          car:icon="@*android:drawable/ic_audio_ring_notif"/>
    <item car:usage="voice_communication_signalling"
          car:title="@*android:string/volume_call"
          car:icon="@*android:drawable/ic_audio_ring_notif"/>
    <item car:usage="media"
          car:title="@*android:string/volume_music"
          car:icon="@*android:drawable/ic_audio_media"/>
    <item car:usage="game"
          car:title="@*android:string/volume_music"
          car:icon="@*android:drawable/ic_audio_media"/>
    <item car:usage="alarm"
          car:title="@*android:string/volume_alarm"
          car:icon="@*android:drawable/ic_audio_alarm"/>
    <item car:usage="assistance_navigation_guidance"
          car:title="@string/navi_volume_title"
          car:icon="@drawable/ic_audio_navi"/>
    <item car:usage="notification_ringtone"
          car:title="@*android:string/volume_ringtone"
          car:icon="@*android:drawable/ic_audio_ring_notif"/>
    <item car:usage="assistant"
          car:title="@*android:string/volume_unknown"
          car:icon="@*android:drawable/ic_audio_vol"/>
    <item car:usage="notification"
          car:title="@*android:string/volume_notification"
          car:icon="@*android:drawable/ic_audio_ring_notif"/>
    <item car:usage="notification_communication_request"
          car:title="@*android:string/volume_notification"
          car:icon="@*android:drawable/ic_audio_ring_notif"/>
    <item car:usage="notification_communication_instant"
          car:title="@*android:string/volume_notification"
          car:icon="@*android:drawable/ic_audio_ring_notif"/>
    <item car:usage="notification_communication_delayed"
          car:title="@*android:string/volume_notification"
          car:icon="@*android:drawable/ic_audio_ring_notif"/>
    <item car:usage="notification_event"
          car:title="@*android:string/volume_notification"
          car:icon="@*android:drawable/ic_audio_ring_notif"/>
    <item car:usage="assistance_accessibility"
          car:title="@*android:string/volume_notification"
          car:icon="@*android:drawable/ic_audio_ring_notif"/>
    <item car:usage="assistance_sonification"
          car:title="@*android:string/volume_unknown"
          car:icon="@*android:drawable/ic_audio_vol"/>
    <item car:usage="unknown"
          car:title="@*android:string/volume_unknown"
          car:icon="@*android:drawable/ic_audio_vol"/>
</carVolumeItems>
```

上述配置中使用的属性和值在 `packages/apps/Car/Settings/res/values/attrs.xml` 中声明。音量设置界面使用以下基于 `VolumeGroup` 的 `CarAudioManager` API：

- `getVolumeGroupCount()`：用于了解应绘制多少个控件。
- `getGroupMinVolume()` 和 `getGroupMaxVolume()`：用于获取音量上限和下限。
- `getGroupVolume()`：用于获取当前音量。
- `registerVolumeChangeObserver()`：用于获取音量更改通知。