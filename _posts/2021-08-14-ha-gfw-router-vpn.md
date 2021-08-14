---
title: 高可用性翻墙路由：2. VPN 的搭建
date: 2021-08-14 10:00:00+08
category: 技术
tags: 网络 高可用性翻墙路由
---

为了翻出墙外，VPN 是必不可少的。你需要购买至少一个国外 VPS 作为出口。而为了高可用性，建议购买至少两台国外 VPS。它们最好是不同线路，不同地区的，以便在某一个服务质量下降时使用另一个作为备份。而国内最好也有一台 VPS 作为流量中转。VPS 应装有 GNU/Linux 操作系统，这里建议直接选择 Debian。

当你有了两台国外 VPS 和一台国内 VPS 后，可以在它们之间搭建如下的 VPN 网络：
* 家（`10.0.0.1/30`） &harr; 国内 VPS（`10.0.0.2/30`）
* 国内 VPS（`10.0.1.2/30`） &harr; 国外 VPS1（`10.0.1.2/30`）
* 国内 VPS（`10.0.2.2/30`） &harr; 国外 VPS2（`10.0.2.2/30`）

其中每条 VPN 连接分配一个 `/30` 的网段，相互之间不重叠。

VPN 方案的选择
----

### IPSec

IPSec 是最标准的 VPN 方案，所以几乎所有的终端与路由器都对其有良好的支持。这还产生了 IPSec 过墙能力很好的都市传说，原因是很多大公司都在使用它。

IPSec 分为隧道模式与传输模式。隧道模式在外层 IP 包中包裹需要传输的内层 IP 包，可用于桥接两个子网。而传输模式只会封装内部数据，没有额外的 IP 包头，只能用于保护两个主机之间的直接通信。但隧道模式的缺点在于一个 IP 包走不走 IPSec 隧道是由 IPSec 的 Policy 控制的，而不是通常的路由表，这会给后期动态选择路由带来麻烦。因此需要选择 IPSec 传输模式 + GRE 封装的解决方案。也就是在两台主机间配置 IPSec 传输模式，再在 IPSec 中通过 GRE 包来封装内部的 IP 包。

{% include figure image_path="https://upload.wikimedia.org/wikipedia/commons/6/6b/Ipsec-modes.svg" alt="IPSec 模式" caption="IPSec 模式" %}

每种模式又可以选择 AH 或 ESP 封装。AH 只提供数据认证，不能加密数据，而 ESP 可以同时提供数据加密与认证。另外，ESP 封装可以自动侦测路径上的 NAT 设备，从而自动改用 UDP + ESP 穿越 NAT。

{% include figure image_path="https://upload.wikimedia.org/wikipedia/commons/6/64/Ipsec-esp-tunnel-and-transport.svg" alt="ESP 封装" caption="ESP 封装" %}

IPSec 本身是可以支持动态地址的，只需把对端地址设为 `%any` 即可。但 GRE 隧道并不原生支持动态 IP，只能通过 DDNS 等方案间接解决。另一种方法是 IPSec 隧道模式 + GRE 封装。这样应该是可以工作的，但每个数据包等于套了三层 IP 头，开销太大。所以推荐 IPsec 最好用在两端有固定 IP 的 VPS 互联中。

### WireGuard

