# 为 VirtualAP 做贡献

[English](CONTRIBUTING.md) | [简体中文](CONTRIBUTING.zh-CN.md) | [繁體中文](CONTRIBUTING.zh-TW.md) | [Русский](CONTRIBUTING.ru.md) | [Français](CONTRIBUTING.fr.md) | [Deutsch](CONTRIBUTING.de.md) | [日本語](CONTRIBUTING.ja.md) | [한국어](CONTRIBUTING.ko.md)

不存在的功能好过坏掉的功能。VirtualAP 以 root 身份运行在
已 root 的手机上，运行在那些对基本问题都各执一词的驱动的 Wi-Fi 芯片上，所以一个补丁
在你的手机上能用、在别人的手机上却弄坏了东西，那是回归，不是贡献。而且
十行能完成工作的代码胜过一百行做同样事情的代码：先删再加，先复用
再动手写。

## 必须成立的条件

任何人改动本仓库（包括助手）都要遵守的规则在
[AGENTS.zh-CN.md](./AGENTS.zh-CN.md)：风格、提交格式、关键关口和硬性
约束。简而言之：只做 Android，引擎是 musl 上的单线程 C，直接与
内核对话，一个模块只负责一件事，应用要能在 Android 8 上运行，二进制只能
来自 `scripts/build-static.sh`，而 `virtualap` 还来自根目录的 `Makefile`。

VirtualAP 运行在 Qualcomm、Broadcom FullMAC（Samsung）和 MediaTek 芯片上。针对某一
芯片怪癖的修复，只有在后端于运行时适配它时才被接受：通过检测、
条件分支或者回退，且不改变其他芯片上的行为；
`nl80211_sta_freq` 读取所关联 BSS 的频率，因为 Samsung 省略了信道，
以及 `ensure_5g_ir` 只在 phy 报告 NO-IR 时才触发，就是这种范式。如果修复无法这样门控，它
就属于一个分支版本。要在真机上测试，Wi-Fi 芯片没有模拟器，并在
你的 PR 中说明你测试了哪些设备、驱动、频段和上游；未经测试的说法算作
未测试。

## 构建

```
./scripts/build-static.sh    # from the repository root: both arches, needs Docker
./build.sh [release]         # from Android/: the APK, needs JDK 17 or newer, ANDROID_HOME and musl-cross gcc
```

脚本做了什么、CI 做了什么，见
[How-It-Works.zh-CN.md](./Documentation/How-It-Works.zh-CN.md) 的 Building 一节。发布就是一个标签：`make bump NEW=x.y.z`，
以 `VirtualAP: bump vX.Y.Z` 提交，然后推送 `main` 和标签 `vX.Y.Z`。

## 代码风格

四条规则在 AGENTS.md 的 Style 一节。在此之上：

- 后端：coreutils 使用 `$ECHO`、`$GREP`、`$SED` 等，绝不裸写，也绝不加引号。
  输出用 `log`、`warn`、`error`。子命令是 `cmd_*`，Droidspaces 辅助函数是 `ds_*`，
  射频探测是 `phy_*`，策略是 `pick_*` 和 `resolve_*`。拆卸与装配逐步对应，
  每一步都容忍不存在。每个守护进程都记录到 `logs/` 下自己的文件。
- 应用：状态放在 ViewModel 里，不放在有状态的 composable 里，也不放在 `util`
  单例里。列表用 `LazyColumn` 和稳定的 key 渲染。颜色来自
  `MaterialTheme.colorScheme`，字体来自 `MaterialTheme.typography`，动画时长来自
  `AnimationUtils`，其他所有视觉数值来自 [DESIGN.zh-CN.md](./DESIGN.zh-CN.md)。任何到达
  root shell 的值都要经过 `Backend.quote()` 或白名单校验器。

提交都要签名，并按 AGENTS.md 的 Commits 一节加前缀。

## 翻译

译者在 Weblate 上工作，它直接提交到 `values-<lang>/` 目录；一个 pull request
只改动 `res/values/` 里的英文。各组件如何配置，见
[How-It-Works.zh-CN.md](./Documentation/How-It-Works.zh-CN.md) 的 Translations 一节。

新增一个字符串：

- 把它放进 `strings.xml` 中对应界面的注释之下，如果数量会改变措辞，就放进
  `plurals.xml`。Kotlin 通过 `stringResource` 读取它，在组合
  之外则用 `getString`。
- 占位符是位置式的，`%1$s`，这样译文可以调整它们的顺序。要说明每个占位符
  是什么，就在字符串正上方的注释里写清楚，Weblate 会把这条注释展示给译者；
  对可能有歧义的短词也这样做。
