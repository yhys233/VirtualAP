# AGENTS.md

[English](AGENTS.md) | [简体中文](AGENTS.zh-CN.md) | [繁體中文](AGENTS.zh-TW.md) | [Русский](AGENTS.ru.md) | [Français](AGENTS.fr.md) | [Deutsch](AGENTS.de.md) | [日本語](AGENTS.ja.md) | [한국어](AGENTS.ko.md)

本仓库中面向 AI 编码助手的说明。人类读者请阅读
[CONTRIBUTING.zh-CN.md](./CONTRIBUTING.zh-CN.md)，其中载有这里已有之物的清单。
它的工作原理见
[Documentation/How-It-Works.zh-CN.md](./Documentation/How-It-Works.zh-CN.md)。

## 这是什么

VirtualAP 把一台已 root 的手机变成 Wi-Fi 接入点。它分为两半：

- `src/` 是引擎 `virtualap`，一个以 root 身份运行的静态 C 二进制，它还会驱动另外两个
  程序：`hostapd` 和 `dnsmasq`，二者由 `scripts/build-static.sh` 从 `externals/` 下的
  git 子模块构建。`make aarch64 armhf` 构建 `virtualap`。三者都会落到
  `backend/aarch64/` 和 `backend/armhf/`，这两个目录已被 gitignore。
- `Android/` 是 Compose 应用，它通过 root shell 驱动 `virtualap`。

`prepareAssets` gradle 任务会把两个架构的目录树放入 `Android/app/build/` 下生成的
assets 根目录，应用会在首次运行以及每次 APK 更新之后，把对应架构部署到 `/data/local/virtualap`，
并为 root 管理器安装一个开机模块。
这里没有 chroot。

## 构建

```
./scripts/build-static.sh    # from the repository root: both arches, needs Docker
./build.sh [release]         # from Android/: the APK, needs JDK 17 or newer, ANDROID_HOME and musl-cross gcc
```

首次构建应用之前先运行第一条，之后每当 `externals/` 或构建脚本发生变化时再运行一次。
应用构建自身会执行 `make aarch64 armhf`，所以 `virtualap` 始终与 `src/` 一致。
`prepareAssets` 在 `backend/<arch>/` 缺少三者之一时会失败。`make check` 运行
测试。版本号来自 `src/include/virtualap.h` 中的 `VAP_VERSION`。只能用
`make bump NEW=x.y.z` 提升它，不要在其他任何地方改。构建与 CI 的细节见
How-It-Works.md 的 Building 一节。

## 提交

- 每次提交都要签名：`git commit -s`。
- 绝不要为 AI 助手添加 `Co-Authored-By:` 尾注。人类共同作者没问题。
- 前缀与现有历史保持一致：
  - `app:` 用于 Android 应用，对应情形下使用 `app: fix:` 和 `app: refactor:`
  - `backend:` 用于 `src/` 下的引擎，`fix:` 用于引擎 bug 修复
  - `scripts:` 用于 `scripts/` 和 `externals/` 下的静态二进制构建，`ci:` 用于
    `.github/workflows/`，`docs:` 用于文档
  - `fix(security):` 用于任何与安全相关的内容，两半都算
  - `VirtualAP: bump vX.Y.Z` 用于版本号提升，该提交里不放别的内容

`ap:`、`build:`、`ui:`、`feat:` 和 `refactor:` 出现在较早的历史中。新的提交不要使用
它们。

## 风格

四条规则。它们适用于代码、注释、提交信息和文档。

**不使用 em dash（破折号）。** 改用逗号、句号，或者重写句子。

**不使用 ASCII 横幅注释。** 不要成排的 `-----`，不要 `=====`，不要框线式的分节标题。

**注释要像人写的。** 说明代码为什么这么做，或者不这么做会坏在哪里。跳过那些
只是复述下方那一行的注释。

```sh
# Bad: bump the priority
PRIO_FROM_AP=7010

# Good: netd's own rules start at 10000, so 7010 wins before the VPN catch-all
PRIO_FROM_AP=7010
```

**十行能用的代码胜过一百行做同样事情的代码。** 先删再加。目标是改动最小
且位置正确，而不是随便哪里改动更小。

