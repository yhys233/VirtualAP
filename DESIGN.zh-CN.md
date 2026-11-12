# DESIGN.md

[English](DESIGN.md) | [简体中文](DESIGN.zh-CN.md) | [繁體中文](DESIGN.zh-TW.md) | [Русский](DESIGN.ru.md) | [Français](DESIGN.fr.md) | [Deutsch](DESIGN.de.md) | [日本語](DESIGN.ja.md) | [한국어](DESIGN.ko.md)

VirtualAP Android 应用的视觉语言，也就是从 Droidspaces 沿用过来的设计语言
以及实现它的那些组件。后端是一个没有 UI 的 shell 脚本，所以这些都不适用于
`backend/`。

组件优先，数值其次：
[CONTRIBUTING.zh-CN.md](./CONTRIBUTING.zh-CN.md) 中的复用清单列出了已有的东西，本文件用于处理
清单里没有覆盖你所需要的东西的情形。绝不要写死一个十六进制颜色、
一个 sp 字号，或者一个你凭眼睛挑的圆角。

## 应用长什么样，以及为什么

这个应用里没有任何东西是抬升的。深度通过把表面颜色提升一级并画上 1dp 边框来表达；
如果你伸手去拿阴影，你就已经离开了这套设计。`Surface` 是基本单元：按钮是可点击的 Surface，
而不是 Material 的 `Button`，对话框是 `Dialog { Surface { } }`，而不是 `AlertDialog`。
Material 组件并未被禁止，但它们自带高度和圆角默认值，
要用就得给出理由。

颜色是推导出来的，绝不是写死的。`ui/theme/Theme.kt` 用三种配色颜色混合出完整的配色方案，
或者整体采用 Material You 的方案。动态颜色默认开启，
为旧设备和关掉它的用户提供了回退配色，而 AMOLED 会把每个表面压成黑色，
所以一个写死的 `Color(0xFF...)` 会同时破坏这四条路径。

边框和着色承载状态，填充保持安静。一个正在运行的接入点不是一张绿色卡片；
它是一张普通卡片，配一个着色的按钮和一条强调色边框。

## 颜色

角色来自 `MaterialTheme.colorScheme`。配色方案本身在 `ui/theme/Theme.kt` 中
由 `ui/theme/Color.kt` 里的配色构建，这是仅有的两个直接写出颜色值的文件。

| 表面 | 角色 |
| --- | --- |
| 界面背景 | `background` |
| 位于背景之上的卡片 | `surfaceContainer` |
| 嵌套在另一表面内的卡片或胶囊，以及可交互的设置行 | `surfaceContainerHigh` |
| 对话框和底部面板的外壳 | `surfaceContainer` |

| 文本 | 角色 |
| --- | --- |
| 主要 | `onSurface` |
| 次要、辅助、元数据 | `onSurfaceVariant`，alpha 为 `0.7f` |
| 三级、刻意低调 | `onSurfaceVariant`，alpha 为 `0.6f` |
| 禁用内容 | `onSurface`，alpha 为 `0.38f` |
| 禁用容器的填充 | `onSurface`，alpha 为 `0.12f` |

边框始终是 1dp。变化的是 alpha：

| 边框 | 颜色 |
| --- | --- |
| 卡片 | `outlineVariant`，`0.35f` |
| 对话框外壳 | `outlineVariant`，`0.4f` |
| 操作胶囊的外层容器及其他嵌套表面 | `outlineVariant`，`0.2f` |
| 分隔线（`HorizontalDivider`） | `outlineVariant`，`0.3f` |
| 文本输入框，未聚焦 | `outlineVariant`，`0.5f` |
| 文本输入框，已聚焦 | `primary`，`0.8f` |
| 着色按钮，与其强调色一致 | 对应的强调色角色，`0.2f` |

### 这些颜色的含义

接入点的生命周期在整个应用中是一致的，新状态应当加入这套体系，
而不是另造一套平行词汇。