- 一个字符串只放一整句话。绝不要拼接翻译片段。
- 路径、命令、偏好键以及引擎输出的内容留在代码里，保持英文。文件里
  任何语言都不会改动的值，例如产品名或示例地址，标为
  `translatable="false"`。

## 复用清单

写任何新东西之前，先用 grep 搜这份清单。如果已有接近的东西，就扩展它，而
不是再加一个同类实现。Android 路径相对于
`Android/app/src/main/java/com/virtualap/app/`；引擎模块是 `src/` 下的路径，
每个目录的头文件在 `src/include/<folder>/` 下。
这份清单回答“我该用什么”；当这里没有合适的东西时，DESIGN.md 回答“它应该长什么样”。
共享组件是从 Droidspaces 移植来的；当它的其他组件出现调用方时，
复制上游文件，而不是自己写一个新的。

表单与输入：

- `ui/component/DsDropdown.kt` 中的 `DsDropdown(label, selected, options, displayName, onSelect, ...)`：
  任何选择字段。绝不要手写 `ExposedDropdownMenuBox`。
- `ui/component/DsTextFieldDefaults.kt` 中的 `DsTextFieldDefaults.colors()`：每个
  `OutlinedTextField`。
- `ui/util/FocusUtils.kt` 中的 `rememberClearFocus()`（`DsDialog` 使用它）和
  `ClearFocusOnClickOutside`（主界面的根）：在外部点击时收起键盘。

对话框与面板：

- `ui/component/DsDialog.kt` 中的 `DsDialog(onDismiss, modifier, footer) { }`：每个对话框。
  绝不要设置宽度、内边距或滚动。
- `ui/component/DialogFooterRow.kt` 中的
  `DialogFooterRow(dismissLabel, confirmLabel, onDismiss, onConfirm, confirmEnabled)`：
  带选择的对话框里的两个按钮，位于 `footer` 槽位。
- `ui/component/DialogCloseButton.kt` 中的 `DialogCloseButton(onClick, enabled)`：头部行里的
  36dp 关闭方块，用于从顶部关闭的内容。
- `ui/component/LogActionRow.kt` 中的 `LogActionRow(logs, isBlocking, onClear)`：日志控制台上方的
  Clear 与 Copy 行；`onClear = null` 时只有 Copy。
- `ui/component/DeviceList.kt` 中的 `DeviceRow`、`DeviceSheet(title, onDismiss, content)`、
  `BlockedList`、`AllowedList`、`BlockedDevicesSheet`、`AllowedDevicesSheet` 和
  `rememberDeviceAction(reload)`：一台设备及其操作、它们所在的面板、两个列表，
  以及执行对它们的改动并把失败说出来。
- `ui/component/AddDeviceDialog.kt` 中的 `AddDeviceDialog` 和 `AllowOnlyDialog`：只接受十六进制数字的
  MAC 字段，以及只允许白名单设备加入时谁会被断开。
- `ui/component/Caption.kt` 中的 `Caption(text)`：一行次要文本。
- Wi-Fi 二维码面板、设备面板和日志面板是 `ui/screen/MainScreen.kt` 私有的，
  关于对话框和语言选择器则属于 `ui/screen/SettingsScreen.kt`。不要导入或复制它们。

栏、设置流程与反馈：

- `ui/component/PrimaryActionBottomBar.kt` 中的
  `PrimaryActionBottomBar(label, icon, onClick, ...)`：启动和停止栏。接受
  `content` 槽位的重载用于会替换自身内容的按钮。
- `ui/component/SetupHero.kt` 中的 `SetupPage`、`SetupFrame`、`SetupHero`、`SetupActionBar`：
  设置流程的每个界面、欢迎翻页、root 检查和安装器。见
  DESIGN.md 的“The setup flow”一节。
- `ui/component/PullToRefreshWrapper.kt` 中的 `PullToRefreshWrapper(onRefresh) { }`：任何
  下拉刷新的界面主体。
- `ui/component/ToggleCard.kt` 中的 `ToggleCard`：单独成卡的开关行，用于主界面的选项。
  `ui/component/SwitchItem.kt` 中的 `SwitchItem`：分组表面内部的扁平开关行，
  用于设置界面。
- `ui/component/StatusPill.kt` 中的 `StatusPill(label, color)`：任何小的状态标签。
  `ui/component/SectionHeader.kt` 中的 `SectionHeader(text)`：一组卡片上方的标题。
- `ui/component/CardMetrics.kt` 中的 `CardContentPadding`、`CardHeaderHeight`：任何带标题和
  状态标签头部的卡片，这样分隔线才能对齐。
- `ui/util/LoadingIndicator.kt` 中的 `LoadingIndicator(size, color)` 与 `LoadingSize`，以及
  `FullScreenLoading(message)`：每个转圈指示器和每个整屏加载状态。
