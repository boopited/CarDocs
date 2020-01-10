---
layout: post
title:  "停车模式"
parent: "电源"
grand_parent: "专题"
date:   2020-01-10
nav_order: 2
---

# 停车/车库模式

车载**Garage Mode**保持系统唤醒，提供周期性的空闲窗口，从而让JobScheduler中受控制的任务可以被执行。

## 什么是车库模式

在连接的设备（例如电话）上，用户依靠系统来确保设备稳定，最新，且经过优化。为了达到该状态，Android平台提供了[IDLE](https://developer.android.com/reference/android/app/job/JobInfo.Builder#setRequiresDeviceIdle(boolean))窗口，在此期间应用程序可以在以下情况下执行任务用户不与设备互动。如果用户长时间（60分钟或更长时间）不触摸手机，并且屏幕关闭，则该手机被视为“空闲”。与电话不同，当不使用汽车时，它会关闭，这意味着汽车没有[空闲时间](https://developer.android.com/reference/android/app/job/JobInfo.Builder#setRequiresDeviceIdle(boolean))窗口。车库模式可确保汽车空闲时间。

当用户关闭汽车时，系统进入车库模式。当汽车处于车库模式时，系统将打开电源，显示屏将关闭，并执行JobScheduler队列中的空闲作业。要实现车库模式，请参阅下面的*设备实施准则*。

##设备实施准则

要激活车库模式，在关闭车辆时，车辆HAL（VHAL）必须发送`AP_POWER_STATE_REQ`，状态为`SHUTDOWN_PREPARE`，且参数设置为`SHUTDOWN_ONLY`或`CAN_SLEEP`。

为了使状态`SHUTDOWN_PREPARE`生效，VHAL必须为`AP_POWER_STATE_REQ`命令指定两个参数（状态和附加参数）。这使设备能够进入车库模式，该模式可以在`JobScheduler`中检测计划的任务，并防止任务完成前系统暂停或关闭。

## 设备实现如何连接到Android framework

对于车库模式，framework要求VHAL延长关闭时间，直到超过了所需的持续时间或执行了所有任务为止，然后系统将关闭。 在CDD中定义的特定情况下，设备实现可以更快地关闭系统。 （有关Android兼容性要求的详细信息，请参阅Android兼容性定义文档CDD。）如果VHAL必须在车库模式完成之前关闭系统，则VHAL可以发出`SHUTDOWN_PREPARE`，并将参数设置为`SHUTDOWN_IMMEDIATELY`或`SLEEP_IMMEDIATELY`。 设备只能在特定情况下使用此功能，通常是在保持系统运行所需的资源不足时。 例如，当电池容量不足时。

![](/assets/images/garage_mode.png)

## 应用开发人员如何使用车库模式

应用和服务不直接通过车库模式交互，应用可以通过 [JobScheduler](https://developer.android.com/reference/android/app/job/JobScheduler)计划任务，然后在车库模式下执行。 这些任务受限于 [idleness](https://developer.android.com/reference/android/app/job/JobInfo.Builder#setRequiresDeviceIdle(boolean))。

以下代码展示了怎么计划一个任务：

```Java
public class MyGarageModeJob extends JobService { ... }

Context context = ...;
int jobId = ...;

ComponentName myGarageModeJobName = new componentName(context,MyGarageModeJob.class);

JobInfo.Builder infoBuilder = new JobInfo.Builder(jobId, myGarageModeJobName).setRequiresDeviceIdle(true);

// Example of an optional constraint:
infoBuilder.setRequiredNetworkType(NetworkType.NETWORK_TYPE_UNMETERED);

JobScheduler jobScheduler = (JobScheduler) context.getSystemService(Context.JOB_SCHEDULER_SERVICE);

jobScheduler.schedule(infoBuilder.build());
```