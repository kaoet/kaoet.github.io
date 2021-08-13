---
title: pfSense + Wireguard + OSPF 高可用性翻墙路由
date: 2021-08-13
tags: 网络
---

必要的准备
----
首先准备一个 x86_64 架构的软路由。软路由上具有至少两个以太网接口。如果你的“软路由”只有一个以太网接口，可能通过设置 VLAN 等也可以工作，但我并没有研究过。

如果家中是光纤到户，则光猫一定要设置为桥接模式，使得接在后端的软路由可以 PPPoE 拨号上网。

同时为了翻墙，也要购买至少一个国外 VPS 作为出口。VPS 应装有 GNU/Linux 操作系统，这里建议直接选择 Debian。国内最好也有一台 VPS 作为流量中转。

pfSense 的安装
----

这里选用 [pfSense](https://pfsense.org) 作为软路由的操作系统。pfSense 是一款基于 FreeBSD 的软件防火墙。它的社区版本是免费且开源的，可以在其[下载页面](https://www.pfsense.org/download/)直接下载到 ISO 或 USB 安装镜像。

鉴于一般软路由并不会配置光驱，我们需要下载 USB 安装镜像，并写入U盘中。在 macOS 或 Linux 下可以直接用 `dd` 命令：
~~~~shell
$ dd if=pfSense-CE-memstick-x.y.z-RELEASE-amd64.img of=/dev/diskX bs=4m
~~~~

安装软路由前请将 BIOS 设置中 CSM 一项选为 **UEFI Only**，否则后续 ZFS 启动会遇到问题。安装过程比较简单，其中建议文件系统一定要选择默认的 ZFS 而非 UFS。

基本网络的配置
----
pfSense 安装完成后，可以在控制台中设置 WAN 与 LAN 对应的物理端口以及 LAN 的静态地址。这时先只需设置 LAN 的 IPv4 即可。如果软路由有多个物理端口用于局域网，也只需先设置第一个端口。这里不妨假设你设置的地址是 `192.168.0.1/24`。

之后就可以将计算机连接到刚刚配置好的端口。应该会自动通过 DHCP 得到一个 IPv4 地址。此时就可以在浏览器中打开 `https://192.168.0.1` 访问 pfSense 设置界面了。在设置向导中就可以设置 WAN 的 PPPoE 连接，也可以直接跳过向导之后再设置。

如果软路由有多个局域网物理端口，此时需要有一些巧妙的方法将这些端口桥接在一起。假设你的广域网端口是 `re0`，局域网端口是 `re1`~`re3`，则具体步骤如下：
1. 在 **Interfaces &raquo; Interface Assignments** 中将 `re2` 和 `re3` 添加进去。并进入其设置界面启用它们。
2. 在 **Interfaces &raquo; Bridges** 中添加一个 Bridge Interface，并设置其关联的物理端口为 `re2` 和 `re3`。
3. 回到 **Interfaces &raquo; Interface Assignments**，并将 LAN 对应的端口设置为刚刚新建的 Bridge。
4. 保存并应用，此时你的计算机应该无法继续访问设置界面。需要将计算机改插入 `re2` 或 `re3` 端口，方能恢复设置界面的访问。
5. 在 **Interfaces &raquo; Bridges** 中修改刚才新建的 Bridge，把 `re1` 添加到物理端口列表中。 

LAN 部分设置完成，之后是 WAN 部分。这里要注意的是如果你在用 PPPoE 拨号，请务必在 WAN 的设置中将 **MTU** 和 **MSS** 设置为 1492，否则会出现由于 TCP MSS 探测错误而部分 HTTPS 网站无法成功访问的问题。到这一步路由器与你的计算机应该都可以访问公网了。

注意。如果你的软路由网卡是 Realtek，那么由于 FreeBSD 默认驱动的问题，可能会出现高负载下网卡掉线重启的情况。在系统日志中会出现：
~~~~
re0: watchdog timeout
re0: link state changed to DOWN
~~~~
如果遇到了请根据[这篇文章](https://www.robpeck.com/2021/04/using-realtek-nics-in-pfsense/)的方式安装相应驱动程序。

IPv6
----
相比 IPv4 直接无脑 NAT，IPv6 对于局域网的方式配置稍显复杂。首先通过 DHCPv6 在 PPPoE 连接中可以获取 WAN 端口自己的 IPv6 地址。之后要通过 DHCPv6 [Prefix Delegation](https://en.wikipedia.org/wiki/Prefix_delegation) 从ISP另外请求一段 IP 地址分发给 LAN 端口以及局域网中的计算机。最后路由器需要 RA 以及 DHCPv6 向局域网中的计算宣告这段地址。

具体的，要配置 WAN 端口：
* 勾选 **Use IPv4 connectivity as parent interface**。
* 勾选 **Do not allow PD/Address release**。

之后配置 LAN 端口：
* **IPv6 Configuration Type** 一项选择 **Track Interface**。
* 设置 **IPv6 Interface** 为 **WAN**。
* 设置 **IPv6 Prefix ID** 为 `0`。

经过这两步应该 WAN 与 LAN 都获取到了各自的 IPv6 地址。我的情况是 WAN 端口获取到了一个 `/64` 的地址，而 LAN 得到了一个 `/60` 的地址。

之后在 **Services &raquo; DHCPv6 Server & RA** 中启用 **DHCPv6 Server**，并在 **Router Advertisements** 选项卡中设置 **Router mode** 为 **Managed**。这样局域网中的计算机会得到 DHCPv6 有状态方式配置的 IPv6 地址。我试图不通过 DHCPv6，只用 RA 在局域网中无状态配置 IPv6。但由于 ISP 通过 DHCPv6 PD 分配的地址段是 `/60` 而非 SLAAC 所需的 `/64` 因而不能正常工作。

最后还需要配置防火墙允许 IPv6 流量通过。在 **Firewall &raquo; Rules &raquo; LAN** 中修改默认的 IPv4 默认允许条目的 Destination 为 **any**。同时新建一个对应的 IPv6 条目，也是允许 **any** 到 **any** 的通信。

*未完待续*
