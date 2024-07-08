# 配置自己的 VPN——内置 FreeBSD、Wireguard、IPv6 和广告拦截（基于 FreeBSD）

由 STEFANO MARINELLI

注意：本文假定基于 FreeBSD 设置。如果您更喜欢基于 OpenBSD 的版本，可以在这里找到。

VPN 是一种安全连接到自己的服务器和设备的基本工具。许多人出于各种原因使用商业 VPN，从不信任其提供者（特别是在连接到公共热点时）到希望使用不同 IP 地址“出网”，也许是来自另一个国家的。在这里，我想强调一些已经纳入基本堆栈的新功能，其中许多功能已默认启用，有些可能需要特别激活。每个功能将被描述，并提供可能有助于改善网络体验的详细信息。

无论原因是什么，解决方案都不会缺少。我始终设置管理 VPN，以允许服务器和/或客户端使用安全通道相互通信。最近，我在所有设备上（包括桌面/服务器和移动设备）激活了 IPv6 连接，并且需要快速创建一个节点，集中一些网络并允许它们在 IPv6 上出网。我使用的工具将进行描述：

* VPS - 在这种情况下，我使用了一个基本的 Hetzner Cloud VPS，但是任何提供 IPv6 连接的提供商都可以 - 如果你想要 IPv6 的话。
* FreeBSD - 一种多功能、稳定和安全的操作系统。
* Wireguard - 轻量级、安全，同时又不会很“啰嗦”，因此它对移动设备的电池也很友好。在没有流量时，它就简单地不发送/接收任何数据。受到所有主要桌面和服务器操作系统以及 Android 和 iOS 设备的良好支持。
* Unbound – 可直接向根服务器进行 DNS 查询，而不通过转发器。它还允许您插入阻止列表，从而实现类似 Pi-Hole 的结果（即广告拦截）。
* SpamHaus 列表 – 立即停止与用户黑名单上的连接。

第一步是激活 VPS 并安装 FreeBSD。在 Hetzner Cloud 控制台上，可能没有预构建的 FreeBSD 镜像，而只有一些 Linux 发行版可供选择。不用担心，只需选择其中任何一个并创建 VPS。完成后，在“ISO 镜像”中将可用 FreeBSD ISO 镜像。只需插入虚拟 CD，重新启动 VPS，FreeBSD 安装将显示在控制台中。

我不会详细说明，操作简单明了。唯一的注意事项（对于 Hetzner Cloud VPS 的情况）是使用 IPv4 的“DHCP”，但目前不要配置 IPv6。稍后会进行配置。

安装所有 FreeBSD 更新（使用 freebsd-update fetch install 命令），然后重新启动。

在 FreeBSD 上，Wireguard 现在作为内核模块可用，并且用户空间可以通过 pkg install wireguard-tools 软件包管理器安装。这意味着您可以轻松地将其与系统上的其他软件一起更新。

第一步是在 VPS 上配置 IPv6。在 Hetzner 的情况下，遗憾的是，他们只提供/64，因此需要对分配的网络进行分段。在本例中，它将被分成/72 子网 - 要找到有效的子类，可以使用计算器来查找。

/etc/rc.conf 文件应该有类似的条目：

`ifconfig_vtnet0=”DHCP”ifconfig_vtnet0_ipv6=”inet6 2a01:4f8:cafe:cafe::1 prefixlen 72”ipv6_defaultrouter=”fe80::1%vtnet0”`

简而言之，保留 Hetzner 分配的基础地址，但将前缀长度更改为 72 - 从而使其他网络可用。现在需要为 IPv4 和 IPv6 启用转发。将这些行添加到 /etc/sysctl.conf 文件中。

`net.inet.ip.forwarding=1net.inet6.ip6.forwarding=1`

重启后，进行测试：

`ping6 google.com`

如果一切配置正确，将执行 ping 操作，google.com 会做出回复。

要配置 Wireguard，将需要一些步骤。首先，需要创建私钥：

`wg genkey | tee /dev/stderr | wg pubkey | grep --label PUBLIC -H .`

你将获得一个私钥和一个公钥。注意公钥 — 配置客户端所需。

现在创建一个名为 /usr/local/etc/wireguard/wg0.conf: 的新文件

`[Interface]Address = 172.14.0.1/24,2a01:4f8:cafe:cafe:100::1/72ListenPort = 51820PrivateKey = YUkS6cNTyPbXmtVf/23ppVW3gX2hZIBzlHtXNFRp80w=`

正在创建一个名为 wg0 的新 Wireguard 接口。启动 Wireguard 接口

`service wireguard enablesysrc wireguard_interfaces=”wg0”service wireguard start`

如果一切输入正确，界面应该会启动。检查其状态：

`wg`

