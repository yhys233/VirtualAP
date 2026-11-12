# VirtualAP へのコントリビューション

[English](CONTRIBUTING.md) | [简体中文](CONTRIBUTING.zh-CN.md) | [繁體中文](CONTRIBUTING.zh-TW.md) | [Русский](CONTRIBUTING.ru.md) | [Français](CONTRIBUTING.fr.md) | [Deutsch](CONTRIBUTING.de.md) | [日本語](CONTRIBUTING.ja.md) | [한국어](CONTRIBUTING.ko.md)

存在しない機能は、壊れた機能より優れています。VirtualAP は root 化したスマートフォン上で、基本的な事柄についてドライバの見解が分かれる Wi-Fi チップ上で、root として動作します。そのため、あなたのスマートフォンでは動き、他人のスマートフォンでは壊れるパッチは、貢献ではなくリグレッションです。そして仕事を成し遂げる 10 行は、同じ仕事をする 100 行に勝ります。追加する前に削り、書く前に再利用してください。

## 守るべきこと

エージェントを含め、このリポジトリを変更するすべての人に向けたルールは [AGENTS.ja.md](./AGENTS.ja.md) にあります。スタイル、コミット形式、チョークポイント、厳守すべき制約です。要するに、Android のみ、エンジンは musl 上のシングルスレッドの C でカーネルと直接対話する、モジュールごとに 1 つの関心事に分割する、アプリは Android 8 で動作する、バイナリは `scripts/build-static.sh` から、そして `virtualap` はルートの `Makefile` からのみ生成する、ということです。

VirtualAP は Qualcomm、Broadcom FullMAC（Samsung）、MediaTek のチップで動作します。あるチップのクセに対する修正が受け入れられるのは、バックエンドが実行時にそれへ適応する場合だけです。検出、条件付きの経路、フォールバックのいずれかで適応し、他のチップの挙動を変えない場合に限ります。Samsung がチャンネルを省略するため関連付けられた BSS の周波数を `nl80211_sta_freq` が読むこと、phy が NO-IR を報告したときにだけ `ensure_5g_ir` が発火すること、これらがそのパターンです。そのように条件分岐できない修正はフォークに属します。実機のスマートフォンでテストしてください。Wi-Fi チップのエミュレーターはありません。PR には、テストした端末、ドライバ、帯域、上流を書き添えてください。テストしていない主張は未テストとして扱います。

## ビルド

```
./scripts/build-static.sh    # from the repository root: both arches, needs Docker
./build.sh [release]         # from Android/: the APK, needs JDK 17 or newer, ANDROID_HOME and musl-cross gcc
```

スクリプトが何をするか、CI が何をするかは [How-It-Works.ja.md](./Documentation/How-It-Works.ja.md) のビルドセクションにあります。リリースはタグです。`make bump NEW=x.y.z` を実行し、`VirtualAP: bump vX.Y.Z` としてコミットし、`main` とタグ `vX.Y.Z` をプッシュします。

## コードスタイル

4 つのルールは AGENTS.md のスタイルにあります。その上で：

- バックエンド：coreutils は `$ECHO`、`$GREP`、`$SED` などを使い、裸の名前は決して使わず、クォートもしません。出力は `log`、`warn`、`error` です。サブコマンドは `cmd_*`、Droidspaces のヘルパーは `ds_*`、無線のプローブは `phy_*`、ポリシーは `pick_*` と `resolve_*` です。後片付けはセットアップを 1 ステップずつ鏡写しにし、すべてのステップが不在を許容します。各デーモンは `logs/` 配下の自分用のファイルにログを書きます。
- アプリ：状態は ViewModel に置き、ステートフルなコンポーザブルにも `util` のシングルトンにも置きません。リストは `LazyColumn` と安定したキーで描画します。色は `MaterialTheme.colorScheme`、書体は `MaterialTheme.typography`、タイミングは `AnimationUtils`、その他の視覚的な値はすべて [DESIGN.ja.md](./DESIGN.ja.md) から取ります。root シェルに届く値はすべて `Backend.quote()` か許可リストの検証を通ります。