- `ui/component/TerminalConsole.kt` 中的 `TerminalConsole(logs, isProcessing, maxHeight)` 和
  `ShimmerAnimation(enabled) { }`：滚动的日志视图及其外框。

主题：

- `ui/theme/Theme.kt` 中的 `VirtualAPTheme(darkTheme, dynamicColor, amoledMode, themePalette)`：
  唯一的主题根，在 `MainActivity` 中应用。
- `ui/theme/ThemeStateHolder.kt` 中的 `rememberThemeState()` 和 `ThemeState`：实时的主题
  偏好设置。
- `ui/theme/Color.kt` 中的 `ThemePalette`：新增强调色配色，只在这里，不在别处。
  `Theme.kt` 和 `Color.kt` 是唯一直接写出颜色值的文件。
- `ui/theme/Type.kt` 中的 `MaterialTheme.typography.*` 和 `JetBrainsMono`：所有文本样式，
  以及用于日志和代码文本的等宽字体。
- `util/AnimationUtils.kt` 中的 `AnimationUtils`：补间规格。绝不要写死 `tween(300)`。
- `ui/component/` 中的 `AccentColorPicker` 和 `ColorPaletteSwatch`：配色选择器。
- 没有间距或形状的 token 对象；这些数字写在 DESIGN.md 里，并以字面 dp
  的形式书写。

导航：

- `MainActivity.kt` 中的 `Screen` 以及对它的 `when`：选择顶层界面的唯一位置，先看欢迎标志，
  再看 root 和安装状态，最后看设置标志。
  新增界面放在那里。

Shell、root 与后端：

- `util/Backend.kt` 中的 `Backend.quote(value)` 和 `Backend.startAp`：每个到达 root 命令的动态值，
  以及引擎的命令前缀（并回退到旧版本释放出的脚本，
  直到设置流程替换它）。
- `util/APManager.kt` 中的 `APManager.getStatus/start/stop/getInterfaces/getCaps/isInstalled`：
  驱动 `virtualap`，以及直接向 Droidspaces 询问正在运行的容器的 `getContainers`/`parseContainers`。
  绝不要手工拼装这些命令中的任何一条。
- `util/Hotspot.kt` 中的 `Hotspot.start(cfg)/stop()/refresh()`、`phase`、`status`、`actionLogs`：
  从任何地方启动或停止 AP 并读取会话状态；它还会发出运行中的通知
  并唤醒磁贴。
- `util/APConfig.kt` 中的 `APConfig` 及其 `isValid()`、`isValidIpv4`、`isValidDnsList`、`channelsForBand`：
  保存的热点配置、唯一的启动校验，
  以及各字段与它共用的地址检查。
- `util/BackendLogger.kt` 中的 `BackendLogger` 和 `classifyLine(line)`：AP 调用所写入的日志汇，
  以及 `[ERROR]`/`[WARN]` 级别标注。
- `util/PreferencesManager.kt` 中的 `PreferencesManager`：应用自己的设置。
  `APManager.loadConfig/saveConfig`：热点配置，在 `ap.conf` 中整份读写。
  `util/Constants.kt` 中的 `Constants`：每个路径和偏好键。
- `util/RootChecker.kt` 中的 `RootChecker.checkRootAccess(reprompt)` 和 `RootStatus`；
  `util/VirtualAPInstaller.kt` 中的 `VirtualAPInstaller.deviceArch/bundledPayloadVersion/payloadUpdateAvailable/install`；
  `util/ModuleInstaller.kt` 中的 `ModuleInstaller.install`，即开机模块；
  `util/QrCodeGenerator.kt` 中的 `QrCodeGenerator.wifiPayload/encode`，
  唯一的 zxing 调用方。
- `HotspotTileService`、`HotspotStopReceiver` 和 `HotspotBootReceiver`：快捷设置开关、
  通知里的停止操作，以及开机模块报告它已启动热点的消息。
  它们都不持有自己的状态。
- `util/LocaleHelper.kt` 中的 `LocaleHelper.availableLanguages/currentLanguage/changeLanguage`：
  应用的语言，`AppCompatDelegate.setApplicationLocales` 的唯一调用方。
- `AppViewModel` 负责欢迎标志、root 状态和安装状态；`APViewModel` 负责可编辑的配置、
  接口和容器列表以及被跟随的日志，并把启动和停止转发给
  `Hotspot`。3 秒的状态轮询在 `MainScreen` 中，绑定到它的生命周期。
- libsu 的全局配置在 `VirtualAPApplication.kt` 中，且只在那里。

引擎（`src/`）：

- `core/`：`proc`（每次程序运行、pid 文件，以及让一次只有一个启动或
  停止运行的锁）、`io`（日志：`vap_log`、`vap_warn`、`vap_error`、`log_stamp`；
  `read_file`、`write_file`、`write_atomic`；解析器 `valid_ip4`、`prefix_parse`、
  `subnet_from_gw`，以及用于 `-d` 的逗号列表的 `csv_each`）。
