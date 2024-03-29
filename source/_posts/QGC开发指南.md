---
title: QGC开发指南
tags:
  - qt
  - QGC
  - QML
categories:
  - qt
  - QGC
toc: true
recommend: 1
keywords: categories-java
uniqueId: 2023-05-10 10:56:24/QGC架构.html
mathJax: false
date: 2023-05-10 18:56:24
---

> QGC开发指南翻译+个人注解
> <!-- more -->

# 通信流 Communication Flow

描述载具在自动连接期间进行的高级通信流程。

- `LinkManager`总是有一个UDP端口打开等待车辆心跳
- `LinkManager`检测到一个新的已知设备类型(Pixhawk, SiK Radio, PX4 Flow)，使UDP连接到计算机
  - `LinkManager`在计算机和设备之间创建一个新的`SerialLink`
- 来自链路的传入字节被发送到`MAVLinkProtocol`
- `MAVLinkProtocol`将字节转换为MAVLink消息
- 如果消息是`HEARTBEAT`，则通知`MultiVehicleManager`(多机管理类)
- `MultiVehicleManager`收到`HEARTBEAT`的通知，并根据`HEARTBEAT`消息中的信息创建一个新的`vehicle `车辆对象
- `vehicle`对象实例化与车辆匹配的插件
- 与`vehicle`对象相关联的ParameterLoader发送一个 `PARAM_REQUEST_LIST`到连接的设备，以使用参数协议加载参数( to load parameters using the parameter protocol)
- 参数加载完成后，与`vehicle`对象相关联的`MissionManager`使用任务协议(mission protocol)从连接的设备请求任务项(mission items)
- 当参数加载完成后，`VehicleComponents`对象将在Setup视图中显示其UI

</br>

# 插件架构设计 Plugin Architecture

虽然MAVLink规范定义了与载具通信的标准通信协议。该规范的许多方面需要固件开发人员来解释。正因为如此，在许多情况下，为了完成相同的任务，与运行一个固件的车辆的通信与运行不同固件的车辆的通信略有不同。此外，每个固件可以实现MAVLink命令集的不同子集。

另一个主要问题是MAVLink规范不包括载具配置或通用参数集。 因此，所有与车辆设置相关的代码最终都是固件特定的。此外，任何必须引用特定参数的代码也是特定于固件的。

鉴于固件实现之间的所有这些差异，创建单个地面站应用程序可能非常棘手，可以支持每个应用程序而不会使代码库降级为基于车辆使用的固件在任何地方遍布的大量if / then / else语句。

QGC使用插件架构将固件特定代码与所有固件通用代码隔离开来。有两个主要的插件可以完成这个 `FirmwarePlugin`和`AutoPilotPlugin `。

自定义构建也使用此插件架构，以允许超出标准QGC所能提供的范围的进一步自定义。

## FirmwarePlugin

这是用来创建一个标准接口到Mavlink的部分，这通常是不标准化的。

## AutoPilotPlugin

这用于为车辆设置提供用户界面。

## QGCCorePlugin

这用于通过标准接口公开与车辆无关的QGC应用程序本身的特性。然后由自定义构建使用它来调整QGC特性集以满足其需求。

</br>

# 类层次结构（上层）

## (LinkManager)链接管理器类，(LinkInterface)链接接口类

QGC中的“链接”是QGC与载具间的一种特定类型的通信管道，例如串行端口或基于WiFi的UDP端口。 `LinkInterface`为所有链接的基类。 每个链接都在它自己的线程上运行，并将字节发送到`MAVLinkProtocol`。

`LinkManager`类所生成对象管理系统中的所有打开链接。 `LinkManager`还通过串行和UDP链接管理自动连接

## MAVLink协议类

系统中有一个MAVLink协议对象。 它的功能是从链接获取传入的字节并将它们转换为MAVLink消息。 MAVLink HEARTBEAT消息被分发到`MultiVehicleManager`(多机管理类)。 所有MAVLink消息都将分发到与链接相对应的载具。

## (MultiVehicleManager)多机管理类

系统中有一个`MultiVehicleManager`多机管理类生成的对象, 当它接收到一个新的心跳包(通过心跳包里面的系统ID识别)，它会自动生成一个载具对象，来表示一个新的载具加入到系统。`MultiVehicleManager`还可以保持对系统中所有载具的跟踪，对于激活状态的载具可以自由切换，而对于正在被移除的也能够正确处理。

## (Vehicle)载具类

`Vehicle`类所生成的对象是QGC代码与物理载具通信的主要接口。

注意：还有一个与每个Vehicle相关联的UAS对象，这是一个已弃用的类，并且正逐渐被逐步淘汰，所有功能都转移到`Vehicle`类。 这里不应该添加新代码。

## (FirmwarePlugin)固件插件类，( FirmwarePluginManager)固件插件管理器类

`FirmwarePlugin`类为固件插件的基类。 固件插件包含固件特定代码，因此`Vehicle`对象相对于它是识别的，支持UI的单个标准接口。

`FirmwarePluginManager`是一个工厂类，它根据`Vehicle`类的成员`MAV_AUTOPILOT / MAV_TYPE`组合创建`FirmwarePlugin`类的实例。

</br>

#  用户界面设计

QGC中UI设计的主要模式是用QML编写的UI页面，多次与用C ++编写的自定义“Controller”进行通信。 这种设计模式有点沿用MVC设计模式，但也有显著不同之处。

QML代码通过以下机制绑定到与系统关联的信息：

- 自定义“Controller”
- 全局QGroundControl对象，提供对active Vehicle等内容的访问
- FactSystem提供对参数的访问，在某些情况下提供自定义Facts。

注意：由于QGC中使用的QML的复杂性以及它依赖于与C ++对象的通信来驱动ui，因此无法使用Qt提供的QML Designer来编辑QML。

## 多设备设计模式

QGroundControl设计用于从桌面到笔记本电脑，平板电脑和使用鼠标和触摸的小型手机屏幕等多种设备类型。 以下是QGC如何做到以及背后原理的描述。

### 目标设备

