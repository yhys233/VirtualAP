# AGENTS.md

[English](AGENTS.md) | [简体中文](AGENTS.zh-CN.md) | [繁體中文](AGENTS.zh-TW.md) | [Русский](AGENTS.ru.md) | [Français](AGENTS.fr.md) | [Deutsch](AGENTS.de.md) | [日本語](AGENTS.ja.md) | [한국어](AGENTS.ko.md)

このリポジトリで作業する AI コーディングエージェント向けの指示です。人間は [CONTRIBUTING.ja.md](./CONTRIBUTING.ja.md) を読んでください。そこには、ここにすでに存在するものの一覧があります。仕組みについては [Documentation/How-It-Works.ja.md](./Documentation/How-It-Works.ja.md) にあります。

## これは何か

VirtualAP は root 化したスマートフォンを Wi-Fi アクセスポイントに変えます。2 つの部分から成ります：

- `src/` はエンジンである `virtualap` です。root として動作する静的 C バイナリで、さらに 2 つ、`hostapd` と `dnsmasq` を動かします。この 2 つは `externals/` 配下の git サブモジュールから `scripts/build-static.sh` でビルドします。`make aarch64 armhf` は `virtualap` をビルドします。3 つとも `backend/aarch64/` と `backend/armhf/` に出力され、これらのディレクトリは gitignore されています。
- `Android/` は、root シェル経由で `virtualap` を操作する Compose アプリです。

`prepareAssets` gradle タスクは両方のアーキテクチャのツリーを `Android/app/build/` 配下の生成されたアセットルートに配置し、アプリは初回起動時と APK の更新のたびに、一致するアーキテクチャを `/data/local/virtualap` に配備します。root マネージャー用の起動モジュールも付きます。chroot はありません。

## ビルド

```
./scripts/build-static.sh    # from the repository root: both arches, needs Docker
./build.sh [release]         # from Android/: the APK, needs JDK 17 or newer, ANDROID_HOME and musl-cross gcc
```

最初のアプリビルドの前に 1 つ目を実行し、`externals/` またはビルドスクリプトが変わったときにもう一度実行してください。アプリのビルドは `make aarch64 armhf` を自身で実行するため、`virtualap` は常に `src/` と一致します。`backend/<arch>/` に 3 つのうち 1 つでも欠けていると `prepareAssets` は失敗します。テストは `make check` で実行します。バージョンは `src/include/virtualap.h` の `VAP_VERSION` から取得します。更新は `make bump NEW=x.y.z` で行い、それ以外の場所では行いません。ビルドと CI の詳細は How-It-Works.md のビルドセクションにあります。

## コミット

- すべてのコミットにサインオフを付けてください：`git commit -s`。
- AI エージェントのための `Co-Authored-By:` トレーラーは決して付けないでください。人間の共同作成者は問題ありません。
- プレフィックスは既存の履歴に合わせます：
  - `app:` は Android アプリ向けで、該当する場合は `app: fix:` と `app: refactor:` を使います
  - `backend:` は `src/` 配下のエンジン向け、エンジンのバグ修正には `fix:` を使います
  - `scripts:` は `scripts/` と `externals/` 配下の静的バイナリビルド向け、`ci:` は `.github/workflows/` 向け、`docs:` はドキュメント向けです
  - `fix(security):` はセキュリティに関わるものすべて向けで、どちらの部分でも使います
  - バージョン更新には `VirtualAP: bump vX.Y.Z` を使い、そのコミットには他に何も入れません

`ap:`、`build:`、`ui:`、`feat:`、`refactor:` は古い履歴に現れます。新しいコミットでは使わないでください。

## スタイル

4 つのルールです。コード、コメント、コミットメッセージ、ドキュメントに適用されます。

**em ダッシュを使わない。** 読点、句点を使うか、文を書き直してください。

**ASCII バナーコメントを使わない。** `-----` の行も、`=====` も、枠で囲んだセクション見出しも使いません。

**コメントは人間が話すように書く。** コードがなぜそうするのか、そうしないと何が壊れるのかを書いてください。すぐ下の行を言い換えるだけのコメントは省きます。

```sh
# Bad: bump the priority
PRIO_FROM_AP=7010

# Good: netd's own rules start at 10000, so 7010 wins before the VPN catch-all
PRIO_FROM_AP=7010
```

**動く 10 行は、同じことをする 100 行に勝る。** 追加する前に削ってください。目的は正しい場所でのより小さな差分であり、どこであれ小さな差分ではありません。

