# VirtualAP 如何運作

[English](How-It-Works.md) | [简体中文](How-It-Works.zh-CN.md) | [繁體中文](How-It-Works.zh-TW.md) | [Русский](How-It-Works.ru.md) | [Français](How-It-Works.fr.md) | [Deutsch](How-It-Works.de.md) | [日本語](How-It-Works.ja.md) | [한국어](How-It-Works.ko.md)

VirtualAP 在手機的 Wi-Fi 晶片上建立一個虛擬 AP 介面 `ap0`，在它上面執行 `hostapd`，
然後以兩種模式之一運作。以下所有事情都由 `virtualap` 完成，那是一個以 root 執行、
由 `src/` 建置的靜態 C 二進位檔；應用程式只呼叫它的指令
（`start`、`stop`、`status`、`clients`、`interfaces`、`caps`）。它直接透過
netlink 與核心溝通（連結、位址、路由與規則用 rtnetlink；無線用 nl80211），並透過
原始的 `ip_tables` socket。它會啟動 `hostapd` 與 `dnsmasq`，只為 TTL 修正啟動
`iptables`，而在管理模式之下，啟動 `droidspaces` 以找出容器並在其中執行 OpenWrt 設定。
至於哪些容器正在執行，也就是應用程式在管理模式之下提供的清單，應用程式
直接問 Droidspaces 本身（`droidspaces show --json`，或在 `--json` 之前的發行版上用
`show --format`）。

## 路由模式（應用程式中的「介面」）

VirtualAP 擁有熱點的整個網路層。它給 `ap0` 閘道位址
（`192.168.42.1/24`，除非你選了別的），用自家的 `dnsmasq` 提供 DHCP 與 DNS
（租約從 `.10` 到 `.50`，為期 12 小時，DNS 轉送到你輸入的伺服器，或預設的
`1.1.1.1` 與 `8.8.8.8`），並透過上游對用戶端做 NAT。

流量用釘在 Android 自身規則之上的策略路由規則來導向，那些規則從
優先權 10000 開始，所以 VPN 的攔截全部規則無法搶走熱點的流量：

```
client -> ap0
       -> ip rule pref 7010: from all iif ap0 lookup <upstream table>
       -> MASQUERADE (-s <subnet> ! -d <subnet>)
       -> internet or VPN tunnel

reply  -> ip rule pref 7000: to <subnet> lookup main -> ap0
```

在 Auto 之下，上游是下列第一個介面已啟動的：Android 的 netd
預設網路規則，也就是現在有網際網路的網路，從核心的規則
清單讀出，包含 Android 15 寫它的形式（涵蓋每個 uid 的 uid 範圍）；主
表格的預設路由，給會填入它的 ROM 用；對已知上行名稱的掃描（`wlan*`、
`eth*`、`v4-*`、`rmnet*`、`*ccmni*`），只掃 IPv4，因為永遠啟動的 IMS 承載會
在 IPv6 上符合它們。IPv4 會先走完這三者，才會嘗試 IPv6。熱點、網路共享與 VPN
介面在 Auto 下永遠不會被選中。`-o` 會關掉這一切，改為接受以逗號分隔、
依優先順序排列的清單，沒有排除項：第一個已啟動且有路由的項目
勝出，IPv4 路由走完整份清單之後才輪到 IPv6。只有比 /8 更寬的路由才算數，所以預設路由或
分割通道 VPN 的 `0.0.0.0/5` 算，而介面自己的子網路不算。

`interfaces`（應用程式的下拉式選單會顯示）列出 `-o` 會接受的東西：在任何表格中
有這種路由的連結，兩個位址族都算，所以 IMS 承載也會出現。它略過
VirtualAP、Droidspaces 與 USB Arsenal 建立的東西，以及 netd 的 `dummy0`。

沒有上游是正常狀態，不是錯誤：AP 只帶著規則 7000 起來，
用戶端立刻被拒絕，而監視器在有上游出現時會加上規則 7010。防火牆
規則透過原始的 `ip_tables` socket 在 Android 的 xtables 鎖之下寫入，
那是 netd 與 `iptables -w` 持有的同一把鎖，所以雙方永遠不會
覆蓋對方的變更。

