# 為 VirtualAP 貢獻

[English](CONTRIBUTING.md) | [简体中文](CONTRIBUTING.zh-CN.md) | [繁體中文](CONTRIBUTING.zh-TW.md) | [Русский](CONTRIBUTING.ru.md) | [Français](CONTRIBUTING.fr.md) | [Deutsch](CONTRIBUTING.de.md) | [日本語](CONTRIBUTING.ja.md) | [한국어](CONTRIBUTING.ko.md)

不存在的功能勝過壞掉的功能。VirtualAP 以 root 執行於
已 root 的手機上，用的 Wi-Fi 晶片驅動程式對基本的事情意見不一，所以一個
在你手機上有效、卻在別人手機上壞掉的補丁是回歸，不是貢獻。而且
十行能完成工作的程式碼勝過一百行做同樣工作的程式碼：先刪再增，
先重用再寫。

## 必須成立的事

給任何要變更此儲存庫的人（包含代理）的規則在
[AGENTS.zh-TW.md](./AGENTS.zh-TW.md)：風格、提交格式、瓶頸點與硬性
限制。簡言之：只支援 Android，引擎是在 musl 上的單執行緒 C，直接與
核心溝通，每個模組只負責一件事，應用程式在 Android 8 上執行，而二進位檔
只來自 `scripts/build-static.sh`，以及（對 `virtualap` 而言）根目錄的 `Makefile`。

VirtualAP 在 Qualcomm、Broadcom FullMAC（Samsung）與 MediaTek 晶片上執行。針對單一
晶片怪癖的修正，只有在後端於執行時因應它時才會被接受，方式是偵測、
條件式路徑或後備，且不改變其他晶片上的行為；
`nl80211_sta_freq` 讀取關聯 BSS 的頻率，因為 Samsung 省略了頻道，
以及 `ensure_5g_ir` 只在 phy 回報 NO-IR 時觸發，就是這個模式。如果修正無法這樣設條件，它
屬於分支版本。在真實手機上測試，Wi-Fi 晶片沒有模擬器，並且在你的 PR 中
說明你測試過的裝置、驅動程式、頻段與上游；未經測試的主張視同
未測試。

## 建置

```
./scripts/build-static.sh    # from the repository root: both arches, needs Docker
./build.sh [release]         # from Android/: the APK, needs JDK 17 or newer, ANDROID_HOME and musl-cross gcc
```

這些腳本做什麼、CI 做什麼，都在
[How-It-Works.zh-TW.md](./Documentation/How-It-Works.zh-TW.md) 的 Building 一節。發行版就是一個標籤：`make bump NEW=x.y.z`，
以 `VirtualAP: bump vX.Y.Z` 提交它，然後推送 `main` 與標籤 `vX.Y.Z`。

## 程式碼風格

四條規則在 AGENTS.md 的 Style 一節。在此之上：

- 後端：coreutils 用 `$ECHO`、`$GREP`、`$SED` 之類，絕不單獨使用，也絕不加引號。
  輸出用 `log`、`warn`、`error`。子命令是 `cmd_*`，Droidspaces 輔助函式是 `ds_*`，
  無線探測是 `phy_*`，策略是 `pick_*` 與 `resolve_*`。拆除逐步對應設定，
  而且每一步都容忍東西不存在。每個常駐程式記錄到 `logs/` 下自己的檔案。
- 應用程式：狀態放在 ViewModel，不放在有狀態的 composable，也不放在 `util`
  單例。清單用 `LazyColumn` 與穩定的鍵呈現。顏色來自
  `MaterialTheme.colorScheme`，字型來自 `MaterialTheme.typography`，時間設定來自
  `AnimationUtils`，其他每個視覺數值來自 [DESIGN.zh-TW.md](./DESIGN.zh-TW.md)。任何送到
  root shell 的數值都經過 `Backend.quote()` 或允許清單驗證器。

提交都要簽署並加上前綴，如 AGENTS.md 的 Commits 一節所述。

