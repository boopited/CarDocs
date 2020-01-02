---
layout: post
title:  "连接"
parent: "专题"
date:   2020-01-02
nav_order: 1
---

# 连接

Android 8.0开始，把蓝牙设备连接到IVI有个更顺滑的体验。因为IVI可以监听一些事件，比如打开车门、发动车，这样可以自动扫描范围内的蓝牙设备。而且IVI可以同时连接多个不同设备，用户可以无障碍使用任意设备。

## 蓝牙连接管理

在 Android 7.x 及更低版本中，仅当蓝牙适配器已接通电源时，IVI 蓝牙堆栈才会搜索设备。当 IVI 蓝牙处于开启状态时，如果用户的设备进入检测范围，他们必须手动连接才能连接到其设备。使用 Android 8.0，用户将无需再通过 IVI 设置应用手动连接设备。

当蓝牙适配器处于开启状态且未连接到设备时，可自定义的触发事件（例如解锁车门或启动发动机）会导致 IVI 蓝牙搜索检测范围内的设备，以便与可用配置文件配对。然后，IVI 蓝牙会根据优先级列表来确定要连接到哪台设备。

除了触发事件，您还可以针对每个配置文件指定设备优先级。在默认实现中，IVI 蓝牙会优先考虑最近连接过的设备。对于每项蓝牙服务，都有一个单独的优先级列表，以便每个服务配置文件可以分别连接到不同的设备。此外，每个蓝牙服务配置文件都有不同的连接限制。

每种蓝牙profile有不同的连接限制：

| Service | Profile  | Connection limit |
| ------- | -------- | ---------------- |
| Phone   | HFP/PBAP | 2                |
| Message | MAP      | 1                |
| Audio   | A2DP     | 1                |

### 连接管理实现

Android 8.0为了实现自动连接，蓝牙连接的的逻辑从Bluetooth Service移到了CarService里的一个policy。Policy里定义了什么时候扫描新设备，以及先尝试连接哪个设备。

您可以在 [service/src/com/android/car/BluetoothDeviceConnectionPolicy.java](https://android.googlesource.com/platform/packages/services/Car/+/master/service/src/com/android/car/BluetoothDeviceConnectionPolicy.java) 查看默认连接策略。要使用自定义电话政策，请将 enable_phone_policy 设为 false，以停用 [res/values/config.xml](https://android.googlesource.com/platform/packages/apps/Bluetooth/+/master/res/values/config.xml) 中的默认电话政策。

要创建触发事件，请将监听器注册到相应的汽车服务。新传感器事件示例：
```
  /
  * Handles events coming in from the {@link CarSensorService}
  * Upon receiving the event that is of interest, initiate a connection attempt by
  * calling the policy {@link BluetoothDeviceConnectionPolicy}
  */
  private class CarSensorEventListener extends ICarSensorEventListener.Stub {
    @Override
    public void onSensorChanged(List<CarSensorEvent> events) throws RemoteException {
      if (events != null & !events.isEmpty()) {
        CarSensorEvent event = events.get(0);
        if (event.sensorType == CarSensorManager.SOME_NEW_SENSOR_EVENT) {
          initiateConnection();
        }
      ...
  }
```

要针对每个配置文件配置设备优先级，请在 BluetoothDeviceConnectionPolicy 中修改以下方法：

- updateDeviceConnectionStatus()
- findDeviceToConnect()

### 验证连接管理

通过在 Android Automotive IVI 上开启/关闭蓝牙来验证蓝牙连接管理功能的行为，看看它是否能够自动连接到适当的设备。

可以使用以下 adb 命令通过模拟 CarCabinManager.ID_DOOR_LOCK 事件测试您的 IVI：
```
adb shell dumpsys activity service com.android.car inject-vhal-event 0x16200b02 1
```

## 蓝牙多设备连接

在 Android 8.0 中，IVI 可以支持通过蓝牙同时连接多台设备。通过多设备蓝牙电话服务，用户可以连接单独的设备（例如个人电话和工作电话），并使用任一设备进行免提通话。该功能有助于减少干扰，提升驾驶安全性；而且可以充分利用汽车音频系统，提升通话体验。

### 多设备连接

当触发事件发生时，蓝牙连接管理政策会确定在每个蓝牙配置文件上连接哪些设备。然后，该政策会将连接 intent 发送到配置文件客户端服务。接下来，配置文件客户端服务会创建客户端状态机实例来管理与每台设备的连接。

### 免提模式配置文件 (HFP) 

Android 8.0 中的蓝牙免提模式配置文件允许两台设备并发连接，并且其中每台设备都可以拨打或接听电话。为了做到这一点，每个连接都会向电信管理器注册一个单独的电话帐号，而电信管理器会将这些电话帐号告知 IVI 应用。

当用户从设备拨打或接听电话时，相应的电话帐号会创建一个 HfpClientConnection 对象。拨号器应用会与 HfpClientConnection 对象进行交互以管理通话功能，例如接听电话或挂断。不过，拨号器应用不会指出来电来自哪台设备。在用户拨打电话时，应用将使用上次连接的设备。制造商可以通过自定义拨号器应用来更改此行为。

### 电话簿访问配置文件 (PBAP)

蓝牙电话簿访问配置文件会从所有已连接的设备下载联系人信息。PBAP 会维护一个可搜索的联系人信息汇总列表，该列表由 PBAP 客户端状态机进行更新。由于每台设备都是与单独的 PBAP 客户端状态机交互，因此在用户拨打电话时，联系人会与适当的设备相关联。当 PBAP 客户端断开连接时，内部数据库会移除与该电话关联的所有联系人信息。

### 实现多设备 HFP

借助默认的 Android 蓝牙堆栈，IVI 可以自动连接到 HFP 上的多台设备。 HeadsetClientService 中的 MAX_STATE_MACHINES_POSSIBLE 指定了可同时进行的 HFP 连接的最大数量。

> 注意：对于 HFP 同步连接导向型 (SCO) 连接和电话音频路由，需要在应用 Binder 中使用 connectAudio 手动完成。

实现多设备 HFP 时，应自定义拨号器应用界面，以便用户在拨打电话时选择使用哪个设备帐号。这样，应用便可以使用正确的帐号来调用 telecomManager.placeCall。

### 验证多设备 HFP

要通过蓝牙检查多设备连接是否能够正常工作，请执行以下操作：

1. 通过蓝牙将设备连接到 IVI 并从设备流式传输音频。
2. 通过蓝牙将两部手机连接到 IVI。
3. 选择一部手机。直接从该手机拨打电话，然后使用 IVI 拨打电话。
  - 在两次呼叫中，验证流式传输的音频是否会暂停，以及电话音频是否能够通过连接 IVI 的音响系统播放。
4. 直接在前面使用的手机上接听来电，然后使用 IVI 接听来电。
  - 在两次接听来电中，验证流式传输的音频是否会暂停，以及电话音频是否能够通过连接 IVI 的音响系统播放。
5. 使用连接的其他手机重复第 3 步和第 4 步。

> 注意：蓝牙 PBAP 配置文件（例如 MAP 配置文件）是单向的。要在这些配置文件上连接设备，需要从 IVI（而不是源设备）将连接实例化。