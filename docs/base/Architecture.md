---
layout: post
title:  "架构"
parent: "基本"
date:   2019-12-29
nav_order: 1
---

# 架构

## 1、功能

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

## 2、安全性

车载 HAL 支持 3 个级别的数据访问安全性：

- 仅限系统（由 vns_policy.xml 控制）
- 允许拥有权限的应用访问（通过汽车服务）
- 无需任何权限即可访问（通过汽车服务）

仅允许部分系统组件直接访问车辆属性，而车辆网络服务是把关程序。大多数应用需通过汽车服务的额外把关（例如，只有系统应用可以控制 HVAC，因为这需要仅授予系统应用的系统权限）。

## 3、验证

AOSP 包含开发过程中使用的以下测试资源：

- hardware/interfaces/automotive/vehicle/2.0/default/tests/：车载 HAL 测试。
- packages/services/Car/tests/carservice_test/：包含使用模拟车载 HAL 属性进行的汽车服务测试。每个属性的预期行为都会在测试中实现，这是了解预期行为的绝佳起点。
- hardware/interfaces/automotive/vehicle/2.0/default/：基本参考实现。