## 翻譯

譯者在 Weblate 上工作，它會提交到 `values-<lang>/` 資料夾；拉取請求
只會動到 `res/values/` 中的英文。各元件如何設定，在
[How-It-Works.zh-TW.md](./Documentation/How-It-Works.zh-TW.md) 的 Translations 一節。

新增字串：

- 放在 `strings.xml` 中它所屬畫面的註解之下，或放在 `plurals.xml`，如果數量
  會改變措辭。Kotlin 以 `stringResource` 讀取它，或在 composition 之外用 `getString`
  讀取。
- 預留位置是有序的，`%1$s`，所以翻譯可以重新排序。在字串正上方的註解中
  說明每一個是什麼，Weblate 會把它顯示給譯者；對可能意指兩件事的
  短詞也照做。
- 每個字串一個完整片語。絕不把翻譯好的片段黏在一起。
- 路徑、指令、偏好設定鍵與引擎印出的東西留在程式碼中，用英文。檔案中
  沒有任何語言會改變的值，例如產品名稱或範例位址，是
  `translatable="false"`。

## 可重複使用清單

在寫任何新東西之前，先 grep 這份清單。如果已有接近的東西，就擴充它，而
不是新增一個同類的。Android 路徑相對於
`Android/app/src/main/java/com/virtualap/app/`；引擎模組是 `src/` 下的路徑，每個
資料夾的標頭檔在 `src/include/<folder>/`。
這份清單回答「我該用什麼」；當這裡都不合適時，DESIGN.md 回答「它應該長什麼樣」。
共用元件是從 Droidspaces 移植來的；當它的其他元件的呼叫者出現時，
複製上游的檔案，而不是寫一個新的。

表單與輸入：

- `DsDropdown(label, selected, options, displayName, onSelect, ...)` 位於
  `ui/component/DsDropdown.kt`：任何選取欄位。絕不手寫 `ExposedDropdownMenuBox`。
- `ui/component/DsTextFieldDefaults.kt` 中的 `DsTextFieldDefaults.colors()`：每個
  `OutlinedTextField`。
- `DsDialog` 使用的 `rememberClearFocus()`，以及主畫面根部的 `ClearFocusOnClickOutside`，
  兩者都在 `ui/util/FocusUtils.kt`：在外側點擊時收起鍵盤。

對話方塊與彈出面板：

- `ui/component/DsDialog.kt` 中的 `DsDialog(onDismiss, modifier, footer) { }`：每個對話方塊。絕不
  設定寬度、內距或捲動。
- `ui/component/DialogFooterRow.kt` 中的
  `DialogFooterRow(dismissLabel, confirmLabel, onDismiss, onConfirm, confirmEnabled)`：有選項的對話方塊中的
  兩個按鈕，放在 `footer` 插槽。
- `ui/component/DialogCloseButton.kt` 中的 `DialogCloseButton(onClick, enabled)`：標題列中 36dp 的關閉
  方塊，用於從頂部關閉的內容。
- `ui/component/LogActionRow.kt` 中的 `LogActionRow(logs, isBlocking, onClear)`：日誌主控台上方的
  Clear 與 Copy 列；`onClear = null` 只給 Copy。
- `DeviceRow`、`DeviceSheet(title, onDismiss, content)`、`BlockedList`、`AllowedList`、
  `BlockedDevicesSheet`、`AllowedDevicesSheet` 與 `rememberDeviceAction(reload)`，位於
  `ui/component/DeviceList.kt`：一台裝置與它的動作、它們所在的彈出面板、那兩份清單，
  以及對它們執行變更並把失敗說出來。
- `ui/component/AddDeviceDialog.kt` 中的 `AddDeviceDialog` 與 `AllowOnlyDialog`：只接受十六進位
  數字的 MAC 欄位，以及只有允許的裝置能加入時誰會被踢掉。