コミットはサインオフを付け、AGENTS.md のコミットに書かれているとおりにプレフィックスを付けます。

## 翻訳

翻訳者は Weblate で作業し、Weblate は `values-<lang>/` フォルダにコミットします。プルリクエストが触れるのは `res/values/` の英語だけです。コンポーネントの設定方法は [How-It-Works.ja.md](./Documentation/How-It-Works.ja.md) の翻訳セクションにあります。

文字列を追加する場合：

- その画面用のコメントの下の `strings.xml` に置きます。数によって文言が変わる場合は `plurals.xml` に置きます。Kotlin は `stringResource` で、コンポジションの外では `getString` で読み出します。
- プレースホルダーは位置指定の `%1$s` なので、翻訳側で順序を変えられます。それぞれが何であるかを文字列のすぐ上のコメントに書き、Weblate がそれを翻訳者に表示します。2 つの意味に取れる短い語についても同じようにしてください。
- 1 つの文字列には 1 つの完全なフレーズを入れます。翻訳された断片を決してつなぎ合わせないでください。
- パス、コマンド、設定キー、エンジンが出力するものはコード内に英語のまま置きます。どの言語でも変わらないファイル内の値、製品名、例示アドレスは `translatable="false"` です。

## 再利用の一覧

何か新しいものを書く前にこのリストを grep してください。近いものがあれば、兄弟を追加するのではなく拡張してください。Android のパスは `Android/app/src/main/java/com/virtualap/app/` からの相対パスです。エンジンのモジュールは `src/` 配下のパスで、各フォルダのヘッダは `src/include/<folder>/` 配下です。このリストは「何と呼ぶか」に答え、ここに合うものがない場合に「どう見えるべきか」に答えるのは DESIGN.md です。共有コンポーネントは Droidspaces から移植されました。その他のコンポーネントの呼び出し元が必要になったときは、新しいものを書くのではなく上流のファイルをコピーしてください。

フォームと入力：

- `ui/component/DsDropdown.kt` の `DsDropdown(label, selected, options, displayName, onSelect, ...)`：選択フィールド全般。`ExposedDropdownMenuBox` を手書きしないでください。
- `ui/component/DsTextFieldDefaults.kt` の `DsTextFieldDefaults.colors()`：すべての `OutlinedTextField`。
- `ui/util/FocusUtils.kt` の、`DsDialog` が使う `rememberClearFocus()` と、メイン画面のルートである `ClearFocusOnClickOutside`：外側のタップでキーボードを閉じます。

ダイアログとシート：

- `ui/component/DsDialog.kt` の `DsDialog(onDismiss, modifier, footer) { }`：すべてのダイアログ。幅、パディング、スクロールを自分で設定しないでください。
- `ui/component/DialogFooterRow.kt` の `DialogFooterRow(dismissLabel, confirmLabel, onDismiss, onConfirm, confirmEnabled)`：選択肢のあるダイアログの 2 つのボタンで、`footer` スロットに入れます。
- `ui/component/DialogCloseButton.kt` の `DialogCloseButton(onClick, enabled)`：ヘッダ行にある 36dp の閉じる正方形で、上から閉じるコンテンツ向けです。
- `ui/component/LogActionRow.kt` の `LogActionRow(logs, isBlocking, onClear)`：ログコンソールの上にあるクリアとコピーの行。`onClear = null` にするとコピーだけになります。
- `ui/component/DeviceList.kt` の `DeviceRow`、`DeviceSheet(title, onDismiss, content)`、`BlockedList`、`AllowedList`、`BlockedDevicesSheet`、`AllowedDevicesSheet`、`rememberDeviceAction(reload)`：操作を伴う端末、それらを載せるシート、2 つのリスト、そして失敗を明示しながらそれらへの変更を実行する仕組みです。
- `ui/component/AddDeviceDialog.kt` の `AddDeviceDialog` と `AllowOnlyDialog`：16 進数字だけを受け付ける MAC フィールドと、許可された端末だけが参加できるときに誰が外れるか。
- `ui/component/Caption.kt` の `Caption(text)`：補助テキストの 1 行。
- Wi-Fi QR シート、デバイスシート、ログシートは `ui/screen/MainScreen.kt` 専用で、About ダイアログと言語ピッカーは `ui/screen/SettingsScreen.kt` 専用です。これらをインポートしたりコピーしたりしないでください。

