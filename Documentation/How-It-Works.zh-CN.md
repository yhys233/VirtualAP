# VirtualAP 如何工作

[English](How-It-Works.md) | [简体中文](How-It-Works.zh-CN.md) | [繁體中文](How-It-Works.zh-TW.md) | [Русский](How-It-Works.ru.md) | [Français](How-It-Works.fr.md) | [Deutsch](How-It-Works.de.md) | [日本語](How-It-Works.ja.md) | [한국어](How-It-Works.ko.md)

VirtualAP 在手机的 Wi-Fi 芯片上创建一个虚拟 AP 接口 `ap0`，在它上面运行 `hostapd`，
然后以两种模式之一工作。下面所有事情都由 `virtualap` 完成，一个以 root 身份运行、
由 `src/` 构建的静态 C 二进制；应用只调用它的命令
（`start`、`stop`、`status`、`clients`、`interfaces`、`caps`）。它通过 netlink 直接
与内核通信（链路、地址、路由和规则用 rtnetlink；射频用 nl80211），也通过
原始的 `ip_tables` 套接字通信。它会启动 `hostapd` 和 `dnsmasq`，仅为 TTL 修正启动
`iptables`，并且在托管模式下启动 `droidspaces` 以找到容器并在其中运行 OpenWrt 的配置。
哪些容器正在运行，也就是应用为托管模式提供的列表，
应用是直接问 Droidspaces 本身的（`droidspaces show --json`，或者在没有 `--json` 的
旧版本上用 `show --format`）。

## 路由模式（应用中的“接口”）

VirtualAP 掌管热点的整个网络层。它给 `ap0` 配上网关地址
（`192.168.42.1/24`，除非你另选了一个），用自带的 `dnsmasq` 提供 DHCP 和 DNS
（租约从 `.10` 到 `.50`，有效期 12 小时，DNS 转发到你输入的服务器，默认是
`1.1.1.1` 和 `8.8.8.8`），并把客户端经上游做 NAT 送出。

流量用策略路由规则来引导，这些规则固定在 Android 自己的规则之上，Android 的规则从
优先级 10000 开始，所以 VPN 的兜底规则夺不走热点的流量：

```
client -> ap0
       -> ip rule pref 7010: from all iif ap0 lookup <upstream table>
       -> MASQUERADE (-s <subnet> ! -d <subnet>)
       -> internet or VPN tunnel

reply  -> ip rule pref 7000: to <subnet> lookup main -> ap0
```

在 Auto 模式下，上游是这些当中第一个接口已启动的：Android netd 的
默认网络规则，也就是当前能上网的那个网络，从内核的规则列表中
读出，包括 Android 15 写它的那种形式（覆盖所有 uid 的 uid 范围）；主表的
默认路由，用于那些会填充它的 ROM；以及对已知上行接口名的扫描（`wlan*`、
`eth*`、`v4-*`、`rmnet*`、`*ccmni*`），只扫 IPv4，因为始终在线的 IMS 承载会在 IPv6 上
命中它们。IPv4 走完这三项之后才会尝试 IPv6。热点、USB 网络共享和 VPN 的
接口在 Auto 下从不被选中。`-o` 会把这一切关掉，改为接受一个按优先级排序的、
逗号分隔的列表，没有任何排除项：列表里第一个已启动且有路由的条目胜出，
整份列表里的 IPv4 路由都优先于 IPv6。只有宽于 /8 的路由才算数，所以默认路由或者分离隧道
VPN 的 `0.0.0.0/5` 算，而接口自己的子网不算。

`interfaces` 也就是应用下拉框显示的内容，列出的是 `-o` 会接受的东西：在任何表中
拥有这样一条路由的链路，两个地址族都算，所以 IMS 承载也会出现。它会排除
VirtualAP、Droidspaces 和 USB Arsenal 创建的接口，以及 netd 的 `dummy0`。