- `ui/component/Caption.kt` 中的 `Caption(text)`：一行次要文字。
- Wi-Fi QR 彈出面板、裝置彈出面板與日誌彈出面板是 `ui/screen/MainScreen.kt` 的私有內容，About
  對話方塊與語言選擇器則屬於 `ui/screen/SettingsScreen.kt`。不要匯入或複製它們。

列、設定、回饋：

- `ui/component/PrimaryActionBottomBar.kt` 中的
  `PrimaryActionBottomBar(label, icon, onClick, ...)`：Start 與 Stop 列。接受
  `content` 插槽的多載版本是給會替換內容的按鈕用。
- `ui/component/SetupHero.kt` 中的 `SetupPage`、`SetupFrame`、`SetupHero`、`SetupActionBar`：
  設定流程的每個畫面、歡迎分頁、root 檢查與安裝程式。見
  DESIGN.md 的「The setup flow」。
- `ui/component/PullToRefreshWrapper.kt` 中的 `PullToRefreshWrapper(onRefresh) { }`：任何下拉
  重新整理的畫面主體。
- `ui/component/ToggleCard.kt` 中的 `ToggleCard`：自己一張卡片中的開關列，主畫面的
  選項。`ui/component/SwitchItem.kt` 中的 `SwitchItem`：分組介面內
  的平面開關列，設定畫面用。
- `ui/component/StatusPill.kt` 中的 `StatusPill(label, color)`：任何小型狀態標籤。
  `ui/component/SectionHeader.kt` 中的 `SectionHeader(text)`：一組卡片上方的標題。
- `ui/component/CardMetrics.kt` 中的 `CardContentPadding`、`CardHeaderHeight`：任何有
  標題與膠囊標頭的卡片，讓分隔線對齊。
- `ui/util/LoadingIndicator.kt` 中的 `LoadingIndicator(size, color)` 搭配 `LoadingSize`，以及 `FullScreenLoading(message)`：
  每個轉圈指示器與每個全螢幕載入狀態。
- `ui/component/TerminalConsole.kt` 中的 `TerminalConsole(logs, isProcessing, maxHeight)` 與 `ShimmerAnimation(enabled) { }`：
  可捲動的日誌檢視與它的外框。

佈景主題：

- `ui/theme/Theme.kt` 中的 `VirtualAPTheme(darkTheme, dynamicColor, amoledMode, themePalette)`：
  唯一的佈景主題根，套用於 `MainActivity`。
- `ui/theme/ThemeStateHolder.kt` 中的 `rememberThemeState()` 與 `ThemeState`：即時的佈景主題
  偏好設定。
- `ui/theme/Color.kt` 中的 `ThemePalette`：新增強調色調色盤，只在這裡，不在別處。
  `Theme.kt` 與 `Color.kt` 是唯二直接寫出顏色名稱的檔案。
- `ui/theme/Type.kt` 中的 `MaterialTheme.typography.*` 與 `JetBrainsMono`：所有文字樣式，
  以及日誌與程式碼文字用的等寬字型。
- `util/AnimationUtils.kt` 中的 `AnimationUtils`：補間動畫的規格。絕不寫死 `tween(300)`。
- `ui/component/` 中的 `AccentColorPicker` 與 `ColorPaletteSwatch`：調色盤選擇器。
- 沒有間距或形狀的權杖物件；數字放在 DESIGN.md，並以
  字面 dp 寫出。

導覽：

- `MainActivity.kt` 中的 `Screen` 與對它的 `when`：選擇頂層畫面唯一的地方，
  先看歡迎旗標，然後是 root 與安裝狀態，然後是設定旗標。新的
  畫面放在那裡。

Shell、root 與後端：

- `util/Backend.kt` 中的 `Backend.quote(value)` 與 `Backend.startAp`：每個送到 root 指令的
  動態值，以及引擎的指令前綴（在設定流程取代它之前，會後備到較舊發行版
  解出的腳本）。
