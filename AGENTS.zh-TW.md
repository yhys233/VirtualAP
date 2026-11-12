# AGENTS.md

[English](AGENTS.md) | [简体中文](AGENTS.zh-CN.md) | [繁體中文](AGENTS.zh-TW.md) | [Русский](AGENTS.ru.md) | [Français](AGENTS.fr.md) | [Deutsch](AGENTS.de.md) | [日本語](AGENTS.ja.md) | [한국어](AGENTS.ko.md)

給在此儲存庫工作的 AI 程式設計代理的指示。人類請閱讀
[CONTRIBUTING.zh-TW.md](./CONTRIBUTING.zh-TW.md)，裡面有這裡既有項目的清單。
運作方式寫在
[Documentation/How-It-Works.zh-TW.md](./Documentation/How-It-Works.zh-TW.md)。

## 這是什麼

VirtualAP 把已 root 的手機變成一個 Wi-Fi 存取點。兩個部分：

- `src/` 是引擎 `virtualap`，一個以 root 執行的靜態 C 二進位檔，它再驅動另外兩個：
  `hostapd` 與 `dnsmasq`，由 `scripts/build-static.sh` 從 `externals/` 下的 git 子模組
  建置而成。`make aarch64 armhf` 建置 `virtualap`。三者都落在
  `backend/aarch64/` 與 `backend/armhf/`，那兩個目錄被 gitignore。
- `Android/` 是透過 root shell 驅動 `virtualap` 的 Compose 應用程式。

`prepareAssets` gradle 工作會把兩個架構樹放進 `Android/app/build/` 下產生的 assets
根目錄，應用程式在首次執行時以及每次 APK 更新後，把相符的架構部署到
`/data/local/virtualap`，並為 root 管理員準備一個開機模組。
沒有 chroot。

## 建置

```
./scripts/build-static.sh    # from the repository root: both arches, needs Docker
./build.sh [release]         # from Android/: the APK, needs JDK 17 or newer, ANDROID_HOME and musl-cross gcc
```

在第一次建置應用程式之前先執行第一個，之後只要 `externals/` 或建置腳本
有變更就再執行一次。應用程式建置本身會執行 `make aarch64 armhf`，所以 `virtualap` 永遠
與 `src/` 相符。當 `backend/<arch>/` 缺少三者之一時，`prepareAssets` 會失敗。
`make check` 會執行測試。版本來自 `src/include/virtualap.h` 中的 `VAP_VERSION`。只用
`make bump NEW=x.y.z` 提升它，不要在其他地方改。建置與 CI 的細節在
How-It-Works.md 的 Building 一節。

## 提交

- 每次提交都要簽署：`git commit -s`。
- 絕不為 AI 代理加上 `Co-Authored-By:` 標記。人類共同作者則可以。
- 前綴，與既有歷史一致：
  - `app:` 用於 Android 應用程式，必要時用 `app: fix:` 與 `app: refactor:`
  - `backend:` 用於 `src/` 下的引擎，引擎錯誤修正用 `fix:`
  - `scripts:` 用於 `scripts/` 與 `externals/` 下的靜態二進位檔建置，`ci:` 用於
    `.github/workflows/`，`docs:` 用於文件
  - `fix(security):` 用於任何與安全性相關的事，兩個部分皆然
  - `VirtualAP: bump vX.Y.Z` 用於版本提升，該提交中不放其他東西

`ap:`、`build:`、`ui:`、`feat:` 與 `refactor:` 出現在較舊的歷史中。新的提交
不要使用它們。

## 風格

四條規則。它們適用於程式碼、註解、提交訊息與文件。

**不用 em dash。** 用逗號、句號，或改寫句子。

**不用 ASCII 橫幅註解。** 不要成排的 `-----`，不要 `=====`，不要方框式的區段標題。

**註解要像人說的話。** 說明程式碼為何這樣做，或不這樣做會壞掉什麼。
略過只是重述下一行的註解。

```sh
# Bad: bump the priority
PRIO_FROM_AP=7010

# Good: netd's own rules start at 10000, so 7010 wins before the VPN catch-all
PRIO_FROM_AP=7010
```

**十行能用的程式碼勝過一百行做同樣事情的程式碼。** 先刪再增。目標是在對的地方
有更小的差異，而不是在任何地方有更小的差異。