从触摸的角度和屏幕尺寸的角度来看，QGC UI设计的优先目标是平板电脑（比如三星Galaxy）。 由于此决定，其他设备类型和大小可能会看到一些视觉和/或可用性的错误。 基于优先级的决策时的当前顺序是平板电脑，笔记本电脑，台式机，电话（任何小屏幕）。



### 使用的开发工具

#### Qt布局控件

QGC没有针对不同屏幕尺寸和/或形状因子的不同编码的UI。 通常，它使用QML Layout功能来重排一组QML UI代码以适应不同的外形。 在某些情况下，它提供了较小的屏幕尺寸细节，使事情适合。 但这是一个简单的可见性模式。

#### FactSystem（事实系统）

QGC的内部是一个系统，用于管理系统内的所有单个数据块。然后将此数据模型连接到控件。

#### 严重依赖可重用控件[^注1]

QGC UI是由一组基本的可重用控件和UI元素开发的。这样，添加到可重用控件中的任何新特性现在都可以在整个UI中使用。这些可重用控件还连接到FactSystem Facts，然后自动提供适当的UI。

[^注1]: 意思是 QGC 内部重写了QML基本控件，方便自适应各种屏幕，还可以关联 Fact 系统，实现数据持久化。



## 字体和调色板

QGC有一套标准的字体和调色板，应该由所有用户界面使用。

```qml
import QGroundControl.Palette       1.0
import QGroundControl.ScreenTools   1.0
```

### QGCPalette(QGC调色板)

此项目显示QGC调色板。 这个调色板有两种变体：浅色和深色。 较浅的调色板适合户外使用，黑色调色板适用于室内。 通常，您不应该直接为UI指定颜色，您应该使用调色板中的颜色。 如果不遵循此规则，则您创建的用户界面将无法通过浅色/深色样式更改。

### QGCMapPalette(QGC地图调色板)

地图调色板用于绘制地图的颜色。 由于不同的地图样式，特别是卫星和街道，您需要有不同的颜色来清晰地绘制它们。 卫星地图需要更浅的颜色，而街道地图需要更深的颜色。 QGCMapPalette项目为此提供了一组颜色，以及在地图上切换浅色和深色的功能。

### ScreenTools (屏幕工具)

ScreenTools项提供可用于指定字体大小的值。 它还提供有关屏幕大小以及QGC是否在移动设备上运行的信息。



## 用户界面控件

QGC为构建用户界面提供了一组基本控件。 一般来说，它们往往是Qt支持的基本QML控件上一层，这些控件遵守QGC调色板。

```
import QGroundControl.Controls 1.0
```

### Qt控件

以下控件是标准Qt QML控件的QGC变体[^注2]。 除了使用QGC调色板绘制。 它还们提供与相应Qt控件相同的功能。

- QGCButton
- QGCCheckBox
- QGCColoredImage
- QGCComboBox
- QGCFlickable
- QGCLabel
- QGCMovableItem
- QGCRadioButton
- QGCSlider
- QGCTextField

[^注2]: 即QML基本控件的子类对象



### QGC 控件

这些自定义控件是QGC独有的，用于创建标准UI元素。

- DropButton - RoundButton，单击时会删除一组选项。 示例是平面视图中的同步按钮。
- ExclusiveGroupItem - 用于支持QML ExclusiveGroup 概念的自定义控制的基础项目。
- QGCView - 系统中所有顶级视图的基本控件。 提供对FactPanels的支持并显示QGCViewDialogs和QGCViewMessages。
- QGCViewDialog - 从QGC视图右侧弹出的对话框。 您可以指定对话框的接受/拒绝按钮以及对话框内容。 使用示例是当您单击某个参数并显示值编辑器对话框时。[^注3]
- QGCViewMessage - QGCViewDialog的简化版本，允许您指定按钮和简单的文本消息。
- QGCViewPanel - QGCView内部的主要视图内容。
- RoundButton - 一个圆形按钮控件，它使用图像作为其内部内容。
- SetupPage - 所有安装载具组件页面的基本控件。 提供标题，说明和组件页面内容区域。

[^注3]:FactTextField对象是关联了Fact系统的文本编辑框，点击编辑会在右侧显示帮助按钮，点击帮助按钮会弹出该对话框。

</br>

# Fact System

Fact System[^注4]提供一组标准化和简化QGC用户界面创建的功能。

[^注4]: 提供了数据的定义模板，只需要定义一个json并使用宏声明即可

## Fact

Fact[^注5]代表系统中的单个数据。

[^注5]:Fact是对单个数据的封装并对外提供接口，支持QML数据交互。数据变更时，会相应发出信号

## FactMetaData

每个Fact都有一个FactMetaData[^注6]。它提供了有关Fact的详细信息，以便驱动自动用户界面生成和验证。

[^注6]:用于存储单个数据，数据由json解析而来，支持数字、字符串、枚举等；数据信息包含数据键值，类型，范围，默认值等，以供校验；还提供了翻译功能，可以在qgc_json_zh_CN.ts进行翻译，翻译时需要注意有些内容是以','作为分割符，不可翻译为中文符号

##  Fact Controls

Fact Controls[^注7]是一个QML用户界面控件，它连接到Fact和它的FactMetaData，为用户提供控件以修改/显示与Fact相关的值。

[^注7]:与Fact系统关联的QML控件，能够同步更新数据

## FactGroup

FactGroup[^注8]是一组Facts。它用于组织事实和管理用户定义的Facts。

[^注8]:将Fact编为一组数据进行封装，增加层次关系，加强管理。

## 补充：

**SettingsFact**

继承自`Fact`，增加了数据持久化，使用`QSettings`保存至ini配置文件

**SettingsGroup**

将单个json文件编为一组，并解析保存至`_nameToMetaDataMap`;

提供了持久化数据(`SettingsFact`)的创建模板。

## 自定义构建支持

用户定义的`Fact`可以通过在自定义固件插件类中覆盖`FirmwarePlugin`的`factGroups`函数来添加。这些函数返回事实组映射的名称，该名称用于标识添加的事实组。可以通过扩展`FactGroup`类来添加自定义事实组。通过提供包含必要信息的json文件，可以使用适当的`FactGroup`构造函数来定义`FactMetaData`。通过重写`FirmwarePlugin`类的`adjustMetaData`，也可以更改现有事实的元数据。与车辆相关的Fact(包括属于车辆固件插件的`factGroups`函数返回的事实组的事实)可以使用`getFact(“factName”)`或`getFact(“factGroupName.factName”)`

