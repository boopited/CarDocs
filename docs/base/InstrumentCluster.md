---
layout: post
title:  "仪表盘"
parent: "基本"
date:   2019-12-31
nav_order: 5
---

# 仪表盘

Android的Instrument Cluster API可以用来在外接屏上显示导航(Google Maps...)，比如在方向盘的仪表板上安装的屏。本文说明如何创建一个服务控制这样的外接屏，以及把它集成到CarService用来显示导航信息。

## 0、术语

| 术语                              | 说明                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| CarInstrumentClusterManager       | 一种CarManager，允许外部应用在仪表盘启动一个Activity，当仪表盘准备显示时应用会收到回调 |
| CarManager                        | 所有基于CarService实现的车载服务，封装给外部应用使用的Manager类的基类 |
| CarService                        | Android平台提供给外部应用，来跟车载特性通信的服务            |
| ETA                               | 预估到达(目的地)时间                                         |
| Head Unit (HU)                    | 嵌入汽车的主要计算单元，上面安装完整的Android，并且连接到汽车中控屏幕 |
| Instrument Cluster                | 汽车仪表盘上面安装的外接(相对于中控屏)显示屏。可以是通过CAN Bus或其他方式连接到车内网络的独立计算单元，也可以是HU上连接的一个显示屏 |
| InstrumentClusterRenderingService | 用来与仪表屏交互的服务基类，OEM要基于硬件提供一个该类的实现  |
| KitchenSink app                   | Android Automotive上的测试应用                               |
| Singleton service                 | 单例服务，添加`android:singleUser`属性，任何时候Android只有一个服务实例运行 |

## 1、预置条件

