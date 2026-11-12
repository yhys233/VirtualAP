# DESIGN.md

[English](DESIGN.md) | [简体中文](DESIGN.zh-CN.md) | [繁體中文](DESIGN.zh-TW.md) | [Русский](DESIGN.ru.md) | [Français](DESIGN.fr.md) | [Deutsch](DESIGN.de.md) | [日本語](DESIGN.ja.md) | [한국어](DESIGN.ko.md)

VirtualAP Android 應用程式的視覺語言，也就是 Droidspaces 的設計語言
連同實作它的元件一起沿用過來。後端是沒有介面的 shell 腳本，
所以這些都不適用於 `backend/`。

元件優先，數值其次：[CONTRIBUTING.zh-TW.md](./CONTRIBUTING.zh-TW.md) 中的可重複使用清單
列出既有的東西，而本檔案是為了那裡都沒有涵蓋你需求的情況。
絕不寫死十六進位顏色、sp 尺寸，
或憑眼睛挑的圓角。

## 應用程式看起來如何，以及為什麼

這個應用程式中沒有任何東西是浮起的。深度是靠把表面顏色往上提一階
並畫一條 1dp 邊框來表達；如果你伸手去拿陰影，你就已經離開這套設計了。
`Surface` 是基礎元件：按鈕是可點擊的 Surface，不是 Material 的 `Button`，而
對話方塊是 `Dialog { Surface { } }`，不是 `AlertDialog`。Material 元件並未被禁止，
但它們帶著自己的浮起與圓角預設值，必須能說得出理由。

顏色是推導出來的，從不寫死。`ui/theme/Theme.kt` 從三個
調色盤顏色混出完整的配色方案，或整份採用 Material You 配置。動態顏色預設開啟，
舊裝置與關閉它的使用者有後備調色盤，而 AMOLED 會把所有表面收成黑色，
所以寫死的 `Color(0xFF...)` 會一次弄壞這四條路徑。

邊框與著色承載狀態，填色保持低調。執行中的存取點不是一張綠色卡片；
它是一張普通卡片，配一個著色按鈕與一條強調色邊框。

## 顏色

角色來自 `MaterialTheme.colorScheme`。配色方案本身建構於 `ui/theme/Theme.kt`，
取自 `ui/theme/Color.kt` 中的調色盤，那是唯二直接寫出顏色名稱的檔案。

| 表面 | 角色 |
| --- | --- |
| 畫面背景 | `background` |
| 位於背景上的卡片 | `surfaceContainer` |
| 巢狀於另一個表面內的卡片或膠囊，以及可互動的設定列 | `surfaceContainerHigh` |
| 對話方塊與底部彈出面板的外殼 | `surfaceContainer` |

| 文字 | 角色 |
| --- | --- |
| 主要 | `onSurface` |
| 次要、輔助、中繼資料 | `onSurfaceVariant`，alpha `0.7f` |
| 第三層、刻意低調 | `onSurfaceVariant`，alpha `0.6f` |
| 停用內容 | `onSurface`，alpha `0.38f` |
| 停用容器填色 | `onSurface`，alpha `0.12f` |

邊框一律是 1dp。變化的是 alpha：

| 邊框 | 顏色 |
| --- | --- |
| 卡片 | `outlineVariant`，`0.35f` |
| 對話方塊外殼 | `outlineVariant`，`0.4f` |
| 動作膠囊外框與其他巢狀表面 | `outlineVariant`，`0.2f` |
| 分隔線（`HorizontalDivider`） | `outlineVariant`，`0.3f` |
| 文字欄位，未聚焦 | `outlineVariant`，`0.5f` |
| 文字欄位，已聚焦 | `primary`，`0.8f` |
| 著色按鈕，搭配它的強調色 | 該強調色角色，`0.2f` |

### 顏色代表什麼

存取點的生命週期在整個應用程式中是一致的，新的狀態應該加入它，而
不是另造一套平行的詞彙。