ドキュメントでは、すべての事実を一度だけ、読者がそれを探すページに書き、他のページからはリンクします。分担は固定です：README.md は何を、誰のために、どの手順でを述べ、How-It-Works.md はすべての技術的事実を保持し、このファイルはエージェント向けのルールを保持し、CONTRIBUTING.md は人間が変更をマージしてもらう方法と一覧を保持し、DESIGN.md は UI の値を保持します。DESIGN.md の表は残します。それらはロールの対応表とサイズであり、文章では機能しないからです。それ以外の場所ではリストか段落を使います。

## UI の変更

`Android/` の視覚に関わるものはすべて [DESIGN.ja.md](./DESIGN.ja.md) に従います。色、書体、余白、角丸、アクションピルのパターンはすべてそこに規定されており、発明されたものではなく既存のアプリから読み取られたものです。したがって従うことが最小の差分でもあります。

ルールが本当に自分のケースに合わない場合は、外れてかまいません。ただしその場所にコメントを残し、その逸脱が何をもたらすかを書き、PR でも述べてください。文書化されていない逸脱はドリフトであり、次のコントリビューターがそれを「修正」してしまいます。DESIGN.md の「決定済みの例外」セクションには、決定としての逸脱が並んでいます。それらを「修正」しないでください。

## そもそも書く前に

このリストを上から順にたどり、最初に当てはまる答えで止めてください。

1. これは存在する必要があるか？ 誰も求めていないフラグ、決して変わらない値のためのつまみ、実装が 1 つしかないインターフェースなら、省いて 1 行でそう述べてください。
2. これはすでにここにあるか？ 下の「書く前に再利用する」を参照してください。まず grep します。
3. プラットフォームがすでにやっているか？ エンジンでは musl の libc とカーネル自身のインターフェース（netlink、`/proc/sys`）、アプリ側では Kotlin と Java の標準ライブラリと Android フレームワークです。`minSdk = 26` に注意してください。
4. インストール済みの依存関係で足りるか？ アプリ側では libsu、Compose、lifecycle、zxing、appcompat です。数行でできることのために新しいものを追加しないでください。
5. 1 行でできるか？ ならば 1 行です。
6. それから初めて、動く最小のものを書きます。

2 つの答えが成り立つ場合は、より上のものを選んで先に進んでください。

このリストは解決策を短くするものであり、読む量を短くするものではありません。段を選ぶ前に、変更が触れる流れをたどってください。誤った場所での小さな差分は 2 つ目のバグであり、手抜きの修正ではありません。バグ報告も同じです。報告は症状を述べるものなので、編集する前にすべての呼び出し元を grep してください。共有関数に 1 つのガードを置くほうが、各呼び出し元に 1 つずつ置くより小さく、報告が触れていない兄弟も直ります。

信頼境界での入力検証、フェイルクローズのチェック、状態を失うエラー処理、依頼者が名指しで求めたものを、単純化して取り除かないでください。

## 書く前に再利用する

CONTRIBUTING.md の一覧を grep し、次にツリーを grep してください。以下はチョークポイントです。1 つを迂回することはバグであり、近道ではありません。