在文档中，每个事实只陈述一次，放在读者会去查找它的那一页，其他页面链接过去。
分工是固定的：README.md 说明它是什么、面向谁以及操作步骤；
How-It-Works.md 承载所有技术事实；本文件承载面向助手的规则；
CONTRIBUTING.md 说明人类如何让一处改动被合并，并承载清单；DESIGN.md
承载 UI 数值。DESIGN.md 的表格保留，它们是角色映射和尺寸，写成散文并不好用；
其他地方用列表或段落。

## UI 改动

`Android/` 中任何视觉相关的内容都遵循 [DESIGN.zh-CN.md](./DESIGN.zh-CN.md)。颜色、字体、间距、圆角以及
操作胶囊的模式都在那里规定，它们是从现有应用中读出来的，而不是凭空发明的，
所以遵循它们同时也是最小的改动。

如果某条规则确实不适用于你的情况，可以偏离，但要在相应位置留一条注释，说明偏离带来了什么，并在 PR 中说明。
未被记录的偏离就是漂移，下一位贡献者会去“修好”它。
DESIGN.md 的“已决定的例外”一节列出了那些属于决策的偏离，
不要“修好”它们。

## 动手写之前

逐条往下看，在第一个成立的答案处停下。

1. 这个功能需要存在吗？没人要求的开关、给永不变化的取值准备的旋钮、
   只有一个实现的接口：跳过它，并用一句话说明。
2. 这里已经存在了吗？见下文“写之前先复用”。先用 grep 搜。
3. 平台本身已经做了吗？引擎侧是 musl 的 libc 和内核自己的接口（netlink、
   `/proc/sys`）；应用侧是 Kotlin 和 Java 标准库以及 Android framework。
   注意 `minSdk = 26`。
4. 已安装的依赖能覆盖吗？应用侧有 libsu、Compose、lifecycle、zxing、appcompat。
   绝不要为几行代码就能做到的事新增一个依赖。
5. 能写成一行吗？那就写成一行。
6. 只有到这时，才写能工作的最小实现。

有两个答案都成立？取更靠前的那个，然后继续。

这份清单缩短的是解决方案，绝不是阅读。选定某一级之前，
先把改动所触及的流程走一遍。位置不对的小改动是第二个 bug，不是省事的修复。
bug 报告也一样：报告给出的是一个症状，所以改之前先用 grep 查遍每个调用方。
在共享函数里加一处保护，比在每个调用方各加一处更小，
而且它能一并修好报告没有提到的同类问题。

绝不要以简化为由，去掉信任边界上的输入校验、fail-closed 检查、会丢失状态的错误处理，
或者请求者点名要求的东西。

## 写之前先复用

先在 CONTRIBUTING.md 的清单里 grep，再在代码树里 grep。下面这些是关键关口。绕过其中
任何一处都是 bug，而不是捷径。

- `Android/app/src/main/java/com/virtualap/app/util/Backend.kt` 中的 `Backend.quote()` 会包装
  每一个到达 root shell 的动态值。`APManager` 是唯一运行
  `virtualap` 的类，`Backend.startAp` 是唯一拼出它路径的方式。
- `Hotspot` 是 `APManager.start` 和 `APManager.stop` 的唯一调用方。它负责
  启动/停止阶段、命令日志、运行中的通知，以及快捷设置磁贴的同步，
  所以界面和磁贴始终一致。`APConfig.isValid()` 是唯一的
  启动校验；界面的按钮和磁贴都会询问它。
- `src/` 按层一目录：`core`、`cli`、`net`（含 `net/ipt/`）、`radio`、`mode`、
  `monitor`、`cmd`，然后是 `main.c`。一个文件只包含自己这一层和它下面的层，
  顺序也是如此。每个目录有一个公开头文件 `src/include/<folder>/<folder>.h`，其中的
  原型按定义所在的文件分组，用标注该文件的标签隔开；只被本目录自己的
  文件和测试使用的内容放在 `src/<folder>/internal.h`，以 `"internal.h"` 方式包含
  （测试按路径包含它）。其余一律是 `static`。文件长度在 200 到 800 行之间：新
  函数放进它所属子系统的文件，新文件用于新的子系统，或者已超过 800 行的
  文件。`net/ipt/` 有自己的头文件，因为在 musl 上内核的 netfilter
  头文件和 libc 的网络头文件无法共存于同一个文件。从
  Droidspaces 和 USB-Arsenal 移植的代码遵循这些规则；第三方代码留在 `externals/`，
  不做改动。