在文件中，每個事實只陳述一次，放在讀者會去找它的那一頁，其他頁面連過去。
分工是固定的：README.md 說明是什麼、給誰用以及步驟；
How-It-Works.md 持有所有技術事實；本檔案持有給代理的規則；
CONTRIBUTING.md 說明人類如何讓變更被合併，並持有清單；DESIGN.md
持有 UI 的數值。DESIGN.md 的表格保留，它們是角色對應與尺寸，寫成散文不管用；
其他地方用清單或段落。

## UI 變更

`Android/` 中任何視覺相關的東西都遵循 [DESIGN.zh-TW.md](./DESIGN.zh-TW.md)。顏色、字型、間距、圓角與
動作膠囊的模式都在那裡規定，而且是從既有應用程式讀出來的，不是憑空發明，
所以遵循它們同時也是最小的差異。

如果某條規則實在不適合你的情況，可以偏離，但要在該處留註解說明
偏離換到什麼，並在 PR 中說出來。未記錄的偏離是漂移，下一位貢獻者
會把它「修好」。DESIGN.md 的「已決定的例外」一節列出屬於決定的偏離，
不要去「修」那些。

## 在你開始寫之前

依序往下看這份清單，停在第一個成立的答案。

1. 這個東西需要存在嗎？沒人要求的旗標、給永不改變的數值用的旋鈕、
   只有一個實作的介面：跳過它，並用一行說明。
2. 這裡已經有嗎？見下方「先重複使用再寫」。先 grep。
3. 平台已經做了嗎？引擎上是 musl 的 libc 與核心自己的介面（netlink、
   `/proc/sys`）；應用程式端是 Kotlin 與 Java 標準函式庫以及 Android 框架。
   注意 `minSdk = 26`。
4. 已安裝的相依套件涵蓋了嗎？應用程式端有 libsu、Compose、lifecycle、zxing、appcompat。
   絕不為幾行就能做到的事新增一個。
5. 能寫成一行嗎？那就寫成一行。
6. 只有到那時，才寫最小可行的東西。

兩個答案都成立？取較高的那個，然後繼續。

這份清單縮短的是解法，從來不是閱讀。在你選定一階之前，
先追過變更會碰到的流程。在錯的地方做小差異是第二個錯誤，不是偷懶的修正。
錯誤報告也一樣：報告只指出症狀，所以編輯前先 grep 每個呼叫者。在
共用函式中放一個防護比在每個呼叫者各放一個更小，而且它會順帶修好
報告沒提到的同類問題。

絕不為了簡化而拿掉信任邊界上的輸入驗證、fail-closed 檢查、會遺失狀態的
錯誤處理，或請求者明確要求的任何東西。

## 先重複使用再寫

先在 CONTRIBUTING.md 的清單中 grep，然後是整棵樹。這些是瓶頸點。繞過
其中一個是錯誤，不是捷徑。

- `Android/app/src/main/java/com/virtualap/app/util/Backend.kt` 中的 `Backend.quote()` 會包住
  每個送到 root shell 的動態值。`APManager` 是唯一執行
  `virtualap` 的類別，而 `Backend.startAp` 是唯一拼出它路徑的方式。
- `Hotspot` 是 `APManager.start` 與 `APManager.stop` 唯一的呼叫者。它擁有
  啟動/停止的階段、指令日誌、執行中的通知與快速設定
  磚的同步，所以畫面和磚永遠一致。`APConfig.isValid()` 是唯一的
  啟動驗證；畫面的按鈕和磚都會問它。
- `src/` 每一層一個資料夾：`core`、`cli`、`net`（含 `net/ipt/`）、`radio`、`mode`、
  `monitor`、`cmd`，然後是 `main.c`。一個檔案只包含自己這一層與其下的層，
  依那個順序。每個資料夾有一個公開標頭檔 `src/include/<folder>/<folder>.h`，其
  原型依定義它的檔案分組標示；只有該資料夾自己的檔案與測試會用的東西放在
  `src/<folder>/internal.h`，以 `"internal.h"` 引入
  （測試則以路徑引入）。其他一切都用 `static`。檔案長度 200 到 800 行：新的
  函式放進其子系統所屬的檔案，新檔案是為了新的子系統，或為了
  超過 800 行的檔案。`net/ipt/` 有自己的標頭檔，因為在 musl 上核心的 netfilter
  標頭檔與 libc 的網路標頭檔無法共存於同一個檔案。從
  Droidspaces 與 USB-Arsenal 移植的程式碼遵循這些規則；外來的程式碼留在 `externals/`，
  原封不動。