AP 子網路的路由也會映照到 Android 的 `local_network` 表格（97）。沒有
這樣做的話，來自 Droidspaces 容器（來源 `172.28.0.0/16`）回給熱點用戶端的
回覆，會落到 WAN 表格並從實體上行離開，而不是 `ap0`，所以
用戶端無法連到容器內的服務。

### IPv6

用戶端只有在上游有全域位址時才會拿到 IPv6；只發放 ULA 的 Wi-Fi 網路
會讓 AP 停留在只有 IPv4。有兩種給法，依這個順序嘗試：

- NAT66，Droidspaces 的做法，當核心有帶 MASQUERADE 的 `ip6table_nat`，且
  上游的表格有 IPv6 預設路由。`ap0` 在一個私有 /64 中取 `::1`，該 /64 依
  IPv4 子網路編號（`192.168.42.1` 對應 `fd76:6170:42::/64`），`dnsmasq` 用它自己
  作為 DNS 伺服器來通告它，而用戶端經上游被偽裝出去。它
  在上游或前綴改變時原封不動地存活。
- 上游自己的 /64，也就是 Android 網路共享的做法，用在沒有 IPv6 NAT 的核心
  （原廠 GKI 不帶）而上游持有全域 /64 時。`ap0` 在其中取一個 EUI-64
  位址，而 `dnsmasq` 以 off-link 方式通告該前綴，所以用戶端把所有東西都
  送到閘道。每個用戶端位址都以 /128 路由到 `ap0`，並在上游以 NDP 代理項目
  代為回應。監視器每 3 秒從 `ap0` ping `ff02::1` 來找出這些位址，
  那是唯一能揭露它們的方法；它只接受對自己 echo 的回覆，
  絕不接受手機自己的位址，或上游鄰居的位址。規則 7001 會把
  沒有用戶端持有的 /64 位址的上游封包黑洞掉，否則在行動網路上
  它會一直彈跳直到躍點限制耗盡。沒有監視器時，
  前綴不會被通告。

兩者都會加上規則 7002，它給手機自己送往用戶端連結本位址的封包一條
路由：Android 把介面自己的路由歸檔在沒有規則會讀的表格，而 `dnsmasq` 對
路由器請求的答覆就是那些封包之一。IPv6 是盡力而為：當某一步
失敗，AP 就停留在只有 IPv4。

## 監視器

成功啟動後會讓 `virtualap` 以監視器身分持續執行（`run/monitor.pid`），一個
沒有執行緒的行程，排在 Droidspaces 的路由監視器與 USB-Arsenal 的網路共享
常駐程式之後。它監聽核心的連結、位址、路由與規則事件，在一串事件的最後
一個之後 200 ms 查看一次，並且不論如何每 3 秒查看一次，因為 netd 會
在沒有事件的情況下改寫 `iptables`。每次查看：

- 重新解析上游，把規則 7010 與 `run.state` 移到新的上游，所以 Wi-Fi 到
  行動數據的交接不會讓用戶端失去網際網路，而沒有網路的啟動也會以同樣方式
  得到它的第一個上游。什麼都沒找到時，Auto 會放著規則不管，沒有更好的
  東西可以指向，而 `-o` 會移除它，直到清單中的介面回來，
  所以它的流量永遠不會從別的東西離開；
- 把 netd 清掉的任何規則或防火牆項目放回去，以及 `ip_forward`，只在
  缺少時才做，所以它自己的寫入永遠不會再把它喚醒；
- 重做 IPv6、`dnsmasq` 的通告以及隨之的 TTL 修正，當上游的 IPv6
  有變時：新的前綴、另一個介面、IPv6 出現或不見了。

在管理模式中，它轉而監看 veth，並在容器重新啟動時把新的 veth 接進容器。
當 AP 消失時它會結束；`stop` 會在做其他事情之前先殺掉它。

## 管理模式（`-K <container>`）

