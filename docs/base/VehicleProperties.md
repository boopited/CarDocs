---
layout: post
title:  "属性"
parent: "基本"
date:   2019-12-29
nav_order: 2
---

# Vehicle Properties

车载 HAL定义 OEM 可以实现的车辆属性的接口。包含属性元数据（例如，车辆属性的类型(int, float...) 以及允许使用哪些更改模式）。VHAL接口基于每个属性的访问方式定义了一组抽象方法，如读、写、订阅等。

## 属性接口

VHAL有如下接口：

- `vehicle_prop_config_t const *(*list_properties)(..., int* num_properties)`：列出VHAL支持的所有属性
- `(*get)(..., vehicle_prop_value_t *data)`：获取属性的值
- `(*set)(..., const vehicle_prop_value_t *data)`：修改属性的值
- `(*subscribe)(..., int32_t prop, float sample_rate, int32_t zones)`：订阅一个属性值的变化，`zones`稍后解释
- `(*release_memory_from_get)(struct vehicle_hw_device* device, vehicle_prop_value_t *data)`：释放get调用申请的内存

其中用到的回调：

- `(*vehicle_event_callback_fn)(const vehicle_prop_value_t *event_data)`：订阅的属性发生变化时回调
- `(*vehicle_error_callback_fn)(int32_t error_code, int32_t property, int32_t operation)`：返回VHAL全局或者某个属性的错误。全局错误可能导致HAL重启，这可能导致其他模块重启(比如应用)。

## 属性定义

属性可以是只读、只写，也可以是允许读写，一般可写的属性用于向VHAL传递信息。每个属性都有一个32位int类型的ID，其中有预定义的值类型。

### 值类型

支持的值类型有：

- BYTES
- BOOLEAN
- FLOAT
- FLOAT[]
- INT32
- INT32[]
- INT64
- INT64[]
- STRING

### 区域类型

有一类属性属于分区属性，如座椅角度，因为座椅区分前排左/右，后排左/右等，所以需要区分不同的座椅，也意味着每个属性会有多个值。

VHAL定义的区域类型：

- GLOBAL：通用类型，无分区
- WINDOW：基于窗户分区，枚举类型`VehicleAreaWindow`
- MIRROR：基于镜子分区，枚举类型`VehicleAreaMirror`
- SEAT：基于座椅分区，枚举类型`VehicleAreaSeat`
- DOOR：基于车门分区，枚举类型`VehicleAreaDoor`
- WHEEL：基于车轮分区，枚举类型`VehicleAreaWheel`

每个属性都有预定义的区域类型，每个区域类型有一组不同bit表示的flag值。比如VehicleAreaSeat：

- ROW_1_LEFT = 0x0001
- ROW_1_CENTER = 0x0002
- ROW_1_RIGHT = 0x0004
- ROW_2_LEFT = 0x0010
- ROW_2_CENTER = 0x0020
- ROW_2_RIGHT = 0x0040
- ROW_3_LEFT = 0x0100
- ...

这类属性读、写或者订阅需要使用分区ID进一步区分。

### 属性状态

每个属性都有一个状态值，类型VehiclePropertyStatus，可能的值有：

- AVAILABLE：属性可用且有效
- UNAVAILABLE：属性暂时不可见，用于表示一个支持的属性暂时被禁用。
- ERROR：属性发生错误

不支持的属性，不应该设置为不可见。

### 配置属性

每个属性都有一个配置`vehicle_prop_config_t`，包括以下信息：

- access：访问权限（r/w/rw）
- changeMode：变化模式（离散还是持续）
- areaConfigs：区域设置（区域ID、最小值、最大值）
- configArray：附加配置数组
- configString：附加配置字符
- maxSampleRate/minSampleRate：最大/最小采样率
- prop：属性ID

## 属性使用

### 调用流程

以空调温度为例，`get`和`set`流程分别如下：
> CS = CarService, VNS = VehicleNetworkService, VHAL = Vehicle HAL

![](/assets/images/vehicle_hvac_get.png)

![](/assets/images/vehicle_hvac_set.png)

### 分区属性

分区属性相当于一组可以通过分区ID访问的子属性集合。

- `get`传入分区ID就可以获取对应的分区属性值，如果是非分区属性，分区ID为0
- `set`必须传入分区ID，所以只能修改对应分区的属性值
- `subscribe`会收到对应属性所有分区的事件

### 自定义属性

为支撑合作方需要，VHAL允许系统应用使用自定义属性，要求如下：

- 属性ID如下配置：
  - VehiclePropertyGroup使用VENDOR
  - VehicleArea按需设置
  - VehiclePropertyType按需设置
- 通过CarPropertyManager (Java) 或Vehicle Network Service API (native)访问属性

车载地图服务(Vehicle Map Service)通过发布/订阅机制，共享地图数据给客户端，这个是给Android Automotive用的，AOSP中没有相关例子。但VHAL 2.0中定义了VmsMessageType，其中有VMS支持V消息类型