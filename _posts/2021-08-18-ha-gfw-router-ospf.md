---
title: 高可用性翻墙路由：3. OSPF 与动态路由
category: 技术
tags: 网络 高可用性翻墙路由
---

在有多于一个国外线路的基础上，实现翻墙高可用性的关键在于能够自动侦测线路的连通性，并在一条线路失效的情况下自动切换的其他线路。这就需要动态设置路由表。

当线路A连通时，出国流量路由到A线路，而当线路A失效时路由到线路B。即使你只有一条国外线路，动态路由也是有意义的，以便在这唯一的一条线路失效时，自动将国外 IP 段的路由回退到默认的 ISP 网关。以此防止彻底断网的情况发生。

OSPF
----
实现动态路由可以直接采用已有的路由协商协议。常见的协议有 BGP、OSPF、RIP 等。这里我们采用 OSPF 协议，它容易配置且路由表收敛速度快。

### Linux

首先确保 `/etc/sysctl.conf` 中 IP 转发已经开启。如果没有开启，可以加入如下内容后运行 `sysctl -p` 开启：
~~~~
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
~~~~

另外也需要确保 Linux 内部的防火墙不会阻止 VPN 网络接口中的 OSPF 以及 ICMP 协议。

之后安装 [BIRD](https://bird.network.cz/) 来运行 OSPF 协议。如果是 Debian 系统直接 `apt install bird` 即可。

安装完成后编辑 `/etc/bird/bird.conf`：
~~~~
router id <路由器ID>;

protocol kernel {
    scan time 60;
    import none;
    export all;
}

protocol device {
    scan time 60;
}

protocol ospf {
    import all;
    export all;
    area 0 {
        interface "gre1" {
            cost <花销>;
            type pointopoint;
            authentication cryptographic;
            password "<共享密钥>";
        };
        interface "wg0" {
            cost <花销>;
            type pointopoint;
            authentication cryptographic;
            password "<共享密钥>";
        }
        ……
    };
}
~~~~

其中`<路由器ID>`一般填写你的公网 IP。而在 `protocol ospf` &raquo; `area 0` 一节，需要为每一个 VPN 网卡添加一个 `interface` 节。在一个 VPN 隧道两端，`<共享密钥>`需要相同，而`<花销>`也可以配置成一样的。

路由花销会被用于计算最佳路由。在一个 OSPF 网络中，系统会选择路径花销加和最小的路径作为两个节点之间的路由。花销是一个整数，其标准配置方法是根据网络接口的带宽决定，带宽越大则花销越小。网上有带宽与花销的[转换表格](https://www.watchguard.com/help/docs/help-center/en-US/Content/en-US/Fireware/dynamicrouting/ospf_interface_cost_c.html)。但其实我们更常关心的是延迟，总希望选一条延迟尽量小的线路承载出国流量。因此此处将花销定为两端之间的 ping 延迟也可以。

当两端配置完成后，使用 `systemctl restart bird` 重启 BIRD。之后可以用 `birdc` 命令行查看 OSPF 运行状况。
~~~~
# birdc
bird> show ospf neighbor
ospf1:
Router ID  Pri State     DTime Interface  Router IP
1.1.1.1    1   Full/PtP  00:36 gre1       1.1.1.1
2.2.2.2    1   Full/PtP  00:39 wg0        2.2.2.2
~~~~
当显示 **State** 为 **Full** 的时候说明两端交换路由信息成功。

### pfSense

pfSense 中同样需要保证 VPN 网络接口中的 OSPF 与 ICMP 协议不会被防火墙阻挡。这需要在 **Firewall &raquo; Rules &raquo; 相应网络接口**中添加两条规则实现。

之后在 **System &raquo; Package Manager** 中安装 **FRR** 软件包，即可在 **Services** 菜单中看到相应设置。

在 **Services &raquo; FRR OSPF &raquo; OSPF** 中，将 OSPF 功能开启，并设置 **Default Area** 为0。

在 **Services &raquo; FRR OSPF &raquo; Areas** 中，添加一个新的 Area，ID 设置为0。

在 **Services &raquo; FRR OSPF &raquo; Interfaces** 中，为每个 VPN 网络接口添加一个新的 Interface：
* **Network Type** 选择 **Point-To-Point**。
* **Metric** 填写相应开销。
* **Authentication Type** 选择 **Message Digest (MD5 Hash)**。
* **Password** 填写共享密钥。
同时为了将局域网对应的 IP 端也发布到 OSPF 中，也要为 LAN 接口添加一个 Interface。其中勾选 **Interface is Passive**，使其成为一个 stub interface，而其他选项保持默认即可。

最后，在 **Services &raquo; FRR Global/Zebra** 中启用 FRR，并填写 **Default Router ID** 以及 **Master Password**。保存后 OSPF 正式开启。

在 **Status &raquo; FRR** 中可以查看 OSPF 的协商状态。成功后会在 **OSPF Neighbors** 一栏显示所有邻居都是 **Full** 的状态。
~~~~
Neighbor ID Pri  State        Dead Time  Address   Interface        RXmtL RqstL DBsmL
1.1.1.1     1   Full/DROther  35.996s    10.0.0.1  tun_wg0:10.0.0.2 0     0     0
2.2.2.2     1   Full/DROther  37.479s    10.0.1.1  tun_wg1:10.0.1.2 0     0     0
~~~~

### 测试
当所有节点配置好后，所有的私有 IP 端应该在任何节点处都可以 ping 通。例如可以在 VPS 中 ping 一下家中局域网的地址。
~~~~shell
$ ping 192.168.0.1
PING 192.168.0.1 (192.168.0.1) 56(84) bytes of data.
64 bytes from 192.168.0.1: icmp_seq=1 ttl=64 time=42.0 ms
64 bytes from 192.168.0.1: icmp_seq=2 ttl=64 time=42.0 ms
64 bytes from 192.168.0.1: icmp_seq=3 ttl=64 time=42.0 ms
64 bytes from 192.168.0.1: icmp_seq=4 ttl=64 time=42.0 ms
--- 192.168.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 42ms
rtt min/avg/max/mdev = 42.0/42.0/42.0 ms
~~~~

**未完待续**
{: .text-center}
{: .notice--info}

**上一篇文章：**[高可用性翻墙路由：2. VPN 的搭建]({% post_url 2021-08-14-ha-gfw-router-vpn %})<br>
**下一篇文章：**高可用性翻墙路由：4. DNS 分流
{: .notice}