熱點的 LAN 交給一個執行中的 Droidspaces 容器。VirtualAP 不給 `ap0`
位址、不執行 `dnsmasq`，也不安裝 NAT：它建立一條第二層路徑，容器
就是路由器。

```
client -> ap0 (bridge port, no address)
       -> vap-br0 -> vaplan0 (inside the container)
       -> container LAN (DHCP, DNS, firewall)
       -> container NAT -> container WAN -> internet
```

`ap0` 被納入一個主機橋接 `vap-br0`，它不帶位址，並停用 IPv6，
所以它永遠不會從容器的通告設定自己。一對 veth 在主機側
接入橋接；另一端在容器的網路命名空間內建立，已經命名為 `vaplan0`
並帶著它的 MAC，在同一個請求中完成。在主機上建立、移動後再改名的連結，
在 5.x 之前的核心上會先以舊名稱出現，而 OpenWrt 的 netifd
不會再看它一眼。當容器是 OpenWrt 時，VirtualAP 會用一個經由
`droidspaces run` 執行的 `uci` 腳本佈建它：在設定的閘道上一個靜態的 `vaplan`
介面、一個從路由器 ULA 取來並釘住 IPv4 子網路編號的 IPv6 /64，讓 netifd
無法重新編號它、一個帶路由器通告的 DHCP 集區，以及
重新載入 iptables 防火牆，或當映像檔沒有防火牆套件時加一條 MASQUERADE 規則。
任何其他容器收到 `vaplan0` 後自行設定它。

veth 對在 MTU 1508 運作，而 `ap0` 與用戶端維持在 1500。Samsung 的 Broadcom
韌體會把大約四分之一的接收訊框送上來，末端還留著 8 位元組的 CCMP MIC，
而橋接或 veth 會悄悄釋放任何比它的 MTU 加 18 位元組更長的訊框，所以
在 1500 時，來自用戶端的一個滿尺寸訊框每四次會遺失一次，而傳入
容器的上傳會慢到大約 1 MB/s。容器內的 IP 堆疊會修掉 MIC，
而 RA 用 `ra_mtu` 把用戶端的 IPv6 MTU 釘在 1500，因為 odhcpd 否則會通告
連接埠的 1508。

veth 的兩端都取得 RPS，在每個 CPU 上，但 CPU0 除外。veth 會把每個封包排在
送出它的那個 CPU 上，而一個流會從好幾個 CPU 抵達，因為 Wi-Fi 驅動程式把它
的接收工作分散到各核心，而 TCP 從哪個核心拿到最後一個 ACK 就從那裡送出。
沒有 RPS 時，另一端會看到順序錯亂的流，而 TCP 會重傳從未遺失的資料。
`vaplan0` 的遮罩由一個子行程寫入，它進入容器的網路命名空間並在
那裡掛載自己的 `/sys`，絕不透過 `/proc/<pid>/root`，那些符號連結由
容器選擇。核心在寫入遮罩時只保留當時在線的 CPU，所以在那一刻
停用的核心會留在外面，直到 AP 重新啟動。

新的 veth 每次啟動都有新的隨機 MAC，而 Windows 用閘道的 MAC 作為網路設定檔的
鍵，所以應用程式可以釘住一個：跨啟動保留的隨機 MAC，或手機自己的
Wi-Fi MAC。當容器重新啟動時，
監視器會接上新的 veth，而從重新啟動中存活的 OpenWrt 設定會接手它。
舊的命名空間通常會把 veth 一起帶走，
但只要還有東西持有參照，它就會活得比最後一個行程久，然後主機端
會留下來，凝視著虛無。所以監視器兩邊都檢查：主機端還在，而啟動時
接入的那個行程，仍然在它被接入時所在的命名空間裡。

## TTL 修正