至于防火墙，FreeBSD 并不带有 pf 配置。在我的设置中，我倾向于阻止不必要的内容，并允许可能有用的内容。然而，我喜欢把“坏人”挡在门外，所以我使用黑名单。pf 允许在运行时向表中插入和删除元素，因此防火墙可以相应地进行配置。

要下载并应用 Spamhaus 列表，我使用了在互联网上找到的一个简单但有效的脚本，不过是针对 OpenBSD 的。

针对 Spamhaus 列表，继续进行 FreeBSD 脚本创建。

在 /usr/local/sbin/spamhaus.sh: 中创建脚本。

`#!/bin/sh##this is normally run once per day via cron.#echo updating Spamhaus DROP lists:(  { fetch -o - https://www.spamhaus.org/drop/drop.txt && \<br/>    fetch -o - https://www.spamhaus.org/drop/edrop.txt && \<br/>    fetch -o - https://www.spamhaus.org/drop/dropv6.txt ; \<br/>  } 2>/dev/null | sed “s/;/#/” > /var/db/drop.txt)pfctl -t spamhaus -T replace -f /var/db/drop.txt`

将其设为可执行并运行。由于 Pf 未启用，您将收到一个错误消息 —— 但这将创建 /var/db/drop.txt 文件：

`chmod a+rx /usr/local/sbin/spamhaus.sh/usr/local/sbin/spamhaus.sh`

在 FreeBSD 上有许多配置 pf 的可能性。一个相当简单的例子可能是这样的：

`ext_if="vtnet0"wg0_if="wg0"wg0_networks="172.14.0.0/24"set skip on lonat on $ext_if from { $wg0_networks } to any -> ($ext_if)# Spamhaus DROP list:table <spamhaus> persist file "/var/db/drop.txt"block drop log quick from <spamhaus># Pass ICMP on ipv6pass quick proto ipv6-icmp# Block from ipv6 to wg0 networkblock in quick on $ext_if inet6 to { 2a01:4f8:cafe:cafe:100::/72 }# Pass Wireguard traffic - in and outpass quick on $wg0_if# default denyblock inblock outpass in on $ext_if proto tcp to port sshpass in on $ext_if proto udp to port 51820pass out on $ext_if`

这是一个非常简单的配置：它阻止从 Spamhaus 下载的列表中存在的所有内容，允许来自 Wireguard 网络到公共接口的 NAT，允许 IPv6 中的 ICMP 流量（对网络正常运行至关重要），同时阻止来自 Wireguard IPv6 LAN 的传入流量（请记住，IP 将是公共的并且可以直接访问，所以我们不希望默认情况下暴露我们的设备）。 Wireguard 接口上的所有流量都将被允许通过。 然后将阻止所有内容，并指定例外情况，即允许 SSH 和 Wireguard 连接（当然）。 还将授权允许从公共网络接口退出流量。

保存这个配置到/etc/pf.conf。

启用并启动 pf：

`service pf enableservice pf start`

您可能会被系统踢出。不用担心，只需重新连接。pf 正在执行其工作。

如果一切正常，防火墙应该已加载新的规则。

要获取 DNS 查询的缓存和相关的广告拦截功能，现在是配置 Unbound 的时候了。让我们使用以下命令安装它：

`pkg install unbound`

前段时间，我找到了一个脚本，稍作调整后使用。我不记得从哪里获取的了，所以我会在这里粘贴它而不引用原始创建者。

在 /usr/local/sbin/unbound-adhosts.sh 中创建一个脚本来更新 unbound 广告拦截：

`#!/bin/sh## Using blacklist from pi-hole project https://github.com/pi-hole/# to enable AD blocking in unbound(8)#PATH=”/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin”# Available blocklists - comment line to disable blocklist_disconad=”https://s3.amazonaws.com/lists.disconnect.me/simple_ad.txt”_discontrack=”https://s3.amazonaws.com/lists.disconnect.me/simple_tracking.txt”_stevenblack=”https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts”# Global variables_tmpfile=”$(mktemp)” && echo '' > $_tmpfile_unboundconf=”/usr/local/etc/unbound/unbound-adhosts.conf”# Remove comments from blocklistsimpleParse() {fetch -o - $1 | \<br/>sed -e ‘s/#.*$//’ -e ‘/^[[:space:]]*$/d’ >> $2}# Parse DisconTrack[ -n “${_discontrack}” ] && simpleParse $_discontrack $_tmpfile# Parse DisconAD[ -n “${_disconad}” ] && simpleParse $_disconad $_tmpfile# Parse StevenBlack[ -n “${_stevenblack}” ] && \<br/>  fetch -o - $_stevenblack | \<br/>  sed -n '/Start/,$p' | \<br/>  sed -e 's/#.*$//' -e '/^[[:space:]]*$/d' | \<br/>  awk '/^0.0.0.0/ { print $2 }' >> $_tmpfile# Create unbound(8) local zone filesort -fu $_tmpfile | grep -v “^[[:space:]]*$” | \<br/>awk '{  print “local-zone: \”” $1 “\” redirect”  print “local-data: \”” $1 “ A 0.0.0.0\””}' > $_unboundconf && rm -f $_tmpfileservice unbound reload 1>/dev/nullexit 0`

