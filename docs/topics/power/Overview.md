---
layout: post
title:  "电源管理"
parent: "电源"
grand_parent: "专题"
date:   2020-01-10
nav_order: 0
---

# 电源管理

Android 在电源管理状态机中引入了一种新的状态，称为“深度睡眠”。为了实现深度睡眠，Android 提供了 `CarPowerManagementService` 服务和 `CarPowerManager` 接口。

状态转变由车载主控单元 (VMCU) 触发。为了能够与 VMCU 通信，集成器必须实现几个组件。集成器负责与车载硬件抽象层 (VHAL) 和内核实现相集成。集成器还负责停用唤醒源，并确保不会无限期地推迟关闭。

## 术语

本文档中使用了以下术语：

| 术语                             | 说明                                                         |
| :------------------------------- | :----------------------------------------------------------- |
| 应用处理器 (AP)                  | 系统芯片 (SoC) 的一部分。                                    |
| 板级支持处理器 (BSP)             | 产品正常运行所需的所有芯片和硬件特定代码。通常由 SoC 供应商和硬件制造商提供。它涵盖设备驱动程序、PMIC 序列代码和 SoC 启动等多项内容。 |
| CarPowerManager (CPM)            | 公开一个 API 以供应用记录电源状态的更改。                    |
| CarPowerManagementService (CPMS) | 实现汽车电源状态机、与 VHAL 连接，并执行对 `suspend()` 和 `shutdown()` 的最终调用。 |
| CarServiceHelperService (CSHS)   | 提供一个接入 SystemServer 的钩子以便正常运行，前提是该服务是汽车服务。 |
| 通用输入/输出 (GPIO)             | 用于一般用途的数字信号引脚。                                 |
| 休眠                             | 也称为“挂起到磁盘”(S2D/S4)。将 SoC 置于 S4 电源模式（休眠），将 RAM 内容写入非易失性介质（如闪存或磁盘），并且整个系统断电。Android 当前未实现休眠。 |
| 介质处理器 (MP)                  | 请参阅系统芯片 (SoC)。                                       |
| 电源管理集成电路 (PMIC)          | 芯片，用于管理主机系统的电源要求。                           |
| 系统芯片 (SoC)                   | 运行 Android 的主处理器，通常由 Intel、Mediatek、Nvidia、Qualcomm、Renesas 和 Texas Instruments 等制造商提供。 |
| 挂起                             | 也称为“挂起到 RAM”（S2R 或 STR）。将 SoC 置于 S3 电源模式，CPU 断电而 RAM 仍通电。 |
| 车载 HAL (VHAL)                  | 用于与车载网络连接的 Android API。第 1 级合作伙伴或原始设备制造商 (OEM) 负责编写此模块。车载网络可以使用任何物理层（如 CAN、LIN、MOST 和以太网）。VHAL 可以抽象化处理此车载网络，以使 Android 能够与车辆交互。 |
| 车载接口处理器 (VIP)             | 请参见“车载 MCU”。                                           |
| 车载主控单元 (VMCU)              | 微控制器，可提供车载网络与 SoC 之间的接口。SoC 通过 USB、UART、SPI 和 GPIO 信号与 VMCU 通信。 |

## 系统设计

本节介绍了 Android 如何表示应用处理器的电源状态以及哪些模块会实现电源管理系统。本材料还介绍了这些模块如何协同工作以及通常如何发生状态转换。

### 汽车电源状态机

Android 使用状态机表示 AP 的电源状态。状态机提供了五种状态，如下所示：

![](/assets/images/automotive_power_states.png)

**图 1.** 汽车电源状态机

此状态机的初始状态为 OFF。这种状态可以转换为两种状态，即 ON: DISP OFF 和 ON: FULL。这两种状态都指示 AP 已打开。不同之处在于显示屏的电源状态。ON: DISP OFF 表示 AP 正在运行且显示屏已关闭。当显示屏打开时，ON: DISP OFF 状态将转换为 ON: FULL（反之亦然）。

AP 在两种情况下会关闭。在这两种情况下，状态机会首先将状态更改为 SHUTDOWN PREPARE，再转换为 OFF 或 DEEP SLEEP：

- 断电
- 挂起到 RAM

当电源管理状态机进入 DEEP SLEEP 状态时，AP 将运行“挂起到 RAM”。例如，AP 将在 RAM 中挂起其状态（如寄存器存储的值）。唤醒 AP 时，所有状态都会恢复。