加上 `-T 1` 時，轉送的封包以 TTL 64 離開（IPv6 的躍點限制為 64），所以對
電信業者而言它們看起來像手機自己的。規則住在它們自己的 mangle 鏈
`vap_ttl`，在 `FORWARD` 中依用戶端來源子網路比對，那裡只看得到核心拿走
一跳之後的轉送封包。後端會探測核心的 TTL 目標
（`CONFIG_NETFILTER_XT_TARGET_HL`），方式是在一個沒有東西會跳過去的鏈中加上真正的規則；
探測失敗時應用程式會把該選項變灰。這些是仍然由 `iptables` 二進位檔
寫入的唯一規則。

## 無線

大多數手機晶片只能在工作站連線的頻道上發送信標，所以後端在頻道
適合頻段時跟隨工作站的頻道。工作站的頻道來自掃描結果中它所關聯的 BSS 的
頻率，那正是 `iw dev wlan0 link` 讀的東西，因為 Samsung 的 Broadcom 驅動程式
從介面資訊中省略了頻道。位於 DFS 頻道（52 到 144）的工作站會後備到
2.4 GHz：手機晶片無法在那裡發送信標。位於其他頻段（包含 6 GHz）的工作站
則讓 AP 用它自己的預設值。沒有工作站時，5 GHz 從頻道 36 開始，2.4 GHz 從
頻道 6 開始。

Samsung 的 Broadcom FullMAC 韌體以世界法規網域「99」開機，而框架
有時從不推送真正的國別，所以每個 5 GHz 頻道都維持 NO-IR，`hostapd` 會在
大約五秒後死亡，就在存活檢查之後。wiphy 是自我管理的，所以
`iw reg set` 與 `hostapd` 的 `country_code` 都被忽略；唯一的槓桿是驅動程式的
私有 `COUNTRY` 指令，經由 supplicant 的控制 socket 送出，帶著框架
回報的國別（`ensure_5g_ir`）。它只在 phy 把該頻道回報為
NO-IR 時觸發，而如果之後頻道仍是 NO-IR，AP 會後備到 2.4 GHz。
`hostapd.conf` 保留 `country_code=US`。

寬度跟隨晶片與頻道能做的事，來自該頻段的能力：2.4 GHz 上的 40 MHz
只有在明確要求時才用，5 GHz 上最高到 80 MHz，當頻段列出 VHT 且
頻道位於一個 80 MHz 區塊內。HT40 與 VHT 需要 `hostapd` 跳過它的 20/40 MHz
共存掃描，FullMAC 晶片無法在 AP 介面上執行那個掃描；做到這件事的
`no_pri_sec_switch` 選項是我們 `hostapd` 分支中的補丁，而建置會
斷言該補丁存在。如果 `hostapd` 拒絕 2.4 GHz 上的 40 MHz，後端會
以 20 重試。

那個分支也從不告訴 `hostapd` 某個驅動程式有 MAC ACL，建置也會
斷言這一點。驅動程式的 ACL 能容納的位址比清單少，超過之後 `hostapd` 就
跳過自己的檢查，而驅動程式什麼都沒持有。所以 MAC 清單在每個
晶片上都在 `hostapd` 中檢查：被列出的裝置會短暫關聯，然後在
握手前被移除。Samsung 的 Broadcom 建置把 ACL 編譯掉，gen4m 沒有，
所以它們本來就是這樣運作。Qualcomm 的 qcacld 有一個，所以它是這個變更所針對的驅動程式，而且沒有 qcacld 手機跑過它。

MediaTek 的 gen4m 驅動程式沒有 `dump_station`，所以用戶端數量來自 `hostapd`
自己的工作站清單，經由它的控制 socket，綁定在一個抽象位址上，因為 Android
沒有 `/tmp`，並有一秒逾時，所以卡住的 `hostapd` 不會拖住 `status`。