| 状态 | 强调色 |
| --- | --- |
| 运行中、健康、启动操作 | `primary` |
| 已停止、失败、破坏性操作、停止操作 | `error` |
| 重启操作 | `secondary` |
| 进行中、正在重启、正在下载 | `tertiary` |
| 空闲、未知、无可报告 | `onSurfaceVariant`，`0.6f` |
| 禁用 | 填充 `onSurface`，`0.08f`，强调色 `outlineVariant` |

着色按钮的填充取自对应的容器角色、alpha 为 `0.4f`，例如启动用
`primaryContainer.copy(alpha = 0.4f)`，停止用 `errorContainer.copy(alpha = 0.4f)`，
再加一条 1dp、alpha 为 `0.2f` 的强调色边框。

控制台把 `[WARN]` 行着成 `tertiary`，把 `[ERROR]` 行着成 `error`，依据是
`classifyLine` 从标签读出的级别；后端只输出纯文本，不用转义码。

## 字体

字阶是 Material 3 自己的，即 `ui/theme/Type.kt` 中的 `Typography()`。使用具名样式，
并在使用处传入字重：标题是 `titleLarge` 配 `FontWeight.Bold`，卡片标题
是 `titleMedium` 配 SemiBold。应用里任何地方都没有 `fontSize` 覆盖。

UI 文本使用 `FontFamily.Default`。任何由机器产生的东西，日志行、配置值、
路径，都用 JetBrains Mono，从 `ui/theme/Type.kt` 导入。不要声明局部的
字体族。

| 元素 | 样式 |
| --- | --- |
| `TopAppBar` 中的界面标题 | `titleLarge`，Bold |
| 对话框标题 | `titleLarge`，Bold，左对齐 |
| 卡片标题 | `titleMedium`，SemiBold |
| 界面内的分节标题 | `titleSmall`，Bold，`primary` |
| 按钮文字 | `labelLarge`，SemiBold |
| 对话框中的正文 | `bodyMedium` |
| 帮助文本、元数据、副标题 | `bodySmall`，`onSurfaceVariant`，`0.7f` |
| 状态标签文字 | `labelSmall`，Black，`letterSpacing = 0.5.sp`，大写 |
| 机器输出 | `bodySmall`，使用 JetBrains Mono 字族 |

## 间距

字阶是 **4、8、12、16、20、24、32**。没有别的。如果某个间隙感觉想要 14，
那它想要的是 12 或 16。

| 间距 | 数值 |
| --- | --- |
| 界面边缘，列表和仪表盘 | 水平 16 |
| 界面边缘，向导和设置流程 | 水平 24 |
| 列表中卡片之间 | 16 |
| 卡片内区块之间 | 12 |
| 卡片内边距 | 16 |
| 对话框内边距 | 24 |
| 图标到其标签、芯片到芯片 | 8 |
| 紧凑的元数据行 | 4 |

没有 tab 栏。主界面位于 `Scaffold` 内的 `PrimaryActionBottomBar` 之上，
所以该栏量出的高度会作为 Scaffold 的内边距传入，列表不必有人写一个让位数值
就能避开它。保持这样：一个不是 Scaffold `bottomBar` 的栏
需要写一个具体数值，而这个数值会漂移。

### 卡片头部对齐

当前网络卡片没有标题：它的第一行左边是网络名称、右边是状态标签，位于分隔线之上，
最小高度为 `CardHeaderHeight`，内边距位于 `CardContentPadding` 之内，
两者都来自 `ui/component/CardMetrics.kt`。高度是 48dp，
因为这一行里有 Wi-Fi 二维码图标按钮，而 48dp 是触控底线。

确实带标题和标签头部的卡片使用同样的两个数值，这样它的分隔线和标签
与前者处在同一条线上。如果你改动它们，你就是在移动应用中每一张卡片头部，
而这正是预期的方向。

## 形状