### 电源管理模块

电源管理系统由以下模块组成：

| **模块名称**              | **说明**                         |
| :------------------------ | :------------------------------- |
| CarPowerManager           | Java/C++ API。                   |
| CarPowerManagementService | 协调睡眠/挂起电源状态。          |
| 车载 HAL                  | 连接到 VMCU。                    |
| libsuspend                | 用于将设备置于挂起状态的原生库。 |
| 内核                      | 挂起到 RAM 实现。                |

深度睡眠功能（将 Android 挂起到 RAM）在内核中实现。此功能以位于 `/sys/power/state` 的特殊文件形式提供给用户空间。Android Auto 通过将 `mem` 写入此文件进行挂起。

`libsuspend` 是一个用于实现 `forcesuspend()` 的原生库。此函数使用 `/sys/power/state` 挂起 Android Auto。 您可以通过系统服务（包括 CPMS）调用 `forcesuspend()`。

CPMS 与其他服务和 HAL 协调电源状态。CPMS 实现上述状态机，并在发生电源状态转换时向每个观察者发送通知。此服务还使用 `libsuspend` 和 VHAL 向硬件发送消息。

某些属性是在 VHAL 中定义的。为了与 VMCU 通信，CPMS 将读取和写入这些属性。应用可以使用在 CPM 中定义的接口来监控电源状态的更改。应用还能通过此接口获取启动原因并发送关闭请求。此 API 可通过 Java 和 C++ 进行调用，并且带有 @hide/@System API 注解，这意味着它仅供特权应用使用。这五个模块、应用和服务之间的关系如下所示：

![](/assets/images/automotive_power_reference_diagram.png) 

**图 2.** 电源组件参考图

### 消息序列

上一节介绍了组成电源管理系统的模块。本节使用“进入深度睡眠”和“退出深度睡眠”示例来说明模块与应用之间的通信方式：

#### 进入深度睡眠

只有 VMCU 可以启动深度睡眠。启动深度睡眠后，VMCU 将通过 VHAL 向 CPMS 发送通知。CPMS 通过使用由 CPM 提供的新状态 ID 调用 `onStateChanged()` 方法，将状态更改为 SHUTDOWN PREPARE 并向所有观察者（监控 CPMS 的应用和服务）广播此状态转换。

CPM 在应用/服务与 CPMS 之间进行协调。应用/服务的 `onStateChanged()` 方法在 CPM 的 `onStateChanged()` 方法中进行同步调用。之后，调用 CPMS 的完成方法来通知 CPMS，告知它目标应用或服务已准备好挂起。CPM 的 `onStateChanged()` 方法异步运行。因此，在向所有 CPM 对象调用 `onStateChanged()` 方法与从所有这些对象接收完成消息之间会发生一些延迟。在此期间，CPMS 将一直向 VHAL 发送推迟关闭的请求。

CPMS 从所有 CPM 对象接收完成消息后，CPMS 将向 VHAL 发送 `AP_POWER_STATE_REPORT`，VHAL 随后通知 VMCU，告知它 AP 已准备好挂起。CPMS 还会调用其挂起方法，该方法将使用 `libsuspend` 提供的功能挂起内核。

下图演示了上述流程：

![](/assets/images/automotive_power_deep_sleep.png) 

**图 3.** 进入深度睡眠

#### 退出深度睡眠

退出挂起的序列也由 VMCU 启动。VMCU 将开启 AP，随后 AP 从 RAM 中恢复挂起的 Android。唤醒后，CPMS 将使用 `onStateChanged` 方法向应用和服务发送 `SUSPEND_EXIT` 消息。

要获知原因，应用和服务可以调用 CPM 提供的 `getBootReason()` 方法。此方法会将由 VMCU 通知的以下值返回到 VHAL：

- BOOT_REASON_USER_POWER_ON
- BOOT_REASON_DOOR_UNLOCK
- BOOT_REASON_TIMER
- BOOT_REASON_DOOR_OPEN
- BOOT_REASON_REMOTE_START

![](/assets/images/automotive_power_deep_sleep_exit.png)

**图 4.** 退出深度睡眠

## CPM 提供的编程接口

本节介绍了 CPM 为系统应用和服务提供的 Java 和 C++ API。在 C++ 中调用 CPM 的进程与 Java API 使用的进程完全相同。借助此 API，系统软件可以：