获取更多信息，请参阅FirmwarePlugin.h中的注释。

</br>

# 顶级视图

本节包含有关顶级视图代码的主题：设置，设置，计划和飞行。

## Settings View

- 顶级QML代码是AppSettings.qml
- 每个按钮都会加载一个单独的QML页面

##  Setup View

- 在SetupView.qml中实现的顶级QML代码
- 固定的按钮/页面集：摘要，固件
- 按钮/页面的剩余部分来自AutoPilot Plugin Vehicle Component列表

## Plan View

- 最高级别的 QML 代码是 [PlanView.qml](https://github.com/mavlink/qgroundcontrol/blob/master/src/PlanView/PlanView.qml)
- 主视图UI是FlightMap控件
- QML与MissionController（C ++）通信，后者为视图提供任务项数据和方法

### 用于任务项目编辑的动态UI

#### 任务指令树

QGC 创建用户界面，用于从 json 元数据的层次结构中动态编辑特定任务项命令。 此层次结构称为任务命令树。 这样，在添加新命令时，只能创建 json 元数据。

##### 为什么是一颗树？

需要该树以不同的方式处理不同固件和／或不同的车辆类型，以支持不同的命令。 最简单的例子是 mavlink 规范可能包含了并非所有固件都支持的命令参数。 或着命令参数仅对某些车辆类型有效。 此外，在某些情况下，GCS 可能会决定将某些命令参数在视图中对最终用户进行隐藏，因为它们过于复杂或导致可用性问题。

该树是MissionCommandTree类： [MissionCommandTree.cc](https://github.com/mavlink/qgroundcontrol/blob/master/src/MissionManager/MissionCommandTree.cc), [MissionCommandTree.h](https://github.com/mavlink/qgroundcontrol/blob/master/src/MissionManager/MissionCommandTree.h)

##### 树根目录

树的根目录是与 mavlink 规范完全匹配的json元数据。

您可以在这里看到`MAV_CMD_NAV_WAYPOINT`根目录[json](https://github.com/mavlink/qgroundcontrol/blob/master/src/MissionManager/MavCmdInfoCommon.json#L27)的示例：

```json
{
    "id":                   16,
    "rawName":              "MAV_CMD_NAV_WAYPOINT",
    "friendlyName":         "Waypoint",
    "description":          "Travel to a position in 3D space.",
    "specifiesCoordinate":  true,
    "friendlyEdit":         true,
    "category":             "Basic",
    "param1": {
        "label":            "Hold",
        "units":            "secs",
        "default":          0,
        "decimalPlaces":    0
    },
    "param2": {
        "label":            "Acceptance",
        "units":            "m",
        "default":          3,
        "decimalPlaces":    2
    },
    "param3": {
        "label":            "PassThru",
        "units":            "m",
        "default":          0,
        "decimalPlaces":    2
    },
    "param4": {
        "label":            "Yaw",
        "units":            "deg",
        "nanUnchanged":     true,
        "default":          null,
        "decimalPlaces":    2
    }
},
```

注意：在现实中，基于此的信息应由 mavlink 本身提供，而不需要成为 GCS 的一部分。

##### 叶节点

然后，叶节点提供元数据，这些元数据可以覆盖命令的值和/或从显示给用户的参数中删除参数。 完整的树层次结构是这样的：

- 根－通用Mavlink
  - 特定的车辆类型－特定于车辆的通用规范
  - 特定的硬件类型－每个固件类型有一个可选的叶节点（PX4/ArduPilot）
    - 特定的车辆类型－每个车辆类型有一个可选的叶节点（FW/MR/VTOL/Rover/Sub）

注意：实际上，此替代功能应该是mavlink规格的一部分，并且应该可以从车辆中查询。

##### 从完整树中构建实例树

由于 json 元数据提供了所有固件／车辆类型组合的信息，因此必须根据用于创建计划的固件和车辆类型来构建要使用的实际树。 这是通过一个进程调用“collapsing”的完整树到一个固件／车辆的特定树来完成的 ([code](https://github.com/mavlink/qgroundcontrol/blob/master/src/MissionManager/MissionCommandTree.cc#L119))。

步骤如下：

- 在实例树种添加根
- 将特定的车辆类型重写实例树
- Apply the firmware type specific overrides to the instance tree
- 将特定的硬件／车辆类型重写实例树

然后，生成的任务命令树将为平面项目编辑器构建UI。 实际上，它不仅用于此，还有许多其他地方可以帮助您了解有关特定命令 id 的更多信息。

#### 层次结构示例 `MAV_CMD_NAV_WAYPOINT`

让我们来看看`MAV_CMD_NAV_WAYPOINT`的示例层次结构。 根信息如上图所示。

##### 根－车辆类型的特定叶节点

层次结构的下一个层级是通用的 mavlink，但只针对特定的车辆。 这里的Json文件：[MR](https://github.com/mavlink/qgroundcontrol/blob/master/src/MissionManager/MavCmdInfoMultiRotor.json), [FW](https://github.com/mavlink/qgroundcontrol/blob/master/src/MissionManager/MavCmdInfoFixedWing.json), [ROVER](https://github.com/mavlink/qgroundcontrol/blob/master/src/MissionManager/MavCmdInfoRover.json), [Sub](https://github.com/mavlink/qgroundcontrol/blob/master/src/MissionManager/MavCmdInfoSub.json), [VTOL](https://github.com/mavlink/qgroundcontrol/blob/master/src/MissionManager/MavCmdInfoVTOL.json)。 这个是重写（固定翼）　

```json
{
    "id":           16,
    "comment":      "MAV_CMD_NAV_WAYPOINT",
    "paramRemove":  "4"
},
```

这样做是删除参数4的编辑 UI，固定翼没有使用航向（Yaw）参数。 由于这是根的叶节点，因此无论固件类型如何，这都适用于所有固定翼车辆。

##### 根－硬件类型的特定叶节点

层次结构的下一层级是特定于固件类型但适用于所有车辆类型的替代。 再次让我们看看航点（Waypoint）覆盖：

[ArduPilot](https://github.com/mavlink/qgroundcontrol/blob/master/src/FirmwarePlugin/APM/MavCmdInfoCommon.json#L6)：

```json
{
    "id":           16,
    "comment":      "MAV_CMD_NAV_WAYPOINT",
    "paramRemove":  "2"
},
```

[PX4](https://github.com/mavlink/qgroundcontrol/blob/master/src/FirmwarePlugin/PX4/MavCmdInfoCommon.json#L7)：

```json
{
    "id":           16,
    "comment":      "MAV_CMD_NAV_WAYPOINT",
    "paramRemove":  "2,3"
},
```

您可以看到，对于两个固件参数参数2，即接受半径，从编辑 ui 中删除。 这是QGC的特性决定。 与指定值相比，使用固件通用接受半径会更加安全和容易。 因此，我们决定对用户隐藏它。

您还可以看到，对于 PX4 param3/PassThru，由于 PX 不支持它，因此已被删除。

##### 根－特定于固件的类型－特定于车辆类型的叶子节点

层次结构的最后一个级别既针对固件又针对车辆类型。

[ArduPilot/MR](https://github.com/mavlink/qgroundcontrol/blob/master/src/FirmwarePlugin/APM/MavCmdInfoMultiRotor.json#L7):

```JSON
{
    "id":           16,
    "comment":      "MAV_CMD_NAV_WAYPOINT",
    "paramRemove":  "2,3,4"
},
```

在这里你可以看到，ArduPilot的多电机车辆参数2/3/4 Acceptance/PassThru/Yaw 已被移除。 例如，航向（Yaw）是因为不支持所以被移除。 由于此代码的工作原理的怪癖，您需要从较低级别重复重写。

#### 任务命令 UI 信息

两个类定义与命令相关联的元数据：

- MissionCommandUIInfo－整个命令的元数据
- MissionCmdParamInfo－命令中参数的元数据

源中注释了支持 json 键的完整详细信息。

[MissionCommandUIInfo](https://github.com/mavlink/qgroundcontrol/blob/master/src/MissionManager/MissionCommandUIInfo.h#L82)：

```
/// 与任务命令关联的 UI 信息 （MAV_CMD）
///
///MissionCommandUIInfo用于自动为MAV_CMD生成编辑ui。 此对象还支持仅具有一组命
/// 令的部分信息的概念。 这用于创建基本命令信息的替代。 对于覆盖，只需从基本命
/// 令 ui 信息中指定要修改的键即可。 若要重写 ui 参数信息，必须指定整个MissionParamInfo对象。
///
/// MissionCommandUIInfo对象的json格式为：
///
/// 键值                   　类型    默认值     描述
/// id                      int     reauired    MAV_CMD id
/// comment                 string              用于添加评论
/// rawName                 string  required    MAV_CMD 枚举名称，仅应设置基础树信息
/// friendlyName            string  rawName     命令的简单描述
/// description             string              命令的详细描述
/// specifiesCoordinate     bool    false       true: 命令指定一个纬／经／高坐标
/// specifiesAltitudeOnly   bool    false       true: 命令仅指定高度（非坐标）
/// standaloneCoordinate    bool    false       true: 车辆无法通过与命令关联的坐标（例如：ROI）
/// isLandCommand           bool    false       true: 命令指定着陆指令 (LAND, VTOL_LAND, ...)
/// friendlyEdit            bool    false       true: 命令支持友好的编辑对话框，false：Command仅支持“显示所有值”样式的编辑
/// category                string  Advanced    该命令所属的类别
/// paramRemove             string              由替代使用以删除参数，例如：“ 1,3”将删除替代上的参数1和3
/// param[1-7]              object              MissionCommandParamInfo 对象
///
```

[MissionCmdParamInfo](https://github.com/mavlink/qgroundcontrol/blob/master/src/MissionManager/MissionCommandUIInfo.h#L25)：

```
/// 与任务命令 （MAV_CMD） 参数关联的 UI 信息
///
/// MissionCommandParamInfo 用于自动为与 MAV_CMD 关联的参数生成编辑ui。
///
/// MissionCmdParamInfo 对象的 Json 文件格式为：
///　
/// 键值             类型    默认值     描述
/// label           string  required    文本字段标签
/// units           string              值的单位，应使用 FactMetaData Units 字符串以获得自动转换
/// default         double  0.0/NaN     默认参数值。 如果未指定默认值且 nanunchange==true，默认值为NaN。
/// decimalPlaces   int     7           显示值得小数位数
/// enumStrings     string              要在组合框中显示以供选择的字符串
/// enumValues      string              与每个枚举字符串关联的值
/// nanUnchanged    bool    false       True: 值可以设置为NaN表示信号不变
```



##  Fly View

- 顶级QML代码在FlightDisplayView.qml中
- QML代码与MissionController（C ++）通信以进行任务显示
- 仪器小部件与活动的Vehicle对象通信
- 两个主要的内部view是：
  - `FlightDisplayViewMap`
  - `FlightDisplayViewVideo`

</br>

# 文件格式

本节包含有关QGroundControl使用/支持的文件格式的主题。

## 开发者工具

QGroundControl主要为自动驾驶开发人员提供了许多工具。 这些简化了常见的开发人员任务，包括设置模拟连接以进行测试，以及通过MAVLink访问系统Shell。

> 注意:在调试模式下构建源以启用这些工具。

工具包括：

- **Mock Link** 模拟链接（仅限每日构建） - 创建和停止多个模拟载具链接。
- **Replay Flight Data** 重播飞行数据 - 重播遥测日志（用户指南）。
- **MAVLink Inspector** - 显示收到的MAVLink消息/值。
- **MAVLink Analyzer** - 绘制MAVLink消息/值的趋势图。
- **Custom Command Widget** 自定义命令小组件 - 在运行时加载自定义/测试QML UI。
- **Onboard Files** 板载文件 - 导航车辆文件系统和上载/下载文件。
- **HIL Config Widget** - HIL模拟器的设置.
- **MAVLink Console**（仅限PX4） - 连接到PX4 nsh shell并发送命令。

### 模拟链接

Mock Link允许您在QGroundControl调试版本中创建和停止指向多个模拟（模拟）载具的链接。

模拟不支持飞行，但允许轻松测试：

- 任务上传/下载
- 查看和更改参数
- 测试大多数设置页面
- 多个载具用户界面

对于任务上载/下载的单元测试错误情况尤其有用。

### 使用Mock Link

为了使用Mock Link：

1. 1.通过构建源来创建调试版本。

2. 通过选择顶部工具栏中的“应用程序设置”图标，然后选择侧栏中的“模拟链接”来访问“模拟链接”：

   ![img](https://dev.qgroundcontrol.com/master/assets/tools/mocklink_waiting_for_connection.jpg)

3. 1. 可以单击面板中的按钮以创建相关类型的车辆链接。

4. 每次单击按钮时，都会创建一个新连接。

5. 当存在多个连接时，将显示多车辆UI。

   ![img](https://dev.qgroundcontrol.com/master/assets/tools/mocklink_connected.jpg)

6. 1. 单击停止一个模拟链接以停止当前活动的车辆。

然后使用模拟链接或多或少与使用任何其他载具相同，只是模拟不允许飞行。

</br>

# 命令行选项

您可以使用命令行选项启动QGroundControl。 这些用于启用日志记录，运行单元测试以及模拟不同的主机环境以进行测试。

## 使用选项启动QGroundControl

您需要打开命令提示符或终端，将目录更改为存储qgroundcontrol.exe的位置，然后运行它。 每个平台如下所示（使用--logging：full选项）：

Windows命令提示符：

```bash
cd "\Program Files (x86)\qgroundcontrol"
qgroundcontrol --logging:full
```

OSX终端应用程序（应用程序/实用程序）：

```bash
cd /Applications/qgroundcontrol.app/Contents/MacOS/
./qgroundcontrol --logging:full
```

Linux终端：

```bash
./qgroundcontrol-start.sh --logging:full
```

## 选项

下表列出了选项/命令行参数。

| Option                                                    | Description                                                  |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| `--clear-settings`                                        | 清除应用程序设置(将QGroundControl恢复到默认设置)。           |
| `--logging:full`                                          | 打开完整日志记录。参见 [Console Logging](https://docs.qgroundcontrol.com/en/SettingsView/console_logging.html#logging-from-the-command-line). |
| `--logging:full,LinkManagerVerboseLog,ParameterLoaderLog` | 打开完整日志记录并关闭以下以逗号分隔的日志记录选项。         |
| `--logging:LinkManagerLog,ParameterLoaderLog`             | 打开指定的以逗号分隔的日志记录选项                           |
| `--unittest:name`                                         | (Debug builds only)运行指定的单元测试。去掉“:name”以运行所有测试。 |
| `--unittest-stress:name`                                  | (Debug builds only)连续运行指定的单元测试20次。去掉:运行所有测试的名称。 |
| `--fake-mobile`                                           | 模拟在移动设备上运行。                                       |
| `--test-high-dpi`                                         | 模拟在高DPI设备上运行                                        |

Notes:

- 单元测试自动包含在调试版本中（作为QGroundControl的一部分）。 QGroundControl在单元测试的控制下运行（它不能正常启动）。

</br>

# 自定义构建

自定义构建允许第三方创建自己的QGC版本，使其能够轻松跟上常规QGC中所做的更改。QGC内置了一个架构，允许自定义构建修改并添加到常规QGC的功能集中。

定制构建的一些可能性

- 为您的构建打造完全品牌化
- 定义单个外部测试版堆栈以避免携带不必要的代码
- 实现您自己的、自动驾驶仪和固件插件覆盖
- 实现您自己的相机管理器和插件覆盖
- 实现您自己的QtQuick接口模块
- 实现您自己的工具栏、工具栏指示器和 UI 导航
- 实现您自己的飞行视图叠加层（以及如何隐藏QGC中的元素，例如飞行小部件）
- 实现您自己的自定义QtQuick相机控制
- 实施您自己的自定义飞行前检查清单
- 为上述所有内容定义您自己的资源

QGC为任何支持mavlink的车辆提供通用支持，并为PX4 Pro和ArduPilot提供固件特定支持的缺点之一是用户界面的复杂性。由于QGC事先不知道有关您的车辆的任何信息，因此需要UI位，如果您驾驶的车辆仅使用PX4 Pro固件并且是多旋翼车辆，则可能会妨碍您。如果这是已知的事情，那么可以在不同的地方简化UI。此外，QGC既针对从头开始构建自己车辆的DIY用户，也针对现成车辆的商业用户。从头开始设置 DIY 无人机需要各种功能，而这些功能对于现成车辆的用户来说是不需要的。因此，对于现成的车辆用户来说，所有 DIY 特定的东西都只是他们需要查看的额外噪音。创建自定义构建允许您为车辆指定确切的详细信息并隐藏不相关的内容，从而为您的用户创建比常规通用 QGC 更简单的用户体验。

QGC 中有一个插件架构，允许这种自定义构建创建。它们可以在QGCCorePlugin.h，FirmwarePlugin.h和AutoPilotPlugin.h相关类中找到。要创建自定义构建，您需要创建标准插件的子类，覆盖适合您使用的方法集。

还有一种机制允许您覆盖资源，以便您可以更改 QGC 中较小的可视元素。

QGC内部还有“高级模式”的概念。而标准 QGC 构建始终在高级模式下运行。自定义构建始终以常规/非高级模式启动。构建中有一种更简单的机制可以打开高级模式，即相当快地连续单击 5 次飞行视图按钮。如果在自定义版本中执行此操作，则会警告您进入高级模式。这里的概念是隐藏普通用户不应该在高级模式后面访问的内容。例如，商用车将不需要访问大多数面向DIY设置的设置页面。因此，自定义构建可以隐藏这一点。自定义示例代码演示如何执行此操作。

如果您想了解可能性，第一步是通读那些记录可能性的文件。接下来查看[' custom-example '](https://github.com/mavlink/qgroundcontrol/tree/master/custom-example)源代码，包括[README](https://github.com/mavlink/qgroundcontrol/blob/master/custom-example/README.md)



## 自定义构建的初始存储库设置

在QGC和QGC的自定义构建版本上工作的建议机制是有两个单独的存储库。第一个存储库是你的主 QGC 分叉。第二个存储库是自定义生成存储库。

### 主要QGC存储库

此存储库用于处理对主线 QGC 的更改。在创建自己的自定义构建时，发现您可能需要对自定义构建进行调整/添加以实现所需的内容的情况并不少见。通过与 QGC 开发人员直接讨论这些需要的更改并提交拉取以使自定义构建架构更好，您可以使 QGC 对每个人来说更强大，并回馈社区。

创建此存储库的最佳方法是将常规 QGC 存储库分叉到您自己的 GitHub 帐户。

### 自定义构建存储库

这是您将进行主要自定义构建开发的地方。此处的所有更改都应在自定义目录中，而不是渗入常规的QGC代码库。

由于只能分叉一次存储库，因此创建此存储库的方法是在 GitHub 帐户中“创建新存储库”。不要向其添加任何其他文件，如 gitignore、readme 等。创建后，您可以选择设置存储库。现在您可以选择“从另一个存储库导入代码”。只需使用“导入代码”按钮导入常规 QGC 存储库即可。

## 自定义构建插件

为自定义构建自定义 QGC 的机制是通过现有的 和体系结构。通过在自定义构建中创建这些插件的子类，您可以更改 QGC 的行为以满足您的需求，而无需修改上游代码。`FirmwarePlugin``AutoPilotPlugin``QGCCorePlugin`

### QGCCorePlugin

这允许您修改 QGC 中与车辆没有直接关系但与 QGC 应用程序本身相关的部分。这包括应用程序设置、调色板、品牌等内容。



## 资源覆盖

QGC 源代码术语中的“资源”是指[在 qgroundcontrol.qrc](https://github.com/mavlink/qgroundcontrol/blob/master/qgroundcontrol.qrc) 和 [qgcresources.qrc](https://github.com/mavlink/qgroundcontrol/blob/master/qgcresources.qrc) 文件中找到的任何内容。通过覆盖资源，您可以将其替换为您自己的版本。这可以像单个图标一样简单，也可以像替换 qml UI 代码的整个车辆设置页面一样复杂。

请注意，使用资源覆盖并不像插件架构那样将您与上游 QGC 更改隔离开来。从某种意义上说，您直接修改了主代码使用的上游 QGC 资源。

### 排除文件

覆盖资源的第一步是将其从上游构建的标准部分中“排除”。这意味着您将在自己的自定义生成资源文件中提供该资源。有两个文件可以实现这一点：[qgroundcontrol.exclusion和qgcresources.exclusion](https://dev.qgroundcontrol.com/master/zh/custom_build/ResourceOverride.html)。它们直接与 *.qrc 对应项对应。要排除资源，请将资源行从 .qrc 文件复制到相应的 .exclusion 文件中。

### 排除资源的自定义版本

必须在自定义生成资源文件中包含重写资源的自定义版本。资源别名必须与上游别名完全匹配。资源的名称和实际位置可以位于自定义目录结构中的任何位置。

### 生成标准 QGC 资源文件的新修改版本

这是使用python脚本`updateqrc.py` 完成的。它将读取上游 `qgroundcontrol.qrc` 和`qgcresources.qrc`和相应的 exclusion files，并在自定义目录中输出这些文件的新版本。这些新版本将不包含指定要排除的资源。定制构建的构建系统使用这些生成的文件(如果存在的话)来代替上游版本进行构建。这些文件的生成版本应该添加到您的repo中。此外，每当您在自定义repo中更新QGC的上游部分时，您必须重新运行`python updateqrc.py`以生成新版本的文件，因为上游资源可能已经更改。

### 自定义构建示例

可以在存储库自定义构建示例中查看自定义生成 qgcresource 覆盖的示例：

- [qgcresources.qrc](https://github.com/mavlink/qgroundcontrol/blob/master/custom-example/qgcresources.exclusion)
- [custom.qrc](https://github.com/mavlink/qgroundcontrol/blob/master/custom-example/custom.qrc)

## 定制

以下主题说明如何自定义 UI 的各种视图和其他部分。

- [首次运行提示](https://dev.qgroundcontrol.com/master/zh/custom_build/FirstRunPrompts.html)
- [工具栏自定义](https://dev.qgroundcontrol.com/master/zh/custom_build/Toolbar.html)
- [飞视图定制](https://dev.qgroundcontrol.com/master/zh/custom_build/FlyView.html)

</br>

# 首次运行提示

当QGC第一次启动时，它会提示用户指定一些初始设置。在撰写本文档时，它们是:

- 单位设置-用户希望使用什么单位来显示。

- 离线车辆设置-未连接车辆时创建计划的车辆信息。

自定义构建体系结构包括用于自定义构建的机制，以覆盖这些提示的显示和/或创建您自己的首次运行提示。

## 首次运行提示对话框

每个首次运行提示符都是一个简单的对话框，可以向用户显示ui。特定的对话框是否已经显示给用户存储在一个设置中。下面是上游第一次运行提示对话框的代码:

- [Units Settings](https://github.com/mavlink/qgroundcontrol/blob/master/src/FirstRunPromptDialogs/UnitsFirstRunPrompt.qml)
- [Offline Vehicle Settings](https://github.com/mavlink/qgroundcontrol/blob/master/src/FirstRunPromptDialogs/OfflineVehicleFirstRunPrompt.qml)

## 标准的首次运行提示对话框

每个对话框都有一个唯一的ID。当该对话框显示给用户时，该ID被注册为已经显示过，因此它只发生一次(除非您清除设置)。包含在上游QGC中的首次运行提示符集被认为是“标准”集。QGC从` QGCCorePlugin::first strunpromptstdids` 调用中获取要显示的标准提示列表。

```cpp
    /// Returns the standard list of first run prompt ids for possible display. Actual display is based on the
    /// current AppSettings::firstRunPromptIds value. The order of this list also determines the order the prompts
    /// will be displayed in.
    virtual QList<int> firstRunPromptStdIds(void);
```

如果想隐藏其中一些，可以在自定义构建中重写此方法。

## 自定义首次运行提示对话框

自定义构建可以根据需要创建自己的一组额外的首次运行提示，通过使用以下QGCCorePlugin方法重写:

```cpp
    /// Returns the custom build list of first run prompt ids for possible display. Actual display is based on the
    /// current AppSettings::firstRunPromptIds value. The order of this list also determines the order the prompts
    /// will be displayed in.
    virtual QList<int> firstRunPromptCustomIds(void);
    /// Returns the resource which contains the specified first run prompt for display
    Q_INVOKABLE virtual QString firstRunPromptResource(int id);
```

您的QGCCorePlugin应该覆盖这两个方法，并为新的首次运行提示符的id提供静态常量。看看标准集是如何实现的，并采用相同的方法。

## 显示顺序

显示给用户的第一次运行提示的集合是按照 `QGCCorePlugin::first strunpromptstdids` 和 `QGCCorePlugin::first strunpromptcustomids` 返回的顺序，在自定义提示之前显示标准提示。只显示以前没有显示给用户的提示。

## 始终打开提示

通过在提示ui实现中设置 `markAsShownOnClose: false` 属性，你可以创建一个每次QGC启动时都会显示的提示符。这可以用于向用户显示使用提示之类的事情。如果您这样做，最好确保最后显示它。

</br>

# 定制工具栏

可以通过多种方式定制工具栏，以满足您的定制构建需求。工具栏内部由从左到右的几个部分组成:

- View Switching
- Indicators
  - App Indicators
  - Vehicle Indicators
  - Vehicle Mode Indicators
- Connection Management
- Branding

根据当前显示的视图，Indicators “指标”部分有所不同:

- Fly View - 显示所有指标
- Plan View - 不显示任何指示器，并且有自己的自定义指示器部分用于计划状态值
- Other Views - 不显示车辆模式指示

## 定制的可能性

### 指示器

您可以添加自己的指标显示或删除任何上游指标。您使用的机制取决于指示器类型。

#### App Indicators

这些向用户提供的信息与车辆无关。例如RTK状态。使用 `QGCPlugin::toolbarIndicators` 来操作应用程序指标列表

#### Vehicle Indicators

这些是与车辆信息相关的指示器。它们只有在车辆联网时才可用。要操作车辆指标列表，您可以覆盖 `FirmwarePlugin::toolIndicators` 。

#### Vehicle Mode Indicators

这些是与车辆信息相关的指示器。它们需要Fly View提供的额外UI来完成它们的操作。Arming and Disarming《武装与解除武装》就是一个例子。它们只有在车辆联网时才可用。要操作车辆模式指标列表，您可以覆盖`FirmwarePlugin::modeIndicators` 。

### 修改工具栏UI本身

这是通过在与工具栏关联的qml文件上使用资源覆盖来实现的。这提供了高级别的定制，但也增加了复杂性。工具栏的主要用户界面在 `MainToolBar.qml` 中。主窗口代码`MainRootWindow.qml`与工具栏交互，根据当前视图显示不同的指示符部分，以及模式指示符是否显示也基于当前视图。

如果你想完全控制工具栏，那么你可以重写`MainToolBar.qml`，并制作自己完全不同的版本。您需要特别注意主工具栏与`MainRootWindow.qml`的交互。因为您将需要在自己的自定义版本中复制这些交互。

工具栏有两个标准的指示器ui部分:

#### `MainToolBarIndicators.qml`

这用于除Plan之外的所有视图。以一行方式显示所有指标。虽然您可以覆盖这个文件，但实际上它除了为指示器布局之外并没有做什么。

#### `PlanToolBarIndicators.qml`

Plan视图使用它来显示状态值。如果您想更改ui，您可以覆盖该文件并提供您自己的自定义版本。

</br>

# 飞行视图定制

Fly View是这样设计的，它可以从简单到更复杂的多种方式进行定制。它被设计成三个独立的层，每个层都是可定制的，提供不同级别的更改

## Layers

- 从上到下视觉上有三层:
  - [`FlyView.qml`](https://github.com/mavlink/qgroundcontrol/blob/master/src/FlightDisplay/FlyView.qml) 这是ui和业务逻辑的基础层，用于控制地图和视频切换。
  - [`FlyViewWidgetsOverlay.qml`](https://github.com/mavlink/qgroundcontrol/blob/master/src/FlightDisplay/FlyViewWidgetLayer.qml) 这一层包括飞行视图的所有剩余小部件。
  - [`FlyViewCustomLayer.qml`](https://github.com/mavlink/qgroundcontrol/blob/master/src/FlightDisplay/FlyViewCustomLayer.qml) 这是一个图层，您可以使用资源覆盖来添加自己的自定义图层。

### Inset Negotiation using `QGCToolInsets`

Fly View的一个重要方面是，它需要了解它的地图窗口中间有多少中央空间，这些空间没有被窗口边缘的ui小部件所阻碍。当车辆从视野中消失时，它会利用这些信息平移地图。这不仅需要对窗口边缘进行操作，还需要对小部件本身进行操作，以便地图在进入小部件下面之前平移。

这是通过使用[` QGCToolInsets` ](https://github.com/mavlink/qgroundcontrol/blob/master/src/QmlControls/QGCToolInsets.qml)对象包含在每一层。该对象为每个窗口边缘提供插入信息，告知系统基于边缘的ui占用了多少空间。每个图层都通过`parentToolInsets`获得下面图层的插图，然后通过“toolInsets”报告新的插图，考虑到下面的图层和它自己的添加。然后将最终结果总插入值提供给地图，以便它可以做正确的事情。理解这一点的最好方法是查看上游和自定义示例代码。

### `FlyView.qml`

从ui交互和业务逻辑来看，视图的基础层也是最复杂的。它包括地图和视频的主要显示元素以及引导控件。尽管您可以覆盖此层，但不建议这样做。如果你这样做，你最好真的知道你在做什么。它是一个单独的层的原因是使上面的层更简单，更容易定制。

### `FlyViewWidgetsOverlay.qml`

这一层包含飞行视图的所有剩余控件。你可以通过使用[ `QGCFlyViewOptions `](https://github.com/mavlink/qgroundcontrol/blob/master/src/api/QGCOptions.h)隐藏控件。但是，为了更改上游控件的布局，您必须使用资源覆盖。如果你查看源代码，你会发现控件本身被很好地封装，因此创建自己的重写来重新定位它们和/或添加自己的ui应该不是那么困难。同时保持与控件的上游实现的连接。

### `FlyViewCustomLayer.qml`

这为Fly View提供了最简单的自定义能力。允许您添加ui元素，添加到现有的上游控件。上游代码没有添加ui元素，它是您自己的自定义代码的基础，用作此qml的资源覆盖。自定义示例代码为您提供了如何执行此操作的示例。

## 推荐规范

### 简单的定制

最好的开始是使用自定义层覆盖加上关闭widget层的ui元素(如果需要的话)。如果可能的话，我建议只坚持这样做。它提供了最大的能力，不会被下面层中的上游更改搞砸。

### 中等复杂度定制

如果你真的需要重新定位上游ui元素，那么你唯一的选择是重写 `flyviewwidgetoverlay .qml` 。通过这样做，您可以将自己与上游变化保持一定距离。尽管您仍然可以免费获得上游控件的更改。如果有一个全新的控件被添加到飞行视图的上游，你不会得到它，直到你把它添加到你自己的覆盖。

### 高度复杂的定制

最后也是最不推荐的定制机制是重写 `FlyView.qml` 。通过这样做，您将进一步远离免费获得上游更改

</br>

# 自定义构建的发布过程[WIP文档]

创建您自己的自定义构建的一个更棘手的方面是使其与常规QGC保持同步的过程。本文档描述了建议遵循的流程。但实际上，我们欢迎您为定制构建使用任何分支和发布策略。

## 上游QGC发布/分支策略

最好的起点是理解QGC用来发布自己版本的机制。我们将在此基础上分层定制构建发布过程。您可以在这里找到标准的QGC[发布过程](https://dev.qgroundcontrol.com/master/en/ReleaseBranchingProcess.html)。

## 自定义构建/发布类型

常规QGC有两种主要的构建类型:稳定版和每日版。自定义构建的构建类型更为复杂。在整个讨论中，我们将使用术语“上游”来指代主要的QGC回购(https://github.com/mavlink/qgroundcontrol)。此外，当我们谈论“新的”上游稳定版本时，这意味着一个主要/次要版本，而不是补丁版本。

### 同步稳定版

这种类型的发布与上游稳定版的发布同步。一旦QGC发布稳定版，您就可以发布基于此稳定版的自定义构建版本。此构建将包括来自上游的所有新功能，包括您自己的自定义代码中的新功能。

### 带外稳定版 Out-Of-Band Stable

这是自定义构建的后续版本，在发布同步稳定版之后，但在上游发布新稳定版之前。它只包括来自您自己的定制构建的新特性，不包括来自上游的新特性。这种类型发布的工作将发生在一个分支上，该分支要么基于您最新的同步稳定版本，要么基于您的最后一个带外发布版本(如果存在的话)。您可以在第一个同步稳定版本之后的任何时间发布带外稳定版本。

### 每日版 Daily

您的自定义每日构建是从您的分支构建的。重要的是要使您的定制主与QGC主保持同步。如果你落后了，你可能会对上游特性感到惊讶，这些特性需要一些努力才能集成到你的构建中。或者您甚至可能需要更改“核心”QGC以便与您的代码一起工作。如果您不尽快让QGC开发团队了解情况，可能会导致更改为时已晚。

## 第一次构建的选项

### 从同步稳定版本开始

建议您从已发布的同步稳定版开始。这不是必须的，但这是最简单的开始方式。要设置自己的同步稳定，您需要创建自己的开发分支，该分支基于上游当前稳定。

### 从每日构建开始

你可能会认为这是你的起点，因为你需要的功能，只有在上游主为您自己的定制构建。在这种情况下，您将不得不忍受发布自定义的每日构建，直到下一个上游稳定版本。这时你会发布第一个同步稳定版本。对于这种设置，您可以使用主分支，并在开发时使其与上游主分支保持同步。

## 在你发布第一个同步稳定版之后

### 补丁发布

由于上游QGC在稳定版上发布补丁，您也应该基于上游版本发布自己的补丁版本，以使您的稳定版与最新的关键错误修复保持同步。

### 带外版, 每日版: One or the other or both?

此时，您可以决定要遵循哪种类型的发布。你也可以决定两者都做。你可以使用带外发行版来实现更小的新功能，而不需要新的上游功能。你可以做一些主要的新功能工作，比如daily/master，直到你可以做一个新的同步稳定版。

</br>

# MAVLink 定制

QGC使用[MAVLink](https://mavlink.io/en/)与飞行堆栈通信，这是一种为无人机生态系统设计的非常轻量级的消息传递协议。QGC默认包含[ArduPilotMega.xml](https://mavlink.io/en/messages/ardupilotmega.html)语言，它允许它与PX4和Ardupilot (PX4使用[common.xml](https://mavlink.io/en/messages/common.html)，它包含在ArduPilotMega中)进行通信。

为了增加对一组新消息的支持，您最终需要将它们添加到或分叉*QGroundControl*并包含您自己的语言。`ArduPilotMega.xml` `common.xml`

To do this:

- 替换预构建的C库[/qgroundcontrol/libs/mavlink/include/mavlink](https://github.com/mavlink/qgroundcontrol/tree/master/libs/mavlink/include/mavlink)。

  - 默认情况下，这是一个子模块导入https://github.com/mavlink/c_library_v2

  - 你可以改变子模块，或[建立自己的库](https://mavlink.io/en/getting_started/generate_libraries.html)使用MAVLink工具链。

- 当运行*qmake*时，您可以通过在[`MAVLINK_CONF`](https://github.com/mavlink/qgroundcontrol/blob/master/QGCExternalLibs.pri#L52)中设置所使用的整个语言。





# 参考链接：

[Overview · QGroundControl Developer Guide](https://dev.qgroundcontrol.com/master/en/)
