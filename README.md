# 互动直播 Android Demo 源码导读

## <span id="工程概述">工程概述</span>
在线互动 Demo 是网易云通信的一款针对目前市场比较热门的互动直播解决方案。在方案中结合了网易云通信 IM 能力的聊天室模型和网易云通信的音视频能力的多人会议模型。在使用本解决方案之前请务必了解聊天室和多人会议能力。

 * [IM即时通讯](/docs/product/IM即时通讯/SDK开发集成/Android开发集成) 的聊天室能力
 * [音视频通话](/docs/product/音视频通话/SDK开发集成/Android开发集成) 的多人会议能力。

## <span id="互动连麦总体逻辑">互动连麦总体逻辑</span>

互动连麦的总体逻辑如下：
* 主播开始视频预览，如果不需要预览可以跳过这个步骤。
* 主播开始直播(在多人会议中推流)，观众可以观看(用播放器拉流)。
* 观众申请连麦，主播端显示连麦队列。
* 主播选择连麦队列中的某一个观众，观众结束播放器拉流并进入主播的多人会议。此时旁路直播自动开启，其他观众会看到主播和连麦观众合并的画面；主播和连麦观众和收到对方的画面数据，用特定控件(AVChatSurfaceViewRenderer或者AVChatTextureViewRenderer)渲染显示。
* 主播结束连麦，连麦观众自动退出多人会议，并开启播放器进行拉流操作。

由于聊天室和多人会议都不是直接针对直播的方案模型，所以需要在应用上层补充一些控制指令来保证连麦直播业务逻辑。
控制指令分为两套：

* 点对点自定义系统通知，用于主播和连麦者的控制交互，用于保证连麦者的上下麦时机。
* 聊天室广播消息，用于全局通知所有观众当前的连麦状态，观众需要根据连麦状态显示或隐藏一些控件。

### 点对点系统通知

* 进入麦序队列

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:加入连麦队列通知 | PushMicNotificationType#JOIN_QUEUE|
| roomid      | 房间ID      |   聊天室ID |
| style | 网络通话类型      |    AVChatType枚举 |
| info | 进入聊天队列用户信息      |  {"nick" : "","avatar" : ""} 字典|

* 退出麦序队列

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:退出连麦队列通知 | PushMicNotificationType#EXIT_QUEUE|
| roomid      | 房间ID      |   聊天室ID |

* 主播同意连麦

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:主播同意连麦通知 | PushMicNotificationType#CONNECTING_MIC|
| roomid      | 房间ID      |   聊天室ID |
| style | 连麦者连麦方式      |    AVChatType枚举 |

* 连麦者拒绝连麦

**当连麦者收到主播同意连麦通知时，会检查自身的连麦状态，如果连麦状态过期则需要发送一条拒绝消息告诉主播**

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:连麦者拒绝连麦通知 | PushMicNotificationType#REJECT_CONNECTING |
| roomid      | 房间ID      |   聊天室ID |

* 主播强制连麦者断开

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:主播强制连麦者断开 | PushMicNotificationType#DISCONNECT_MIC|
| roomid      | 房间ID      |   聊天室ID |



### 聊天室广播消息

* 连麦者已连麦

| 参数           | 说明           | 
| :-------------: |:-------------:|
| uid       | 连麦者的 accid   |
| nick      | 连麦者的昵称      |
| avatar    | 连麦者的头像      |
| style     | 连麦者的连麦方式   |

* 连麦者已断开

| 参数           | 说明           |
| :-------------: |:-------------:|
| uid      | 连麦者的 accid |

## <span id="互动PK总体逻辑">互动PK总体逻辑</span>

互动PK的总体逻辑如下：
* 主播A和B开始视频预览，如果不需要预览可以跳过这个步骤。
* 主播A和B开始直播(在多人会议中推流)，各自的观众可以观看(用播放器拉流)。
* 主播A向主播B发送PK邀请，主播B显示收到的PK邀请。
* 主播B接受PK邀请，主播A和B都进入PK模式，各自的观众拉到的流都会看到主播A和B的PK画面。
* 主播A或者B结束PK，主播A和B都恢复之前的直播模式，各自的观众拉到的流切换回看到各自的主播的画面。

由于聊天室和多人会议都不是直接针对直播的方案模型，所以需要在应用上层补充一些控制指令来保证PK直播业务逻辑。
控制指令分为两套：

* 点对点自定义系统通知，用于主播与主播的控制交互，用于保证PK时机。
* 聊天室广播消息，用于全局通知所有观众当前的PK直播状态，观众需要根据PK直播状态显示或隐藏一些控件。

### 点对点系统通知


* 尝试发起PK邀请

** 因为无法判断对方是否在线，因此先尝试发送邀请，等对方回复在线后进行真正的PK邀请

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:邀请主播PK通知 | PushMicNotificationType#TRY_INVITE_ANCHOR|
| pkRoomName      | pk房间名称      |   String |
| style | 网络通话类型      |    AVChatType枚举 |
| info | 进入聊天队列用户信息      |  {"nick" : "","avatar" : ""} 字典|

* 回复当前在线

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:邀请主播PK通知 | PushMicNotificationType#REPLY_INVITATION|
| pkRoomName      | pk房间名称      |   String |
| style | 网络通话类型      |    AVChatType枚举 |
| info | 进入聊天队列用户信息      |  {"nick" : "","avatar" : ""} 字典|

