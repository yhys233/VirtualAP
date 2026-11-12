# VirtualAP

[English](README.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md) | [Русский](README.ru.md) | [Français](README.fr.md) | [Deutsch](README.de.md) | [日本語](README.ja.md) | [한국어](README.ko.md)

root 化した Android スマートフォンを、本物の Wi-Fi ルーターに変えます。

VirtualAP はスマートフォンの Wi-Fi チップ上で正式なアクセスポイントを動作させます。
ネットワーク名、パスワード、帯域とチャンネル、端末から見えるゲートウェイアドレス、
そして通信の出口（モバイルデータ、Wi-Fi、VPN）を自分で選べます。

## できること

- **OpenWrt をホットスポットのルーターとして動かす。** ファイアウォールゾーン、
  ポート転送、通信ルール、帯域制限、LuCI の Web インターフェースを、すべて
  スマートフォン上で。下記の管理モードを参照してください。
- **VPN を共有する。** ホットスポットにつながるすべての端末がスマートフォンの VPN
  を経由します。テレビ、ゲーム機、ノート PC に何もインストールする必要は
  ありません。
- **Wi-Fi を中継する。** ある Wi-Fi ネットワークにつながったまま、それを自分用の
  ネットワークとして再配信します。
- **固定のゲートウェイを保つ。** ゲートウェイアドレスは変わらないため、ポート転送
  や保存済みの SSH 接続先がそのまま使えます。
- **無線を調整する。** 2.4 / 5 GHz、チャンネルと帯域幅、WPA2 / WPA3、ネットワーク
  名の非表示。
- **回線事業者にテザリングを隠す**：対応カーネルでは、任意の TTL 補正を利用
  できます。

## 管理モード：OpenWrt がホットスポットを動かす

これが VirtualAP の際立った特徴です。ホットスポットを
[Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) 上で動作する OpenWrt
コンテナに任せると、OpenWrt は本物の Wi-Fi ルーターと同じように接続端末を管理します。
ブラウザで選んだゲートウェイアドレスを開けば、そこが LuCI です。

OpenWrt に任せると、次のことができます。

- 好きなだけゾーンを持つファイアウォールを構築する
- ポート転送と通信ルールを設定する
- 端末ごとに帯域と速度を制限する
- 他の端末に到達できないゲストネットワークを運用する
- 接続中の端末をすべて確認し、固定アドレスと名前を割り当てる
- ネットワーク全体の DNS サーバーを選び、OpenWrt のパッケージフィードから全端末に
  広告ブロックを追加する
- 端末に IPv4 だけでなく IPv6 も提供する
- Droidspaces コンテナを同じルーターの背後に置き、Wi-Fi 端末と同じネットワークに
  参加させる、あるいは完全に分離する
- `opkg` で OpenWrt のその他のソフトウェアをインストールする

OpenWrt の WAN 側は Droidspaces で、Droidspaces はモバイルデータ、Wi-Fi、VPN トンネル
の間でアップリンクをリアルタイムに切り替えます。ホットスポットはこれをそのまま
利用できます。

1. Droidspaces で、OpenWrt コンテナの最初の上流インターフェースに `tun0` を設定し、
   続けて `wlan0` と `rmnet0` を指定します。VPN はまだ動いていないため、`tun0` は
   存在しません。
2. VirtualAP を起動します。ホットスポットの通信は Wi-Fi またはモバイルデータから
   出ていきます。
3. VPN アプリを起動します。Droidspaces は `tun0` の出現を検知して OpenWrt をそちらへ
   移すので、直後にはホットスポット上のすべての端末が VPN の内側に入ります。
   再起動は不要です。

OpenWrt はスマートフォン自身のカーネル上のコンテナで動作し、間に仮想マシンは
ありません。

管理モードに必要なもの：

- root 権限
- インストール済みで動作する
  [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS)
- [Droidspaces 公式 rootfs リポジトリ](https://github.com/Droidspaces/Droidspaces-rootfs-builder/releases)
  の OpenWrt コンテナをインストールして実行しておくこと

## インターフェースモード

コンテナも Web インターフェースも使わないシンプルなモードです。VirtualAP 自身が
アドレスを配布し、上流が対応していれば端末に IPv6 を提供し、選んだインターフェース
から通信を送り出すか、Android が現在使用中のネットワークに追従します。VPN の共有や
Wi-Fi の中継にはこれで十分です。

## 動作要件

- root 化されたスマートフォン
- Android 8.0 以降
- ARM 搭載のスマートフォン（64 ビット／32 ビット）。1 つの APK で両方に対応します。
- アクセスポイントの自動起動には Magisk または KernelSU
- 管理モードには Droidspaces とその OpenWrt コンテナ（上記参照）
- TTL 補正には対応カーネル。アプリが判定し、非対応の場合はオプションがグレーアウト
  されますが、ホットスポットはそのまま動作します。

## はじめかた

1. [Releases](https://github.com/ravindu644/VirtualAP/releases) から最新の APK を
   ダウンロードしてインストールします。
2. アプリを開き、root を許可します。必要な設定はすべて自動で行われ、任意の
   Magisk/KernelSU 起動モジュールも含まれます。書き込みも再起動も不要です。
3. ネットワーク名とパスワードを入力します。
4. 上流を選びます。**インターフェース**（または自動）、管理モードなら OpenWrt
   **コンテナ**です。
5. ホットスポットを起動し、端末を接続します。

設定で **起動時にアクセスポイントを有効化** をオンにすると、再起動後も自動で復帰
します。

管理モードでは、接続した端末から `http://<your gateway address>` を開くと LuCI に
アクセスできます。

## 詳細情報

- [仕組み](Documentation/How-It-Works.md)：ルーティング、IPv6、ブリッジ、無線、
  スマートフォン上のファイル、バイナリと APK のビルド方法
- [コントリビューション](CONTRIBUTING.md)

## ライセンス

GNU General Public License v3.0。[LICENSE](LICENSE) を参照してください。

VirtualAP は [Ubuntu-Chroot](https://github.com/ravindu644/Ubuntu-Chroot) の
[`start-hotspot`](https://github.com/ravindu644/Ubuntu-Chroot/blob/main/tools/start-hotspot)
スクリプトから発展したもので、アプリは AI アシスタントの助けを借りて書かれています。