WireGuard 固定用 UDP 传输，并使用 ChaCha20-Poly1305 加密。配置起来比 IPSec 方便很多。据其[官网评测](https://www.wireguard.com/performance/)，无论在延迟还是吞吐方面，WireGuard 都超越了其他方案。

WireGuard 支持漫游，即两端地址的动态变化。这得益于 WireGuard 只通过公钥，而不是对端地址来认证彼此。一旦某一个 IP 地址发来的包通过了公钥验证，后续向对方发回去的包都会选择这个 IP 地址，而不是静态配置中的地址。

WireGuard 比较新，所以大部分硬件路由器是不支持的。好在 pfSense 是支持 WireGuard 的。

### OpenVPN / PPTP / L2TP

这些协议基本都不推荐。它们要么无法用于穿墙，要么性能低下，要么配置复杂，要么以上缺点都具备。

IPSec + GRE
----
如果你只需要一个可用的 IPSec 隧道，其配置方式其实并不复杂。由于家里一般具有动态 IP，不太适合 IPSec，这里只介绍两台 VPS 主机间的互联方法。

### 防火墙

首先确保两端的各种防火墙都允许了以下协议通过：
1. **ESP**，即 IP 协议号50。
2. **UDP 500**，用于 IKEv2 协商密钥。
3. **UDP 4500**，用于 UDP 封装的 ESP 通过 NAT，即 NAT-T。

其中 UDP 500 是必须的。如果两台主机间存在 NAT，则 UDP 4500 是必须的。如果没有 NAT，则要么允许 ESP 通过，要么允许 UDP 4500 并开启强制 NAT-T 封装。

这里判断是否有 NAT 的简单方法是通过 `ip a` 命令看两端网卡上绑定的地址，如果都是公网地址，则没有 NAT，否则会有 NAT。一些云计算平台会将机器放置在 VPC 中，机器只能得到 VPC 中的私有地址。虽然此时会有公网地址与这些私有地址一一对应，但也属于存在 NAT 的情况。

### IPSec

安装 strongSwan，各大发行版都有[现成的软件包](https://wiki.strongswan.org/projects/strongswan/wiki/InstallationDocumentation#Distribution-packages)。如果是 Debian 的话，直接 `apt install strongswan` 即可。

安装后只需要修改两个文件。先在 `/etc/ipsec.conf` 加入如下内容：
~~~~
conn <对端名称>
    leftid=<本端名称>
    left=<本端IP>
    rightid=<对端名称>
    right=<对端IP>
    authby=secret
    type=transport
    auto=start
~~~~

再在 `/etc/ipsec.secrets` 中加入下面一行：
~~~~
<本端名称> <对端名称> : PSK '<连接密码>'
~~~~

其中`<本端名称>`和`<对端名称>`可以随意选取。`<本端IP>`填写网卡上绑定的 IP，而`<对端IP>`填写对端的公网 IP。

当两台主机都设置好后，运行 `ipsec restart`，没有问题的话就可以成功建立连接。可以运行 `ipsec statusall` 来查看连接情况。也可以 ping 对方公网 IP 并抓包来测试封装效果。

### GRE

如果是 Debian 系统，并且用 ifupdown 管理网络，则在 `/etc/network/interfaces` 中新加入一个网络接口即可：
~~~~
auto gre1
iface gre1 inet tunnel
    address <隧道内本端IP>
    netmask <隧道内子网掩码>
    mode gre
    endpoint <对端IP>
    ttl 64
    mtu 1400
~~~~

其中请务必设置 `ttl` 选项，否则内部 IP 包的 TTL 值会被直接复制到外部 IP 包，后续运行 OSPF 时造成错误。

另外网络接口的名称不能为 `gre0`，因为这是一个[保留名称](https://serverfault.com/a/975704)。

MTU 的设置请参考 [Cisco 给出的分析](https://www.cisco.com/c/en/us/support/docs/ip/generic-routing-encapsulation-gre/25885-pmtud-ipfrag.html#anc19)。这里直接沿用了建议值 `1400`。

两台主机都配置好之后运行 `ifup gre1` 就可以开启 GRE 隧道。可以用 `ping <隧道内对端IP>` 的方式验证其是否正常工作。

WireGuard
----

主流的发行版同样提供了 WireGuard 的软件包。如果是 Debian，则直接 `apt install wireguard`；如果是 pfSense，则在 **System &raquo; Package Manager** 中安装 `WireGuard` 包；其他系统请参考官网的[安装说明](https://www.wireguard.com/install/)。

WireGuard 只需用到一个 UDP 端口，且端口号可以任意指定。相互连接的两端只要有一端具有公网 IP 以及可接受入站连接的 UDP 端口即可。另一端即使位于 NAT 后且不可接受入站连接也没有问题。这里我们假设使用了默认的 UDP 端口 51820。注意在防火墙上放通此端口。

### 单网络接口 vs. 多网络接口

WireGuard 支持通过同一个网络接口与多于一个 Peer 通信。其实现方式是要求同一个网络接口中各个 Peer 的 `AllowedIPs` 不重合。这样当发送一个 IP 包时，就可以根据 `AllowedIPs` 自动选择向哪个 Peer 发送。但这样的问题在于无法做到动态选择路由。

例如有三台主机 A、B、C，它们各自把另外两台主机加入自己的 Peer 列表中。如果 A 希望与 C 通信，那么它只能向 C 发送数据包，而不能绕道 A &rarr; B &rarr; C。在一般情况下这是没问题的，但如果 A 与 C 之间无法直接正常通信，就会导致两者之间完全断联。

因此我们采用每个 WireGuard 网络接口只设置一个 Peer 的方式。此时只需把 `AllowedIPs` 设为 `0.0.0.0/0` 和 `::/0` 即可。而对于 VPN Mesh 中的路由问题，我们在[下一篇文章]({% post_url 2021-08-14-ha-gfw-router-ospf %})中通过 OSPF 解决。

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
