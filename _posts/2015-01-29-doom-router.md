---
title: 自建智能宿舍路由
category: 技术
tags: 网络
---

教育网的网络环境可谓十分复杂。为了存活，你不得不从物理层到应用层，层层打通障碍，最终实现无障碍访问。从校内地址到免费地址、收费地址再到全球地址，每次开机后台都要运行若干个代理软件。这是计算机的情况，而手机访问则更加麻烦。由于我的手机并没有root，各种代理软件几乎难以奏效，可谓难上加难。

为了统一解决这些网络困难，减少网络问题带来的不适，我开始了改造宿舍路由的漫漫征程。

路由购置
----
改造路由的第一步自然是刷成开源ROM，想靠原版ROM生存想必是不够的。宿舍以前的路由是路边80+乱买的TP-LINK WR740N v5.7，是一款中国定制型路由。这款路由虽然支持刷机，但Flash只有区区2M，以至于OpenWrt和DD-WRT都不对其提供支持。摆在面前的路有两条：更换闪存或另外购买。作为被电路基础实验虐暴的我，果断选择了后者。

HuaWei HG255D，这是一款电信的定制路由。虽说是定制机，但功能还是十分强大的：16M的Flash、32M的RAM、386MHz的Ralink3052处理器、一个USB口、一个WAN口、四个LAN口，支持300M的802.11bgn无线。在淘宝上有很多二手货，30元的价格加上10元的运费，完虐80+的TP-LINK一条街。

刷机
----
拿到路由时，上面装的是电信原装的天翼宽带的固件。看都不用看，直接刷掉。这款路由原装的uboot据说比较奇怪，如果不更换后期OpenWrt无法成功安装。我用了lintel的uboot，详细刷uboot的方法可以参考[恩山上的这篇文章](http://www.right.com.cn/forum/thread-103361-1-1.html)。刷好uboot后，以后每次刷机就十分简单了，把自己网卡设为192.168.1.xxx，然后按住Wi-Fi按钮启动路由，待Power灯闪烁时打开tftp刷机工具刷入固件即可。网上说固件传输完毕后需要等待5分钟才能刷好。以我实际经验来看，如果固件没有问题一般两分钟就能刷完并重启完毕；但如果固件有问题，等待20分钟都没用的。

固件编译
--------
目前网上资料最多的应属OpenWrt的14.07（Barrier Breaker）了。这一版本网上有许多别人为HG255D定制好的固件，但这些固件大多集成了我不需要的很多软件包，例如PPPoE等。另一方面，后期需要根据需求自己选择软件包加入，这样一来自己编译固件是跑不掉了。

OpenWrt的编译还是十分人性化的，只需运行区区几个命令，配置好需要的软件包，就可以一句make完成从软件包下载到ROM生成的一系列工作了。详细的固件编译过程可以参考[Demon's Blog上的这篇文章](http://demon.tw/hardware/hg255d-compile-openwrt-barrier-breaker.html)。

OpenWrt默认安装的软件包很少，甚至没有Web配置界面。因此需要在make menuconfig中好好配置一下。下面是一些我添加的软件包：

* ebtables: 以太网桥防火墙，用于搭建ipv6网桥。
* lcui: Web管理界面。
* openssh-sftp-server: 为SSH加入SFTP支持。
* openvpn-openssl: 搭建到收费地址代理的隧道。
* vsftpd: FTP服务器，提供SFTP服务。
* wifitoggle: 使路由器侧面的Wi-Fi按钮起效。

收费网址隧道
------------
教育网把所有ipv4网络地址划分为三个范围：校内网址、免费地址和收费地址。校内地址插上网线即可访问，无需任何设置；免费地址需要去校园网关登录，之后即可不限时访问，主要是国内的地址，可以满足聊天、交友、闲逛等低层次生活需求，但想看一下国外的技术文章就无能为力了；收费地址也需要在校园网关登录，需要每月缴纳100元才可不限时访问，包括国内的一些地址和国外的大部分地址。

我并不是壕，因此让路由7×24小时登录收费地址是不用想了。不过好在校内有一台可以访问收费地址的服务器，因此解决方案是在路由与服务器间搭建隧道，将访问外网的流量通过服务器代理。搭建隧道最方便的方式是使用OpenVPN。

首先是服务器一端的配置，使用网络层隧道，本地为10.0.206.1，远程为10.0.206.2。由于只是校内通信，为了效率没有使用身份验证和加密，一切明文传输。下面是OpenVPN的配置文件：

~~~~
dev tun
ifconfig 10.0.206.1 10.0.206.2
keepalive 10 120
comp-lzo
persist-tun
verb 3
~~~~

只构建隧道是不行的，隧道相当于在路由和服务器之间插上了一根虚拟的网线，但要想实现收费地址代理的效果，还需要配置NAT。服务器本身是Windows 2012 R2的系统，因此配置NAT相当简单，在服务器管理器中安装“路由与远程访问”功能，之后在路由与远程访问中开启NAT，按照向导依次选择公网的网卡和OpenVPN隧道的网卡即可。整个过程可谓傻瓜式操作。

接下来是OpenWrt端的OpenVPN配置。`remote`为服务器的校内地址，而两项`route`则为隧道建立后在路由器的路由表中添加的内容。这里最关键的是要设置好路由规则。第一个`remote`是对于全部ipv4地址用隧道转发但这样就无法连接服务器本身了，因此需要第二条规则，即对于校内地址`xxx.xxx.0.0/16`则使用原有的网关`xxx.xxx.yyy.1`（这是宿舍接通网线后的默认网关）。设置完毕运行`/etc/init.d/openvpn restart`或重启路由就行了。下面是`/etc/config/openvpn`的内容：

~~~~
package openvpn

config openvpn
    option enabled 1
    option dev tun
    option remote "xxx.xxx.xxx.xxx"
    option ifconfig "10.0.206.2 10.0.206.1"
    list route "0.0.0.0 0.0.0.0"
    list route "xxx.xxx.0.0 255.255.0.0 xxx.xxx.yyy.1"
    option persist_tun 1
    option comp_lzo yes
    option verb 3
~~~~

IPv6桥接
--------
教育网的ipv6是原生的，而且不需要登录网关。因此最好的策略是对于ipv4流量用NAT转发，而对于ipv6流量直接转发，不做处理。这里我用到了一个小技巧，方法来源于[6bridge](https://wiki.tuna.tsinghua.edu.cn/openwrt/6bridge)。首先在LCUI中将wan口加入br-lan网桥中。之后wan口与lan口之间的数据都会被转发，但如此一来内外的ipv4流量也会被转发，造成对外部的DHCP污染。解决方法是利用`ebtables`工具将br-lan中通过wan口的非ipv6流量禁用掉。运行如下命令：

~~~~
ebtables -t broute -A BROUTING -i eth0.2 -p ! ipv6 -j DROP
~~~~

其中`eth0.2`是wan口的名称。为了每次开机自动生效，可以将上述命令加入开机启动脚本中。这里还有一个问题，实际应用中，上述命令似乎对OpenVPN隧道产生了影响，运行后需重启OpenVPN。因此我将OpenVPN的开机项禁用掉，转而在脚本最后加入启动OpenVPN的命令：

~~~~
ebtables -t broute -A BROUTING -i eth0.2 -p ! ipv6 -j DROP
/etc/init.d/openvpn start
~~~~

未完成事项
----------
* 利用USB接口外接硬盘，将路由配置为野生Time Capsule。
* 实现“全球”访问。