没有上游是一种正常状态，不是错误：AP 只带着规则 7000 起来，
客户端立刻被拒绝，等上游出现时监控器再把规则 7010 加上。防火墙规则
通过原始的 `ip_tables` 套接字、在 Android 的 xtables 锁之下写入，
这与 netd 和 `iptables -w` 持有的锁相同，所以任何一方都不会
覆盖对方的改动。

AP 子网的路由还会被镜像到 Android 的 `local_network` 表（97）。没有
这一步，从 Droidspaces 容器（源 `172.28.0.0/16`）发往热点客户端的回复
就会落到 WAN 表，经物理上行离开而不是从 `ap0` 出去，
于是客户端无法访问容器内的服务。

### IPv6

只有当上游有全局地址时客户端才会拿到 IPv6；只下发 ULA 的 Wi-Fi 网络
会让 AP 停留在只有 IPv4。提供 IPv6 有两种方式，按这个顺序尝试：

- NAT66，Droidspaces 的做法，前提是内核有带 MASQUERADE 的 `ip6table_nat`，
  并且上游的表里有 IPv6 默认路由。`ap0` 取得一个私有 /64 里的 `::1`，该 /64 的
  编号接在 IPv4 子网之后（`192.168.42.1` 对应 `fd76:6170:42::/64`），`dnsmasq` 以
  自己为 DNS 服务器通告它，客户端经上游做 masquerade。它在上游或前缀
  变化时原样存活。
- 上游自己的 /64，Android 网络共享的做法，用于没有 IPv6 NAT 的内核
  （原版 GKI 不带），前提是上游持有一个全局 /64。`ap0` 在其中取得一个
  EUI-64 地址，`dnsmasq` 以 off-link 方式通告该前缀，这样客户端把一切都
  发给网关。每个客户端地址都作为 /128 路由到 `ap0`，并在上游用一条
  NDP 代理条目代为应答。监控器每 3 秒从 `ap0` ping 一次 `ff02::1`
  来发现这些地址，这是唯一能揭示它们的手段；它只接受对自己发出的
  echo 的回复，绝不接受本机自己的地址或者上游邻居的地址。
  规则 7001 会把 /64 中没有任何客户端持有的地址的上游数据包黑洞掉，
  否则在蜂窝网络上它们会一直弹跳直到跳数上限耗尽。没有
  监控器，这个前缀就不会被通告。

两种方式都会加上规则 7002，它给手机自己发往客户端链路本地地址的包
提供一条路由：Android 会把接口自己的路由放在没有任何规则读取的表里，而
`dnsmasq` 对路由请求的应答正是这类包之一。IPv6 是尽力而为：当某一步
失败时，AP 就停留在只有 IPv4。

## 监控器

成功启动之后，`virtualap` 会作为监控器继续运行（`run/monitor.pid`），一个
没有线程的进程，排在 Droidspaces 的路由监控器和 USB-Arsenal 的网络共享守护进程之后。它
监听内核的链路、地址、路由和规则事件，在一次事件爆发的最后一个事件之后 200 毫秒查看一次，
并且无论如何每 3 秒查看一次，因为 netd 会不带事件地重写 `iptables`。
每次查看：

- 重新解析上游，并把规则 7010 和 `run.state` 移到新的上游，这样 Wi-Fi 到
  移动数据的切换不会让客户端失去网络，而没有网络时的启动也能以
  同样的方式拿到第一个上游。什么都没找到时，Auto 保持规则不变，
  因为没有更好的目标可指向，而 `-o` 会移除它，直到列表中的接口回来，
  这样它的流量绝不会从其他任何地方出去；
- 把 netd 清掉的任何规则或防火墙条目以及 `ip_forward` 放回去，只在
  缺失时才写，这样它自己的写入不会再把自己唤醒；
- 当上游的 IPv6 发生变化时，重做 IPv6、`dnsmasq` 的通告以及随之的 TTL 修正：
  例如新的前缀、另一个接口、IPv6 出现或消失。