| 狀態 | 強調色 |
| --- | --- |
| 執行中、健康、啟動動作 | `primary` |
| 已停止、失敗、破壞性、停止動作 | `error` |
| 重新啟動動作 | `secondary` |
| 進行中、重新啟動中、下載中 | `tertiary` |
| 閒置、未知、無可回報 | `onSurfaceVariant`，`0.6f` |
| 停用 | 填色 `onSurface`，`0.08f`，強調色 `outlineVariant` |

著色按鈕從相符的容器角色取填色，alpha 為 `0.4f`，例如
start 用 `primaryContainer.copy(alpha = 0.4f)`，stop 用 `errorContainer.copy(alpha = 0.4f)`，
並加上 1dp、alpha 為 `0.2f` 的強調色邊框。

主控台把 `[WARN]` 行著成 `tertiary`，把 `[ERROR]` 行著成 `error`，依據 `classifyLine`
從標籤讀出的等級；後端輸出純文字，沒有跳脫碼。

## 文字排版

級數採用 Material 3 自己的，`ui/theme/Type.kt` 中的 `Typography()`。使用具名樣式並
在該處傳入字重：標題是 `titleLarge` 搭配 `FontWeight.Bold`，卡片標題
`titleMedium` SemiBold。應用程式中任何地方都沒有覆寫 `fontSize`。

UI 文字是 `FontFamily.Default`。機器產生的任何東西，日誌行、設定值、
路徑，都是 JetBrains Mono，從 `ui/theme/Type.kt` 匯入。不要宣告本地的字型
家族。

| 元素 | 樣式 |
| --- | --- |
| `TopAppBar` 中的畫面標題 | `titleLarge`，Bold |
| 對話方塊標題 | `titleLarge`，Bold，靠左對齊 |
| 卡片標題 | `titleMedium`，SemiBold |
| 畫面內的區段標題 | `titleSmall`，Bold，`primary` |
| 按鈕標籤 | `labelLarge`，SemiBold |
| 對話方塊中的內文 | `bodyMedium` |
| 輔助文字、中繼資料、副標題 | `bodySmall` 搭配 `onSurfaceVariant`，`0.7f` |
| 狀態膠囊標籤 | `labelSmall`，Black，`letterSpacing = 0.5.sp`，大寫 |
| 機器輸出 | `bodySmall` 搭配 JetBrains Mono 字型家族 |

## 間距

級數是 **4, 8, 12, 16, 20, 24, 32**。沒有別的。如果某個間隙感覺想要 14，它
其實想要 12 或 16。

| 間隙 | 數值 |
| --- | --- |
| 畫面邊緣，清單與儀表板 | 水平 16 |
| 畫面邊緣，精靈與設定流程 | 水平 24 |
| 清單中卡片之間 | 16 |
| 卡片內區塊之間 | 12 |
| 卡片內距 | 16 |
| 對話方塊內距 | 24 |
| 圖示到它的標籤、標籤到標籤 | 8 |
| 緊湊的中繼資料列 | 4 |

沒有分頁列。主畫面在 `Scaffold` 內位於 `PrimaryActionBottomBar` 上方，
所以列的實測高度會成為 Scaffold 的內距，清單會避開它，
不需要任何人輸入避讓數值。保持這樣：不是 Scaffold 的
`bottomBar` 的列需要一個字面數值，而字面數值會漂移。

### 卡片標頭對齊

使用中的網路卡片沒有標題：它的第一列左邊是網路名稱、右邊是
狀態膠囊，位於分隔線上方，最小高度為 `CardHeaderHeight`，
並在 `CardContentPadding` 之內，兩者都來自 `ui/component/CardMetrics.kt`。高度是 48dp，
因為這一列放了 Wi-Fi QR 圖示按鈕，而 48dp 是觸控下限。

