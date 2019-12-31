---
layout: post
title:  "HMI"
parent: "基本"
date:   2019-12-31
nav_order: 3
---

# HMI

Android Automotive是AOSP提供的一个IVI(In-Vehicle Infotainment)平台方案，这部分介绍其中的核心应用的功能。

- Car Settings：针对汽车的图形界面，基本的驾驶干扰优化，以及OEM定制入口
- Media：只要一些设置和一个服务，开发者就可以从已有的媒体应用扩展出自己的。应用要遵循Automotive的媒体应用模板。
- System UI：导航栏、状态栏和音量控制这些UI元素，都要在这定制。

## 1、定制化

AOSP中的SystemUI等核心系统应用可以作为HMI开发的基础，在此基础上进行修改（比如资源overlay）以达到OEM要求，称为”定制化“。

Automotive设计为不同的模块以不同的方式定制：

- SystemUI：OEM可以直接修改或者替换，只要满足CDD和CTS的要求
- 非核心应用：定制或替换
- 核心应用：每个应用有各自的定制指导，强烈建议OEM使用AOSP的实现，在此基础上有限制的定制。

## 2、UX限制引擎

`CarUxRestrictionsManager`给应用提供了一个hook，可以用来监听驾驶状态相关的变化，然后修改UX。OEM可以覆盖配置文件`packages/services/Car/service/res/xml/car_ux_restrictions_map.xml`来影响系统行为。

## 3、系统主题

系统默认属性为DeviceDefault，建议OEM从修改此主题开始定制。默认情况AOSP中所有系统应用都从这个主题继承，当然也可以扩展DeviceDefault。第三方应用可以使用androidx.car中的Theme.Car。

- Core. /frameworks/base/core/res/res/values/themes_device_defaults.xml
- Colors. /frameworks/base/core/res/res/values/colors_car.xml
- Styles. /frameworks/base/core/res/res/values/styles_car.xml
- Car overlay. /packages/services/Car/car_product/overlay/.../values/themes_device_defaults.xml

OEM应该在vendor新建一个与 car_product 类似的overlay结构，用来覆盖car_product的overlay。

`/packages/services/Car/tests/ThemePlayground`可以用来检查主题修改。

## 4、系统应用

Android Automotive除了SystemUI外，包含了一组核心应用：

- **Communication Center**
- HVAC
- IME (keyboard)
- Launcher (home screen)
- Local Media Player
- **Media**
- Messenger
- **Notifications**
- Radio
- **Settings**

### 主屏

主屏也就是Car Launcher，是HMI的登录页，AOSP中的实现只是一个参考，OEM应该用自己的实现替换它，一般会包含导航、媒体、通信和其他系统状态。

### 通知

Automotive的通知是和Android中相同结构的完整模块，可以直接使用，不建议大幅改动。为车载做了以下优化：

- 用户可见的通知削减：正在播放，正在导航，以及系统应用的"不重要"的前台服务通知都被从通知中移除，或者有其他替代，或者不再有用
- 移除复杂的上下文控件：长按、左滑设置
- 遵循UX限制引擎：场景限制、字符串限长
- Android P添加了新的通知Category，只对`android.uid.system`应用开放
- `CATEGORY_CAR_EMERGENCY`：通知顶部显示，不受DND控制
- `CATEGORY_CAR_WARNING`：在emergency下面，其他上面，不受DND控制
- `CATEGORY_CAR_INFORMATION`：与其他通知根据规则排序

`packages/apps/Car/Notification/res/values/config.xml`提供了很小一部分的定制项。

### 设置

Settings暴露了很多配置（超过200个OS特性），用户可以用来配置Android OS和汽车其余部分的各个方面。为了保证可正常升级，防止碎片化，强烈建议OEM在AOSP的基础上有限改动。

#### 定制

- 主题：Preference类型的视觉定制，包括：
  - Preference.DeviceDefault.CheckBoxPreference
  - Preference.DeviceDefault.DialogPreference.EditTextPreference
- 结构定制：
  - 修改`Settings/res/values/config.xml`中的`config_settings_hierarchy_root_fragment`属性，可以登入任意root fragment
  - 修改`Settings/res/xml/*.xml`可以定制顺序、分组、文字、图标等
- 静态注入：通过overlay方式，OEM可以添加其他的Fragment和Controller
- 动态注入：通过动态Preference的方式，可以从Settings中打开第三方APP

### Media

Media是一个用来给实现了`MediaSession`和`MediaBrowser`的媒体应用提供前端界面的应用。媒体应用可以是音乐应用，也可以是其他媒体源，比如蓝牙音频流和本地音乐。为了保证兼容性，强烈建议OEM使用AOSP实现。

#### 定制

标准的DeviceDefault主题修改可以影响到Media，进一步可以用overlay修改风格，只要保持在UX指导要求内。