在托管模式下它改为监视 veth，并在容器重启时
把新的 veth 接入容器。AP 消失时它就退出；`stop` 会在做任何其他事之前先杀掉它。

## 托管模式（`-K <container>`）

热点的 LAN 被交给一个正在运行的 Droidspaces 容器。VirtualAP 不给 `ap0`
地址，不运行 `dnsmasq`，也不安装 NAT：它构建一条二层通路，
由容器充当路由器。

```
client -> ap0 (bridge port, no address)
       -> vap-br0 -> vaplan0 (inside the container)
       -> container LAN (DHCP, DNS, firewall)
       -> container NAT -> container WAN -> internet
```

`ap0` 被从属于一个宿主网桥 `vap-br0`，该网桥不携带地址，并禁用了 IPv6，
这样它绝不会根据容器的通告来配置自己。一对 veth
在宿主侧接入该网桥；另一端在容器的网络命名空间内创建，已经命名为 `vaplan0` 并带有它的 MAC，
全部在一次请求里完成。在宿主上创建、移动之后再改名的链路，在 5.x 之前的内核上
会先以旧名字出现，而 OpenWrt 的 netifd 不会再看它一眼。当容器是 OpenWrt 时，VirtualAP
用一个通过 `droidspaces run` 执行的 `uci` 脚本为它做配置：在配置好的网关上建一个静态的 `vaplan`
接口，一个取自路由器 ULA 的 IPv6 /64，其编号固定为 IPv4
子网的编号，这样 netifd 无法重新编号，一个带路由通告的 DHCP 池，以及
重载 iptables 防火墙，或者在镜像没有防火墙
软件包时加一条 MASQUERADE 规则。
任何其他容器收到 `vaplan0` 后自行配置。

veth 对以 MTU 1508 运行，而 `ap0` 和客户端保持在 1500。Samsung 的 Broadcom
固件在递交约四分之一的接收帧时，末尾仍带着 8 字节的 CCMP MIC，
而网桥或 veth 会静默释放任何长于其 MTU 加 18 字节的帧，所以在
1500 时，客户端发来的满尺寸帧四次里会丢一次，上传到
容器的速度只有大约 1 MB/s。容器内的 IP 协议栈会截掉 MIC，而
RA 通过 `ra_mtu` 把客户端的 IPv6 MTU 钉在 1500，因为 odhcpd 否则会通告
端口的 1508。

veth 的两端都设置 RPS，除 CPU0 之外的每个 CPU 都算。veth 会把每个包排到
发送它的那个 CPU 上，而一条流会来自多个 CPU，因为 Wi-Fi 驱动把
接收工作分散到多个核上，而 TCP 从拿到最后一个 ACK 的那个核发送。没有 RPS，
远端会看到这条流乱序，TCP 就会重传从未丢失的数据。
`vaplan0` 的掩码由一个进入容器网络命名空间并在其中挂载自己的 `/sys` 的子进程写入，
绝不通过 `/proc/<pid>/root`，那里面的符号链接由容器决定。写入掩码时内核
只保留当时在线的 CPU，所以那一刻被停用的核
会一直缺席，直到 AP 重启。

新建的 veth 每次启动都会有一个新的随机 MAC，而 Windows 的网络配置文件以
网关的 MAC 为键，所以应用可以把 MAC 固定下来：一个跨启动保持的随机 MAC，或者手机自己的
Wi-Fi MAC。容器重启时，监控器会接入一个新的 veth，而重启后仍然存在的
OpenWrt 配置会把它捡起来。旧的命名空间通常会带着 veth 一起走，
但只要还有东西持有引用，它就会比最后一个进程活得更久，于是宿主这一端
留了下来，对面却什么都没有。所以监控器两边都检查：宿主这一端在，
并且启动时接入的那个进程
仍然在它被接入的那个命名空间里。

## TTL 修正