- `util/APManager.kt` 中的 `APManager.getStatus/start/stop/getInterfaces/getCaps/isInstalled`：
  驅動 `virtualap`，以及 `getContainers`/`parseContainers`，它們直接向 Droidspaces 詢問
  正在執行的容器。絕不手工組裝其中任何一個指令。
- `util/Hotspot.kt` 中的 `Hotspot.start(cfg)/stop()/refresh()`、`phase`、`status`、`actionLogs`：
  從任何地方啟動或停止 AP，並讀取工作階段狀態；也會發出
  執行中的通知並喚醒磚。
- `util/APConfig.kt` 中的 `APConfig` 與 `isValid()`、`isValidIpv4`、`isValidDnsList`、`channelsForBand`：
  儲存的熱點設定、唯一的啟動驗證，以及欄位與它共用的位址
  檢查。
- `util/BackendLogger.kt` 中的 `BackendLogger` 與 `classifyLine(line)`：AP 呼叫所接收的
  日誌接收端，以及 `[ERROR]`/`[WARN]` 等級標記。
- `util/PreferencesManager.kt` 中的 `PreferencesManager`：應用程式自己的設定。
  `APManager.loadConfig/saveConfig`：熱點設定，在 `ap.conf` 中整份讀寫。
  `util/Constants.kt` 中的 `Constants`：每個路徑與偏好設定鍵。
- `util/RootChecker.kt` 中的 `RootChecker.checkRootAccess(reprompt)` 與 `RootStatus`；
  `util/VirtualAPInstaller.kt` 中的 `VirtualAPInstaller.deviceArch/bundledPayloadVersion/payloadUpdateAvailable/install`；
  `util/ModuleInstaller.kt` 中的 `ModuleInstaller.install`，即開機
  模組；`util/QrCodeGenerator.kt` 中的 `QrCodeGenerator.wifiPayload/encode`，
  唯一的 zxing 呼叫者。
- `HotspotTileService`、`HotspotStopReceiver` 與 `HotspotBootReceiver`：快速設定
  開關、通知的 Stop 動作，以及開機模組表示它啟動了熱點的訊息。
  它們本身都不持有狀態。
- `util/LocaleHelper.kt` 中的 `LocaleHelper.availableLanguages/currentLanguage/changeLanguage`：
  應用程式的語言，`AppCompatDelegate.setApplicationLocales` 唯一的呼叫者。
- `AppViewModel` 擁有歡迎旗標、root 狀態與安裝狀態；`APViewModel` 擁有可編輯的設定、
  介面與容器清單以及尾隨的日誌，並把啟動與停止轉發給
  `Hotspot`。3 s 狀態輪詢位於 `MainScreen`，綁定於它的生命週期。
- 全域 libsu 設定位於 `VirtualAPApplication.kt`，而且只在那裡。

引擎（`src/`）：

- `core/`：`proc`（每個執行的程式、pid 檔案，以及讓一次只能有一個啟動或
  停止在執行的鎖）、`io`（日誌：`vap_log`、`vap_warn`、`vap_error`、`log_stamp`；
  `read_file`、`write_file`、`write_atomic`；解析器 `valid_ip4`、`prefix_parse`、
  `subnet_from_gw`，以及用於 `-d` 逗號清單的 `csv_each`）。
- `cli/`：`options`（`vap_keys` 表、`opt_set`、`opts_parse`、`print_usage`）、`conf`
  （`ap.conf` 與 `run.state`）。
- `net/`：`nl`（rtnetlink 傳輸、`nl_dump`，以及命名空間輔助函式 `netns_open`、
  `in_netns`）、`link`（連結、位址與鄰居）、`route`（路由、策略規則，以及
  netd 的表格：`upstream_resolve`、`upstream_v6_iface`）、`ipt/`，之所以分開，是因為在
  musl 上核心的 netfilter 標頭檔與 libc 的網路標頭檔無法共存於同一個檔案：
  `ipt`（`ip_tables` 引擎）、`ipt_rules`（FORWARD、MASQUERADE、存在檢查、
  NAT 探測、`ipt_bin`、TTL 修正）。