- `cli/`：`options`（`vap_keys` 表、`opt_set`、`opts_parse`、`print_usage`）、`conf`
  （`ap.conf` 和 `run.state`）。
- `net/`：`nl`（rtnetlink 传输层、`nl_dump`，以及命名空间辅助函数 `netns_open`、
  `in_netns`）、`link`（链路、地址和邻居）、`route`（路由、策略规则和 netd 的表：
  `upstream_resolve`、`upstream_v6_iface`）、`ipt/`，它被单独分出来，因为在 musl 上
  内核的 netfilter 头文件和 libc 的网络头文件无法共存于同一个文件：
  `ipt`（`ip_tables` 引擎）、`ipt_rules`（FORWARD、MASQUERADE、存在性检查、
  NAT 探测、`ipt_bin`、TTL 修正）。
- `radio/`：`driver`（通过 genetlink 的 nl80211：创建和删除 ap0、phy、站点的频率、
  宽度；wpa 控制套接字；`wifi_mac`，出厂 MAC）、`channel`
  （频段、信道和宽度策略，以及 `ensure_5g_ir`）。
- `mode/`：`ap`（ap0、守护进程、射频规划、两种模式共用的信号保持，
  以及 `hostapd` 和 `dnsmasq`：它们的配置和启动、客户端计数）、
  `routed`（含两种地址族都用到的固定策略规则）、`ipv6`（`ipv6_decide`、装配、拆卸、
  NAT66 以及共用的 /64 及其客户端发现）、`managed`（Droidspaces 容器：它的 pid、
  OpenWrt 的供给配置、RPS 掩码）、
  `maclist`（被阻止和允许的设备的文件，也就是 `hostapd` 的 `deny_mac_file` 和
  `accept_mac_file`，以及通知正在运行的 `hostapd`）。
- `monitor/`：`monitor`（守护进程的循环）、`upstream`（`reprobe`）。
- `cmd/`：应用调用的命令，由 `main.c` 分发：`start`（`start` 和
  `stop`）、`status`（只读的那些：`status`、`clients`、`interfaces`、`caps`）
  以及 `block`（`block`、`unblock`、`blocked`、`allow`、`disallow`、`allowed`、`allow-only`）。

## 发起 PR

Fork，在分支上工作，对着 `main` 发起 PR，并保持响应：无人回应的 PR 会被关闭，
反馈在同一个 PR 里处理，而不是新开一个。本项目没有 CLA；提交即表示你同意
你的贡献可以按本项目的许可证分发。

每个 PR 都要说明：

1. 它解决了什么现实问题。“我想要这个”不算；要说明在真实硬件上，
   真实用户遇到了什么损坏、失败或缺失。
2. 展示它能工作的截图或终端输出。
3. 测试过的环境：应用侧要写设备、Android 版本和 root 提供方；
   后端侧要写设备、SoC 和 Wi-Fi 芯片、驱动、频段和宽度、上游
   （移动数据、Wi-Fi、VPN），以及是否跑过托管模式。
4. UI 改动要写你遵循了哪些 DESIGN.md 的规则，以及你偏离了哪一条、为什么。
   与 DESIGN.md 不一致又没有理由的圆角或颜色会被退回。

会被合并的：带复现步骤并验证过修复的 bug 修复、在多种设备上验证过的功能、
安全加固、有实测数据的性能改进、删除，以及
文档更正。会被拒绝的：本可用 netlink 或内核自己的文件做到、
却去启动一个工具的引擎改动，或者让某个模块超出它那一件事的改动；
预编译或动态链接的二进制；
没有真实用户报告过、或者只在一种硬件配置上有效的功能；
作者自己解释不了的代码；破坏 Android 8 的应用改动；清单里已有之物的第二份拷贝；
任何未经引号处理就到达 root shell 的动态值；
任何让原本可用的东西出现回归的改动，无论新行为多有用。

被合并的功能从此归你维护：当新的 Android 版本或驱动把它弄坏时，
你需要去修，作者联系不上的功能会被移除。如果一位贡献者的 PR 总是因为
同样的原因被拒绝，没有真实问题、未经测试，或者只是在给代码库注水，
该贡献者就会被拉黑；没有警告次数，这由维护者决定。

## 安全

任何让热点客户端、或者输入到应用里的值，未经引号处理就到达 root shell 的问题
都是漏洞。不要开公开 issue；请私下报告到
droidcasts@protonmail.com 或 [t.me/ravindu](https://t.me/ravindu)，并附上复现步骤、
受影响的配置以及设备或驱动，在修复发布之前不要公开。
加固补丁会被优先审查。