`clients` 走同一份清單，為每個完成握手的工作站印出 `mac ip name`，
不知道的用 `-`。`status` 中的數量是同一次走訪，所以密碼錯誤的裝置
兩者都不在。應用程式只在它的已連線裝置彈出面板開著時才詢問，因為
管理模式會啟動 `droidspaces`，而那個啟動有五秒限制，因為 libsu 一次
只執行一個指令。位址與名稱來自閘道 /24 中最新的租約：
路由模式下是我們自己的 `dnsmasq.leases`，
管理模式則是容器的 `/tmp/dhcp.leases`。後者是透過
`/proc/<pid>/root` 讀取的，因為 OpenWrt 的 `/tmp` 是容器掛載命名空間中的
tmpfs，一次讀一個元件，而且只接受 tmpfs 上的普通檔案，
所以容器留在那裡的符號連結、FIFO 或綁定掛載都會被拒絕。
沒有送出主機名稱的用戶端，或沒有那個檔案的容器，會讓那些欄位留下 `-`。

使用靜態位址的裝置會保留它原本那個位址的租約，所以核心的 ARP 表
會修正它：路由模式下是 `/proc/net/arp`，管理模式則是 `/proc/<pid>/net/arp`，
在那裡用戶端的 LAN 結束於容器內。租約在表格確認它，或不知道該 MAC
的其他位址時成立，而表格可以把位址給一個沒有租約的裝置。它
只有對送過東西的裝置才有記錄，而且它也會把過期項目標為完成，
所以裝置可能顯示它剛離開的位址。

`block <mac> [name]` 把裝置加入 `/data/local/virtualap/blocklist`，而 `unblock <mac>`
把它移除；`blocked` 為每個印出 `mac name`，沒有名稱的用 `-`。`allow`、`disallow` 與
`allowed` 對 `allowlist` 做同樣的事。`allow-only on` 擋掉不在
允許清單上的每個裝置，踢掉任何已連線的，並且在清單為空時被拒絕；
清單上的最後一個裝置離開會把它關掉，因為那樣就沒人能加入了。沒有參數時
它印出 `on` 或 `off`。那個狀態是檔案 `allow_only`，就在清單旁邊，因為
和它們一樣，它一次只由一個指令改變，熱點開著或關著都一樣。一個裝置只會在一份清單上，
把它加入一份就會從另一份移除，因為 `hostapd` 先檢查允許清單，
否則會讓同時在兩份清單上的裝置進來。

這些清單放在 `ap.conf` 之外，因為應用程式一次只編輯一個項目，不是啟動時
傳入的設定。這些檔案就是 `hostapd` 自己的 `deny_mac_file` 與 `accept_mac_file`，只有在
它們存在時才會在其設定中指名，因為 `hostapd` 不會在它打不開的清單上啟動。一行是
`mac #name`：`hostapd` 透過 `atoi` 把 MAC 之後的東西讀成 VLAN id，而 `#` 讓它成為
0，也就是沒有 VLAN。正在執行的熱點會經由控制 socket 聽到變更：`DENY_ACL ADD_MAC`
會在裝置已連線時踢掉它，而一旦只有允許的裝置能加入，`ACCEPT_ACL DEL_MAC` 也一樣。
啟動時會先改寫這些檔案，記錄並丟棄任何它不確定
`hostapd` 能否讀取的行，而每個檔案大約容納 400 台裝置。

應用程式從它的裝置彈出面板封鎖時不確認，因為 Unblock 就在旁邊，並
在設定中保留兩份清單，供熱點關閉時使用。它的 MAC 欄位除了十六進位
數字之外什麼都容不下，並拒絕引擎拒絕的東西：全零與群組位址。任一份清單
都只保存那個 MAC。自己換位址的裝置會繞過封鎖，也會從允許清單上
掉下來，而手機預設為每個網路使用隨機位址，所以要輸入的是
裝置 Wi-Fi 詳細資料中針對這個熱點的那一個。

## 在手機上

所有東西都在 `/data/local/virtualap` 下：

- `bin/`：`virtualap`、`hostapd` 與 `dnsmasq`，為手機架構準備的完全靜態 musl 二進位檔。
  `iptables` 是 Android 自己的。
- `ap.conf`：熱點的設定，以 `KEY='value'` 行表示。每當畫面上的內容
  有效時以及每次啟動前，應用程式都會寫它，而半填妥的表單會留在畫面上。
  引擎只讀它，而旗標會在該次執行中覆寫它。它存有 Wi-Fi 密碼，所以
  只有 root 可以讀取。
