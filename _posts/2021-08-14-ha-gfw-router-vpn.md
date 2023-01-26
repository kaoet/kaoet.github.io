---
title: 高可用性翻墙路由：2. VPN 的搭建
last_modified_at: 2022-09-17T00:00:00+08
category: 技术
tags: 网络 高可用性翻墙路由
---

为了翻出墙外，VPN 是必不可少的。你需要购买至少一个国外 VPS 作为出口。而为了高可用性，建议购买至少两台国外 VPS。它们最好是不同线路，不同地区的，以便在某一个服务质量下降时使用另一个作为备份。而国内最好也有一台 VPS 作为流量中转。VPS 应装有 GNU/Linux 操作系统，这里建议直接选择 Debian。

当你有了两台国外 VPS 和一台国内 VPS 后，可以以国内 VPS 为中心建立星型 VPN 网络。每个设备都与国内 VPS 建立一个 P2P 隧道。隧道两端地址可以选择一个 `/31` 的子网。隧道之间的地址不要重合，例如：
* 国内 VPS（`10.0.0.0/31`） &harr; 家（`10.0.0.1/31`）
* 国内 VPS（`10.0.1.0/31`） &harr; 国外 VPS1（`10.0.1.1/31`）
* 国内 VPS（`10.0.2.0/31`） &harr; 国外 VPS2（`10.0.2.1/31`）

VPN 方案的选择
----

### IKEv2 + IPSec

IKEv2 + IPSec 是最标准的 VPN 方案，所以几乎所有的终端与路由器都对其有良好的支持。这还产生了其过墙能力很好的都市传说，原因是很多大公司都在使用它。其中 IKEv2 协议负责协商网络两端的通信参数，而 IPSec 负责传输数据负载。

IKEv2 + IPSec 也支持漫游，即网络的一端可以不具备固定的 IP 地址，甚至可以随时切换地址（例如从 Wi-Fi 切换到移动网络）而仍然保持连接。

### WireGuard