- 监控 AP 中的电源状态更改。
- 从 CPMS 获取启动原因。
- 请求 VMCU 以便在发出下一条挂起指令时将 AP 关闭。

`com.google.android.car.kitchensink.power` 中的 `PowerTestFragment.java` 说明了如何在 Java 中使用这些 API。您可以按照以下步骤调用 CPM 提供的 API：

1. 调用 Car API，以获取 CPM 实例。
2. 在第 1 步中创建的对象上调用适当的方法。

### 创建 CarPowerManager 对象

要创建 CPM 对象，请调用 Car 对象的 `getCarManager()` 方法。此方法是用于创建 CM 对象的外观模式。指定 `android.car.Car.POWER_SERVICE` 作为创建 CPM 对象的参数。

```
Car car = Car.createCar(this, carCallbacks);car.connect();CarPowerManager powerManager = (CarPowerManager)car.getCarManager(android.car.Car.POWER_SERVICE);
```

## CarPowerStateListener 与注册

系统应用和服务可以通过实现 `CarPowerManager.CarPowerStateListener` 来接收电源状态更改通知。此接口定义了一种方法 `onStateChanged()`，它是在 CPMS 的电源状态发生更改时调用的回调函数。以下示例定义了一个新的匿名类，用来实现此接口：

```
private final CarPowerManager.CarPowerStateListener listener =
  new CarPowerManager.CarPowerStateListener () {
    @Override
     public void onStateChanged(int state) {
       Log.i(TAG, "onStateChanged() state = " + state);
     }
};
```

要指示此监听器对象监控电源状态转换，请创建一个新的执行线程，并将监听器和该线程注册到 PM 对象：

```
executer = new ThreadPerTaskExecuter();
powerManager.setListener(powerListener, executer);
```

当电源状态发生更改时，系统将调用监听器对象的 `onStateChanged()` 方法，并使用一个值表示新的电源状态。`CarPowerManager.CarPowerStateListener` 中定义了实际值与电源状态之间的关系，如下表所示：

| **名称**          | **说明**                                   |
| :---------------- | :----------------------------------------- |
| SHUTDOWN_CANCELED | 已取消关闭，且电源状态已返回到正常状态。   |
| SHUTDOWN_ENTER    | 进入关闭状态。应用应该进行清理并准备关闭。 |
| SUSPEND_ENTER     | 进入挂起状态。应用应该进行清理并准备挂起。 |
| SUSPEND_EXIT      | 从挂起状态唤醒或从取消的挂起状态恢复。     |

### CarPowerStateListener 取消注册

要取消注册已注册到 CPM 的所有监听器对象，请调用 `clearListener` 方法：

```
powerManager.clearListener();
```

#### 启动原因获取

要获取启动原因，请调用 `getBootReason` 方法，该方法将与 CPMS 进行通信并返回以下五个启动原因之一：

| **名称**                  | **说明**                                 |
| :------------------------ | :--------------------------------------- |
| BOOT_REASON_USER_POWER_ON | 用户按下电源键或旋转点火开关以启动设备。 |
| BOOT_REASON_DOOR_UNLOCK   | 门解锁，从而引起设备启动。               |
| BOOT_REASON_TIMER         | 定时器到期，车辆唤醒 AP。                |
| BOOT_REASON_DOOR_OPEN     | 门打开，引起设备启动。                   |
| BOOT_REASON_REMOTE_START  | 用户激活远程启动。                       |

这些启动原因在 CPM 中进行定义。以下示例代码演示了启动原因获取：

```
try{
  int bootReason = carPowerManager.getBootReason();
  if (bootReason == CarPowerManager.BOOT_REASON_TIMER){
    doSomething;
  }else{
    doSomethingElse();
  }
}catch(CarNotConnectedException e){
   Log.e("Failed to getBootReason()" + e);
}
```

当无法与 CPMS 通信时，此方法会抛出 `CarNotConnectedException`。

#### 下次挂起时的关闭请求

`requestShutdownOnNextSuspend()` 方法指示 CPMS 在下次有机会时关闭而不是进入深度睡眠。

## Android 实现上的系统集成

集成器负责实现以下几项：

- 用于挂起 Android 的内核接口。
- VHAL，用途如下：
  - 将挂起或关闭的启动消息从汽车传播到 Android。
  - 将关闭就绪消息从 Android 发送到汽车。
  - 通过 Linux 内核接口启动 Android 的关闭或挂起操作。
  - 从挂起状态中恢复后，将唤醒原因从汽车传播到 Android。
