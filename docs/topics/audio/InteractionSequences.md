---
layout: post
title:  "交互序列"
parent: "音频"
grand_parent: "专题"
date:   2020-01-10
nav_order: 3
---

# 交互序列

在下列汽车音响示例中，汽车音响主机运行 Android 9，其中安装有收音机应用和导航应用。此外，车辆调谐器在外部布线，并通过扬声器播放。在实际使用场景中，采用如下做法可能会大有好处：将调谐器处理为 Android 的输入，让收音机应用从调谐器中读取数据并将其写入 AudioTrack 对象。

## 用户启动收音机

在此交互序列中，当用户对收音机应用中的某个预设频率按**播放**时，车辆内不播放任何媒体内容。收音机应用必须获得焦点，然后调谐器才能通过扬声器播放声音。

![](/assets/images/audio_auto_focus_radio.png)

**图 1.** 收音机获得焦点，然后调谐器通过扬声器播放声音

1. 收音机：“调到 FM 96.5。”
2. 收音机：请求焦点 GAIN。
3. AudioManager：授予 GAIN。
4. 收音机：`createAudioPatch()`
5. 收音机：“播放调谐器输出。”
6. 外部布线的调谐器：混音器使调谐器音频路由到放大器。

## 收音机闪避导航提示

在此交互序列中，当导航应用为下一个转弯通知生成导航提示时，收音机正在播放内容。导航应用必须先从 `AudioManager` 获得瞬时焦点，然后才能播放导航提示。

![](/assets/images/audio_auto_radio_ducks.png)

**图 2.** 收音机播放闪避导航提示

5. 收音机：“播放调谐器输出。”
6. 外部布线的调谐器：混音器使调谐器音频路由到放大器。
7. 导航：从 `AudioManager` 请求焦点 GAIN TRANSIENT。
8. AudioManager：向导航提供 GAIN TRANSIENT。
9. 导航：打开数据流，发送数据包。
  a. 导航：在 bus1 上路由上下文 GUIDANCE。
  b. 混音器：闪避调谐器，以通过扬声器播放 bus1 GUIDANCE。
10. 导航：通知结束，关闭数据流。
11. 导航：放弃焦点。

`AudioManager` 认为收音机播放可以闪避，并且通常会在不通知收音机应用的情况下对音乐流应用一个闪避因子。不过，通过叠加 `framework/base/core/res/res/values/config.xml` 并将 `config_applyInternalDucking` 设置为 `false`，框架闪避被绕过了，因此，外部调谐器会继续提供声音，并且收音机应用未发现任何变化。混音器（位于 HAL 下游）负责合并这两个输入，并且可以选择是闪避收音机播放，还是将收音机播放移至后置扬声器。

导航提示播放完毕后，导航应用将释放焦点，收音机播放将恢复。

## 用户启动有声读物应用

在此交互序列中，用户启动有声读物应用，导致收音机播放停止（按流式传输音乐应用中的“播放”是类似的触发器）。

![](/assets/images/audio_auto_focus_book.png)

**图 3**. 有声读物从收音机播放处夺取焦点

12. 有声读物：从 AudioManager 请求 GAIN 上下文 MEDIA。
13. 收音机失去焦点：
  a. AudioManager：LOSS。
  b. 收音机：releaseAudioPatch()
14. 有声读物获得焦点：
  a. 授予 GAIN，在 bus0 上路由上下文 MEDIA
  b. 打开数据流，发送 MEDIA 数据包。

有声读物应用发起的焦点请求不是瞬时的，因此上一个焦点持有者（收音机应用）将收到永久性焦点丢失信号；收音机应用会拆除连接到调谐器的补丁程序，以此作为响应。混音器停止接收调谐器信号，并开始处理通过音频 HAL 传输的音频（它还可以选择性地在从收音机过渡到有声读物的过程中执行交错淡出）。

## 导航提示获得焦点

在此交互序列中，当导航应用生成导航提示时，系统正在播放有声读物。

![](/assets/images/audio_auto_focus_nav.png)

**图 4.** 导航提示从有声读物处夺取焦点

15. 有声读物：正在流式播放 MEDIA 数据包，焦点未并发。
16. 导航：请求 GAIN TRANSIENT。
17. AudioManager：LOSS TRANSIENT。
18. 有声读物：停止。
19. AudioManager：授予 GAIN TRANSIENT。
20. 导航：打开数据流，发送数据包。
  a. 导航：在 bus1 上路由上下文 GUIDANCE。
  b. 混音器：播放 bus1 (GUIDANCE)。
21. 导航：通知结束，关闭数据流。
22. 导航：放弃焦点。
23. 有声读物：GAIN。
24. 有声读物：重新启动。

由于有声读物应用的原始 `AudioFocusRequest`（在启动 `AudioTrack` 时发送）包含 `AUDIOFOCUS_FLAG_PAUSES_ON_DUCKABLE_LOSS` 标记，因此 `AudioManager` 发现无法对有声读物应用进行闪避处理。取而代之的是，`AudioManager` 会向有声读物应用发送一条 `AUDIOFOCUS_LOSS_TRANSIENT` 消息，有声读物应用应该通过暂停播放来进行响应。

导航应用现在可以不间断地播放导航提示。导航提示播放完毕后，有声读物将重新获得焦点并恢复播放。