- 記錄用 `vap_log`、`vap_warn`、`vap_error`（`core/io`）。它們印到 stdout，應用程式
  把 stdout 串流進它的日誌面板，並讀取其中的 `[WARN]` 與 `[ERROR]` 標籤，同時
  把帶時間戳的副本附加到 `logs/ap.log`。應用程式從不讀那個檔案。單獨的
  `printf` 在日誌中看不見，而送到 stderr 的任何東西都會遺失。
- 程式由 `core/proc` 執行（`run_quiet`、`run_capture`、`run_shown`、`run_logged`、
  `run_lines`），絕不用 `popen` 或 `system`：Android 沒有 `/bin/sh`，而且每個子行程得到
  `/dev/null` 作為 stdin 以及乾淨的訊號遮罩。Pid 檔案經由它的 `pidfile_*`
  函式處理，那些函式只會對執行預期程式的行程送出訊號。
- 設定存在一張表裡，`cli/options.c` 中的 `vap_keys`：旗標字母、`ap.conf`
  鍵、預設值、用法文字。新的設定是 `struct vap_opts` 中的欄位加上那裡的一列，
  以及 `APConfig` 中的一個欄位；解析與用法文字隨之而來。每個值都經過
  `opt_set`，它會拒絕換行。`ap.conf` 是應用程式的：只有 `APManager.saveConfig`
  會寫它，而且只寫通過 `APConfig.isValid()` 的設定。引擎只讀它。
- 與核心的溝通經由 `net/`（rtnetlink：帶位址與鄰居的 `link`、
  帶策略規則與 netd 表格的 `route`；`net/ipt/` 中的 `ip_tables` 引擎：
  `ipt`、`ipt_rules`）與 `radio/`（`driver`：nl80211 與 wpa 控制
  socket）。沒有模組會啟動 `ip`、`iw` 或 `hostapd_cli`；`iptables` 只為
  TTL 修正而啟動（`net/ipt/ipt_rules` 中的 `ttl_*`）。
- `upstream_resolve`（`net/route`）是唯一解析上游表的地方。
  `radio/channel`，包含 `ensure_5g_ir`，是頻段與頻道策略唯一的所在：
  跟隨工作站的頻道、DFS 後備、NO-IR 國別修正。不要在其他任何地方比較
  頻道號碼。`ipv6_decide`（`mode/ipv6`）是唯一決定用戶端
  IPv6 的地方；啟動與監視器都會問它。
- `Constants` 持有應用程式端每個路徑與偏好設定鍵，包含引擎的
  （`Constants.ENGINE`）。`VirtualAPInstaller` 負責部署二進位檔；它不會在
  其他地方重複。
- Kotlin 中不留文字。應用程式顯示的每個字串都在 `res/values/strings.xml` 或
  `plurals.xml`，以 `stringResource` 讀取，或在 composition 之外用 `getString`，讓 Weblate
  可以翻譯它。規則在 [CONTRIBUTING.zh-TW.md](./CONTRIBUTING.zh-TW.md) 的 Translations 一節。

## 硬性限制

- 應用程式的目標是 `minSdk = 26`。在假設某個 API 存在之前，先以 Android 8 的行為測試。
- 引擎是在 musl 上的 C11（`gnu11`），單執行緒：`setns` 只移動呼叫中的
  執行緒，而 netlink socket 會留在它被打開時所在的命名空間。它在 Makefile 的
  gcc 設定下無警告建置，CI 會把警告變成錯誤。
- 二進位檔是來自 `externals/` 子模組的完全靜態 musl 建置，兩個架構皆是。絕不
  使用動態連結的二進位檔，也絕不使用來自其他地方的預先建置檔。新工具就是新的子模組
  加上 `scripts/build-in-container.sh` 中的一個區塊；`virtualap` 是唯一由
  本儲存庫自己的 `src/` 建置的二進位檔，使用 musl-cross gcc 與根目錄的 `Makefile`。
- 已安裝的相依套件涵蓋時，不新增相依套件。應用程式已經有 libsu、Compose、
  lifecycle、zxing 與 appcompat。
- Samsung 的 Broadcom FullMAC 驅動程式是無線假設的下限，MediaTek 的
  gen4m 是第二個；各自缺少什麼寫在 How-It-Works.md 的「The radio」。沒有以
  執行時偵測為條件的怪癖修正不會被納入。