使用 `-T 1` 时，被转发的包以 TTL 64 离开（IPv6 是跳数上限 64），所以在
运营商看来它们就像手机自己发出的一样。规则放在一条自己的 mangle 链里，
即 `vap_ttl`，按客户端的源子网在 `FORWARD` 中匹配，而 `FORWARD` 只在内核减掉
一跳之后才看到被转发的包。后端通过把真正的规则加进一条没人跳转的链，
来探测内核的 TTL 目标（`CONFIG_NETFILTER_XT_TARGET_HL`）；
探测失败时应用会把该选项置灰。
这些是仍然由 `iptables` 二进制写入的唯一一类规则。

## 射频

大多数手机芯片只能在与站点相连的信道上发送信标，所以当站点信道
符合频段时，后端会跟随它。站点信道来自扫描结果中它所关联的
BSS 的频率，这也正是 `iw dev wlan0 link` 读取的东西，
因为 Samsung 的 Broadcom 驱动在接口信息里省略了信道。处于 DFS 信道
（52 到 144）的站点会回退到 2.4 GHz：手机芯片无法在那里发信标。位于其他频段
（包括 6 GHz）的站点会让 AP 使用自己的默认值。
没有站点时，5 GHz 从信道 36 起，2.4 GHz 从信道 6 起。

Samsung 的 Broadcom FullMAC 固件启动在世界管制域 “99”，而 framework
有时从不推送真正的国家码，所以每个 5 GHz 信道都保持 NO-IR，`hostapd`
在存活检查之后大约五秒就死掉。wiphy 是自管理的，所以
`iw reg set` 和 `hostapd` 的 `country_code` 会被忽略；唯一可用的杠杆是驱动
通过 supplicant 控制套接字发送的私有 `COUNTRY` 命令，发送的是 framework
报告的国家码（`ensure_5g_ir`）。它只在 phy 把该信道报告为
NO-IR 时才触发，而如果之后该信道仍是 NO-IR，AP 就回退到 2.4 GHz。
`hostapd.conf` 保留 `country_code=US`。

宽度遵循芯片和信道能做什么，依据该频段的能力：2.4 GHz 上只有在按名字
明确要求时才用 40 MHz，5 GHz 上在该频段列出 VHT 且信道位于 80 MHz 区块内时
最高到 80 MHz。HT40 和 VHT 需要 `hostapd` 跳过它的 20/40 MHz
共存扫描，而 FullMAC 芯片无法在 AP 接口上运行这个扫描；实现这一点的
`no_pri_sec_switch` 选项是我们 `hostapd` 分支里的一个补丁，构建时会
断言该补丁存在。如果 `hostapd` 在 2.4 GHz 上拒绝 40 MHz，后端会以
20 重试。

这个分支也从不会告诉 `hostapd` 某个驱动带有 MAC ACL，构建时同样会断言这一点。
驱动的 ACL 能容纳的地址可能少于列表，超过之后 `hostapd` 会跳过它
自己的检查，而驱动那边什么都没持有。所以 MAC 列表在每个芯片上
都由 `hostapd` 检查：列表中的设备会短暂关联，并在握手之前被移除。Samsung 的
Broadcom 构建把 ACL 编译掉了，gen4m 则根本没有，所以它们本来就是这样工作的。
Qualcomm 的 qcacld 有 ACL，所以这次改动的驱动就是它，而还没有任何 qcacld 手机运行过它。

MediaTek 的 gen4m 驱动没有 `dump_station`，所以客户端数量来自 `hostapd`
自己的站点列表，经由它的控制套接字获取，绑定到一个抽象地址，因为 Android 没有
`/tmp`，并带一秒超时，这样一个卡住的 `hostapd` 不会拖住 `status`。