WireGuard 固定用 UDP 传输，并使用 ChaCha20-Poly1305 加密。配置起来比 IPSec 方便一些。据其[官网评测](https://www.wireguard.com/performance/)，无论在延迟还是吞吐方面，WireGuard 都超越了其他方案。

WireGuard 也支持漫游，这得益于 WireGuard 只通过公钥，而不是对端地址来认证彼此。一旦某一个 IP 地址发来的包通过了公钥验证，后续向对方发回去的包都会选择这个 IP 地址，而不是静态配置中的地址。

WireGuard 比较新，所以大部分硬件路由器是不支持的。好在 pfSense 是支持 WireGuard 的。

### OpenVPN / PPTP / L2TP

这些协议基本都不推荐。它们要么无法用于穿墙，要么性能低下，要么配置复杂，要么以上缺点都具备。

IKEv2 + IPSec
----

### IPSec 封装
IPSec 分为隧道模式与传输模式。隧道模式在外层 IP 包中包裹需要传输的内层 IP 包，可用于桥接两个子网。而传输模式只会封装内部数据，没有额外的 IP 包头，只能用于保护两个主机之间的直接通信。传输模式过去曾被计划用于 IPv6 的加密，目前多用于封装 L2TP 协议。[^ipsec-mode] 这里我们将要用到的是隧道模式。

[^ipsec-mode]: [Encapsulating Security Payload (ESP)](https://docs.strongswan.org/docs/5.9/howtos/ipsecProtocol.html#_encapsulating_security_payload_esp)

{% include figure image_path="https://upload.wikimedia.org/wikipedia/commons/6/6b/Ipsec-modes.svg" alt="IPSec 模式" caption="IPSec 模式" %}

每种模式又可以选择 AH 或 ESP 封装。AH 只提供数据认证，不能加密数据，而 ESP 可以同时提供数据加密与认证。另外，ESP 封装可以自动侦测路径上的 NAT 设备，从而自动改用 UDP + ESP 穿越 NAT。所以我们选择 ESP 封装。

{% include figure image_path="https://upload.wikimedia.org/wikipedia/commons/6/64/Ipsec-esp-tunnel-and-transport.svg" alt="ESP 封装" caption="ESP 封装" %}

### 基于策略 vs. 基于路由

在配置方面，IPSec 又可分为基于策略与基于路由两种配置方式。

默认的是基于策略的方式，即数据包都经过同一个底层网络接口发出，但根据其源、目的地址的不同，在发出前选择封装到某一个 IPSec 连接中。这种方式适合于简单的网络拓扑，例如对于北京子网的包发送到北京的网关，而上海子网的包发送到上海。

而如果两个节点之间不只有一条路径，而需要动态选择最优路径的话，基于策略的配置无法与 BGP、OSPF 等路由协议良好地配合在一起。这时可以采用基于路由的配置方法。基于路由的配置会给每个 IPSec 隧道建立一个虚拟网络接口，之后在策略中规定凡是经过此网络接口的数据包都走某一个 IPSec 隧道。这样我们就可以在路由表中配置不同 IP 段路由到不同的虚拟接口，而无需再关心 IPSec 策略。

需要注意到的是，无论是基于策略还是基于路由，底层网络上传输的数据包是一样的。因此甚至可以在隧道的一端使用基于策略的配置，而在另一端使用基于路由的配置。

### 防火墙

首先确保两端的各种防火墙都允许了以下协议通过：
1. **ESP**，即 IP 协议号50。
2. **UDP 500**，用于 IKEv2 协商密钥。
3. **UDP 4500**，用于 UDP 封装的 ESP 通过 NAT，即 NAT-T。

其中 UDP 500 是必须的。如果两台主机间存在 NAT，则 UDP 4500 是必须的。如果没有 NAT，则要么允许 ESP 通过，要么允许 UDP 4500 并开启强制 NAT-T 封装。

这里判断是否有 NAT 的简单方法是通过 `ip a` 命令看两端网卡上绑定的地址，如果都是公网地址，则没有 NAT，否则会有 NAT。一些云计算平台会将机器放置在 VPC 中，机器只能得到 VPC 中的私有地址。虽然此时会有公网地址与这些私有地址一一对应，但也属于存在 NAT 的情况。

### IPSec


安装 [strongSwan](https://www.strongswan.org)，各大发行版都有[现成的软件包](https://docs.strongswan.org/docs/5.9/install/install.html#_distribution_packages)。如果是 Debian 的话，直接 `apt install charon-systemd` 即可。

{% capture config %}
strongSwan 有两种配置方式：
* 第一种是用 `ipsec.conf` 和 `ipsec.secrets` 作为配置文件。同时使用 `ipsec` 命令行控制连接，使用 `starter` 作为后台服务。这种方式已被淘汰。
* 第二种是用 `swanctl.conf` 文件以及 `swanctl` 目录作为配置。同时使用 `swanctl` 命令行控制，使用 `charon` 作为后台服务。这是目前推荐的方式。

因此对于 Debian 系统，最好卸载 `strongswan` 软件包，只安装 `charon-systemd`，否则可能引发冲突。
{% endcapture %}

<div class="notice--warning">
    {{ config | markdownify }}
</div>

之后生成两端的通信密钥，这里为了方便采用 PSK 方式认证。如果节点繁多，为了方便管理，也可使用基于证书的方法认证。证书链的配置方法在 [strongSwan 文档](https://docs.strongswan.org/docs/5.9/pki/pkiQuickstart.html)中有介绍。

~~~~
$ dd if=/dev/random count=1 bs=33 2>/dev/random | base64
s1Jy/Li0Bkqmd3C7eYG71EoPxmcataF/LL4ATA3gE+au
~~~~

修改 `/etc/swanctl/swanctl.conf` 为如下内容：
~~~~
connections {
    <对端名称> {
        version = 2
        remote_addrs = <对端地址>
        local {
            auth = psk
            id = <对端ID>
        }
        remote {
            auth = psk
            id = <本端ID>
        }
        children {
            all {
                local_ts = 0.0.0.0/0
                remote_ts = 0.0.0.0/0
                start_action = trap
            }
        }
        if_id_in = <接口ID>
        if_id_out = <接口ID>
    }
}

secrets {
    ike-<对端名称> {
        id = <对端ID>
        secret = 0s<PSK>
    }
}
~~~~

* `<对端名称>`可以随意选取，只用于配置文件内标识不同的连接。
* `<本端ID>`，`<对端ID>`用于身份认证，可以随意选取，但网络两端必须恰好相反。
* `<对端地址>`填写对端的公网 IP 或域名。也可以填写多个，使用逗号分隔。
* `<接口ID>`是一个任意整数，用于关联 IPSec 连接与隧道网络接口。不同连接的接口ID不能重复。
* `secret` 一项要在上一步生成的 Base64 格式 PSK 前面加上 `0s` 前缀，例如 `0ss1Jy/Li0Bkqmd3C7eYG71EoPxmcataF/LL4ATA3gE+au`。

如果网络一端没有固定 IP，例如家庭宽带连接到 VPS，则对具有固定 IP 一端的配置做如下修改：
* 删除 `remote_addrs` 一行，使其接受任意对端 IP。
* 删除 `start_action = trap` 一行，使其不会主动发起连接。

完成后运行 `swanctl --load-all` 加载配置。

### XFRM

XFRM 是 Linux 4.19 后支持的虚拟网络接口类型，相比之前的 VTI 接口配置更为简单直接。每个 XFRM 接口有一个接口ID，系统正是通过接口ID，将出入接口的网络流量关联到 IPSec 连接。这里简单起见，我们使 XFRM 接口与 IPSec 连接有1:1的对应关系。

如果是 Debian 系统，并且用 ifupdown 管理网络，则在 `/etc/network/interfaces.d` 中新加入一个网络接口即可：
~~~~
auto xfrm0
iface xfrm0 inet static
    address <隧道本端IP>
    netmask <隧道子网掩码>
    pre-up ip link add xfrm0 type xfrm dev <底层网络接口> if_id <接口ID>
    post-down ip link del xfrm0
    mtu 1410
~~~~

* `<隧道本端IP>`是 VPN 隧道本端的 IP 地址，例如开头例子中的 `10.0.1.0`。
* `<隧道子网掩码>`是 VPN 隧道的子网掩码。如果采用 `/31` 的子网，则子网掩码为 `255.255.255.254`。
* `<底层网络接口>`是可以访问互联网的网络接口名称，例如 `eth0`。

MTU 的设置可以参考[这篇文章](https://kusoneko.blogspot.com/p/ikev2-ipsec.html)。这里我采用的数值 `1410` 是考虑了常见最坏情况下的数值。其计算过程如下：

封装 | 字节数
--- | -----
PPPoE | 8
IPv6 | 40
UDP (NAT-T) | 8
ESP Header | 8
ESP IV (AES-GCM) | 8
ESP Trailer | 2
ESP Authentication (AES-GCM) | 16
总计 | 90
MTU | 1500 - 90 = 1410

完成后运行 `ifup xfrm0` 打开 XFRM 接口。

### 调试

如果你足够幸运，此时在隧道一端 `ping` 另一端隧道内 IP 可以连通。否则可以尝试如下方法逐步调试。

* 确认对端公网 IP 可以 `ping` 通。
* 运行 `ip xfrm policy` 查看有没有相应的 IPSec 策略。
* 运行 `journalctl -f -u strongswan` 或 `swanctl --log`。观察其输出的握手过程。
* 运行 `swanctl --list-sas` 打印正在建立或已建立的连接。
* 运行 `ip route` 打印路由表，以及 `ip route get <IP>` 打印特定 IP 的路由选择。
* 运行 `tcpdump -i <网络接口>` 抓包观察。

WireGuard
----

主流的发行版同样提供了 WireGuard 的软件包。如果是 Debian，则直接 `apt install wireguard`；如果是 pfSense，则在 **System &raquo; Package Manager** 中安装 `WireGuard` 包；其他系统请参考官网的[安装说明](https://www.wireguard.com/install/)。

WireGuard 只需用到一个 UDP 端口，且端口号可以任意指定。相互连接的两端只要有一端具有公网 IP 以及可接受入站连接的 UDP 端口即可。另一端即使位于 NAT 后且不可接受入站连接也没有问题。这里我们假设使用了默认的 UDP 端口 51820。注意在防火墙上放通此端口。

### 单网络接口 vs. 多网络接口

WireGuard 支持通过同一个网络接口与多于一个 Peer 通信。其实现方式是要求同一个网络接口中各个 Peer 的 `AllowedIPs` 不重合。这样当发送一个 IP 包时，就可以根据 `AllowedIPs` 自动选择向哪个 Peer 发送。但这样的问题在于无法做到动态选择路由。

例如有三台主机 A、B、C，它们各自把另外两台主机加入自己的 Peer 列表中。如果 A 希望与 C 通信，那么它只能向 C 发送数据包，而不能绕道 A &rarr; B &rarr; C。在一般情况下这是没问题的，但如果 A 与 C 之间无法直接正常通信，就会导致两者之间完全断联。

因此我们采用每个 WireGuard 网络接口只设置一个 Peer 的方式。此时只需把 `AllowedIPs` 设为 `0.0.0.0/0` 和 `::/0` 即可。而对于 VPN Mesh 中的路由问题，我们在[下一篇文章]({% post_url 2021-08-18-ha-gfw-router-ospf %})中通过 OSPF 解决。

### Linux

在 Linux 下，首先使用 `wg genkey | tee /dev/stderr | wg pubkey` 生成一对私钥与公钥。之后新建一个配置文件 `/etc/wireguard/wg0.conf`，并写入如下内容：
~~~~
[Interface]
PrivateKey = <本端私钥>
ListenPort = 51820

[Peer]
PublicKey = <对端公钥>
Endpoint = <对端IP>:51820
AllowedIPs = 0.0.0.0/0, ::/0
~~~~

如果对端没有公网 IP 与端口，其中 `Endpoint` 一项可以不填。如果本端在 NAT 后面，建议在 `[Peer]` 一节中添加 `PersistentKeepalive = 25`，以便维持 NAT 连接映射。

在使用 ifupdown 的系统中，在 `/etc/network/interfaces` 中加入一个网络接口：
~~~~
auto wg0
iface wg0 inet static
    address <隧道内本端IP>
    netmask <隧道内子网掩码>
    pre-up ip link add wg0 type wireguard
    pre-up wg setconf wg0 /etc/wireguard/wg0.conf
    post-down ip link del wg0
~~~~

两端运行 `ifup wg0` 就可以开启隧道。此时可以运行 `wg` 命令查看连接状态，或 `ping <隧道内对端IP>` 验证其是否正常工作。

### pfSense

当安装完 WireGuard 包后，在 **VPN &raquo; WireGuard &raquo; Tunnels** 就可以新建一个 WireGuard 网络接口。其中 **Interface Addresses** 可以先不填，而是在创建完成后再去 **Interfaces &raquo; Assignments** 中手动给刚建好的 `tun_wg0` 关联一个新的网络接口并设置静态 IP。同时也要将 MTU 和 MSS 设为1420。

之后在 **VPN &raquo; WireGuard &raquo; Peers** 添加 Peer。注意将 **Allowed IPs** 设为 `0.0.0.0/0` 和 `::/0`。

防火墙方面，需要分别设置 WAN 的规则与 WG0 的规则。两者都在 **Firewall &raquo; Rules** 中。对于 WAN，需要允许 any 访问 This Firewall 的 UDP 51820 端口。对于 WG0，至少允许 ICMP 协议以便测试连通性。

配置完成后可以在 **Status &raquo; WireGuard** 界面看到连接状态。并在 **Diagnostics &raquo; Ping** 页面中测试到对端的连通性。

**上一篇文章：**[高可用性翻墙路由：1. pfSense 的安装]({% post_url 2021-08-13-ha-gfw-router-pfsense %})<br>
**下一篇文章：**[高可用性翻墙路由：3. OSPF 与动态路由]({% post_url 2021-08-18-ha-gfw-router-ospf %})
{: .notice}