- 当设备处于挂起状态时要停用的唤醒源。
- 足够快速关闭（以免无限期推迟关闭过程）的应用。

### 内核接口：/sys/power/state

首先，实现 Linux 挂起电源状态。当应用或服务将 `mem` 写入位于 `/sys/power/state.` 的文件时，Android 会将设备置于挂起模式。 此功能可能会向 VMCU 发送 GPIO 以通知 VMCU，告知它设备已完全关闭。集成器还负责消除 VHAL 向 VMCU 发送最终消息与系统进入挂起或关闭模式之间的任何竞争条件。

### VHAL 的责任

VHAL 可提供车载网络与 Android 之间的接口。VHAL 用途如下：

- 将挂起或关闭的启动消息从汽车传播到 Android。
- 将关闭就绪消息从 Android 发送到汽车。
- 通过 Linux 内核接口启动 Android 的关闭或挂起操作。
- 从挂起状态恢复后，将唤醒原因从汽车传播到 Android。

当 CPMS 通知 VHAL 它已准备好关闭时，VHAL 将关闭就绪消息发送到 VMCU。一般情况下，该消息由 UART、SPI 和 USB 等芯片外设进行传输。发送该消息后，VHAL 便会调用内核命令以将设备挂起或关闭。在执行此操作之前（如果是关闭的话），VHAL 或 BSP 可能会对 GPIO 进行相应的切换以指示 VMCU 完全可以断开设备的电源。

VHAL 必须支持以下属性，这些属性通过 VHAL 控制电源管理：

| **名称**               | **说明**                                                     |
| :--------------------- | :----------------------------------------------------------- |
| AP_POWER_STATE_REPORT  | Android 通过此属性向 VMCU 报告状态转换（使用 VehicleApPowerStateSet 枚举值）。 |
| AP_POWER_STATE_REQ     | VMCU 使用此属性指示 Android 转换为不同的电源状态（使用 VehicleApPowerStateReq 枚举值）。 |
| AP_POWER_BOOTUP_REASON | VMCU 向 Android 报告唤醒原因（使用 VehicleApPowerBootupReason 枚举值）。 |

#### AP_POWER_STATE_REPORT

使用此属性报告 Android 的当前电源管理状态。此属性包含两个整数：

- `int32Values[0]`：当前状态的 VehicleApPowerStateReport 枚举。
- `int32Values[1]`：要推迟或进入睡眠/关闭的时间（以毫秒为单位）。此值取决于第一个值。

第一个值可以采用以下值之一。`Types.hal` 包含更具体的说明，这些说明存储在 `hardware/interfaces/automotive/vehicle/2.0.` 中。

| **值名称**        | **说明**                                                    | **第二个值** |
| :---------------- | :---------------------------------------------------------- | :----------- |
| BOOT_COMPLETE     | AP 已完成启动，可以开始关闭。                               |              |
| DEEP_SLEEP_ENTRY  | AP 正在进入深度睡眠状态。                                   | 必须设置     |
| DEEP_SLEEP_EXIT   | AP 正在退出深度睡眠状态。                                   |              |
| SHUTDOWN_POSTPONE | Android 正在请求推迟关闭过程。                              | 必须设置     |
| SHUTDOWN_START    | AP 正在开始关闭。VMCU 可以在第二个值中指定的时间后开启 AP。 | 必须设置     |
| DISPLAY_OFF       | 用户已请求关闭音响主机的显示屏。                            |              |
| DISPLAY_ON        | 用户已请求开启音响主机的显示屏。                            |              |

状态可以进行异步设置（在 `BOOT_COMPLETE` 的情况下）或通过 VMCU 响应请求。当状态设置为 `SHUTDOWN_START`、`DEEP_SLEEP_ENTRY,` 或 `SHUTDOWN_POSTPONE` 时，系统会发送一个整数（以毫秒为单位）以通知 VMCU，告知它 AP 必须推迟关闭或进入睡眠状态的时长。

#### AP_POWER_STATE_REQ

此属性由 VMCU 发送，用于将 Android 转换为不同的电源状态，并包含两个整数：

- `int32Values[0]`：`VehicleApPowerStateReq` 枚举值，表示要转换为的新状态。
- `int32Values[1]`：`VehicleApPowerStateShutdownParam` 枚举值。仅针对 `SHUTDOWN_PREPARE` 消息发送此值，可将其包含的选项发送给 Android。

