---
layout: post
title:  "Media控制电台"
parent: "专题"
date:   2020-01-10
nav_order: 5
---

# Media控制电台

电台 (Radio) 控制实现基于 MediaSession 和 MediaBrowse，两者搭配使用可让媒体服务和 Google 助理控制电台。有关这些接口的详细信息，请参阅[为汽车开发 Android 媒体应用](https://developer.android.com/training/auto/audio/)。

请参阅 `packages/apps/Car/libs` 中提供的 `car-broadcastradio-support` 库中的媒体浏览树实现。该库还提供了 `ProgramSelector` 的扩展，用于与 URI 之间来回进行转换。要实现电台功能，请使用该库构建一个浏览树。

## 媒体来源切换器

为了在电台和媒体服务中的其他应用之间顺畅切换，`car-media-common` 库包含了要集成到电台应用中的类。请在电台应用的 XML 文件中包含 `MediaAppSelectorWidget` 微件（在参考媒体和电台应用中使用的图标或下拉列表）：

```
<com.android.car.media.common.MediaAppSelectorWidget
    android:id="@+id/app_switch_container"
    android:layout_width="@dimen/app_switch_widget_width"
    android:layout_height="wrap_content"
    android:background="@drawable/app_item_background"
    android:gravity="center" />
```

该微件会启动 `AppSelectionFragment`，以显示可供选择的媒体来源的列表。如果您需要的用户界面与所提供的不同，请创建自定义微件以在要显示切换器时启动 `AppSelectionFragment`：

```
AppSelectionFragment newFragment = AppSelectionFragment.create(widget,
            packageName, fullScreen);
    newFragment.show(mActivity.getSupportFragmentManager(), null);
```

在作为参考的电台应用实现中，提供了一个实现示例，它位于 `packages/apps/Car/Radio` 中。

## 详细控制规格

MediaSession 接口（通过 `MediaSession.Callback`）为当前正在播放的电台应用提供控制机制：

| 机制                                | 说明                         |
| :---------------------------------- | :--------------------------- |
| `onPlay` `onStop`                   | 时移暂停（如果支持）。       |
| `onPlayFromMediaId` `onPlayFromUri` | 调到特定电台。               |
| `onSkipToNext` `onSkipToPrevious`   | 调到下一个（或上一个）电台。 |
| `onSetRating`                       | 添加和移除收藏的电台。       |

MediaBrowser 公开的 MediaItem（您可以随时调到这些电台）会划分到三种类型的顶层目录：

- **节目（电台）**。可以收听且在范围内的电台节目。
- **收藏**。已添加到“收藏”列表中的电台节目。有些可能超出范围，无法收听。
- **频段频道**。当前区域中所有可实际接收到的频道（例如，87.9、88.1、88.3、88.5、88.7、88.9、89.1 等）。每个频段都有一个单独的顶层目录。

![](/assets/images/radio_media_01.png)

**图 1.** 广播电台 MediaBrowserService 树结构

这些列表中的每个元素都有一个 `mediaId`，可与 `MediaSession` 配合使用来调台。

由于 `MediaBrowserService` 的实现非常重要，因此强烈建议 OEM 电台应用将实现细节交由所提供的 `car-broadcastradio-support` 库来负责。OEM 仍可以选择自行实现这些规范。

### MediaSession

由于实时广播是无法暂停的，因此播放、暂停和停止操作并不太适用于电台。相对地，停止操作可以视为将实时广播*静音*，而播放操作可视为将实时广播*取消静音*。

为了在实时广播中模拟暂停效果，一些电台应用会先缓存内容，然后通过 `onPause` 命令回放缓存的内容。如果您不这样做，请勿宣称支持 `ACTION_PAUSE`。

根据 `mediaId` 和 URI 开始播放的操作旨在用于调到从 MediaBrowser 接口中提取的电台。`mediaId` 是电台应用提供的任意字符串，用于唯一（一个 ID 仅指向一项内容）且稳定地（一项内容的 ID 在整个会话中保持不变）标识给定的电台。在这种情况下，URI 将采用一个明确定义的架构（一种能够作为 URI 来处理的 ProgramSelector 形式）。尽管 URI 不需要保持稳定，但仍保留了唯一性属性。（如果电台频率发生变动，该属性可能会改变）。

未使用 `onPlayFromSearch`。客户端（Google 助理或配套应用）应自行负责从 MediaBrowser 树中选择搜索结果。将这部分工作转由电台应用执行会增加复杂性，需要就字符串查询的格式达成正式协定，并且会导致在不同硬件平台上出现不可预测的用户体验。

>  **注意**：就电台名称搜索而言，在 MediaBrowser 接口向客户端提供的信息之外，电台应用无法提供任何有用的额外信息。

跳到下一个或上一个电台时，具体会跳到哪个电台取决于当前的使用情景：

- 如果应用当前调到的是“收藏”列表中的某个电台，电台应用可以移到“收藏”列表中的下一个电台。
- 收听的是节目列表中的电台时，可能会调到下一个可供播放的电台（根据电台排序）。
- 收听的是随机电台时，可能会调到下一个实体电台（可能收不到有效的广播信号）。

> **注意**：电台应用必须能够妥善处理这些情景。

#### 处理错误

TransportControls 操作（播放、停止和下一个）不提供有关成功或失败的任何反馈。要指示错误，MediaSession 的状态必须为 `STATE_ERROR` 并带有错误消息。

电台应用必须执行这些操作或设置错误状态。如果不立即执行播放命令，则请将播放状态更改为 `STATE_CONNECTING`（如果是直接调谐）或者更改为 `STATE_SKIPPING_TO_PREVIOUS` 或 `STATE_SKIPPING_TO_NEXT`（执行命令时）。

要验证会话是已顺利将当前节目改为所需节目，还是已被置于错误状态，客户端应该监视 PlaybackState。虽然处于 `STATE_CONNECTING` 状态的时长应该不会超过 30 秒，但直接调谐到特定的 AM/FM 频率应该比这更快。

#### 添加和移除收藏的节目

MediaSession 支持评分功能，可用于控制收藏的节目。使用 `RATING_HEART` 类型的评分调用 `onSetRating` 时，会在“收藏”列表中添加或移除当前调到的电台。

该模型假设“收藏”列表是无序且无界的，这与传统的预设 (preset) 相反，后者中每个收藏的节目都会分配到一个数字槽（通常为 1 到 6）。因此，基于预设的系统与 `onSetRating` 不兼容。

MediaSession API 的局限在于它只能添加和移除当前所调到的电台。这就像如果不先选择某项内容，就无法从列表中删除该内容。这是 MediaBrowser 客户端（例如 Google 助理或配套应用）的一项限制。电台应用不受限制。

**重要提示**：如果应用不支持收藏功能，则您可忽略以下内容。

### MediaBrowser

为了指明在给定的区域中哪些频率或实体频道名称（如果给定的电台技术支持调到任意频道）有效，我们列出了每个频段的所有有效频道（频率）。美国地区提供：

| 频段 | 频道                                                    |
| :--- | :------------------------------------------------------ |
| FM   | 87.8 至 108.0 MHz 范围内的 101 个频道，间隔为 0.2 MHz。 |
| AM   | 530 至 1700 kHz 范围内的 117 个频道，间隔为 10 kHz。    |

>  **注意**：HD 电台使用相同的信道空间，因此不单独列出。

当前可供播放的电台节目列表属于一个简单列表，它不支持按数字音频广播 (DAB) 集合进行分组等显示方案。

“收藏”列表中的条目有时可能不在信号范围内。电台应用不一定能事先检测到哪些条目当前可正常收听。这种情况下，电台应用可能不会将条目标记为可播放。

要识别顶层文件夹，请使用与蓝牙所用相同的机制。

`MediaDescription` 对象的 Extra 捆绑包将包含一个调谐器专属字段，其等效于蓝牙的 `EXTRA_BT_FOLDER_TYPE`。如果是广播电台，则需要在公共 API 中定义以下新字段。

`EXTRA_BCRADIO_FOLDER_TYPE` = `android.media.extra.EXTRA_BCRADIO_FOLDER_TYPE`，以下值之一：

| 字段                            | 值                          |
| :------------------------------ | :-------------------------- |
| `BCRADIO_FOLDER_TYPE_PROGRAMS`  | 1. 当前可供播放的节目。     |
| `BCRADIO_FOLDER_TYPE_FAVORITES` | 2. 收藏。                   |
| `BCRADIO_FOLDER_TYPE_BAND`      | 3. 给定频段的所有实际频道。 |

无需设置电台专属自定义元数据字段，因为所有相关数据都可以容纳在现有 `MediaBrowser.MediaItem` 机制中：

- **节目名称（RDS PS，DAB 服务名称）**。`MediaDescription.getTitle`
- **FM 频率**。URI（请参阅 ProgramSelector 部分）或 `MediaDescription.getTitle`。如果 `BROADCASTRADIO_FOLDER_TYPE_BAND` 文件夹包含条目。
- **电台专属标识符（RDS PI，DAB SId）**。`MediaDescription.getMediaUri` 解析到 ProgramSelector。

通常，您无需为当前节目或“收藏”列表中的条目提取 FM 频率（因为客户端应根据媒体 ID 进行操作）。但是，如果存在这种需要（可能用于显示目的），则该条目会出现在 可以解析到 ProgramSelector 的 URI 中。

不建议在当前会话中使用 URI 进行项目选择。有关详情，请参阅 ProgramSelector。

为避免发生性能和 Binder 相关的问题，MediaBrowser 服务必须支持分页。将 `EXTRA_PAGE` 和 `EXTRA_PAGE_SIZE` 参数用于 `subscribe()`。

>  **注意**：默认情况下，`onLoadChildren()` 变体（没有选项处理）会实现分页。

所有列表类型（原始频道、找到的节目以及收藏的节目）中的相关条目可能具有不同的媒体 ID，这由电台应用决定。支持库会有所不同。在大多数情况下，原始频道和找到的节目的 URI（采用 `ProgramSelector` 形式）有所不同，没有 RDS 的 FM 除外。在大多数情况下，找到的节目和收藏的节目也是如此（AF 更新时除外）。

>  **重要提示**：频率列表中的条目以 `AMFM_FREQUENCY` 作为其主要标识符，而节目列表中的条目很可能以 `RDS_PI`（或其他数字电台标识符类型）作为主要标识符。

对来自不同列表类型的条目使用不同的媒体 ID 可以对它们执行不同的操作：执行 onSkipToNext 操作时遍历“收藏”列表还是“所有节目”列表，具体取决于最近选择的 MediaItem 所在的文件夹（请参阅 MediaSession 部分）。这也意味着 Google 助理会根据找到匹配条目的位置（在“收藏”列表或“所有可供播放的节目”列表中）无意切换这些上下文。如何处理这类情况则由应用决定。

#### 特殊调谐操作

节目列表允许调谐到特定电台，但不允许发出诸如“调谐到 FM”之类的一般请求，这可能会导致调谐到 FM 频段上最近收听的电台。

为了支持此类操作，一些顶层目录设置了 `FLAG_PLAYABLE` 标记，并针对文件夹设置了强制性 `FLAG_BROWSABLE`。

| 操作           | 调谐目标       | 如何发出                                                     |
| :------------- | :------------- | :----------------------------------------------------------- |
| 播放电台       | 任意           | `startService(ACTION_PLAY_BROADCASTRADIO)` 或 `playFromMediaId(MediaBrowser.getRoot())` |
| 播放收藏的节目 | 任意收藏的节目 | 从 `favorites` 文件夹中的 `mediaId` 播放                     |
| 播放 FM        | 任意 FM 频道   | 从 FM 频段的 `mediaId` 播放                                  |

具体调谐到哪个确切节目，则由应用决定。通常是给定列表中最近调到的频道。

有关 `ACTION_PLAY_BROADCASTRADIO` 更多详细信息，请参阅“常规播放 intent”部分。

#### 发现和服务连接

`PackageManager` 可以直接找到提供广播电台树的 `MediaBrowserService`。只需使用 `ACTION_PLAY_BROADCASTRADIO` intent（参见“常规播放 intent”部分）和 `MATCH_SYSTEM_ONLY` 标记调用 `resolveService`。要查找提供电台的所有服务（可能有多项，例如单独的 AM/FM 和卫星），请使用 `queryIntentServices`。已解析的服务也将处理 `android.media.browse.MediaBrowserService` 绑定 intent，因为它是通过 GTS 验证的。

要连接到所选的 MediaBrowserService，请为给定服务组件创建 MediaBrowser 实例并进行连接。建立连接后，可以通过 `getSessionToken` 获取 MediaSession 的句柄。

电台应用可以在其服务的 onGetRoot 实现中限制允许连接的客户端软件包。该应用应允许系统应用直接连接，而不必将其加入白名单。

如果特定于来源的应用（例如，电台应用）安装在不支持此类来源的设备上，则仍建议由其自身处理 `ACTION_PLAY_BROADCASTRADIO` intent，同时各自的 MediaBrowser 树将不包含特定于电台的标记（如果系统映像出现在不同的硬件设备上则同样如此）。

客户端可以检查给定来源在设备上是否可用，具体操作如下：

1. 要发现电台服务，请针对 `ACTION_PLAY_BROADCASTRADIO)` 调用 `resolveService`。
2. 为电台服务创建并连接到 MediaBrowser。
3. 确认带有 `EXTRA_BCRADIO_FOLDER_TYPE` extra 的任何 MediaItem。

>  **注意**：在大多数情况下，客户端必须检查可用的 MediaBrowser 树，以识别可在给定设备上使用的所有来源。

#### 频段名称

频段列表由一组顶层目录表示，文件夹类型标记设置为 `BCRADIO_FOLDER_TYPE_BAND`。各个 MediaItem 的标题是表示频段名称的本地化字符串。在大多数情况下，这些字符串是其对应英文的翻译。但客户端应这样假设。

为了提供一种稳定的机制来查找特定频段，为频段文件夹添加了一个 extra 标记。`EXTRA_BCRADIO_BAND_NAME_EN` 是频段的非本地化名称，它采用一个预定义值：

- TAM
- FM
- DAB
- SXM

如果频段不在此列表中，则不应设置频段名称标记。但是，如果频段在列表中，则必须设置标记。HD 电台不会枚举单独的频段，因为 HD 电台使用与 AM/FM 相同的底层媒介。

## 常规播放 intent

每个专用于播放特定来源（例如电台或 CD）的应用都必须处理常规播放 intent 以便开始播放内容（可能来自非活动状态。例如，在启动后。）如何选择要播放的内容由应用决定，但通常是播放最近播放过的电台应用或 CD 曲目。

为每个音频源定义一个单独的 intent：

| Intent                                          | 说明                                                         |
| :---------------------------------------------- | :----------------------------------------------------------- |
| `android.car.intent.action.PLAY_BROADCASTRADIO` |                                                              |
| `android.car.intent.action.PLAY_AUDIOCD`        | CD-DA 或 CD-Text。                                           |
| `android.car.intent.action.PLAY_DATADISC`       | 数据光盘，如 CD 和 DVD，但不是 CD-DA。 这可能是混合模式 CD。 |
| `android.car.intent.action.PLAY_AUX`            | 不指定 AUX 端口。                                            |
| `android.car.intent.action.PLAY_BLUETOOTH`      |                                                              |
| `android.car.intent.action.PLAY_USB`            | 不指定 USB 设备。                                            |
| `android.car.intent.action.PLAY_LOCAL`          | 本地媒体存储设备，例如内置闪存驱动器。                       |

之所以选择将 intent 与 Play 命令一起使用，是因为 intent 解析能实现两个目的：

### Play 命令服务发现

另一个好处是，intent 可以在不打开 MediaBrowser 会话的情况下执行简单操作。这就是说，服务发现是 intent 实现的更有价值的目的。服务发现的过程因此而变得直接而明确。如需了解详情，请参阅“发现和服务连接”。

为了简化客户端实现，发出 Play 命令的替代方案（也必须由电台应用实现）是发出带有根节点 `rootId`（用作 `mediaId`）的 `playFromMediaId1`。虽然根节点并不是可用来播放的，但其 `rootId` 是可以被指定为可用作 `mediaId` 的任意字符串。

>  **注意**：客户端无需了解这种细微差别。

## ProgramSelector

虽然使用 `mediaId` 足以从 MediaBrowserService 中选择频道，但它仅限于会话，并且在各提供程序之间不一致。在某些情况下，客户端可能需要一个绝对指针（如绝对频率）来在会话和设备之间保持不变。

在当下 DAB 时代，单靠频率不足以调谐到特定电台。为了提供可用于调谐到模拟或数字频道的实体，设计了 `ProgramSelector`。（设计文档、HAL 源代码、Java 源代码）。该实体由以下两部分组成：

- **主要标识符**。给定电台的唯一且稳定的标识符，它不会改变，但可能不足以调谐到该电台。例如，RDS PI 代码，在美国可以转换为呼号。
- **辅助标识符**。用于调谐到电台的任何其他标识符。例如频率，可能包括采用其他电台技术的标识符。DAB 电台可能会回退到模拟广播。

要使 ProgramSelector 适合这个基于 MediaBrowser/MediaSession 的解决方案，请定义 URI 架构以对其进行序列化。架构定义如下：

```
broadcastradio://program/<primary ID type>/<primary ID>?
<secondary ID type>=<secondary ID><secondary ID type>=<secondary ID>
```

辅助标识符（位于 `?` 字符后面）是可选的，可以移除以提供稳定的标识符来用作 mediaId。例如：

- `broadcastradio://program/RDS_PI/1234?AMFM_FREQUENCY=88500&AMFM_FREQUENCY=103300`
- `broadcastradio://program/AMFM_FREQUENCY/102100`
- `broadcastradio://program/DAB_SID_EXT/14895264?RDS_PI=1234`

节目的权威部分（也就是 `host`）为将来扩展该架构提供了一些空间。

`IdentifierType` 的 HAL 2.x 定义中名称后面精确指定了标识符 `type` 字符串，并且 `value` 格式是十进制或十六进制（带有 0x 前缀）数字。

所有供应商专用标识符都用 `VENDOR_` 前缀表示。例如，`VENDOR_0` 表示 `VENDOR_START`，`VENDOR_1` 表示 `VENDOR_START` + 1。

>  **重要提示**：此类 URI 特定于生成它们的电台硬件，不能在其他 OEM 制造的设备之间传输。必须将这些 URI 分配给顶层电台文件夹下的每个 MediaItem，并且 MediaSession 必须支持 `playFromMediaId` 和 `playFromUri`。但是，URI 主要用于电台元数据提取（例如 FM 频率）和持久存储。不保证 URI 可用于所有媒体项目。例如，当框架尚不支持主要 ID 类型时。另一方面，MediaID 始终保证有效。建议客户不要使用 URI 从当前 MediaBrowser 会话中选择项目。相反，请使用 `playFromMediaId`。但是，它对于服务应用来说不是可选的，并且保留缺少的 URI 有完全合理的用途。

最初设计是使用一个冒号 (:) 来代替 `://` 序列，放在架构部分后面。但是，`android.net.Uri` 不支持绝对分层 URI 引用。

## 其他来源类型：

其他音频源的管理方式类似于电台。例如，辅助输入和音频 CD 播放器。

单个应用可以提供多种类型的来源。在这种情况下，强烈建议为每种来源创建单独的 MediaBrowserService。即使在具有多个提供的来源或 MediaBrowserService 的设置中，也强烈建议您在一个应用中使用一个 MediaSession。

### 音频 CD

播放此类磁盘中内容的应用公开具有一个可浏览条目（或多个，如果系统有 CD 换碟机）的 MediaBrowser，该 MediaBrowser 又包含给定 CD 的所有曲目。如果系统未读取每张 CD 上的曲目（例如，一次性将所有磁盘插入磁盘盒并且系统未读取这些磁盘中的内容时），那么整个磁盘的 MediaItem 只是可播放的，而不是既可浏览又可播放。如果没有磁盘占用给定的数字槽，则相关内容既不可播放，也不可浏览，尽管每个数字槽必须始终存在于树中也是如此。

![](/assets/images/radio_media_02.png)

**图 2.** 音频 CD MediaBrowser 树结构

这些条目将以类似于广播电台文件夹的方式进行标记 - 它们将包含 MediaDescription API 中定义的其他 extra 字段：

| 字段             | 值                                                  |
| :--------------- | :-------------------------------------------------- |
| `EXTRA_CD_TRACK` | 对于音频 CD 上的每个 MediaItem，基于 1 的曲目编号。 |
| `EXTRA_CD_DISK`  | 基于 1 的磁盘编号。                                 |

对于支持 CD-Text 的系统和兼容磁盘，顶层 MediaItem 包含磁盘的标题。类似地，曲目的 MediaItem 包含曲目的标题。

#### 辅助输入

提供辅助输入功能的应用公开具有单个条目（或多个，如果有多个端口）的 MediaBrowser 树以表示 AUX 输入端口。在收到 `playFromMediaId` 请求后，相应的 MediaSession 将获取其 `mediaId` 并切换到该来源。

![](/assets/images/radio_media_03.png)

**图 3.** AUX MediaBrowser 树结构

每个 AUX MediaItem 条目都有一个 extra 字段 `EXTRA_AUX_PORT_NAME`，其已设置为端口的非本地化名称，但不包含“AUX”字样。例如，“AUX 1”会设置为“1”，“AUX front”会设置为“front”，而“AUX”会设置为空字符串。在非英语语言区域，名称标记保留英文字符串不变。`EXTRA_BCRADIO_BAND_NAME_EN` 则不可能是这种情况，其值由 OEM 定义且不局限于预定义列表。

如果硬件可以检查设备是否已连接到 AUX 端口，则硬件应仅在连接输入时将 MediaItem 标记为 `PLAYABLE`。如果此端口没有连接任何输入，硬件仍必须执行枚举操作（但不是通过 `PLAYABLE`）。如果硬件没有此类功能，则 MediaItem 必须始终为 `PLAYABLE`。

#### extra 字段

可以定义以下 extra 键：

| 字段                  | 值                                  |
| :-------------------- | :---------------------------------- |
| `EXTRA_CD_TRACK`      | `android.media.extra.CD_TRACK`      |
| `EXTRA_CD_DISK`       | `android.media.extra.CD_DISK`       |
| `EXTRA_AUX_PORT_NAME` | `android.media.extra.AUX_PORT_NAME` |

客户端必须在顶层的 MediaItem 中查找已为 `EXTRA_CD_DISK` 和 `EXTRA_AUX_PORT_NAME` 设置值的元素。

## 详细示例

以下示例说明了此设计中包含的来源类型的 MediaBrowser 树结构。

广播电台 `MediaBrowserService` 处理 `ACTION_PLAY_BROADCASTRADIO`。

| 电台（可浏览）                                           | URI                                                          |
| :------------------------------------------------------- | :----------------------------------------------------------- |
| `EXTRA_BCRADIO_FOLDER_TYPE=BCRADIO_FOLDER_TYPE_PROGRAMS` |                                                              |
| BBC One（可播放）                                        | `broadcastradio://program/RDS_PI/1234?AMFM_FREQUENCY=90500`  |
| ABC 88.1（可播放）                                       | `broadcastradio://program/RDS_PI/5678?AMFM_FREQUENCY=88100`  |
| ABC 88.1 HD1（可播放）                                   | `broadcastradio://program/HD_STATION_ID_EXT/158241DEADBEEF?AMFM_FREQUENCY=88100&RDS_PI=5678` |
| ABC 88.1 HD2（可播放）                                   | `broadcastradio://program/HD_STATION_ID_EXT/158242DEADBEFE`  |
| 90.5 FM（可播放） 没有 RDS 的 FM                         | `broadcastradio://program/AMFM_FREQUENCY/90500`              |
| 620 AM（可播放）                                         | `broadcastradio://program/AMFM_FREQUENCY/620`                |
| BBC One（可播放）                                        | `broadcastradio://program/DAB_SID_EXT/1E24102?RDS_PI=1234`   |

| 收藏夹（可浏览、可播放）                                  | URI                                                          |
| :-------------------------------------------------------- | :----------------------------------------------------------- |
| `EXTRA_BCRADIO_FOLDER_TYPE=BCRADIO_FOLDER_TYPE_FAVORITES` |                                                              |
| BBC One（可播放）                                         | `broadcastradio://program/RDS_PI/1234?AMFM_FREQUENCY=101300` |
| BBC Two（不可播放）                                       | `broadcastradio://program/RDS_PI/1300?AMFM_FREQUENCY=102100` |
| AM（可浏览、可播放）                                      | `EXTRA_BCRADIO_FOLDER_TYPE=BCRADIO_FOLDER_TYPE_BAND`         |

| AM（可浏览、可播放）                                         | URI                                           |
| :----------------------------------------------------------- | :-------------------------------------------- |
| `EXTRA_BCRADIO_FOLDER_TYPE=BCRADIO_FOLDER_TYPE_BAND` `EXTRA_BCRADIO_BAND_NAME_EN="AM"` |                                               |
| 530 AM（可播放）                                             | `broadcastradio://program/AMFM_FREQUENCY/530` |
| 540 AM（可播放）                                             | `broadcastradio://program/AMFM_FREQUENCY/540` |
| 550 AM（可播放）                                             | `broadcastradio://program/AMFM_FREQUENCY/550` |

| FM（可浏览、可播放）                                         | URI                                                          |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `EXTRA_BCRADIO_FOLDER_TYPE=BCRADIO_FOLDER_TYPE_BAND` `EXTRA_BCRADIO_BAND_NAME_EN="FM"` |                                                              |
| 87.7 FM（可播放）                                            | `broadcastradio://program/AMFM_FREQUENCY/87700`              |
| 87.9 FM（可播放）                                            | `broadcastradio://program/AMFM_FREQUENCY/87900`              |
| 88.1 FM（可播放）                                            | `broadcastradio://program/AMFM_FREQUENCY/88100`              |
| DAB（可播放）                                                | `EXTRA_BCRADIO_FOLDER_TYPE=BCRADIO_FOLDER_TYPE_BAND` `EXTRA_BCRADIO_BAND_NAME_EN="DAB"` |

| 音频 CD MediaBrowserService   | URI                                                          |
| :---------------------------- | :----------------------------------------------------------- |
| 处理 `ACTION_PLAY_AUDIOCD`    |                                                              |
| 磁盘 1（可播放）              | `EXTRA_CD_DISK=1`                                            |
| 磁盘 2（可浏览、可播放）      | `EXTRA_CD_DISK=2` 曲目 1（可播放）`EXTRA_CD_TRACK=1`曲目 2（可播放）`EXTRA_CD_TRACK=2` |
| 我的音乐 CD（可浏览、可播放） | `EXTRA_CD_DISK=3` All By Myself（可播放）`EXTRA_CD_TRACK=1`Reise，Reise（可播放）`EXTRA_CD_TRACK=2` |
| 空插槽 4（不可播放）          | `EXTRA_CD_DISK=4`                                            |

| AUX MediaBrowserService | URI                           |
| :---------------------- | :---------------------------- |
| 处理 `ACTION_PLAY_AUX`  |                               |
| AUX front（可播放）     | `EXTRA_AUX_PORT_NAME="front"` |
| AUX rear（可播放）      | `EXTRA_AUX_PORT_NAME="rear"`  |