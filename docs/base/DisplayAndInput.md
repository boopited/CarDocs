---
layout: post
title:  "显示/输入"
parent: "基本"
date:   2019-12-30
nav_order: 4
---

# 显示/输入

从趋势看，以后汽车必然要支持多个屏幕，其中一些屏幕会使用Android来提供内容服务。本文是关于把仪表盘和其他显示设备集成到Automotive IVI系统。

## 1、显示

Android 10使用[android.app.Presentation](https://developer.android.google.cn/reference/android/app/Presentation.html)来支持外接设备。Presentation相当于一个特别的对话框，创建Presentation时关联一个Display(显示器)，根据显示器参数配置相关资源，然后就可以把内容显示到其他屏幕上。

### 仪表盘

![](/assets/images/display_overview_01.png)

典型场景下的仪表盘，使用Presentation的API就可以满足需要了。Presentation API有以下好处，它不需要：

- 单独的音频焦点。
- 运行整个活动或应用程序。
- 考虑并发用户输入。
- 处理触摸事件。

更多细节可以查看[官方文档](https://source.android.google.cn/devices/tech/display/multi_display)。

### 支持的内容类型

有些车可能不想用Android来直接绘制仪表盘，但仍想要导航信息、音乐名字等信息可以显示到仪表盘。Android可以通过几种方式吧这些数据发到仪表盘：

1. 基于元数据：通过接口(比如CAN)把元数据发给仪表盘的驱动系统，然后仪表盘再用元数据进行绘制。
2. 基于图形：基于物理或虚拟显示屏。这个显示屏可能安装在车辆仪表盘内部，或者是图形仪表盘显示屏的一部分。这样的仪表盘显示架构可能类似下图：

![](/assets/images/display_overview_02.png)

这个安全优先的系统和Android可能运行在一个多核SoC上（比如Cortex-R运行real-time OS， Cortex-A运行Android）。接口可能是Ethernet AVB (Audio Video Bridge)、LVDS或HDMI。图形仪表盘可以作为虚拟显示屏连接到Android系统，硬件实现隐藏在Display HAL后面。

### Presentation限制

对于后排的娱乐系统，Presentation API的限制如下：

- 不能投射整个Activity，Presentation是个对话框
- 只有一个音频焦点
- 不能有并发用户
- 外接显示器不支持直接触摸，需要独立的注入流

## 2、状态监控

典型的仪表盘可以在有新数据时更新驾驶、通话或者媒体信息，Android提供了相应的接口。

### 驾驶状态

导航过程中驾驶事件就会被发送，`packages/services/Car/car-lib/src/android/car/cluster/renderer/NavigationRenderer.java`包含仪表盘应用渲染逻辑可用的接口。应用可以扩展`InstrumentClusterRenderingService`来调用自己的NavigationRenderer，接收并显示期望的信息。

### 通话状态

1. 自定义InCallService扩展`android.telecom.InCallService`
2. 在`AndroidManifest.xml`注册这个服务
3. 覆写`onCallAdded`和`onCallRemoved`，转发通话状态变化
4. 注册回调给自定义InCallService，接收状态变化
5. 使用ContentProvider接口获取联系人信息

示例代码：`packages/services/Car/tests/InstrumentClusterRendererSample/src/com/android/car/cluster/sample/ClusterInCallService.java`

### 媒体状态

使用系统API监听媒体元数据和播放状态的变化：

1. 使用`MediaSessionManager`获取控制器`getActiveSessions(null)[0]`
2. 注册回调`MediaController.Callback`
3. 订阅活动回话的变化`MediaSessionManager.addOnActiveSessionsChangedListener()`

## 3、按键

从VHAL的定义可以看到，Automotive OS可以处理来自方向盘按键，物理按键和触摸板上的按键。

按键消息会通过CarInputService转到车载服务/应用中处理。

### KeyEvent数据

VHAL上传的每个按键事件包含：

- 按键action（up/down）
- 按键code（从vendor映射为标准Android code）
- 目标显示器（Android主显示器/仪表盘）

仪表盘可以通过`InstrumentClusterRenderingService`的`onKeyEvent`接收按键事件进行处理。