---
layout: post
title:  "总览"
parent: "音频"
grand_parent: "专题"
date:   2020-01-03
nav_order: 0
---

# 车载音频

这一部分详细介绍了与汽车相关的 Android 实现采用的音频架构。实现汽车音频系统的原始设备制造商 (OEM) 和其他 Android 开发者除了查看主要音频部分的内容外，还应仔细查看本部分中的所有内容。

## 主要概念

Android 负责信息娱乐声音（即媒体、导航和通讯声音），但不直接负责具有严格可用性和时间要求的铃声和警告。外部声源由负责音频焦点的应用表示。不过，您不能依靠音频焦点来选择和混合声音。

对于与汽车相关的音频支持，Android 10 进行了以下更改：

- 音频 HAL 上下文映射到 `AudioAttributes.usage` 以识别声音；音频 HAL 实现负责进行特定于上下文的混音/路由。
- 车辆负责定义用于车载音频系统的通用输出设备 (`AUDIO_DEVICE_OUT_BUS`)；Android 支持一个上下文使用一个 `AUDIO_DEVICE_OUT_BUS` 实例。
- IAudioControl HAL 向音频 HAL 提供车辆专用扩展；有关示例实现，请参阅 `device/generic/car/emulator/audio`。Android 9 不包含 `AUDIO_* VHAL` 属性。

## Android 声音和声音流

汽车音频系统可以处理以下声音和声音流：

![](/assets/images/audio_streams_all.png)

> 以声音流为中心的架构图

Android 管理来自 Android 应用的声音，同时控制这些应用，并根据其声音类型将声音路由到 HAL 中的各个声音流：

- 逻辑声音流：在核心音频命名法中称为声源，使用 AudioAttributes 进行标记。
- 物理声音流：在核心音频命名法中称为设备，在混音后没有上下文信息。

为了确保可靠性，外部声音（来自独立声源，例如安全带警告铃声）在 Android 外部（HAL 下方，甚至是在单独的硬件中）进行管理。系统实现者必须提供一个混音器，用于接受来自 Android 的一个或多个声音输入流，然后以合适的方式将这些声音流与车辆所需的外部声源组合起来。外部声音流可始终处于开启状态，也可以通过 HAL 中的 createAudioPatch 入口点进行控制。

HAL 实现和外部混音器负责确保对保障安全至关重要的外部声音能够被用户听到，而且负责在 Android 提供的声音流中进行混音，并将混音结果路由到合适的音响设备。

### Android 声音

应用可以有一个或多个通过标准 Android API（如用于控制焦点的 AudioManager 或用于流式播放的 MediaPlayer）交互的播放器，以便发出一个或多个音频数据逻辑声音流。这些数据可能是单声道声音，也可能是 7.1 环绕声，但都会作为单个声源进行路由和处理。应用声音流与 AudioAttributes（可向系统提供有关应如何表达音频的提示）相关联。

逻辑声音流通过 AudioService 发送，并路由到一个（并且只有一个）可用的物理输出声音流，其中每个声音流都是混音器在 AudioFlinger 内的输出。音频属性在混合到物理声音流后将不再可用。

然后，每个物理声音流都会传输到音频 HAL，以在硬件上呈现。在汽车应用中，呈现硬件可能是本地编解码器（类似于移动设备），也可能是车辆物理网络中的远程处理器。无论是哪种情况，音频 HAL 实现都需要提供实际样本数据并使其能被用户听见。

### 外部声音流

如果声音流因认证或时间原因而不应经由 Android，则可以直接发送到外部混音器。在许多情况下，Android 都不需要知道这些声音的存在，因为外部混音器可以在 Android 声音之外混合它们。如果需要对某个声音进行闪避处理或需要将其路由到不同的音响设备，外部混音器可以通过 Android 不可见的方式进行此类操作。

如果外部声音流是应与 Android 正在生成的声音环境交互的媒体源（例如，当外部调谐器处于开启状态时，停止 MP3 播放），则这些外部声音流应由 Android 应用表示。此类应用将请求获得音频焦点，并根据需要通过启动/停止外部声音源来响应焦点通知，以符合 Android 声音焦点政策规定。要控制此类外部设备，一种建议使用的机制是 `AudioManager.createAudioPatch()`。

### 音频焦点

在启动逻辑声音流之前，应用应使用将用于其逻辑声音流的同一个音频属性来请求获得音频焦点。虽然我们建议发送此类焦点请求，但系统不会强制要求发送。有些应用可能会明确跳过发送请求的步骤，以实现特定行为（例如，在拨打电话时有意播放声音）。

为此，您应将焦点视为间接控制媒体播放和消除媒体播放冲突的方式，而不是作为主要的音频控制机制；也就是说，车辆不应依赖于焦点系统来操作音频子系统。焦点感知功能不是 HAL 的一部分，不得用于影响音频路由。

### 输出总线

在音频 HAL 级别，设备类型 AUDIO_DEVICE_OUT_BUS 提供用于车载音频系统的通用输出设备。总线设备支持可寻址端口（其中每个端口都是一个物理声音流的端点），并且应该是车辆内唯一受支持的输出设备类型。

