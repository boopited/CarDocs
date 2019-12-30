---
layout: post
title:  "架构"
parent: "基本"
date:   2019-12-29
nav_order: 1
---

# 架构

车载HAL是汽车与Car Service之间的接口定义，下图是Android Automotive的架构：

![](/assets/images/vehicle_hal_arch.png)

整体上是在Android基本架构上进行了针对车载的扩展，如下是对相关类的解析：

Car API：包含 CarHvacManager 和 CarSensorManager 等 API。如需详细了解受支持的 API，请参阅`/platform/packages/services/Car/car-lib`。

CarService：位于`/platform/packages/services/Car/`。

Vehicle Network Service：在最新版本已经被移除了，相关的功能被拆分，放到CarService及相关支持库，Vehicle HAL直接与Car Service通信。相关修改：

- [Migrating Car service to new Vehicle HAL](https://android.googlesource.com/platform/packages/services/Car/+/0d07c76bbc788fba8c77d8e932330ab22ec6ba27)
- [Remove Vehicle Network Service](https://android.googlesource.com/platform/packages/services/Car/+/41aa2192460697dbe1650034d9271cb00c406e7e)
- [Move Vehicle HAL under automotive package](https://android.googlesource.com/platform/hardware/interfaces/+/2579fb792b4d47555515459f372f63c4305ee2ca)

车载 HAL：定义 OEM 可以实现的车辆属性的接口。包含属性元数据（例如，车辆属性是否为 int 以及允许使用哪些更改模式）。要了解基本参考实现的相关信息，请参阅``hardware/interfaces/automotive/vehicle/2.0`。 

这个是整体的架构，下一节会详细介绍车载相关的属性定义。