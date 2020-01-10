---
layout: post
title:  "管理启动时间"
parent: "电源"
grand_parent: "专题"
date:   2020-01-10
nav_order: 1
---

# 管理启动时间

气动管城是一系列动作，从Boot ROM开始，依次是BootLoader、Kernel、**Init**、**Zygote**和**System Server**（加粗的是Android特有的）。在车载的启动流程里，早期服务，如后视摄像头必须在kernel启动时被启动。

| 顺序 | Component     | Android                                                  | Android Automotive                                           |
| :--: | :------------ | :------------------------------------------------------- | :----------------------------------------------------------- |
|  1   | Boot ROM      | 把boot loader第一阶段加载到内部RAM                       |                                                              |
|  2   | Bootloader    | 初始化内存，安全验证，加载kernel                         |                                                              |
|  3   | Kernel        | 启动中断控制器、存储保护、缓存和调度，启动用户空间进程   | **Rear View Camera (RVC)**进程在kernel boot早期启动。这个进程启动后，VMCU的GPIO触发RVC在屏幕上显示摄像头内容 |
|  4   | Init Process  | 解析`init.rc` 脚本，挂在文件系统，启动zygote和系统进程。 | **Vehicle Network Service (VNS)** 在init阶段作为核心服务一部分被启动。(最新版本已移除) |
|  5   | Zygote        | 启动Java runtime，初始化Android对象使用的内存            |                                                              |
|  6   | System Server | 系统中第一个Java组件， 启动核心Android服务               | 所有系统服务启动后，启动**CarService**                       |

## 优化启动事件

根据以下原则优化系统启动时间：

- **Kernel**. 只加载要使用的模块和硬件组件
- **init.rc**
  - 小心使用块操作 (服务：相对于命令调用)
  - 只启动要用的
  - 正确设置服务优先级
- **Zygote**. Class预加载优化 (定义好要加载的类列表).
- **Package Manager**
  - 优化product镜像，只放入必须的APKs
  - [打开DEX预优化](https://source.android.com/devices/tech/dalvik/configure.html#compilation_options)
- **System Server**. 只启动要用的系统服务

Google提供了下面的工具帮助优化：

- `packages/services/Car/tools/bootanalyze/bootanalyze.py`可以用来分析 logcat和dmesg日志
- `packages/services/Car/tools/bootio/`可以记录启动过程的进程I/O，这个必须修改flags，重新编译kernel (依照 `README.md` )

## 早期启动服务

在启动流程中，有些服务可以早于Android启动。

### Rear View Camera (RVC)

后视摄像头(RVC) 应该由kernel处理，当汽车切到倒车档时，VMCU通知kernel进程，然后kernel进程可以在显示屏上显示RVC的图像。 VHAL可以通过`hardware/libhardware/include/hardware/vehicle_camera.h`控制 RVC。

### Vehicle Network Service (VNS)

在boot早期，等待用户空间服务启动时，有些系统可能需要读取和缓存CAN数据，比如车速，档位。 这种场景要求VNS、VHAL以及CAN控制器早早启动，典型情况比如上电几秒内。

- 可以快速加载 `/system` 的系统可以在早期，先启动service manager，然后快点启动VNS
- 不能快速加载 `/system` 的系统必须把 service manager 和 VNS 移到 kernel的boot镜像，并且静态链接依赖库。