保存脚本后，将其设置为可执行并运行：

`chmod a+rx /usr/local/sbin/unbound-adhosts.sh/usr/local/sbin/unbound-adhosts.sh`

现在，可以修改 /usr/local/etc/unbound/unbound.conf 中的 Unbound 配置文件如下：

`server:        verbosity: 1        log-queries: no        num-threads: 4        num-queries-per-thread: 1024        interface: 127.0.0.1        interface: 172.14.0.1        interface: 2a01:4f8:cafe:cafe:100::1        interface: ::1        outgoing-range: 64        chroot: “”        access-control: 0.0.0.0/0 refuse        access-control: 127.0.0.0/8 allow        access-control: ::0/0 refuse        access-control: ::1 allow        access-control: 172.14.0.0/24 allow        access-control: 2a01:4f8:cafe:cafe:100::/72 allow        hide-identity: yes        hide-version: yes        auto-trust-anchor-file: "/usr/local/etc/unbound/root.key"        val-log-level: 2        aggressive-nsec: yes        prefetch: yes        username: “unbound”        directory: "/usr/local/etc/unbound"        logfile: "/var/log/unbound.log"        use-syslog: no        pidfile: "/var/run/unbound.pid"        include: /usr/local/etc/unbound/unbound-adhosts.confremote-control:        control-enable: yes        control-interface: /var/run/unbound.sock`

现在，启用并启动 Unbound：

`service unbound enableservice unbound start`

如果一切设置正确，unbound 将能够响应在 172.14.0.1 和 2a01:4f8:cafe:cafe:100::1 上进行的 DNS 请求。现在可以配置 Wireguard 客户端。通过将“172.14.0.2/32, 2a01:4f8:cafe:cafe:100::2/128”（稍后将在服务器的对等配置中输入）插入本地 IP 地址来创建新配置。将 DNS 服务器地址设置为“172.14.0.1”及其相应的 IPv6 地址（在示例中， 2a01:4f8:cafe:cafe:100::1 – 您的将不同）。在对等方案节中，插入服务器的数据，包括其公钥，IP 地址：port（在示例中，port为 51820），以及允许地址（设置“ 0.0.0.0/0, ::0/0 ”意味着“所有连接将通过 Wireguard 发送” — 所有流量将通过 VPN 传递，无论是 IPv4 还是 IPv6）。每种实现都有自己的流程（Android、iOS、MikroTik、Linux 等），但主要是在服务器和客户端上创建正确的配置就足够了。

重新打开 Wireguard 配置文件 /usr/local/etc/wireguard/wg0.conf 并添加：

`[Interface]Address = 172.14.0.1/24,2a01:4f8:cafe:cafe:100::1/72ListenPort = 51820PrivateKey = YUkS6cNTyPbXmtVf/23ppVW3gX2hZIBzlHtXNFRp80w=[Peer]PublicKey = *client's public key*AllowedIPs = 172.14.0.2/32, 2a01:4f8:cafe:cafe:100::2/128`

客户端的公钥将由客户端本身显示。重新加载 Wireguard 配置：

`service wireguard restart`

只需通过 VPN 路由 DNS 流量，即可将 VPN 仅用作广告拦截器。要实现这一目标，请配置客户端，仅允许的地址是刚刚配置的 unbound 的地址（例如，在本示例中，172.14.0.1 和/或 2a01:4f8:cafe:cafe:100::1） — DNS 解析将通过 VPN 进行，但浏览将继续通过主供应商进行。

要自动更新 spamhaus 和广告拦截列表，我们将使用 cron。首先，创建一个脚本，例如，/usr/local/sbin/update-blocklists.sh：

`#!/bin/sh/usr/local/sbin/unbound-adhosts.sh/usr/local/sbin/spamhaus.sh`

 使其可执行：

`chmod +x /usr/local/sbin/update-blocklists.sh`

然后，将其添加到 crontab 中以每天运行：

`echo “@daily /usr/local/sbin/update-blocklists.sh” >> /etc/crontab`

从更新管理和安全性的角度来看，这种方法都有利。

STEFANO MARINELLI 是一位 IT 顾问，拥有超过二十年的 IT 咨询、培训、研究和出版经验。他的专业涵盖操作系统，特别是 *BSD 系统 — FreeBSD、NetBSD、OpenBSD、DragonFlyBSD — 和 Linux。Stefano 还是 BSD Cafe 的咖啡师，这是 *BSD 爱好者的一个活跃社区中心，并在博洛尼亚大学领导 FreeOsZoo 项目，为虚拟机提供开源操作系统镜像。