系统实现可以针对所有 Android 声音使用一个总线端口，在这种情况下，Android 会将所有声音混合在一起，并将混音结果作为一个声音流进行传输。此外，HAL 可以分别为每个上下文提供一个总线端口，以允许并发传输任何声音类型。这样一来，HAL 实现就可以根据需要混合或闪避不同的声音。

将上下文分配到总线端口是通过音频控制 HAL 进行的，并会在上下文和总线端口之间创建多对一的关系。

## 麦克风输入

在捕获音频时，音频 HAL 会收到 `openInputStream` 调用，其中包含指示应如何处理麦克风输入的 AudioSource 参数。

`VOICE_RECOGNITION` 源（尤其是 Google 助理）需要一个符合以下条件的立体声麦克风流：具有回声消除效果（如果有），但不应用任何其他处理。波束成形应由 Google 助理自行完成。

### 多声道麦克风输入

要从具有两个以上声道（立体声）的设备捕获音频，请使用声道索引掩码，而不是定位索引掩码（例如 `CHANNEL_IN_LEFT`）。例如：
```
final AudioFormat audioFormat = new AudioFormat.Builder()
    .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
    .setSampleRate(44100)
    .setChannelIndexMask(0xf /* 4 channels, 0..3 */)
    .build();
final AudioRecord audioRecord = new AudioRecord.Builder()
    .setAudioFormat(audioFormat)
    .build();
audioRecord.setPreferredDevice(someAudioDeviceInfo);
```

如果 `setChannelMask` 和 `setChannelIndexMask` 均已设置，则 AudioRecord 仅使用由 `setChannelMask` 设置的值（最多两个声道）。

### 并发捕获

对于大多数输入音频设备类型，Android 框架都不允许执行并发捕获，但 `AUDIO_DEVICE_IN_BUS` 和 `AUDIO_DEVICE_IN_FM_TUNER` 例外，因为框架会将它们作为虚拟设备处理。这样做意味着，框架假定这些设备之间不存在资源竞争，因此允许在捕获某个常规输入设备（例如麦克风）的同时并发捕获任何/所有此类设备。如果这些设备之间存在对并发捕获的硬件限制，则此类限制必须由旨在使用这些输入设备的第一方应用中的自定义应用逻辑来处理。

旨在与 `AUDIO_DEVICE_IN_BUS` 设备或辅助 `AUDIO_DEVICE_IN_FM_TUNER` 设备结合使用的应用必须依赖于以下功能：明确识别这些设备，以及使用 `AudioRecord.setPreferredDevice()` 绕过 Android 默认声源选择逻辑。

### 音量和音量组

Android 8.x 及更低版本支持三个音量组（响铃、媒体和闹钟）以及一个用于手机通话的隐藏组。可以根据输出设备将每个组设置不同的音量，例如为音响设备设置较高音量，为耳机设置较低音量。

Android 9 及更高版本添加了一个语音音量组以及与汽车相关的上下文，如下所示。

| 音量组 | 音频上下文              | 说明             |
| ------ | ----------------------- | ---------------- |
| 响铃   | `CALL_RING_CONTEXT`     | 语音通话响铃     |
|        | `NOTIFICATION_CONTEXT`  | 通知             |
|        | `ALARM_CONTEXT`         | Android 闹钟铃声 |
|        | `SYSTEM_SOUND_CONTEXT`  | Android 系统声音 |
| 媒体   | `MUSIC_CONTEXT`         | 音乐播放         |
| 电话   | `CALL_CONTEXT`          | 语音通话         |
| 语音   | `NAVIGATION_CONTEXT`    | 导航指示         |
|        | `VOICE_COMMAND_CONTEXT` | 语音指令会话     |

音量组的值更新后，框架的 CarAudioService 会负责设置受影响的物理声音流增益。车辆中的物理声音流音量基于音量组（而不是 stream_type），每个音量组包含一个或多个音频上下文。`AudioAttributes.USAGE` 的每个实例都会映射到 CarAudioService 中的一个音频上下文，并且可以配置为路由到输出总线（请参阅[配置音量](https://source.android.google.cn/devices/automotive/audio/audio-control.html#configure-volume)和[配置音量组](https://source.android.google.cn/devices/automotive/audio/audio-control.html#configure-volume-groups)）。

Android 9 简化了对放大器中硬件音量的控制：

- 每个音量组都路由到一个或多个输出总线。可以使用汽车设置界面或通过外部生成的 `KEYCODE_VOLUME_DOWN`或 `KEYCODE_VOLUME_UP` 按键事件更改特定组的音量。
- 作为响应，CarAudioService 会调用 `AudioManager.setAudioPortGain()`，同时将音频设备端口绑定到目标音量组。在 HAL 中，这表现为一系列 `IDevice.setAudioPortConfig()` 调用（一次或多次调用），同时将每个物理输出声音流的音量增益值与目标音量组相关联。

您可以在 `audio_policy_configuration.xml` 中为每个音频设备端口配置最大值、最小值和步进增益值。如需示例配置以及关于如何覆盖默认音量组集合的详细信息，请参阅[配置音频设备](https://source.android.google.cn/devices/automotive/audio/audio-hal.html#configure-audio-devices)。