| 元素 | 圆角 |
| --- | --- |
| 卡片 | 20 |
| 对话框外壳 | 24 |
| 底部面板，仅顶部两角 | 24 |
| 全宽主按钮 | 20 |
| 下拉菜单 | 20 |
| 操作按钮 | 16 |
| 文本输入框 | 16 |
| 操作胶囊的外层容器 | 12 |
| 状态标签、小徽标 | 8 |

## 操作胶囊

一行相关的操作放在一个着色的外层容器里：

```kotlin
Surface(
    shape = RoundedCornerShape(12.dp),                    // wrapper
    color = MaterialTheme.colorScheme.surfaceContainerHigh,
    border = BorderStroke(1.dp, MaterialTheme.colorScheme.outlineVariant.copy(alpha = 0.2f))
) {
    Row(Modifier.padding(4.dp), horizontalArrangement = Arrangement.spacedBy(4.dp)) {
        Surface(
            modifier = Modifier.weight(1f).height(48.dp),
            shape = RoundedCornerShape(16.dp),            // button, larger than the wrapper
            color = accentContainer.copy(alpha = 0.4f),
            border = BorderStroke(1.dp, accent.copy(alpha = 0.2f))
        ) { /* icon + labelLarge SemiBold */ }
    }
}
```

按钮比它们所在的盒子更圆，这就是设计意图：4dp 的内缩加上
更大的内圆角，让按钮看起来是凸出于外层容器之上，而不是平贴着嵌进去。
把两个圆角调成一样会抹平这个效果。

主界面上的接口或容器切换（`ui/screen/MainScreen.kt` 中的 `SegmentedSelector`）
就是这套几何用作选择器：选中的段取着色填充和强调色边框，
另一段保持透明。
日志控制台上方的 Clear 与 Copy 行（`LogActionRow`）不是这样：
它是两个 38dp 按钮的裸行，没有外层容器。

## 对话框

每个对话框都是 `DsDialog`，它既管外框也管布局：24dp 边距内一个全宽的 `Surface`，
高度上限为屏幕高度减去这些边距，内部是一个 24dp 内边距的列，
其主体可滚动。这个上限来自 `LocalConfiguration`，而不是窗口，
所以旋转时打开的对话框是按实时配置来度量的。调用方不设置宽度、
自己的内边距或自己的滚动，也绝不传 `wrapContentHeight()`；每个对话框特有的东西
放在 `modifier` 上，例如给文本输入框的 `imePadding()`，或者 `heightIn(max = ...)` 限制。

标题是 `titleLarge` Bold、左对齐，并且是内容里的第一个元素。

## 对话框操作

关于页是一个信息页，用 `DialogCloseButton` 从头部关闭。带选择的对话框，
例如语言选择器，以 `DialogFooterRow` 结束：两个等权重的 48dp 按钮、圆角 16，
一个低调的色调式取消和一个填充式确认，单行标签会省略号截断。它放在 `DsDialog` 的
`footer` 槽位，在可滚动主体之前测量，这样横屏也无法挤压它。
第一个破坏性对话框照搬了 Droidspaces 的 `error` 确认按钮。

## 图标与触控目标

| 用途 | 尺寸 |
| --- | --- |
| 卡片或列表行的前置图标 | 20 |
| 工具栏操作、按钮内图标 | 18 |
| 与 `bodySmall` 文本同行 | 16 |
| 对话框头部 | 24 |
| 设置页主图形 | 96，位于 240 的主图形区内 |
| 状态圆点 | 6 |

任何可点击的东西至少 **48dp**。这是无障碍的底线，不是风格偏好，
也是本文件中唯一一条设计论证不能推翻的规则。

## 对齐

行使用 `Alignment.CenterVertically`。

卡片标题和对话框标题始终左对齐。把标题与控件配在一起的卡片行
使用 `Arrangement.SpaceBetween`。

居中文本只用于整屏状态：空列表、加载、设置页主图形。如果内容
在卡片内部居中，那多半是错的。

## 动效与加载