- `Android/app/src/main/java/com/virtualap/app/util/Backend.kt` の `Backend.quote()` は、root シェルに届くすべての動的な値をラップします。`virtualap` を実行するクラスは `APManager` だけであり、そのパスを書く唯一の方法は `Backend.startAp` です。
- `APManager.start` と `APManager.stop` の唯一の呼び出し元は `Hotspot` です。開始中／停止中のフェーズ、コマンドログ、実行中の通知、クイック設定タイルの同期を所有するため、画面とタイルは常に一致します。開始時の検証は `APConfig.isValid()` だけであり、画面のボタンとタイルの両方がそれを尋ねます。
- `src/` はレイヤーごとに 1 つのフォルダです：`core`、`cli`、`net`（`net/ipt/` を含む）、`radio`、`mode`、`monitor`、`cmd`、そして `main.c` です。ファイルは自分自身のレイヤーとその下のレイヤーだけを、その順でインクルードします。各フォルダには公開ヘッダが 1 つ、`src/include/<folder>/<folder>.h` があり、プロトタイプは定義元のファイル名を付けたラベルの下にまとめます。そのフォルダ自身のファイルとテストだけが使うものは `src/<folder>/internal.h` に置き、`"internal.h"` としてインクルードします（テストはパスで取り込みます）。それ以外はすべて `static` です。ファイルは 200 から 800 行です。新しい関数はそのサブシステムのファイルに入り、新しいファイルは新しいサブシステムか 800 行を超えたファイルのためのものです。`net/ipt/` が独自のヘッダを持つのは、musl ではカーネルの netfilter ヘッダと libc のネットワークヘッダを 1 つのファイルで同時に扱えないからです。Droidspaces と USB-Arsenal から移植したコードはこれらのルールに従います。ベンダー提供のコードは `externals/` にそのまま置きます。
- ログ出力は `vap_log`、`vap_warn`、`vap_error`（`core/io`）です。これらは stdout に出力し、アプリはそれを読み取った `[WARN]` と `[ERROR]` のタグを付けてログシートに流し込み、タイムスタンプ付きのコピーを `logs/ap.log` に追記します。アプリがそのファイルを読むことはありません。素の `printf` はログに現れず、stderr に出したものは失われます。
- プログラムは `core/proc`（`run_quiet`、`run_capture`、`run_shown`、`run_logged`、`run_lines`）で実行し、`popen` や `system` は決して使いません。Android には `/bin/sh` がなく、各子プロセスは stdin に `/dev/null` とクリーンなシグナルマスクを受け取ります。pid ファイルはその `pidfile_*` 関数を通し、期待するプログラムが動いているプロセスにだけシグナルを送ります。
- 設定は `cli/options.c` の 1 つのテーブル `vap_keys` にあります：フラグ文字、`ap.conf` のキー、既定値、使用方法のテキストです。新しい設定は `struct vap_opts` のフィールドとそこへの行、そして `APConfig` のフィールドになります。解析と使用方法のテキストがそれに続きます。すべての値は `opt_set` を通り、これは改行を拒否します。`ap.conf` はアプリのものであり、書くのは `APManager.saveConfig` だけで、しかも `APConfig.isValid()` を通る設定に限られます。エンジンは読むだけです。
- カーネルとの対話は `net/`（rtnetlink：アドレスと近隣を含む `link`、ポリシールールと netd のテーブルを含む `route`、`net/ipt/` の `ip_tables` エンジン：`ipt`、`ipt_rules`）と `radio/`（`driver`：nl80211 と wpa 制御ソケット）を通じて行います。`ip`、`iw`、`hostapd_cli` を起動するモジュールはありません。`iptables` を起動するのは TTL 修正（`net/ipt/ipt_rules` の `ttl_*`）のときだけです。
- 上流のテーブルを解決する唯一の場所は `upstream_resolve`（`net/route`）です。帯域とチャンネルのポリシー、つまりステーションのチャンネルへの追従、DFS のフォールバック、NO-IR の国コード修正が存在する唯一の場所は `radio/channel`（`ensure_5g_ir` を含む）です。他の場所でチャンネル番号を比較しないでください。クライアントの IPv6 を決める唯一の場所は `ipv6_decide`（`mode/ipv6`）であり、開始時とモニターの両方がそれを尋ねます。
- `Constants` はアプリ側のすべてのパスと設定キーを保持し、エンジンのもの（`Constants.ENGINE`）も含みます。`VirtualAPInstaller` はバイナリを配備します。他の場所に重複させません。
- Kotlin にテキストを書かない。アプリが表示するすべての文字列は `res/values/strings.xml` または `plurals.xml` にあり、`stringResource` で、コンポジションの外では `getString` で読み出します。そうすれば Weblate が翻訳できます。ルールは [CONTRIBUTING.ja.md](./CONTRIBUTING.ja.md) の翻訳セクションにあります。

## 厳守すべき制約

- アプリは `minSdk = 26` を対象とします。API が存在すると仮定する前に、Android 8 での挙動をテストしてください。
- エンジンは musl 上の C11（`gnu11`）で、シングルスレッドです。`setns` は呼び出したスレッドだけを移動し、netlink ソケットは開いたときの名前空間に留まります。Makefile の gcc 設定では警告なしでビルドされ、CI は警告をエラーにします。
- バイナリは `externals/` のサブモジュールからの完全な静的 musl ビルドで、両方のアーキテクチャ分あります。動的リンクのバイナリは決して使わず、他所からのビルド済みバイナリも決して使いません。新しいツールは新しいサブモジュールと `scripts/build-in-container.sh` のブロックです。`virtualap` だけは、このリポジトリ自身の `src/` から musl-cross gcc とルートの `Makefile` でビルドされる唯一のバイナリです。
- インストール済みの依存関係で足りる場合は新しい依存関係を追加しない。アプリにはすでに libsu、Compose、lifecycle、zxing、appcompat があります。
- 無線に関する前提の下限は Samsung の Broadcom FullMAC ドライバで、次が MediaTek の gen4m です。それぞれに欠けているものは How-It-Works.md の「無線」にあります。実行時の検出で条件分岐されていないクセ対応の修正は入れません。