`clients` 走同一份列表，为每台完成握手的站点打印 `mac ip name`，
未知的用 `-`。`status` 里的计数也是同一次遍历，所以
密码错误的设备在两者里都不出现。应用只在它的已连接设备面板打开时
才询问，因为托管模式会启动 `droidspaces`，而那次启动有
五秒上限，因为 libsu 一次只运行一条命令。地址
和名称来自网关 /24 中最新的租约：路由模式下是我们自己的 `dnsmasq.leases`，
托管模式下是容器的 `/tmp/dhcp.leases`。后者通过
`/proc/<pid>/root` 读取，因为 OpenWrt 的 `/tmp` 是容器挂载命名空间里的一个 tmpfs，
逐级读取分量，并且只接受 tmpfs 上的普通文件，所以容器留在那里的符号链接、FIFO 或绑定
挂载都会被拒绝。没有发送主机名的客户端，或者没有该文件的容器，
这些字段就保持为 `-`。

使用静态地址的设备会保留它原有地址的租约，所以内核的 ARP 表会
纠正它：路由模式下读 `/proc/net/arp`，托管模式下读 `/proc/<pid>/net/arp`，那里的
客户端 LAN 终结在容器内。租约在该表确认它、或者该表不知道
这个 MAC 的其他地址时成立，而该表也可以给一个没有租约的设备
分配地址。它只有在一个设备发过东西之后才有该设备，而且它也会
把过期的条目标成 complete，所以设备可能显示一个它刚刚离开的地址。

`block <mac> [name]` 把设备加入 `/data/local/virtualap/blocklist`，`unblock <mac>`
把它移出；`blocked` 为每一台打印 `mac name`，没有名字的用 `-`。`allow`、`disallow` 和
`allowed` 对 `allowlist` 做同样的事。`allow-only on` 会把不在
白名单上的设备全部挡在外面，把已连接的踢掉，并且在列表为空时被拒绝；最后
一台设备离开列表就会把它关掉，因为否则没人能加入。不带参数时它打印 `on` 或
`off`。这个状态就是列表旁边的文件 `allow_only`，因为
和列表一样，它一次只被一条命令改动，热点开着或关着都行。一台设备只在一份列表里，
把它加进一份就会从另一份移除，因为 `hostapd` 先检查白名单，否则
会放一台同时在两份列表上的设备进来。

这些列表放在 `ap.conf` 之外，因为应用是一条一条地编辑它们，而不是一个
启动时传入的设置。这些文件就是 `hostapd` 自己的 `deny_mac_file` 和 `accept_mac_file`，只在它们存在时
才写进它的配置，因为 `hostapd` 不会在它打不开的列表上启动。一行是
`mac #name`：`hostapd` 通过 `atoi` 把 MAC 之后的内容读作 VLAN id，而 `#` 使其为
0，即没有 VLAN。正在运行的热点会通过控制套接字听到变化：`DENY_ACL ADD_MAC`
在设备已连接时把它断开，而在只允许白名单设备加入之后，`ACCEPT_ACL DEL_MAC`
也一样。每次启动会先重写这些文件，记录并丢弃任何它不确定
`hostapd` 能否读取的行，每份大约能放 400 台设备。

应用从设备面板中阻止设备时不要求确认，因为 Unblock 就在旁边，
它把两份列表都放在设置里，供热点关闭时使用。它的 MAC 字段只能放
十六进制数字，并拒绝引擎所拒绝的：全零和组地址。任一份列表
都只保存那个 MAC。自己换 MAC 的设备能绕过阻止，也会从允许列表中
掉出去，而手机默认会为每个网络使用随机地址，所以要填的是
设备 Wi-Fi 详情里针对这个热点的那个地址。

## 在手机上

一切都在 `/data/local/virtualap` 下：

- `bin/`：`virtualap`、`hostapd` 和 `dnsmasq`，面向手机架构的完全静态 musl
  二进制。`iptables` 用的是 Android 自己的。
