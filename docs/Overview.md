---
layout: post
title:  "总览"
parent: "基本"
date:   2019-12-27
nav_order: 0
---

# 总览

Google在2019年IO大会发布了针对汽车的Android Automotive OS，当时只看到了成品展示([Polestar 2](https://developer.polestar.com/))。Android Automotive OS适用于车载信息娱乐 (IVI) 系统，即中控系统。官方UI(虚拟机)如下：

![](/assets/images/automotive_launcher.png)

随着Android 10在AOSP上开源，Android Automotive OS的实现也揭开了面纱。从AOSP来看，Automotive OS是在Android原来的基础上添加了一种新的设备类型，为Automotive添加了相关的支持（hal/framework/apps等）。

多亏Android手机的蓬勃发展，把Android也逐步推向成熟。因为Android O添加了Treble，将驱动(HAL)标准化为接口（无需考虑物理传输层），硬件实现对上层软件透明，所以移植到汽车上也更方便。车载硬件的开发主要是HAL接口的实现。

借助各种总线拓扑，很多汽车子系统都可以实现互连以及与车载信息娱乐 (IVI) 系统的连接。不同的制造商提供的确切总线类型和协议之间有很大差异（甚至同一品牌的不同车型之间也是如此），例如控制器区域网络 (CAN) 总线、局域互联网络 (LIN) 总线、面向媒体的系统传输 (MOST) 总线以及汽车级以太网和 TCP/IP 网络（如 BroadR-Reach）。

系统集成商可以将特定于功能的平台 HAL 接口（如 HVAC）与特定于技术的网络接口（如 CAN 总线）连接，以实现车载 HAL 模块。典型的实现可能包括运行专有实时操作系统 (RTOS) 的专用微控制器单元 (MCU)，以用于 CAN 总线访问或类似操作，该微控制器单元可通过串行链路连接到运行 Android Automotive 的 CPU。除了专用的 MCU，还可以将总线访问作为虚拟 CPU 来实现。只要实现符合车载 HAL 的接口要求，每个合作伙伴都可以选择适合硬件的架构。

这个系列的内容关注点在Android Automotive针对车机的设计，不关注HAL的实现。