- Android开发环境
- AOSP源码`pi-car-release`或者更新的分支
- HU：一台运行Android 9+的设备，带显示器，而且可以刷入新系统
- 仪表屏任选其一
  - 连接到HU上的物理屏，要求HU支持外接屏
  - 独立带屏幕的设备，通过网络连接到HU，可以接收和显示视频流
  - 模拟显示屏，开发工具：
    - 模拟外接屏：通过Android系统的 设置 - 开发选项，可以打开，缺点是会覆盖在主屏上
    - 模拟仪表盘：Android Automotive的模拟器有一个[选项](https://android.googlesource.com/platform/external/qemu/+/master/docs/ANDROID-QEMU-PIPE.TXT)可以显示仪表盘。

## 2、集成架构

### 模块

- CarService
- 导航APP
- OEM Instrument Cluster Service

![](/assets/images/cluster_components.png)

#### CarService

CarService在导航应用和车之间协调，确保特定时间只有一个导航应用(需要权限`android.car.permission.CAR_INSTRUMENT_CLUSTER_CONTROL`)可以发送数据给车。

CarService引导所有车相关的特性服务，并提供一系列的管理器，可以被用来访问这些服务。应用只能通过管理器访问服务。

为支持仪表屏功能，OEM必须实现InstrumentClusterRendererService，然后修改[config.xml](https://android.googlesource.com/platform/packages/services/Car/+/master/service/res/values/config.xml)指定为新的实现。CarService在系统启动时会读取配置。AOSP中的配置如下：
```
<string name="instrumentClusterRendererService">
    android.car.cluster/.ClusterRenderingService
</string>
```

### Instrument Cluster Service

OEM添加一个APP，自行实现InstrumentClusterRendererService，[示例ClusterRenderingService](https://android.googlesource.com/platform/packages/apps/Car/Cluster/+/master/src/android/car/cluster/ClusterRenderingService.java)

这个类有两个功能：

- 在Android系统和仪表盘渲染设备之间提供接口
- 接收和渲染导航状态更新，比如转弯信息等

第一个功能，OEM实现的InstrumentClusterRendererService必须初始化用来显示信息的外接屏对应的Display，然后调用`InstrumentClusterRendererService.setClusterActivityOptions()`和`InstrumentClusterRendererService.setClusterActivityState()`来通知CarService。

第二个功能，Instrument Cluster Service必须提供一个`NavigationRenderer`接口的实现，用来接收状态更新事件。

### 流程

下图说明了一个导航状态更新的流程：

![](/assets/images/cluster_sequence.png)

- 黄色：Android平台的CarService和CarNavigationStatusManager
- 青色：OEM实现的InstrumentClusterRendererService
- 紫色：第三方导航应用，如Google Maps
- 绿色：Android平台的CarAppFocusManager

流程说明：

1. CarService初始化InstrumentClusterRenderingService
2. 初始化时，InstrumentClusterRenderingService向CarService更新：
  - 仪表盘的display属性, 比如清晰的边界
  - 在仪表盘的display打开Activity用的ActivityOptions.
3. 导航应用(如Android Automotive平台的Google Maps或其他地图应用)：
  - 使用car-lib中的Car类获取一个CarAppFocusManager
  - 在开始显示细节的导航信息时，调用`CarAppFocusManager.requestFocus()`并传入appType参数`CarAppFocusManager.APP_FOCUS_TYPE_NAVIGATION`
4. CarAppFocusManager将请求发到CarService，如果有权限CarService会扫描导航应用，并找到category为`android.car.cluster.NAVIGATION`的Activity
5. 如果找到导航应用使用InstrumentClusterRenderingService声明的ActivityOptions打开Activity，并把仪表盘的display属性作为extras放到Intent

## 3、集成API

InstrumentClusterRenderingService实现必须：

- 声明为单例服务，在AndroidManifest.xml中添加`android:singleUser="true"`，从而保证在初始化甚至用户切换时都只有一个示例运行
- 持有`BIND_INSTRUMENT_CLUSTER_RENDERER_SERVICE`系统权限，保证只有Android system镜像中的仪表盘渲染服务才会被CarService启动。`<uses-permission android:name="android.car.permission.BIND_INSTRUMENT_CLUSTER_RENDERER_SERVICE"/>`

## 4、实现InstrumentClusterRenderingService

1. 创建一个类扩展InstrumentClusterRenderingService，并在AndroidManifest.xml中声明。这个类是用来控制仪表盘display，也可以用来渲染导航状态数据
2. 在onCreate中，使用此服务初始化与渲染硬件的通信连接，可以包含：
  - 确定仪表盘使用的外接屏
  - 创建一个虚拟Display，让仪表盘应用来渲染和发送渲染好的图像给外接设备
3. 上面ready后，此服务要调用`InstrumentClusterRendererService.setClusterActivityOptions()`来定义用来在仪表屏显示Activity的ActivityOptions，参数：
  - category：`CarInstrumentClusterManager.CATEGORY_NAVIGATION`
  - ActivityOptions：例如AOSP的示例如下
    ```
    getService().setClusterActivityLaunchOptions(
      CATEGORY_NAVIGATION,
      ActivityOptions.makeBasic().setLaunchDisplayId(displayId));
    ```
  - 当仪表屏准备好显示内容，此服务调用`InstrumentClusterRendererService.setClusterActivityState()`，参数：
    - category：`CarInstrumentClusterManager.CATEGORY_NAVIGATION`
    - state：使用ClusterActivityState生成的Bundle，需要提供以下数据：
      - visible：表示仪表屏可以显示内容了
      - unobscuredBounds：仪表屏上可以显示内容的一个矩形区域
  - 覆写`Service.dump()`以便调试时获取有用的状态信息

### InstrumentClusterRenderingService示例

这个示例是创建了VirtualDisplay，把仪表屏内容发到远程物理显示器上。也可以把传入的displayId换成连接到HU的外接显示器。

```Java
/**
* Sample {@link InstrumentClusterRenderingService} implementation
*/
public class SampleClusterServiceImpl extends InstrumentClusterRenderingService {
   // Used to retrieve or create displays
   private final DisplayManager mDisplayManager;
   // Unique identifier for the display that will be used for instrument
   // cluster
   private final String mUniqueId = UUID.randomUUID().toString();
   // Format of the instrument cluster display
   private static final int DISPLAY_WIDTH = 1280;
   private static final int DISPLAY_HEIGHT = 720;
   private static final int DISPLAY_DPI = 320;
   // Area not covered by instruments
   private static final int DISPLAY_UNOBSCURED_LEFT = 40;
   private static final int DISPLAY_UNOBSCURED_TOP = 0;
   private static final int DISPLAY_UNOBSCURED_RIGHT = 1200;
   private static final int DISPLAY_UNOBSCURED_BOTTOM = 680;
   @Override
   public void onCreate() {
      super.onCreate();
      // Create a virtual display to render instrument cluster activities on
      mDisplayManager = getSystemService(DisplayManager.class);
      VirtualDisplay display = mDisplayManager.createVirtualDisplay(
          mUniqueId, DISPLAY_WIDTH, DISPLAY_HEIGHT, DISPLAY_DPI, null,
          0 /* flags */, null, null);
      // Do any additional initialization (e.g.: start a video stream
      // based on this virtual display to present activities on a remote
      // display).
      onDisplayReady(display.getDisplay());
}
private void onDisplayReady(Display display) {
    // Report activity options that should be used to launch activities on
    // the instrument cluster.
    String category = CarInstrumentClusterManager.CATEGORY_NAVIGATION;
    ActionOptions options = ActivityOptions.makeBasic()
        .setLaunchDisplayId(display.getDisplayId());
    setClusterActivityOptions(category, options);
    // Report instrument cluster state.
    Rect unobscuredBounds = new Rect(DISPLAY_UNOBSCURED_LEFT,
        DISPLAY_UNOBSCURED_TOP, DISPLAY_UNOBSCURED_RIGHT,
        DISPLAY_UNOBSCURED_BOTTOM);
    boolean visible = true;
    ClusterActivityState state = ClusterActivityState.create(visible,
       unobscuredBounds);
    setClusterActivityState(category, options);
  }
}
```