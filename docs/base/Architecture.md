---
layout: post
title:  "架构"
parent: "基本"
date:   2019-12-29
nav_order: 1
---

# 架构

车载HAL是汽车与车辆网络服务之间的接口定义，下图是Android Automotive的架构：

![](/assets/images/vehicle_hal_arch.png)

整体上是在Android基本架构上进行了针对车载的扩展，如下是对相关类的解析：

Car API：包含 CarHvacManager 和 CarSensorManager 等 API。如需详细了解受支持的 API，请参阅`/platform/packages/services/Car/car-lib`。

CarService：位于`/platform/packages/services/Car/`。

VehicleNetworkService：通过内置安全机制控制车载 HAL。仅限访问系统组件（第三方应用等非系统组件需使用 Car API）。原始设备制造商 (OEM) 可以通过 `vns_policy.xml` 和 `vendor_vns_policy.xml` 控制访问权限。位于 `/platform/packages/services/Car/vehicle_network_service/`；要查看用于访问车辆网络的库，请参阅 `/platform/packages/services/Car/libvehiclenetwork/`。

车载 HAL：定义 OEM 可以实现的车辆属性的接口。包含属性元数据（例如，车辆属性是否为 int 以及允许使用哪些更改模式）。位于 `hardware/libhardware/include/hardware/vehicle.h`。要了解基本参考实现的相关信息，请参阅 `hardware/libhardware/modules/vehicle/`。

这个是整体的架构，下一节会详细介绍车载相关的属性定义。