* 发送PK邀请命令

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:邀请主播PK通知 | PushMicNotificationType#INVITE_ANCHOR|
| pkRoomName      | pk房间名称      |   String |
| style | 网络通话类型      |    AVChatType枚举 |
| info | 进入聊天队列用户信息      |  {"nick" : "","avatar" : ""} 字典|

* 取消PK邀请

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:邀请主播PK通知 | PushMicNotificationType#CANCEL_INTERACT|
| pkRoomName      | pk房间名称      |   String |
| style | 网络通话类型      |    AVChatType枚举 |
| info | 进入聊天队列用户信息      |  {"nick" : "","avatar" : ""} 字典|

* 同意主播PK邀请

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:邀请主播PK通知 | PushMicNotificationType#AGREE_INVITATION|
| pkRoomName      | pk房间名称      |   String |
| style | 网络通话类型      |    AVChatType枚举 |
| info | 进入聊天队列用户信息      |  {"nick" : "","avatar" : ""} 字典|

* 拒绝主播PK邀请

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:邀请主播PK通知 | PushMicNotificationType#REJECT_INVITATION|
| pkRoomName      | pk房间名称      |   String |
| style | 网络通话类型      |    AVChatType枚举 |
| info | 进入聊天队列用户信息      |  {"nick" : "","avatar" : ""} 字典|

* PK无效指令

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:邀请主播PK通知 | PushMicNotificationType#INVALID|
| pkRoomName      | pk房间名称      |   String |
| style | 网络通话类型      |    AVChatType枚举 |
| info | 进入聊天队列用户信息      |  {"nick" : "","avatar" : ""} 字典|

* 正在PK中

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:邀请主播PK通知 | PushMicNotificationType#ININTER_ACTIONS|
| pkRoomName      | pk房间名称      |   String |
| style | 网络通话类型      |    AVChatType枚举 |
| info | 进入聊天队列用户信息      |  {"nick" : "","avatar" : ""} 字典|

* 对方已经退出PK房间

| 参数           | 说明           | 值  |
| :-------------: |:-------------:| :-----:|
| type      | 自定义系统通知类型:邀请主播PK通知 | PushMicNotificationType#EXITED|
| pkRoomName      | pk房间名称      |   String |
| style | 网络通话类型      |    AVChatType枚举 |
| info | 进入聊天队列用户信息      |  {"nick" : "","avatar" : ""} 字典|

### 聊天室广播消息

* 聊天室信息更新

| 参数           | 说明           |
| :-------------: |:-------------:|
| pkInviter       | pk邀请者      |
| pkInvitee       | pk被邀请者    |
| isPking         | 是否处于pk状态 |
| meetingName     | 房间名称      |
| type            | 直播方式      |



## <span id="源码分析"> 源码分析</span>

### 工程结构说明

源码主要分成四个 package ：base、im、thirdparty 和 entertainment。
- base：封装一些 UI 基类，工具类等。
- im：包含登录页面及登录、注册业务逻辑（ activity / business / config 子包下）、基础 UI 组件( ui 子包下)、 会话页面相关组件( session 子包下)。
- thirdparty：包含网易视频云播放器(拉流)相关的核心组件。
- entertainment：娱乐直播相关的页面和业务逻辑。

下面具体介绍 entertainment 包下的子包结构：
- activity：所有 Activity。
- adapter：聊天页面数据源适配器等。
- constant: 聊天页面数据常量，互动连麦数据常量。
- helper：直播间成员缓存，网络探测缓存，主播收到礼物缓存，礼物动画，互动连麦等帮助类。
- http: 网易云通信直播间 Demo Http Client(与网易云通信 Demo 应用服务器通信)
- model: 聊天界面数据实体。
- module：直播间收发消息模块、直播间自定义消息。
- ui: 直播间界面 ui 控件。
- viewholder：界面相关 ViewHolder。

### 重点类说明

- LivePlayerBaseActivity : 直播间基类。包括直播间的进入/离开的操作，监听直播间在线状态和监听直播间被踢出状态。
- LiveActivity：主播端 Activity。包含主播相关操作。
- AudienceActivity： 观众端 Activity。包含观众相关操作。
- IdentifyActivity： 顶部是网络探测相关控件，底部是选择主播或观众身份相关控件
- EnterRoomActivity： 观众身份填写主播创建的房间id，进入直播间
- ChatRoomMsgViewHolderFactory:  直播间消息项展示ViewHolder工厂类。包括消息展示 ViewHolder 的注册操作。

## <span id="滤镜模块">滤镜模块</span>
从3.9.0版本sdk开始，sdk提供滤镜模块，用于实现对主播和连麦者的视频画面进行美颜、水印等，sdk提供的滤镜模块要求 Android 4.3 以上版本，用户也可以集成第三方视频数据处理的sdk，最低支持到 Android 4.1。
- 使用滤镜时需要集成video_effect.jar和libVideoEffect.so到libs文件夹,或者直接使用gradle依赖的形式集成我们的sdk。
- 使用滤镜模块需要打开外部视频数据处理开关,设置AVChatParameters.KEY_VIDEO_FRAME_FILTER为true，这样视频数据就会在onVideoFrameFilter回调方法中得到。
- 然后使用sdk提供的滤镜模块相关接口对视频数据进行美颜、水印等处理，用户也可以在此接入第三方视频数据处理（美颜、水印等）sdk进行视频处理。