バー、セットアップ、フィードバック：

- `ui/component/PrimaryActionBottomBar.kt` の `PrimaryActionBottomBar(label, icon, onClick, ...)`：開始と停止のバー。`content` スロットを取るオーバーロードは、中身を入れ替えるボタン用です。
- `ui/component/SetupHero.kt` の `SetupPage`、`SetupFrame`、`SetupHero`、`SetupActionBar`：セットアップフローのすべての画面、ウェルカムのページャー、root チェック、インストーラー。DESIGN.md の「セットアップフロー」を参照してください。
- `ui/component/PullToRefreshWrapper.kt` の `PullToRefreshWrapper(onRefresh) { }`：引っ張って更新するあらゆる画面本体。
- `ui/component/ToggleCard.kt` の `ToggleCard`：独自のカード内にあるスイッチ行で、メイン画面のオプション。`ui/component/SwitchItem.kt` の `SwitchItem`：グループ化されたサーフェス内のフラットなスイッチ行で、設定画面。
- `ui/component/StatusPill.kt` の `StatusPill(label, color)`：あらゆる小さな状態チップ。`ui/component/SectionHeader.kt` の `SectionHeader(text)`：カードのグループの上に置く見出し。
- `ui/component/CardMetrics.kt` の `CardContentPadding`、`CardHeaderHeight`：タイトルとピルのヘッダを持つあらゆるカード。区切り線がそろうようにします。
- `ui/util/LoadingIndicator.kt` の、`LoadingSize` を伴う `LoadingIndicator(size, color)` と `FullScreenLoading(message)`：すべてのスピナーと、すべての全画面の読み込み状態。
- `ui/component/TerminalConsole.kt` の `TerminalConsole(logs, isProcessing, maxHeight)` と `ShimmerAnimation(enabled) { }`：スクロールするログビューとその枠。

テーマ：

- `ui/theme/Theme.kt` の `VirtualAPTheme(darkTheme, dynamicColor, amoledMode, themePalette)`：唯一のテーマルートで、`MainActivity` で適用します。
- `ui/theme/ThemeStateHolder.kt` の `rememberThemeState()` と `ThemeState`：実行中のテーマ設定。
- `ui/theme/Color.kt` の `ThemePalette`：アクセントパレットの追加はここでだけ行います。色をリテラルで名指しするファイルは `Theme.kt` と `Color.kt` だけです。
- `ui/theme/Type.kt` の `MaterialTheme.typography.*` と `JetBrainsMono`：すべてのテキストスタイルと、ログとコードのテキスト用の等幅フォント。
- `util/AnimationUtils.kt` の `AnimationUtils`：トゥイーンの定義。リテラルの `tween(300)` は決して使いません。
- `ui/component/` の `AccentColorPicker` と `ColorPaletteSwatch`：パレットピッカー。
- 余白や形状のトークンオブジェクトはありません。数値は DESIGN.md にあり、リテラルの dp として書きます。

ナビゲーション：

- `MainActivity.kt` の `Screen` とそれに対する `when`：トップレベルの画面を選ぶ唯一の場所で、まずウェルカムのフラグ、次に root とインストールの状態、次に設定のフラグです。新しい画面はここに追加します。

シェル、root、バックエンド：