第一个整数值表示 Android 要转换到的目标新状态。相应的语义在 `types.hal` 中进行了定义，如下所示：

| **值名称**       | **说明**                                                    |
| :--------------- | :---------------------------------------------------------- |
| OFF              | AP 已关闭。                                                 |
| DEEP_SLEEP       | AP 处于深度睡眠状态。                                       |
| ON_DISP_OFF      | AP 已开启，但显示屏已关闭。                                 |
| ON_FULL          | AP 和显示屏均已开启。                                       |
| SHUTDOWN_START   | AP 正在开始关闭。VMCU 可以在第二个值中指定的时间后开启 AP。 |
| SHUTDOWN_PREPARE | VMCU 已请求将 AP 关闭。AP 可以进入睡眠状态或开始完全关闭。  |

`VehicleApPowerStateShutdownParam` 也在 `types.hal` 中进行了定义。此枚举具有三个元素，如下所述：

| **值名称**           | **说明**                            |
| :------------------- | :---------------------------------- |
| SHUTDOWN_IMMEDIATELY | AP 必须立即关闭。不得推迟。         |
| CAN_SLEEP            | AP 可以进入深度睡眠而不是彻底关闭。 |
| SHUTDOWN_ONLY        | 只有在允许推迟时，AP 才能关闭。     |

#### AP_POWER_BOOTUP_REASON

每当 Android 启动或从挂起状态中恢复时，VMCU 都会设置此属性。此属性可向 Android 指明触发唤醒的事件。在 Android 重新启动或完成挂起/唤醒周期之前，此值必须保持不变。此属性可以采用 `VehicleApPowerBootupReason` 值，该值在 `types.hal` 中进行了定义，如下所示：

| **值名称**    | **说明**                                                     |
| :------------ | :----------------------------------------------------------- |
| USER_POWER_ON | 因用户按下了电源键或旋转了点火开关而通电。                   |
| USER_UNLOCK   | 由用户解锁门（或其他任何类型的自动用户检测）而触发的自动通电。 |
| TIMER         | 由定时器触发的自动通电。仅当 AP 在特定持续时间之后请求唤醒时，才会发生这种情况（如 `VehicleApPowerSetState`#SHUTDOWN_START 中所指定）。 |

### 唤醒源

当设备处于挂起模式时，可使用集成器停用相应的唤醒源。常见的唤醒源包括检测信号、调制解调器、WLAN 和蓝牙。唯一有效的唤醒源必须是为了唤醒 SoC 而来自 VMCU 的中断。这就假设 VMCU 可以监听调制解调器以获取远程唤醒事件（如远程引擎启动）。如果将此功能推送到 AP，则必须再添加一个为调制解调器提供服务的唤醒源。在当前的设计中，集成器提供了一个文件，其中包含要关闭的唤醒源的列表。CPMS 将遍历此文件，并在挂起时管理唤醒源的关闭和打开。

### 应用

OEM 必须小心地编写应用，以便可以快速关闭应用，而不是无限期地推迟该过程。

## 附录

### 源代码树中的目录

| **内容**                                           | **目录**                                                     |
| :------------------------------------------------- | :----------------------------------------------------------- |
| 与 CarPowerManager 相关的代码。                    | `packages/services/Car/car-lib/src/android/car/hardware/power` |
| CarPowerManagementService 等等。                   | `packages/services/Car/service/src/com/android/car`          |
| 处理 VHAL 的服务，如 `VehicleHal` 和 `HAlClient`。 | `packages/services/Car/service/src/com/android/car/hal`      |
| VHAL 接口和属性定义。                              | `hardware/interfaces/automotive/vehicle/2.0`                 |
| 介绍 `CarPowerManager` 的示例应用                  | `packages/services/Car/tests/EmbeddedKitchenSinkApp/src/com/google/android/car/kitchensink` |
| `libsuspend` 位于此目录中。                        | `system/core/libsuspend`                                     |

### 类图

此类图显示了电源管理系统中的 Java 类和接口：

![](/assets/images/automotive_power_class_diagram.png)

**图 5.** 电源类图

### 对象关系

下图说明了哪些对象会引用其他对象。边缘表示源对象保持对目标对象的引用。例如，VehicleHAL 引用了 PropertyHalService 对象。

![](/assets/images/automotive_power_object_state.png)

**图 6.** 对象引用图