- 日志用 `vap_log`、`vap_warn`、`vap_error`（`core/io`）。它们打印到 stdout，应用会把它
  带着自己识别出的 `[WARN]` 和 `[ERROR]` 标签流式读入日志面板，并
  在 `logs/ap.log` 追加一份带时间戳的副本。应用从不读取那个文件。裸
  `printf` 在日志里是看不到的，写到 stderr 的任何内容都会丢失。
- 程序由 `core/proc` 运行（`run_quiet`、`run_capture`、`run_shown`、`run_logged`、
  `run_lines`），绝不用 `popen` 或 `system`：Android 没有 `/bin/sh`，每个子进程的 stdin 是
  `/dev/null`，并带有干净的信号掩码。pid 文件通过它的 `pidfile_*`
  函数处理，这些函数只向正在运行预期程序的进程发信号。
- 设置集中在一张表里，即 `cli/options.c` 中的 `vap_keys`：标志字母、`ap.conf`
  键名、默认值、用法文本。新增一项设置就是在 `struct vap_opts` 里加一个字段、在表中加一行，
  并在 `APConfig` 里加一个字段；解析和用法文本随之跟上。每个值都经过
  `opt_set`，它会拒绝换行。`ap.conf` 归应用所有：只有 `APManager.saveConfig`
  会写它，而且只写通过 `APConfig.isValid()` 的配置。引擎只读取它。
- 与内核的通信通过 `net/`（rtnetlink：`link` 负责地址和
  邻居，`route` 负责策略规则和 netd 的表；`ip_tables` 引擎在
  `net/ipt/` 中：`ipt`、`ipt_rules`）以及 `radio/`（`driver`：nl80211 和 wpa 控制
  套接字）。没有任何模块会启动 `ip`、`iw` 或 `hostapd_cli`；`iptables` 只为
  TTL 修正而启动（`net/ipt/ipt_rules` 中的 `ttl_*`）。
- `upstream_resolve`（`net/route`）是唯一解析上游表的地方。
  `radio/channel`（包括 `ensure_5g_ir`）是唯一存放频段和信道策略的地方：
  跟随站点所在的信道、DFS 回退、NO-IR 国家码修正。不要在其他任何地方比较
  信道号。`ipv6_decide`（`mode/ipv6`）是唯一决定客户端 IPv6 的地方；
  启动流程和监控器都会询问它。
- `Constants` 承载应用侧的每一个路径和偏好键，包括引擎的
  （`Constants.ENGINE`）。`VirtualAPInstaller` 负责部署二进制文件；它不在
  其他地方重复。
- Kotlin 里不要有文本。应用显示的每个字符串都在 `res/values/strings.xml` 或
  `plurals.xml` 中，通过 `stringResource` 读取，在组合之外用 `getString`，这样 Weblate
  才能翻译它。规则见 [CONTRIBUTING.zh-CN.md](./CONTRIBUTING.zh-CN.md) 的 Translations 一节。

## 硬性约束

- 应用的目标是 `minSdk = 26`。在假定某个 API 存在之前，先在 Android 8 的行为上测试。
- 引擎是 musl 上的 C11（`gnu11`），单线程：`setns` 只移动调用
  线程，netlink 套接字会留在它被打开时所在的命名空间。在 Makefile 的 gcc 选项下它构建时
  没有任何警告，CI 会把警告变成错误。
- 二进制是来自 `externals/` 子模块的完全静态 musl 构建，两个架构都是。绝不使用
  动态链接的二进制，绝不使用别处的预编译产物。新增一个工具就是新增一个子模块
  并在 `scripts/build-in-container.sh` 中加一段；`virtualap` 是唯一由
  本仓库自己的 `src/` 构建的二进制，使用 musl-cross gcc 和根目录的 `Makefile`。
- 已安装的依赖能覆盖时，不要新增依赖。应用已经有 libsu、Compose、
  lifecycle、zxing 和 appcompat。
- Samsung 的 Broadcom FullMAC 驱动是无线相关假设的底线，MediaTek 的
  gen4m 是第二参照；它们各自缺什么，见 How-It-Works.md 的“The radio”一节。不基于
  运行时检测做门控的硬件怪癖修复不予纳入。