- `util/Backend.kt` の `Backend.quote(value)` と `Backend.startAp`：root コマンドに届くすべての動的な値と、エンジンのコマンドプレフィックス（古いリリースが展開したスクリプトへのフォールバックを含み、セットアップフローがそれを置き換えるまで続きます）。
- `util/APManager.kt` の `APManager.getStatus/start/stop/getInterfaces/getCaps/isInstalled`：`virtualap` の操作と、動作中のコンテナを Droidspaces に直接問い合わせる `getContainers`／`parseContainers`。これらのコマンドを手で組み立てないでください。
- `util/Hotspot.kt` の `Hotspot.start(cfg)/stop()/refresh()`、`phase`、`status`、`actionLogs`：どこからでも AP を開始または停止し、セッションの状態を読むためのもの。実行中の通知を投稿し、タイルを起こします。
- `util/APConfig.kt` の `APConfig` と `isValid()`、`isValidIpv4`、`isValidDnsList`、`channelsForBand`：保存されたホットスポット設定、唯一の開始時検証、フィールドがそれと共有するアドレスチェック。
- `util/BackendLogger.kt` の `BackendLogger` と `classifyLine(line)`：AP の呼び出しが使うログの受け口と、`[ERROR]`／`[WARN]` のレベル付け。
- `util/PreferencesManager.kt` の `PreferencesManager`：アプリ自身の設定。`APManager.loadConfig/saveConfig`：ホットスポット設定で、`ap.conf` に丸ごと読み書きします。`util/Constants.kt` の `Constants`：すべてのパスと設定キー。
- `util/RootChecker.kt` の `RootChecker.checkRootAccess(reprompt)` と `RootStatus`、`util/VirtualAPInstaller.kt` の `VirtualAPInstaller.deviceArch/bundledPayloadVersion/payloadUpdateAvailable/install`、起動モジュールである `util/ModuleInstaller.kt` の `ModuleInstaller.install`、zxing の唯一の呼び出し元である `util/QrCodeGenerator.kt` の `QrCodeGenerator.wifiPayload/encode`。
- `HotspotTileService`、`HotspotStopReceiver`、`HotspotBootReceiver`：クイック設定のトグル、通知の停止アクション、ホットスポットを起動したという起動モジュールの連絡。いずれも自身の状態を持ちません。
- `util/LocaleHelper.kt` の `LocaleHelper.availableLanguages/currentLanguage/changeLanguage`：アプリの言語で、`AppCompatDelegate.setApplicationLocales` の唯一の呼び出し元です。
- `AppViewModel` はウェルカムのフラグ、root の状態、インストールの状態を所有します。`APViewModel` は編集可能な設定、インターフェースとコンテナの一覧、追尾しているログを所有し、開始と停止を `Hotspot` に転送します。3 秒ごとの状態ポーリングは `MainScreen` にあり、そのライフサイクルに紐づきます。
- libsu のグローバル設定は `VirtualAPApplication.kt` にあり、そこにだけあります。

エンジン（`src/`）：

