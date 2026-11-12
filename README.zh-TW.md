# VirtualAP

[English](README.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md) | [Русский](README.ru.md) | [Français](README.fr.md) | [Deutsch](README.de.md) | [日本語](README.ja.md) | [한국어](README.ko.md)

把一台已 root 的 Android 手機變成真正的 Wi-Fi 路由器。

VirtualAP 會在手機的 Wi-Fi 晶片上執行完整的無線存取點。網路名稱、密碼、頻段與
頻道，裝置看到的閘道位址，以及它們的流量走向——行動數據、Wi-Fi 或 VPN——全都由你
決定。

## 你可以做什麼

- **讓 OpenWrt 擔任熱點的路由器。** 防火牆區域、連接埠轉送、流量規則、頻寬限制
  與 LuCI 網頁介面，全部在你的手機上執行。詳見下方的管理模式。
- **分享 VPN。** 熱點上的每台裝置都經由手機的 VPN 連外，電視、遊戲主機、筆記型
  電腦都不需要安裝任何東西。
- **中繼 Wi-Fi。** 保持連線到某個 Wi-Fi 網路，再將它當作自己的網路重新廣播。
- **維持固定閘道。** 閘道位址永不變動，連接埠轉送與已儲存的 SSH 目標可以一直
  使用。
- **調整無線參數。** 2.4 或 5 GHz、頻道與頻寬、WPA2 或 WPA3、隱藏網路名稱。
- **對電信業者隱藏熱點分享**：在支援的核心上，可選用 TTL 修正。

## 管理模式：由 OpenWrt 執行你的熱點

這正是 VirtualAP 與眾不同之處。把熱點交給執行於
[Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) 的 OpenWrt 容器，
OpenWrt 就會像真正的 Wi-Fi 路由器那樣管理每一台連線裝置。在瀏覽器開啟你選定的
閘道位址，你就進入了 LuCI。

交由 OpenWrt 管理後，你可以：

- 建立任意數量區域的防火牆
- 設定連接埠轉送與流量規則
- 依裝置限制頻寬與速度
- 執行無法連到其他裝置的訪客網路
- 檢視所有連線裝置，並為它指定固定位址與名稱
- 為整個網路選擇 DNS 伺服器，並透過 OpenWrt 的套件來源為每台裝置加上廣告封鎖
- 同時為裝置提供 IPv6 與 IPv4
- 讓 Droidspaces 容器位於同一台路由器之後，與 Wi-Fi 裝置同處一個網路，或完全
  隔離
- 用 `opkg` 安裝 OpenWrt 提供的其他軟體

OpenWrt 的 WAN 側由 Droidspaces 負責，而 Droidspaces 會在行動數據、Wi-Fi 與 VPN
通道之間即時切換上行連結。你的熱點因此自動享有這項能力：

1. 在 Droidspaces 中，將 `tun0` 設為 OpenWrt 容器第一個上行介面，其後依序為
   `wlan0` 與 `rmnet0`。此時 VPN 尚未執行，因此 `tun0` 並不存在。
2. 啟動 VirtualAP。熱點的流量會經由 Wi-Fi 或行動數據送出。
3. 啟動 VPN 應用程式。Droidspaces 一偵測到 `tun0` 出現，就會把 OpenWrt 移到它
   上面，於是片刻之後熱點上的所有裝置都位於 VPN 之後，無須重新啟動任何東西。

OpenWrt 執行於手機自身核心上的容器中，中間沒有虛擬機器。

管理模式需要：

- root 權限
- 已安裝且可正常運作的
  [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS)
- 來自
  [官方 Droidspaces rootfs 儲存庫](https://github.com/Droidspaces/Droidspaces-rootfs-builder/releases)
  的 OpenWrt 容器，已安裝並執行中

## 介面模式

沒有容器、也沒有網頁介面的簡單模式。VirtualAP 自行配發位址，在上游支援時為裝置
提供 IPv6，並經由你選定的介面送出，或跟隨 Android 目前使用的網路。要分享 VPN 或
中繼 Wi-Fi，這樣就夠了。

## 系統需求

- 已 root 的手機
- Android 8.0 或更新版本
- ARM 手機，64 位元或 32 位元，同一個 APK 兩者通用
- 需要熱點開機自動啟動：Magisk 或 KernelSU
- 需要管理模式：Droidspaces 及其 OpenWrt 容器（見上文）
- 需要 TTL 修正：支援此功能的核心。應用程式會自行偵測，缺少時該選項會顯示為
  灰色，而熱點在沒有它的情況下照常運作。

## 開始使用

1. 從 [Releases](https://github.com/ravindu644/VirtualAP/releases) 下載最新的 APK
   並安裝。
2. 開啟應用程式並授予 root 權限。它會自行完成所有必要設定，包括可選的
   Magisk/KernelSU 開機模組。無須刷入手續，也無須重新開機。
3. 輸入網路名稱與密碼。
4. 選擇上游：一個**介面**（或自動），或管理模式下的 OpenWrt **容器**。
5. 啟動熱點，然後連接你的裝置。

在設定中啟用 **開機時啟動無線存取點**，重新開機後即可自動恢復。

管理模式中，從已連線的裝置開啟 `http://<your gateway address>` 即可進入 LuCI。

## 更多資訊

- [運作原理](Documentation/How-It-Works.md)：路由、IPv6、橋接、無線、手機上的檔案，
  以及二進位檔與 APK 的建置方式
- [參與貢獻](CONTRIBUTING.md)

## 授權條款

GNU General Public License v3.0。請參閱 [LICENSE](LICENSE)。

VirtualAP 源自 [Ubuntu-Chroot](https://github.com/ravindu644/Ubuntu-Chroot) 的
[`start-hotspot`](https://github.com/ravindu644/Ubuntu-Chroot/blob/main/tools/start-hotspot)
腳本；本應用程式在 AI 助理的協助下撰寫。