- `blocklist`、`allowlist` 與 `allow_only`：被封鎖與允許的裝置，以及是否只有
  那些能加入，見[上文](#the-radio)。在第一次使用之前它們都不存在。
- `.boot_mode`：熱點在開機時啟動則為 `1`。設定的開關會寫它，而開機
  模組會讀它。
- `run.state`：執行中的 AP 需要拆除的東西、閘道、上游、IPv6
  模式與介面，以及要還原的 sysctl 值。`status` 以它和
  `hostapd` pid 檔案判斷「執行中」；`stop` 在結束前會移除它。
- `run/`：pid 檔案（`hostapd`、`dnsmasq`、監視器）、產生的 `hostapd.conf`
  與 `dnsmasq.conf`、租約、IPv6 用戶端清單、`hostapd` 的控制 socket，以及
  讓兩次啟動或停止不會同時執行的鎖。
- `logs/`：`ap.log`，應用程式顯示的每一行與監視器所做之事的帶時間戳副本，
  每次啟動時截斷，外加 `hostapd.log` 與 `dnsmasq.log`，因為兩個
  常駐程式否則會把失敗原因一起帶走，以及 `boot-module.log`，也就是
  開機模組在上次開機時做的事。

pid 檔案只有在它指名的行程執行預期程式時才會被處理，所以重新開機後被
重複使用的 pid 永遠不會收到訊號。應用程式在首次執行時部署 `bin/`，之後
每當 APK 附帶不同的二進位檔時再部署：建置會把 `PAYLOAD_VERSION` 寫在
它們旁邊，那是每個架構的二進位檔與開機模組的雜湊，而應用程式會拿它與
上次部署的那個比對。

部署後端也會安裝 `/data/adb/modules/virtualap`，也就是開機模組，方式與
Droidspaces 安裝它自己的相同。沒有 root 管理員的手機會沒有它，而只寫
`.boot_mode` 的設定開關，就沒有東西可讀它。模組的 `service.sh` 在
late start 執行，並等待 `sys.boot_completed`。`.boot_mode` 開啟時，它給
Wi-Fi 介面 25 秒出現，而在管理模式中，再給容器 25 秒執行，任一者
沒有達成時就跳過啟動。它不等網路，因為熱點會跟隨稍後才到的上游。
啟動會讀 `ap.conf`，它位於應用程式的加密儲存之外，所以開機啟動
不需要解鎖。結果會以模組的描述顯示在 root 管理員中。成功啟動後
它等待第一次解鎖，在那之前應用程式無法執行，並向 `HotspotBootReceiver`
送出廣播，由它掛上執行中的通知。

## 建置

`hostapd` 與 `dnsmasq` 由 `scripts/build-static.sh` 從 `externals/` 下的 git 子模組建置，
那是我們自己的分支。每個架構在該架構的 Alpine 3.23 容器中建置，
在 x86 主機上透過 QEMU，帶 `-static -no-pie`，所以結果是沒有解譯器的
獨立 ELF。`dnsmasq` 分支拿掉了 `getpwnam("root")` 特權降級，因為 Android
在 `/etc/passwd` 中沒有 root 條目，而靜態 musl 不會合成一個。`virtualap`
只連結 libc，不需要容器：根目錄的 `Makefile` 以
Droidspaces 的方式建置它，用 `~/toolchains/<triplet>-cross`（或 `MUSL_CROSS`）中的
musl-cross gcc 執行 `make aarch64 armhf`，`buildBackend` gradle 工作會在
每次應用程式建置前執行它。輸出落在被 gitignore 的 `backend/<arch>/`，而
`prepareAssets` gradle 工作會把三個二進位檔放進 APK 的 assets，缺少
任何一個時就失敗。

`src/` 每一層一個資料夾，每個都在 `src/include/<folder>/` 下有一個公開標頭檔，
而當它的檔案共用更多東西時，在原始碼旁有一個私有的 `internal.h`：
`core`、`cli`、`net`（rtnetlink，以及 `net/ipt/` 中的 `ip_tables` 引擎）、`radio`
（nl80211、控制 socket、頻道策略）、`mode`（路由、管理、IPv6、`hostapd` 與
`dnsmasq`）、`monitor` 與 `cmd`。`make check` 會執行 `tests/` 中的斷言，
netlink 的那些會在一個用過即丟的使用者與網路命名空間中對核心執行，
所以既不需要 root 也不需要手機；為手機建置並在手機上以 root 執行時，它們也會
實際驅動真正的 `ip_tables`。

CI 在 ARM runner 上為每個架構建置 `hostapd` 與 `dnsmasq`，快取在
`externals/` 與 `scripts/` 目錄樹上。接著 APK 工作會取得 Droidspaces 發佈的 musl
工具鏈（對它們的 sha256 檢查，並快取在它們的發佈 URL），執行
`make check` 與 `make aarch64 armhf`，並把警告當成錯誤，然後用
JDK 17 建置 APK。推送的 `v*` 標籤若符合 `src/include/virtualap.h` 中的 `VAP_VERSION`，
就會發佈一個包含 APK、`SHA256SUMS` 檔案以及自前一個標籤以來每個提交的
變更記錄的發行版；標籤必須符合它，而且簽署金鑰必須存在，否則執行會
停止。CI 不執行任何格式化工具，也不執行任何 linter。

`make bump NEW=x.y.z` 會改寫 `VAP_VERSION`，而 Makefile、gradle 與 CI 都
從那裡讀它。APK 的 versionCode 是發行版代碼（major·10000 + minor·100 + patch）
乘以 10000 加上 `git rev-list --count HEAD`，所以每個提交的 APK 都能覆蓋安裝前一個。
計數需要完整歷史，這就是 APK 工作以 `fetch-depth: 0` 簽出的原因：
淺層複製計為 1，而沒有 `.git` 的目錄樹計為 0。開機模組的
`module.prop` 只取發行版代碼，因為它會被雜湊進 `PAYLOAD_VERSION`。

## 翻譯

應用程式的文字在 `Android/app/src/main/res/values/strings.xml`，而它的複數形在
旁邊的 `plurals.xml`，兩個 Weblate 元件，`Strings` 與 `Plurals`，與
Droidspaces 相同的版面配置。兩者都使用 Android String Resource 檔案格式，檔案遮罩
`Android/app/src/main/res/values-*/strings.xml` 或 `plurals.xml`，基底檔案在 `values/`
中同時作為單一語言基底檔與新翻譯的範本，以及
Android 風格的語言代碼（`values-pt-rBR`）。開啟 Weblate 的 Cleanup translation files
附加元件：從基底檔案移除卻留在翻譯中的鍵，是 lint 的 `ExtraTranslation`，
那是致命的，所以發行建置會失敗直到它被移除。缺少的翻譯只會
後備到英文。

應用程式跟隨系統語言，直到設定的語言選擇器另有指示。選擇器的
清單與 Android 13 的個別應用程式語言清單是同一個檔案：`generateLocaleConfig` 從
`values-*` 資料夾寫出 `res/xml/_generated_res_locale_config.xml`，
`res/resources.properties` 說 `res/values` 是英文，而 `LocaleHelper` 讀取結果，
所以從 Weblate 合併進來的語言會在下一次建置時提供，不論它翻譯了多少。
選擇會經過 AppCompat，這就是 `MainActivity` 是 `AppCompatActivity` 的原因：
Android 13 與之後的版本把它保存在系統中，而在 8 到 12 上 AppCompat 自己儲存它
並包裝 activity，而不是 application 或服務，所以在那裡通知與快速設定磚
會留在系統語言。在任何版本上，已經顯示的通知會在下次更新時採用新語言。
日誌保持英文，那是引擎印出來的東西。
那個日誌與 QR 彈出面板上的密碼所用的 JetBrains Mono，是唯一
隨附的字型；每個翻譯過的標籤都用系統字型，所以任何文字系統都能繪製。
