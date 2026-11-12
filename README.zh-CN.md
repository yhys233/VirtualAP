# VirtualAP

[English](README.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md) | [Русский](README.ru.md) | [Français](README.fr.md) | [Deutsch](README.de.md) | [日本語](README.ja.md) | [한국어](README.ko.md)

把一台已 root 的 Android 手机变成真正的 Wi-Fi 路由器。

VirtualAP 会在手机的 Wi-Fi 芯片上运行一个完整的接入点。网络名称、密码、频段和信道，
你的设备看到的网关地址，以及它们的流量走向——移动数据、Wi-Fi 还是 VPN——都由你决定。

## 你可以用它做什么

- **让 OpenWrt 担任热点的路由器。** 防火墙区域、端口转发、流量规则、带宽限制以及
  LuCI 网页界面，全部运行在你的手机上。详见下方的托管模式。
- **共享 VPN。** 热点上的每台设备都经由手机的 VPN 出网，电视、游戏机、笔记本上都
  不需要安装任何东西。
- **中继 Wi-Fi。** 保持连接某个 Wi-Fi 网络，再把它作为你自己的网络重新广播出去。
- **固定网关地址。** 网关地址始终不变，端口转发和已保存的 SSH 目标可以一直使用。
- **调整射频参数。** 2.4 或 5 GHz、信道与频宽、WPA2 或 WPA3、隐藏网络名称。
- **对运营商隐藏热点共享**：在内核支持的前提下，可选用 TTL 修正。

## 托管模式：由 OpenWrt 运行你的热点

这是 VirtualAP 与众不同之处。把热点交给运行在
[Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) 中的 OpenWrt 容器，
OpenWrt 就会像真正的 Wi-Fi 路由器那样管理每一台接入设备。在浏览器中打开你选定的
网关地址，你就进入了 LuCI。

交由 OpenWrt 管理后，你可以：

- 构建任意数量区域的防火墙
- 设置端口转发和流量规则
- 按设备限制带宽和速率
- 运行一个无法访问其他设备的访客网络
- 查看所有接入设备，并为其指定固定地址和名称
- 为整个网络选择 DNS 服务器，并通过 OpenWrt 的软件包源为每台设备添加广告拦截
- 为设备同时提供 IPv6 和 IPv4
- 让 Droidspaces 容器位于同一台路由器之后，与 Wi-Fi 设备同处一个网络，或与之隔离
- 用 `opkg` 安装 OpenWrt 提供的任何其他软件

OpenWrt 的 WAN 侧由 Droidspaces 承担，而 Droidspaces 会在移动数据、Wi-Fi 和 VPN 隧道
之间实时切换上行链路。你的热点因此自动获得这一能力：

1. 在 Droidspaces 中，把 `tun0` 设为 OpenWrt 容器的第一个上行接口，其后依次为
   `wlan0` 和 `rmnet0`。此时 VPN 尚未运行，所以 `tun0` 并不存在。
2. 启动 VirtualAP。热点的流量将通过 Wi-Fi 或移动数据发出。
3. 启动 VPN 应用。Droidspaces 一旦发现 `tun0` 出现，就会把 OpenWrt 切换到它上面，
   于是片刻之后热点上的所有设备都处于 VPN 之后，无需重启任何东西。

OpenWrt 运行在手机自身内核上的容器里，中间没有虚拟机。

托管模式需要：

- root 权限
- 已安装并可正常工作的 [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS)
- 来自
  [官方 Droidspaces rootfs 仓库](https://github.com/Droidspaces/Droidspaces-rootfs-builder/releases)
  的 OpenWrt 容器，已安装并正在运行

## 接口模式

这是简单模式，不需要容器，也没有网页界面。VirtualAP 自己分配地址，在上游支持时为
设备提供 IPv6，并通过你选定的接口把它们送出，或者跟随 Android 当前使用的网络。
共享 VPN 或中继 Wi-Fi，用它就够了。

## 环境要求

- 一台已 root 的手机
- Android 8.0 或更高版本
- ARM 手机，64 位或 32 位均可，同一个 APK 两者通用
- 需要接入点开机自启：Magisk 或 KernelSU
- 需要托管模式：Droidspaces 及其 OpenWrt 容器（见上文）
- 需要 TTL 修正：支持该功能的内核。应用会自行检测，缺失时该选项显示为灰色，
  而热点在没有它的情况下照常工作。

## 快速开始

1. 从 [Releases](https://github.com/ravindu644/VirtualAP/releases) 下载最新的 APK
   并安装。
2. 打开应用并授予 root 权限。它会自行完成所有必要的设置，包括可选的
   Magisk/KernelSU 开机模块。无需刷写，也无需重启。
3. 输入网络名称和密码。
4. 选择上游：一个**接口**（或自动），或托管模式下你的 OpenWrt **容器**。
5. 启动热点，然后连接你的设备。

在设置中启用 **开机时启动接入点**，重启后即可自动恢复。

托管模式下，从已连接的设备打开 `http://<your gateway address>` 即可访问 LuCI。

## 更多

- [工作原理](Documentation/How-It-Works.md)：路由、IPv6、网桥、射频、手机上的文件，
  以及二进制文件和 APK 的构建方式
- [参与贡献](CONTRIBUTING.md)

## 许可证

GNU General Public License v3.0。参见 [LICENSE](LICENSE)。

VirtualAP 源自 [Ubuntu-Chroot](https://github.com/ravindu644/Ubuntu-Chroot) 的
[`start-hotspot`](https://github.com/ravindu644/Ubuntu-Chroot/blob/main/tools/start-hotspot)
脚本；本应用在 AI 助手的协助下编写。