確實有標題與膠囊標頭的卡片採用同樣兩個數值，所以它的分隔線與
膠囊會和這張卡落在同一條線上。如果你改了它們，你是在移動應用程式中
每一張卡片的標頭，而那正是預期的方向。

## 形狀

| 元素 | 圓角 |
| --- | --- |
| 卡片 | 20 |
| 對話方塊外殼 | 24 |
| 底部彈出面板，僅上方兩角 | 24 |
| 全寬主要按鈕 | 20 |
| 下拉式選單 | 20 |
| 動作按鈕 | 16 |
| 文字欄位 | 16 |
| 動作膠囊外框 | 12 |
| 狀態膠囊、小型徽章 | 8 |

## 動作膠囊

一排相關的動作位於著色的外框內：

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

按鈕比它們所在的方框更圓，這就是設計所在：4dp 的內縮加上
較大的內圓角，讓按鈕看起來是浮出於外框之上，而不是
平貼嵌在裡面。把兩個圓角改成相同會讓這個效果消失。

主畫面上的介面或容器切換（`ui/screen/MainScreen.kt` 中的
`SegmentedSelector`）就是這套幾何被當作選擇器使用：被選取的區段取用
著色填色與強調色邊框，另一個保持透明。日誌主控台上方的 Clear 與 Copy 列
（`LogActionRow`）不是那樣：它是一排光裸的兩個 38dp 按鈕，
沒有外框。

## 對話方塊

每個對話方塊都是 `DsDialog`，它擁有版面配置與外框：24dp 邊距內的
全寬 `Surface`，高度界於螢幕高度減去這些邊距，以及一個 24dp 內距、主體可捲動的
直向排列。界線來自 `LocalConfiguration`，不是視窗，所以轉向時開著的
對話方塊會依即時配置量測。呼叫者不設定寬度、自己的
內距或自己的捲動，也絕不傳 `wrapContentHeight()`；每個對話方塊特有的東西放在
`modifier` 上，例如文字欄位的 `imePadding()` 或 `heightIn(max = ...)` 上限。

標題是 `titleLarge` Bold，靠左對齊，而且是內容中的第一個東西。

## 對話方塊動作

About 是一個資訊頁，從它的標頭用 `DialogCloseButton` 關閉。有選項的對話方塊，
語言選擇器，以 `DialogFooterRow` 作結：兩個等寬的 48dp 按鈕，圓角 16，一個
低調的色調取消鈕與一個填色的確認鈕，標籤為單行且會省略。它放在 `DsDialog` 的
`footer` 插槽，在可捲動主體之前量測，所以橫向時無法擠壓它。第一個
破壞性對話方塊沿用 Droidspaces 的 `error` 確認鈕。

## 圖示與觸控目標

| 用途 | 尺寸 |
| --- | --- |
| 卡片或清單列的前導圖示 | 20 |
| 工具列動作、按鈕內的圖示 | 18 |
| 與 `bodySmall` 文字同列 | 16 |
| 對話方塊標頭 | 24 |
| 設定主視覺圖示 | 96，在 240 的主視覺內 |
| 狀態圓點 | 6 |

任何可點擊的東西至少 **48dp**。那是無障礙下限，不是風格偏好，
而且它是本檔案中唯一一條設計論述不能推翻的規則。

## 對齊

列使用 `Alignment.CenterVertically`。

卡片標題與對話方塊標題一律靠左對齊。把標題與控制項配成一對的卡片
列使用 `Arrangement.SpaceBetween`。

置中文字保留給全螢幕狀態：空清單、載入、設定主視覺。如果內容在
卡片內置中，那大概是錯的。

## 動態與載入

轉圈指示器來自 `LoadingIndicator` 搭配 `LoadingSize`，絕不在進度指示器上用
原始的 `.size(n.dp)`。全螢幕載入是 `FullScreenLoading`；`LoadingSize.Hero`（144）是
唯一一個設定流程的主視覺載入器。動畫時間設定來自 `AnimationUtils`；主視覺狀態變化
使用 `MaterialTheme.motionScheme`，佈景主題把它設為 `MotionScheme.expressive()`。