转圈指示器来自 `LoadingIndicator` 配一个 `LoadingSize`，绝不在进度指示器上
直接写 `.size(n.dp)`。整屏加载用 `FullScreenLoading`；`LoadingSize.Hero`（144）
是设置流程中唯一的主图形加载器。动画时长来自 `AnimationUtils`；主图形状态变化
使用 `MaterialTheme.motionScheme`，主题把它设为 `MotionScheme.expressive()`。

### 设置流程

欢迎翻页、root 检查和后端安装器共用一套几何，它位于
`ui/component/SetupHero.kt`。`SetupPage` 是 Scaffold 之下的骨架：上方一个 40dp 的行
（欢迎翻页上是页码，其他地方为空）、中间是 frame 槽位，它会把内容居中，
或者在窗口过矮时滚动，下方是一个 44dp 的行（页码圆点）。
每个界面都保留这两行，这样 frame 和按钮在不同界面之间从不移动。
`SetupFrame` 是那个 frame：一个 240dp 的主图形盒，然后是标题和文案，放在固定的
文本块里。`SetupHero` 是真正做事的界面的主图形：一个 240dp 的
`MaterialShapes.Cookie12Sided`，颜色为 `primaryContainer` 或 `errorContainer`，里面放
加载指示器或一个 96dp 的图形，状态之间只有它的颜色和内容变化。欢迎翻页的主图形
则用它自己的 `Shape`，一个柔和的爆裂形状，随着页面滑过逐渐变成
饼干、四叶草和太阳。`SetupActionBar` 是唯一的按钮。用这些来搭建新的设置界面，
主图形、标题和按钮就会和其他界面落在同样的行上。

## 当规则不适用时

有时它确实不适用，而答案不是悄悄换一个数字。

在 PR 中说明，并在相应位置留一条注释，解释偏离带来了什么，
就像下面那些已决定的例外一样。未被记录的偏离就是漂移，下一位贡献者
会去“修好”它。被记录的偏离就是一个决策，它能活下来。

## 已决定的例外

看过，并且刻意保持原样。没有原决策没考虑到的理由，
不要重新讨论这些。

- **二维码底板是 `Color.White`**，在 Wi-Fi 二维码面板中，且 `QrCodeGenerator.encode` 在白色上
  绘制黑色模块。扫描器需要规范所假定的对比度；深色配色下的
  主题化底板会被读成一张坏掉的码。
- **状态卡片会响应点击**（`ui/screen/MainScreen.kt` 中的 `ActiveNetworkCard`），尽管它
  什么也不打开。它是用户为了确认应用还活着而去戳的那张卡片，所以它会泛起涟漪并
  让输入框失去焦点，而不是在手指下毫无反应。代价是 TalkBack 会把它读成一个
  可操作的节点。
- **Wi-Fi 二维码面板、设备面板和日志面板**位于
  `ui/screen/MainScreen.kt`，以及 `ui/component/DeviceList.kt` 中的设备列表面板，
  都仍然是 Material 的 `ModalBottomSheet`，而不是
  `Dialog { Surface { } }`。它们全都是属于底部边缘的
  拖动关闭内容，而日志面板在命令运行时阻止手势关闭，
  这样面板就不会留下一个吞掉触摸的遮罩。它们的外壳是
  `surfaceContainer`，顶部圆角 24dp，与形状表一致。
- **设置流程的按钮**（`ui/component/SetupHero.kt` 中的 `SetupActionBar`）是一个完整的胶囊，
  没有栏表面也没有分隔线，不是那种 20dp 的栏按钮。它单独位于主图形下方的
  背景上，而不是覆盖在滚动内容之上。它的禁用颜色与启用颜色保持一致：
  它只在校验或安装进行中时才禁用，而为了那一瞬间把它变灰
  看起来像闪烁。
- **0.03 和 0.06 这种极淡的填充 alpha** 在 `LogActionRow` 中保持不共享。它们单独看
  是看不见的，把它们归并到一起就意味着为没人能看见的东西
  发明一个 token。