- `radio/`：`driver`（透過 genetlink 的 nl80211：建立與刪除 ap0、phy、工作站的
  頻率、寬度；wpa 控制 socket；`wifi_mac`，出廠 MAC）、`channel`
  （頻段、頻道與寬度策略，以及 `ensure_5g_ir`）。
- `mode/`：`ap`（ap0、常駐程式、無線規劃、兩種模式共用的訊號保持，以及
  `hostapd` 與 `dnsmasq`：它們的設定與啟動、用戶端數量）、`routed`（包含
  兩個位址族都使用的固定策略規則）、`ipv6`（`ipv6_decide`、設定、拆除、NAT66 與
  共用的 /64 及其用戶端探索）、`managed`（Droidspaces 容器：它的 pid、
  OpenWrt 佈建、RPS 遮罩）、`maclist`（被封鎖與允許的裝置的
  檔案，也就是 `hostapd` 的 `deny_mac_file` 與 `accept_mac_file`，以及告知執行中的
  `hostapd`）。
- `monitor/`：`monitor`（常駐程式的迴圈）、`upstream`（`reprobe`）。
- `cmd/`：應用程式呼叫的指令，由 `main.c` 分派：`start`（`start` 與
  `stop`）、`status`（唯讀的那些：`status`、`clients`、`interfaces`、`caps`）與
  `block`（`block`、`unblock`、`blocked`、`allow`、`disallow`、`allowed`、`allow-only`）。

## 提出 PR

建立分支版本，在分支中工作，對 `main` 開 PR，並保持回應：沒有回應的 PR 會被
關閉，而回饋會在同一個 PR 中處理，不是開新的。沒有 CLA；提交即表示
你同意你的貢獻可以依專案的授權條款散布。

每個 PR 都要說明：

1. 它解決什麼真實世界的問題。「我想要這個」不算；要說明真實使用者在真實硬體上
   遇到什麼壞掉、失敗或缺少。
2. 截圖或終端機輸出，證明它可用。
3. 測試過的環境：應用程式要說裝置、Android 版本與 root 提供者；後端要說
   裝置、SoC 與 Wi-Fi 晶片、驅動程式、頻段與寬度、上游（行動數據、
   Wi-Fi、VPN），以及是否實際跑過管理模式。
4. UI 變更要說你遵循了哪些 DESIGN.md 規則，以及你偏離了哪一條、
   為什麼。與 DESIGN.md 不符又沒有理由的圓角或顏色會被退回。

會被合併的：有重現方式與已驗證修復的修正、跨裝置驗證過的功能、
安全性強化、有量測數據的效能改善、刪減，以及文件修正。
會被拒絕的：為了做 netlink 或核心自己的檔案就能做的事而啟動工具的引擎變更，
或讓模組超出它單一職責的變更；預先建置或動態連結的二進位檔；
沒有真實使用者回報過、或只在單一硬體配置上有效的功能；
作者自己解釋不清的程式碼；會破壞 Android 8 的應用程式變更；
清單中某個東西的第二份副本；任何未加引號就送到 root shell 的動態值；
任何會讓原本可用的東西退步的變更，
不論新行為多有用。

已合併的功能之後由你負責：當新的 Android 版本或驅動程式讓它壞掉時，
預期你會修好它，而作者無法聯絡的功能會被移除。貢獻者的 PR 若持續
因為相同理由被拒絕，沒有真實問題、未經測試，或只是在灌水
程式碼庫，就會被封鎖；沒有警告次數，決定權在維護者。

## 安全性

任何讓熱點用戶端，或輸入應用程式中的值，未加引號就到達 root shell 的東西
都是漏洞。不要開公開 issue；請私下回報到
droidcasts@protonmail.com 或 [t.me/ravindu](https://t.me/ravindu)，附上重現方式、
受影響的配置與裝置或驅動程式，並在修復推出前不要公開揭露。
強化補丁會優先審查。