- `core/`：`proc`（すべてのプログラム実行、pid ファイル、1 度に 1 つの開始または停止だけを走らせるロック）、`io`（ログ：`vap_log`、`vap_warn`、`vap_error`、`log_stamp`、`read_file`、`write_file`、`write_atomic`、パーサー `valid_ip4`、`prefix_parse`、`subnet_from_gw`、`-d` のカンマ区切りリスト用の `csv_each`）。
- `cli/`：`options`（`vap_keys` テーブル、`opt_set`、`opts_parse`、`print_usage`）、`conf`（`ap.conf` と `run.state`）。
- `net/`：`nl`（rtnetlink のトランスポート、`nl_dump`、名前空間ヘルパーの `netns_open`、`in_netns`）、`link`（リンク、アドレス、近隣）、`route`（ルート、ポリシールール、netd のテーブル：`upstream_resolve`、`upstream_v6_iface`）、`ipt/`。musl ではカーネルの netfilter ヘッダと libc のネットワークヘッダを 1 つのファイルで同時に扱えないため分離しています：`ipt`（`ip_tables` エンジン）、`ipt_rules`（FORWARD、MASQUERADE、存在チェック、NAT プローブ、`ipt_bin`、TTL 修正）。
- `radio/`：`driver`（genetlink 経由の nl80211：ap0 の作成と削除、phy、ステーションの周波数、幅、wpa 制御ソケット、工場出荷時の MAC である `wifi_mac`）、`channel`（帯域、チャンネル、幅のポリシー、および `ensure_5g_ir`）。
- `mode/`：`ap`（ap0、デーモン、無線の計画、両モードが共有する信号の保持、および `hostapd` と `dnsmasq`：それらの設定と起動、クライアント数）、`routed`（両方のアドレスファミリーが使う固定のポリシールールを含む）、`ipv6`（`ipv6_decide`、セットアップ、後片付け、NAT66、クライアント探索を伴う共有 /64）、`managed`（Droidspaces コンテナ：その pid、OpenWrt のプロビジョニング、RPS マスク）、`maclist`（ブロックされた端末と許可された端末のファイルで、`hostapd` の `deny_mac_file` と `accept_mac_file` にあたり、動作中の `hostapd` にも伝えます）。
- `monitor/`：`monitor`（デーモンのループ）、`upstream`（`reprobe`）。
- `cmd/`：アプリが呼ぶコマンドで、`main.c` からディスパッチします：`start`（`start` と `stop`）、`status`（読み取り専用のもの：`status`、`clients`、`interfaces`、`caps`）、`block`（`block`、`unblock`、`blocked`、`allow`、`disallow`、`allowed`、`allow-only`）。

## PR を開く

フォークし、ブランチで作業し、`main` に対して PR を開き、応答できる状態を保ってください。応答のない PR はクローズされ、フィードバックには新しい PR ではなく同じ PR で対応します。CLA はありません。提出することで、あなたの貢献がプロジェクトのライセンスの下で配布されてよいことに同意したものとみなされます。

すべての PR には次を書きます：

1. どの現実の問題を解決するのか。「これが欲しかった」はそれに当たりません。実際のユーザーが実際のハードウェアで何に困るのか、何が失敗するのか、何が欠けているのかを書いてください。
2. 動作していることを示すスクリーンショットか端末の出力。
3. テストした環境：アプリなら端末、Android のバージョン、root の提供元。バックエンドなら端末、SoC と Wi-Fi チップ、ドライバ、帯域と幅、上流（モバイルデータ、Wi-Fi、VPN）、そして管理モードを試したかどうか。
4. UI の変更なら、従った DESIGN.md のルールと、どのルールから外れたか、その理由。理由なく DESIGN.md と食い違う角丸や色は差し戻されます。

マージされるもの：再現手順と検証済みの解決を伴う修正、複数の端末で検証された機能、セキュリティの強化、計測された性能改善、削除、ドキュメントの訂正。却下されるもの：netlink やカーネル自身のファイルでできることをツールの起動で行うエンジンの変更、またはモジュールをその単一の関心事の外へ膨らませる変更、ビルド済みまたは動的リンクのバイナリ、実際のユーザーが報告していない機能や 1 つのハードウェア構成でしか動かない機能、作者が説明できないコード、Android 8 を壊すアプリの変更、一覧にあるものの 2 つ目のコピー、クォートされずに root シェルに届く動的な値、新しい挙動がどれほど有用でも、動いていたものをリグレッションさせるもの。

マージされた機能は今後あなたのものです。新しい Android のバージョンやドライバがそれを壊したときは、あなたが直すことが期待され、作者に連絡が取れない機能は削除されます。同じ理由で PR が却下され続けるコントリビューター、つまり現実の問題がない、未テスト、コードベースを水増しする、という場合はブロックされます。ストライクの数えはなく、判断はメンテナーのものです。

## セキュリティ

ホットスポットのクライアントやアプリに入力された値が、クォートされずに root シェルに届くものはすべて脆弱性です。公開の issue を開かないでください。再現手順、影響する構成、端末またはドライバを添えて、droidcasts@protonmail.com または [t.me/ravindu](https://t.me/ravindu) へ非公開で報告し、修正が出るまで公開の開示を控えてください。強化のパッチは優先してレビューされます。