- `ap.conf`：热点的设置，以 `KEY='value'` 行表示。应用在屏幕上的内容有效时以及每次启动
  之前写入它，而半填好的表单会留在屏幕上。
  引擎只读取它，某次运行的标志会覆盖它。它保存着 Wi-Fi 密码，
  所以只有 root 可读。
- `blocklist`、`allowlist` 和 `allow_only`：被阻止和允许的设备，以及是否只允许
  这些设备加入，见[上文](#射频)。在首次使用之前它们都不存在。
- `.boot_mode`：热点开机自启时为 `1`。设置里的开关写入它，
  开机模块读取它。
- `run.state`：正在运行的 AP 需要拆除的东西，网关、上游、IPv6
  模式与接口，以及要恢复的 sysctl 值。`status` 依据它和 `hostapd` 的 pid 文件判断
  “running”；`stop` 在退出之前删除它。
- `run/`：pid 文件（`hostapd`、`dnsmasq`、监控器）、生成的 `hostapd.conf`
  和 `dnsmasq.conf`、租约、IPv6 客户端列表、`hostapd` 的控制套接字，
  以及防止两次启动或停止同时运行的锁。
- `logs/`：`ap.log`，应用显示过的每一行以及监控器所做之事的带时间戳副本，
  每次启动时截断，另外还有 `hostapd.log` 和 `dnsmasq.log`，因为否则这两个
  守护进程会把失败原因一起带走，以及 `boot-module.log`，即开机模块
  上一次启动时做了什么。

只有当 pid 文件所指的进程正在运行预期程序时才会对它采取动作，所以重启后
被复用的 pid 永远不会被发信号。应用在首次运行以及每次 APK 附带的二进制
有变化时部署 `bin/`：构建会把 `PAYLOAD_VERSION`，一个由各架构二进制和开机模块
算出的哈希，写在它们旁边，应用把它与
上次部署的那个做比较。

部署后端同时也会安装 `/data/adb/modules/virtualap`，即开机模块，方式与
Droidspaces 安装自己的模块相同。没有 root 管理器的手机就没有它，而设置里的开关
只写 `.boot_mode`，于是没有东西去读它。模块的 `service.sh` 在 late start 阶段运行，
并等待 `sys.boot_completed`。`.boot_mode` 打开时，它给 Wi-Fi 接口 25
秒时间出现，并且在托管模式下再给容器 25 秒运行，
任一未出现就跳过启动。它不等待网络，
因为热点会跟随稍后才出现的上游。启动会读取 `ap.conf`，它位于应用的
加密存储之外，所以开机启动不需要解锁。结果显示为 root 管理器里
模块的描述。成功启动之后，它等待第一次解锁，在那之前应用无法运行，
然后向 `HotspotBootReceiver` 发送一个广播，由它放出运行中的通知。

## 构建

`hostapd` 和 `dnsmasq` 由 `scripts/build-static.sh` 从 `externals/` 下的 git 子模块
构建，那是我们自己的分支。每个架构都在该架构的 Alpine 3.23 容器里构建，
在 x86 宿主上通过 QEMU 运行，使用 `-static -no-pie`，所以产物是一个没有
解释器的独立 ELF。`dnsmasq` 分支去掉了 `getpwnam("root")` 的降权，因为 Android
的 `/etc/passwd` 里没有 root 条目，而且静态 musl 不会合成一个。`virtualap`
只链接 libc，不需要容器：根目录的 `Makefile` 按
Droidspaces 的方式构建它，用 `~/toolchains/<triplet>-cross`（或 `MUSL_CROSS`）中的
musl-cross gcc 执行 `make aarch64 armhf`，`buildBackend` gradle 任务会在
每次应用构建之前运行它。产物落在 `backend/<arch>/`，该目录已被 gitignore，
`prepareAssets` gradle 任务把这三个二进制放入 APK 的 assets，任何一个
缺失就失败。

`src/` 按层一个目录，每个目录在 `src/include/<folder>/` 下有一个公开头文件，
并且当它的文件共享更多东西时，在源码旁边有一个私有的 `internal.h`：
`core`、`cli`、`net`（rtnetlink，以及 `net/ipt/` 中的 `ip_tables` 引擎）、`radio`
（nl80211、控制套接字、信道策略）、`mode`（路由、托管、IPv6、`hostapd` 和
`dnsmasq`）、`monitor` 和 `cmd`。`make check` 运行 `tests/` 中的断言，netlink 那些
在一个一次性的用户和网络命名空间里对内核运行，所以既不需要 root 也不需要
手机；为手机构建并在那里以 root 运行，它们还会实际用到真实的
`ip_tables`。

CI 在 ARM runner 上为每个架构构建 `hostapd` 和 `dnsmasq`，缓存基于
`externals/` 和 `scripts/` 两棵树。APK 任务随后获取 Droidspaces 发布的 musl
工具链（用它们的 sha256 校验，并按它们的发布 URL 缓存），运行
`make check` 和把警告视为错误的 `make aarch64 armhf`，并用
JDK 17 构建 APK。推送的 `v*` 标签如果与 `src/include/virtualap.h` 中的 `VAP_VERSION` 匹配，
就会发布一个 release，包含 APK、一个 `SHA256SUMS` 文件以及自上一个标签以来的
每个提交的变更日志；标签必须与它匹配，签名密钥必须存在，否则运行就停止。
CI 不运行格式化工具，也不运行 linter。

`make bump NEW=x.y.z` 重写 `VAP_VERSION`，Makefile、gradle 和 CI 都从
那里读取它。APK 的 versionCode 是发布编号（major·10000 + minor·100 + patch）
乘以 10000 再加上 `git rev-list --count HEAD`，所以每个提交的 APK 都能覆盖安装上一个。
这个计数需要完整历史，这就是 APK 任务用 `fetch-depth: 0` 检出的原因：浅克隆
算作 1，而没有 `.git` 的树算作 0。开机模块的
`module.prop` 只取发布编号，因为它会被哈希进 `PAYLOAD_VERSION`。

## 翻译

应用的文本在 `Android/app/src/main/res/values/strings.xml`，它的复数形式在
旁边的 `plurals.xml`，两个 Weblate 组件，`Strings` 和 `Plurals`，布局与
Droidspaces 相同。两者都使用文件格式 Android String Resource，文件掩码为
`Android/app/src/main/res/values-*/strings.xml` 或 `plurals.xml`，`values/` 中的基础文件
既是单语基础语言文件，也是新翻译的模板，并且使用
Android 风格的语言代码（`values-pt-rBR`）。打开 Weblate 的 Cleanup translation files
附加组件：从基础文件删掉却留在译文里的键就是 lint 的 `ExtraTranslation`，
它是致命的，所以在它消失之前发布构建都会失败。缺失的翻译只会回退
到英文。

应用跟随系统语言，直到设置里的语言选择器另有指定。选择器的
列表和 Android 13 的按应用语言列表是同一个文件：`generateLocaleConfig` 从
`values-*` 目录写出 `res/xml/_generated_res_locale_config.xml`，`res/resources.properties`
说明 `res/values` 是英文，而 `LocaleHelper` 读取结果，所以从 Weblate 合并进来的
语言会在下一次构建时出现，无论它翻译了多少。选择经过
AppCompat，这就是 `MainActivity` 是 `AppCompatActivity` 的原因：Android 13 及以后把它保存在
系统里，而在 8 到 12 上 AppCompat 自己保存它并包装 activity，而不是 application
或 service，所以在那里通知和快捷设置磁贴仍停留在系统语言。在任何
版本上，已经弹出的通知会在下一次更新时换成新语言。
日志保持英文，那是引擎打印出来的内容。
JetBrains Mono，用于那份日志和二维码面板上的密码，是唯一
捆绑的字体；每个翻译后的标签都使用系统字体，所以任何文字都能绘制。