### 設定流程

歡迎分頁、root 檢查與後端安裝程式共用同一套幾何，它位於
`ui/component/SetupHero.kt`。`SetupPage` 是 Scaffold 底下的骨架：上方一個 40dp 的
列（歡迎分頁上的頁面計數，其他地方為空）、外框插槽（它會置中
內容，或在視窗太短時捲動），以及下方一個 44dp 的列（頁面圓點）。
每個畫面都保留這兩列，所以外框和按鈕在畫面之間永遠不會移動。
`SetupFrame` 就是那個外框：一個 240dp 的主視覺方框，然後是固定文字區塊中的
標題與說明。`SetupHero` 是會做事的畫面的主視覺，一個 240dp 的
`MaterialShapes.Cookie12Sided`，放在 `primaryContainer` 或 `errorContainer`，承載載入器或
一個 96dp 的圖示，狀態之間只有它的顏色與內容改變。歡迎分頁的主視覺
則是它自己的 `Shape`，一個柔和的爆裂形，隨著頁面滑過而定型為餅乾、四葉草與太陽。
`SetupActionBar` 是那唯一的按鈕。用這些東西建一個新的設定畫面，
主視覺、標題與按鈕就會和其他畫面落在同樣的列上。

## 當規則不適合時

有時候它就是不適合，而答案不是悄悄改用另一個數字。

在 PR 中說出來，並在該處留註解說明偏離換到什麼，就像下面
已決定的例外那樣。未記錄的偏離是漂移，下一位貢獻者
會把它「修好」。有記錄的偏離是一個決定，而它會留下來。

## 已決定的例外

看過了，刻意不動。除非有原本理由沒想到的原因，
不要重新打開這些。

- **QR 底板是 `Color.White`**，在 Wi-Fi QR 彈出面板中，而 `QrCodeGenerator.encode` 在白底上
  畫黑色模組。掃描器需要規格所假設的對比；深色調色盤下的
  著色底板讀起來像壞掉的條碼。
- **狀態卡片會回應點擊**（`ui/screen/MainScreen.kt` 中的 `ActiveNetworkCard`），雖然
  它什麼都不會開啟。那是使用者會戳一下確認應用程式還活著的卡片，所以它會泛起漣漪並
  讓欄位失焦，而不是在手指下死氣沉沉。代價是 TalkBack 會把它讀成一個
  可執行動作的節點。
- **Wi-Fi QR 彈出面板、裝置彈出面板與日誌彈出面板**，位於
  `ui/screen/MainScreen.kt`，以及 `ui/component/DeviceList.kt` 中的裝置清單彈出面板，
  維持 Material 的 `ModalBottomSheet`，而不是
  `Dialog { Surface { } }`。它們全都是拖曳關閉的內容，
  屬於底部邊緣，而日誌彈出面板在有指令執行時會阻擋手勢關閉，
  所以面板不會留下一個吃掉觸控的遮罩。它們的外殼是
  `surfaceContainer` 搭配 24dp 的上方圓角，與形狀表一致。
- **設定流程的按鈕**（`ui/component/SetupHero.kt` 中的 `SetupActionBar`）是一個完整的膠囊，
  沒有列的表面也沒有分隔線，不是 20dp 的列按鈕。它單獨坐在主視覺下方的
  背景上，而不是覆蓋在可捲動內容上。它的停用顏色保持等於啟用
  顏色：它只在檢查或安裝進行中時停用，而為了那一瞬間把它變灰
  讀起來像閃爍。
- **0.03 與 0.06 的淡填色 alpha** 在 `LogActionRow` 中保持不共用。它們單獨看是
  看不見的，而把它們湊在一起會意味著為沒有人
  看得見的東西發明一個權杖。
