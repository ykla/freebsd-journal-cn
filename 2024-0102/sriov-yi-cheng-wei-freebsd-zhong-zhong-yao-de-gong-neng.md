# SR-IOV 已成为 FreeBSD 中重要的功能

### 使用 FreeBSD 中支持 SR-IOV 设备的硬件驱动虚拟化的详细设置步骤演练。

 由马克·麦克布赖德

我最喜欢的硬件特性之一是称为单根输入/输出虚拟化（SR-IOV）的功能。它使单个物理设备在操作系统中显现为多个类似设备。FreeBSD 在展示 SR-IOV 功能方面的方法是我倾向于在我的服务器上使用 FreeBSD 的几个原因之一。

## 网络 SR-IOV 概述

虚拟化是一个很好的解决方案，如果您对网络设备的需求超过服务器上物理网络ports的数量。有许多实现此目标的软件方式，但硬件方案是 SR-IOV，它允许一个物理 PCIe 设备向操作系统呈现为多个设备。

使用 SR-IOV 有几个优点。与其他虚拟化手段相比，它提供了最佳性能。如果您对安全性特别重视，SR-IOV 更好地隔离了内存和创建的虚拟 PCI 设备。这也会带来非常整洁的设置，因为一切都是 PCI 设备，即没有虚拟桥接、交换机等。

要利用 SR-IOV 网络，您需要一张支持 SR-IOV 的网络适配器和一台支持 SR-IOV 的主板。多年来，我使用过几款支持 SR-IOV 的网络卡，例如 Intel i350-T4V2 以太网适配器、Mellanox ConnectX-4 Lx 和 Chelsio T520-SO-CR 光纤网卡。在本文中，我将在 FreeBSD 14.0-RELEASE 服务器上使用一款 Intel X710-DA2 光纤网卡（产品简介）。这是一个不错的选择，因为它不需要特殊的固件配置，并且驱动程序支持已被默认构建到 FreeBSD 内核中。而且作为一个额外的好处，它只消耗少量功率，最大功率仅为 3.7 瓦特。

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig1.jpg)

X710-DA2 有两个物理 SFP+光纤端口。在 SR-IOV 术语中，这些对应于物理功能(PFs)。没有启用 SR-IOV 时，PFs 的行为类似于任何网络适配器卡上的端口，并将显示为 FreeBSD 中的两个网络接口。启用 SR-IOV 后，每个 PF 能够创建、配置和管理多个虚拟函数(VFs)。每个 VF 将在操作系统中显示为一个 PCIe 设备。

对于 X710-DA2 具体而言，其 2 个 PF 可以虚拟化多达 128 个 VF。从 FreeBSD 的角度来看，好像你拥有一张具有 128 个端口的网络卡。然后可以将这些 VF 分配给容器和虚拟机，实现网络隔离。

## 在 FreeBSD 中使用 SR-IOV

我们已经简要介绍了 SR-IOV 的概念工作原理，但通过实际示例更容易理解。让我们从头开始在 FreeBSD 中设置 SR-IOV。为此，我们将专注于：

* [ 硬件安装](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/sr-iov-is-a-first-class-freebsd-feature/#hardware_installation)
* [ 硬件配置](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/sr-iov-is-a-first-class-freebsd-feature/#hardware_configuration)
* [FreeBSD SR-IOV 配置](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/sr-iov-is-a-first-class-freebsd-feature/#freebsd_configuration)
* [在 Jail 中使用 SR-IOV 网络 VF](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/sr-iov-is-a-first-class-freebsd-feature/#vf_jail)
* [在 Bhyve 虚拟机中使用 SR-IOV 网络 VF](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/sr-iov-is-a-first-class-freebsd-feature/#vf_bhyve)

## 硬件安装

安装支持 SR-IOV 的 X710-DA2 相对简单，但有一个主要考虑因素。并非所有主板上的 PCIe 插槽都是相同的。在开始之前，我强烈建议您查阅主板的手册。本例中，我将使用 Supermicro X12STH-F 主板。手册提供了两张有见地的图表：

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig2.jpg)

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig3.jpg)

在第一张图中，我们看到我们的 PCIe 插槽从左到右编号为 4、5 和 6。如果仔细观察，您会看到插槽 4 带有“PCH”前缀，而 5 和 6 带有“CPU”前缀。块图示说明了这一点的更多细节。插槽 5 和 6 直接连接到 LGA1200 插座中的 CPU，而插槽 4 连接到平台控制器集线器。根据系统中具体的组件，这可能决定哪些插槽可以如预期般支持 SR-IOV。在稍后配置 FreeBSD 时，这并没有简单的方法来知道，但作为一个经验法则，特别是对于较旧的主板，我发现 CPU 插槽是一个可靠的选择。如果在后续步骤中发现 SR-IOV 未能正常工作，请尝试使用不同的 PCIe 插槽。主板文档并非总是详细，因此有时试错可能是快速找到适合的插槽的方法。

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig5.jpg)

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig6.jpg)

## 硬件配置

X710-DA2 卡在您的主板设置中不支持 SR-IOV 时将表现为非 SR-IOV 能力卡，直到您启用 SR-IOV。这很容易做到，但也很容易忘记，所以务必不要跳过这一重要步骤。

具体的过程会因主板而异，但大多数主板都会有 PCIe 配置选项的屏幕。找到该屏幕并启用 SR-IOV。在那里的时候，检查其他您可能与 SR-IOV 一同使用的设置是否已启用，比如 CPU 虚拟化，是一个好主意。

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig7.jpg)

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig8.jpg)

我们现在可以启动 FreeBSD 并查看 dmesg(8)。这是我机器上的一个片段。

`ixl0: <Intel(R) Ethernet Controller X710 for 10GbE SFP+ - 2.3.3-k> mem      0x6000800000-0x6000ffffff,0x6001808000-0x600180ffff irq 16 at device 0.0 on pci1ixl0: fw 9.120.73026 api 1.15 nvm 9.20 etid 8000d87f oem 1.269.0ixl0: PF-ID[0]: VFs 64, MSI-X 129, VF MSI-X 5, QPs 768, I2Cixl0: Using 1024 TX descriptors and 1024 RX descriptorsixl0: Using 4 RX queues 4 TX queuesixl0: Using MSI-X interrupts with 5 vectorsixl0: Ethernet address: 3c:fd:fe:9c:9e:30ixl0: Allocating 4 queues for PF LAN VSI; 4 queues activeixl0: PCI Express Bus: Speed 2.5GT/s Width x8ixl0: SR-IOV ready ixl0: netmap queues/slots: TX 4/1024, RX 4/1024`

在第三行我们看到一些 SR-IOV 的引用。“PF-ID[0]”与 ixl0 相关联，此 PF 能够支持 64 个 VF。在第十行，我们得到一个很好的确认，这个 PCIe 设备是“SR-IOV ready”。ixl 名称的原因是该网卡使用了 ixl(4) Intel Ethernet 700 系列驱动程序。

配置 X710-DA2 的硬件不需要其他操作。一些网卡（如前面提到的 Mellanox）需要配置网卡的固件，而另一些网卡（如前面提到的 Chelsio）则需要在 /boot/loader.conf 中进行驱动程序配置。不过，使用 X710-DA2 则不需要这些操作，尽管您可能需要检查网卡的固件版本，并在必要时进行更新。

有了这个，我们准备从硬件设置转向 FreeBSD 配置。

## FreeBSD 的 SR-IOV 配置

### 使用 PFs

SR-IOV 的一个好处是，无论您是否告诉 PF 创建 VF，您仍然可以将 PF 用作网络接口。 我将添加以下内容到我的 /etc/rc.conf 并为 PF 指定一个 IP 地址，以便在主机中使用

`ifconfig_ixl0=”inet 10.0.1.201 netmask 255.255.255.0” defaultrouter=”10.0.1.1”`

现在当我启动系统时，无论 SR-IOV 是否已启用，我都可以期望 ixl0 设备具有一个 IP 地址，我可以使用它来连接到系统。

### 告诉 PFs 创建 VFs

FreeBSD 中的 PF 和 VF 管理由 iovctl(8)处理，该程序包含在基础操作系统中。要创建 VF，我们需要在 /etc/iov/ 目录中创建一个文件，其中包含我们希望的一些具体信息。我们将执行一个简单的策略，创建一个 VF 分配给jail，另一个用于 bhyve 虚拟机。iovctl.conf(5)手册页面将为我们提供最重要的参数。

`OPTIONS     The following parameters are accepted by all PF drivers:     device (string)             This parameter specifies the name of the PF device. This             parameter is required to be specified.     num_vfs (uint16_t)             This parameter specifies the number of VF children to create.             This parameter may not be zero. The maximum value of this             parameter is device-specific.`

我喜欢根据需要设置 num_vfs。我们可以将其设置为最大值，但我发现这样做会使得查看 ifconfig 和其他命令输出变得更加困难。

另外，由于不同的网卡使用不同的驱动程序，每个驱动程序都有基于硬件能力的选项可供设置。ixl(4)手册页面列出了几个可选参数。

`IOVCTL OPTIONS     The driver supports additional optional parameters for created VFs     (Virtual Functions) when using iovctl(8):     mac-addr (unicast-mac)             Set the Ethernet MAC address that the VF will use. If             unspecified, the VF will use a randomly generated MAC address.`

或者，您还可以使用 iovctl 命令来获取 PF 及其 VF 的有效参数的简要摘要，以及它们的默认值。

`(host) $ sudo iovctl -S -d ixl0The following configuration parameters may be configured on the PF:     num_vfs : uint16_t (required)     device : string (required)The following configuration parameters may be configured on a VF:     passthrough : bool (default = false)     mac-addr : unicast-mac (optional)     mac-anti-spoof : bool (default = true)     allow-set-mac : bool (default = false)     allow-promisc : bool (default = false)     num-queues : uint16_t (default = 4)`

我们将使用 mac-addr 参数为每个 VF 设置特定的 MAC 地址。在这种情况下，设置 MAC 地址有点随意，但我将演示如何在配置文件中包含 PF 参数、默认 VF 参数以及特定于各个 VF 的参数。

`PF {       device : “ixl0”       num_vfs : 2}DEFAULT {       allow-set-mac : true;}VF-0 {       mac-addr : “aa:88:44:00:02:00”;}VF-1 {       mac-addr : “aa:88:44:00:02:01”;}`

这条指令指示 ixl0 创建两个 VF。默认情况下，每个 VF 都可以设置自己的 MAC 地址。每个 VF 都将分配一个初始 MAC 地址（可以用前面的默认设置覆盖）。

在我们使其生效之前，让我们先查看一下我们目前的环境。我们会发现两个 ixl PCI 设备和两个 ixl 网络接口。

`(host) $ ifconfig -lixl0 ixl1 lo0(host) $ pciconf -lv | grep -e ixl -e iavf -A4ixl0@pci0:1:0:0:        class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086device=0x1572 subvendor=0x8086 subdevice=0x0007    vendor     = 'Intel Corporation'    device     = 'Ethernet Controller X710 for 10GbE SFP+'    class      = network    subclass   = ethernetixl1@pci0:1:0:1:        class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086device=0x1572 subvendor=0x8086 subdevice=0x0000    vendor     = 'Intel Corporation    device     = 'Ethernet Controller X710 for 10GbE SFP+'    class      = network    subclass   = ethernet`

要使我们的/etc/iov/ixl0.conf 配置生效，我们使用 iovctl(8)。

`(host) $ sudo iovctl -C -f /etc/iov/ixl0.conf`

如果您更改了配置文件，请删除并重新创建 VF。

`(host) $ sudo iovctl -D -f /etc/iov/ixl0.conf(host) $ sudo iovctl -C -f /etc/iov/ixl0.conf`

检查它是否工作的方法是，运行之前使用的相同 ifconfig 和 pciconf 命令。

`(host) $ ifconfig -lixl0 ixl1 lo0 iavf0 iavf1(host) $ pciconf -lv | grep -e ixl -e iavf -A4ixl0@pci0:1:0:0:        class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x1572 subvendor=0x8086 subdevice=0x0007    vendor     = 'Intel Corporation'    device     = 'Ethernet Controller X710 for 10GbE SFP+'    class      = network    subclass   = ethernetixl1@pci0:1:0:1:        class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x1572 subvendor=0x8086 subdevice=0x0000    vendor     = 'Intel Corporation'    device     = 'Ethernet Controller X710 for 10GbE SFP+'    class      = network    subclass   = ethernet--iavf0@pci0:1:0:16:        class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x154c subvendor=0x8086 subdevice=0x0000    vendor     = 'Intel Corporation'    device     = 'Ethernet Virtual Function 700 Series'    class      = network    subclass   = ethernetiavf1@pci0:1:0:17:        class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x154c subvendor=0x8086 subdevice=0x0000    vendor     = 'Intel Corporation'    device     = 'Ethernet Virtual Function 700 Series'    class      = network    subclass   = ethernet`

瞧！我们闪亮的新 VF 已经到达。在 pciconf 输出中，我们仍然可以看到我们的 ixl 设备，但现在多了两个 iavf 设备。iavf(4) 手册页告诉我们，这是 Intel 自适应虚拟功能的驱动程序。

除了看到新的 PCI 设备外，ifconfig 确认它们确实被识别为网络接口。对于网络设备的大多数常见方面，您可能无法区分 PF 和 VF 之间的差异。如果您想深入了解详细信息和差异，请查阅驱动程序文档以及 pciconf 的 -c 功能标志，例如 pciconf -lc iavf 。

要使此配置持久跨重启生效，请修改您的 /etc/rc.conf 文件。

`# Configure SR-IOViovctl_files=”/etc/iov/ixl0.conf”`

现在我们有两个准备好使用的虚拟功能(VFs)。让我们把它们投入使用吧！

## 在Jail中使用 SR-IOV 网络 VF。

本节假定您对 FreeBSD 有基本的了解Jails。因此，从头开始设置jail超出了范围。有关如何执行此操作的更多信息，请参阅 FreeBSD 手册的Jails和容器章节。

我不使用任何jail管理ports，依赖于基本操作系统中提供的内容。如果您使用过类似 Bastille 的软件，那么放置配置文件的具体位置/方式可能会有所不同，但概念是相同的。在这个例子中，我们正在处理一个名为“desk”的jail。

`exec.start += “/bin/sh /etc/rc”;exec.stop = “/bin/sh /etc/rc.shutdown”;exec.clean;mount.devfs;desk {        host.hostname = “desk”;        path = “/mnt/apps/jails/desk”;        vnet;        vnet.interface = “iavf0”;        devfs_ruleset=”5”;        allow.raw_sockets;}`

就这些了！jail现在可以访问其专用的 VF 网络设备，通过 vnet(9)进行设置。我将调整jail的/etc/rc.conf 文件以启用它。

`ifconfig_iavf0=”inet 10.0.1.231 netmask 255.255.255.0”defaultrouter=”10.0.1.1”`

现在让我们开始jail并检查它是否工作。

`(host) $ sudo service jail start deskStarting jails: desk.(host) $ sudo jexec desk ifconfig iavf0iavf0: flags=1008843<up,broadcast,running,simplex,multicast,lower_up><span> </span>metric 0 mtu 1500        options=4e507bb<rxcsum,txcsum,vlan_mtu,vlan_hwtagging,jumbo_mtu,vlan_hwcsum,tso4,< br=""><span> </span>TSO6,LRO,VLAN_HWFILTER,VLAN_HWTSO,RXCSUM_IPV6,TXCSUM_IPV6,HWSTATS,MEXTPG>        ether aa:88:44:00:02:0010.0.1.231 netmask 0xffffff00 broadcast 10.0.1.255        media: Ethernet autoselect (10Gbase-SR<span> </span><full-duplex>)        status: active        nd6 options=29<performnud,ifdisabled,auto_linklocal>(host) $ sudo jexec desk ping 9.9.9.9PING 9.9.9.9 (9.9.9.9): 56 data bytes64 bytes from 9.9.9.9: icmp_seq=0 ttl=58 time=19.375 ms64 bytes from 9.9.9.9: icmp_seq=1 ttl=58 time=19.809 ms64 bytes from 9.9.9.9: icmp_seq=2 ttl=58 time=19.963 ms</performnud,ifdisabled,auto_linklocal></full-duplex></rxcsum,txcsum,vlan_mtu,vlan_hwtagging,jumbo_mtu,vlan_hwcsum,tso4,<></up,broadcast,running,simplex,multicast,lower_up>`

正如预期的那样，我们在jail中看到 iavf0 接口，并且它似乎正常工作。但主机操作系统中的那台设备怎么样了？让我们来检查一下。

`(host) $ ifconfig -lixl0 ixl1 lo0 iavf1`

## 在 Bhyve 虚拟机中使用 SR-IOV 网络 VF

通过 bhyve(8)虚拟机，您可以实现类似的结果，尽管方法有所不同。使用jails，我们可以在运行时分配/释放 VF。使用 bhyve，则必须在启动时执行此操作，并需要调整我们的 SR-IOV 配置。首先，在更改任何内容之前，让我们再次查看 pciconf。

`(host) $ pciconf -l | grep iavfiavf0@pci0:1:0:16:      class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x154csubvendor=0x8086 subdevice=0x0000iavf1@pci0:1:0:17:      class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x154csubvendor=0x8086 subdevice=0x0000`

查看未使用的 VF，iavf1。第一列可以解读为“有一个使用驱动程序 iavf，ID 为 1 的 PCI0 设备，具有总线 1、插槽 0、功能 17 的 PCI 选择器”。虽然您现在不需要它们，但最后三个数字是我们最终将告诉 bhyve 要使用哪个设备的方式。在我们开始之前，请确保我们在启动时加载 vmm(4)以启用 bhyve，并调整我们的第二个 VF，以便为其进行 bhyve 透传准备。

`## Load the virtual machine monitor, the kernel portion of bhyvevmm_load=”YES”# Another way to passthrough a VF, or any PCI device, is to# specify the device in /boot/loader.conf. I show this for reference.# We’ll use our iovctl config instead as it keeps things in one place.# pptdevs=”1/0/17”`

要保留 VF 以进行 bhyve 透传，我们使用 iovctl 透传参数。

`    passthrough (boolean)              This parameter controls whether the VF is reserved for the use of              the bhyve(8) hypervisor as a PCI passthrough device. If this              parameter is set to true, then the VF will be reserved as a PCI              passthrough device and it will not be accessible from the host              OS. The default value of this parameter is false.`

`PF {        device : “ixl0”        num_vfs : 2}DEFAULT {        allow-set-mac : true;}VF-0 {        mac-addr : “aa:88:44:00:02:00”;}VF-1 {        mac-addr : “aa:88:44:00:02:01”;        passthrough : true;}`

当我们下次启动系统时，我们会发现 iavf1 不存在，因为 iavf 驱动程序永远不会分配给我们的第二个 VF。相反，它将被标记为“ppt”以进行“PCI 透传”，只有 bhyve 能够使用它。

使用这些调整后，重新启动。

立即您会注意到 dmesg 输出有所不同。这次没有提到 iavf1。还记得我们在 pciconf 中看到的 1:0:17 选择器吗？在这里我们看到它的格式稍有不同。

`ppt0 at device 0.17 on pci1`

pciconf 确认设备已保留用于透传。

`(host) $ pciconf -l | grep iavfiavf0@pci0:1:0:16:      class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x154c subvendor=0x8086 subdevice=0x0000(host) $ pciconf -l | grep pptppt0@pci0:1:0:17:      class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x154c subvendor=0x8086 subdevice=0x0000`

剩下的工作我们在 bhyve 中完成。本文假定您知道如何启动并运行 bhyve 虚拟机。我使用 vm-bhyve 工具来轻松管理虚拟机（但如果您不使用 vm-bhyve，请参考本节末尾的原始 bhyve 参数）。我将将 ppt VF 添加到名为 debian-test 的 Debian 虚拟机中。我们只需在配置中定义要透传的设备，并删除任何涉及虚拟网络的行。

`loader=”grub”cpu=1memory=4Gdisk0_type=”virtio-blk”disk0_name=”disk0.img”uuid=”b997a425-80d3-11ee-a522-00074336bc80”# Passthrough a VF for Networkingpassthru0=”1/0/17”# Common defaults that are not needed with a VF available# network0_type=”virtio-net”# network0_switch=”public”# network0_mac=”58:9c:fc:0c:fd:b7”`

现在我们只需启动我们的 bhyve 虚拟机。

`(host) $ sudo vm start debian-testStarting debian-test  * found guest in /mnt/apps/bhyve/debian-test  * booting...(host) $ sudo vm console debian-testConnecteddebian-test login: rootPassword:Linux debian-test 6.1.0-16-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.67-1 (2023-12-12) x86_64root@debian-test:~# lspci | grep -i intel00:05.0 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series(rev 01)root@debian-test:~# ip addr2: enp0s5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000    link/ether aa:88:44:00:02:01 brd ff:ff:ff:ff:ff:ff    inet 10.0.1.99/24 brd 10.0.1.255 scope global dynamic enp0s5       valid_lft 7186sec preferred_lft 7186sec    inet6 fdd5:c1fa:4193:245:a888:44ff:fe00:201/64 scope global dynamic mngtmpaddr       valid_lft 1795sec preferred_lft 1795sec    inet6 fe80::a888:44ff:fe00:201/64 scope link       valid_lft forever preferred_lft foreverroot@debian-test:~# ping 9.9.9.9PING 9.9.9.9 (9.9.9.9) 56(84) bytes of data.64 bytes from 9.9.9.9: icmp_seq=1 ttl=58 time=20.6 ms64 bytes from 9.9.9.9: icmp_seq=2 ttl=58 time=19.8 ms`

成功！我们现在在我们的 bhyve VM 中拥有了一个用于网络的 SR-IOV VF 设备。如果你是一个纯粹主义者，并且不想使用 vm-bhyve ，当你使用 vm 命令时，细节将被附加到一个 vm-bhyve.log 文件中。在这个文件中，你将看到传递给 grub-bhyve 和 bhyve 启动 VM 的参数。

`create file /mnt/apps/bhyve/debian-test/device.map      -> (hd0) /mnt/apps/bhyve/debian-test/disk0.imggrub-bhyve -c /dev/nmdm-debian-test.1A -S \<br/>      -m /mnt/apps/bhyve/debian-test/device.map \<br/>      -M 4G -r hd0,1 debian-testbhyve -c 1 -m 4G -AHP      -U b997a425-80d3-11ee-a522-00074336bc80 -u -S \<br/>      -s 0,hostbridge -s 31,lpc \<br/>      -s 4:0,virtio-blk,/mnt/apps/bhyve/debian-test/disk0.img \<br/>      -s 5:0,passthru,1/0/17`

### bhyve PCI 透传是一个新兴的功能

虽然在 14.0-RELEASE 版本中，对于 jails 使用 VFs 与 vnet 来说非常稳定，但总体而言，bhyve PCI 透传仍处于快速发展阶段。仅使用透传的 bhyve 运行得非常好。然而，我发现如果我同时使用 VFs 和 jails，某些硬件组合和设备数量可能会产生意外行为。每个版本都会带来改进。如果你发现了边缘案例，请务必提交 bug。

### FreeBSD SR-IOV 总结

要在 FreeBSD 中使用 SR-IOV 启用的虚拟 PCIe 设备，我们：

* 在支持 SR-IOV 的主板上安装 SR-IOV 兼容的网络卡
* 确保主板的 SR-IOV 功能已启用
* 创建 /etc/iov/ixl0.conf 并指定需要多少个 VF
* 在 /etc/rc.conf 中引用 /etc/iov/ixl0.conf 以在启动时保持配置

 这就是它！

为了展示它的工作原理，我们使用 vnet 将一个 VF 分配给 jail。我们还在启动时为 bhyve 虚拟机预分配了另一个 VF 进行透传。在这两种情况下，我们所需做的就是在相应的 jail/VM 配置文件中添加几行代码。

下面的部分将对比 FreeBSD 方法和你在 Linux 发行版中找到的方法，让你感受一下这两种方法的差异。

## Linux 中的 SR-IOV

SR-IOV 在 Linux 中运行非常好。一旦你把它设置好了，你可能很难在 FreeBSD 和 Linux 之间找到明显的区别。然而，设置它可能是一段旅程。

最大的区别在于在 Linux 中没有像 FreeBSD 的 iovctl 这样的标准工具来设置 SR-IOV。有几种方法可以实现一个工作的设置，但它们并不那么明显。我将重点介绍我如何使用 udev 来设置 Mellanox 网卡的 PF 和 VF。

udev 是一个功能强大的工具，可以做很多事情。其中一件事情是在启动时启用 SR-IOV 设备。工具本身非常优秀，但关键在于知道如何准备数据。获取所需的属性可能需要在互联网上进行一些搜索，但一旦拥有了这些属性，生成的 udev 规则就非常简单。

`# DO NOT Probe VFs that will be used for VMsKERNEL==”0000:05:00.0”, SUBSYSTEM==”pci”, ATTRS{vendor}==”0x15b3”, ATTRS{device}==”0x1015”,ATTR{sriov_drivers_autoprobe}=”0”, ATTR{sriov_numvfs}=”4”# DO Probe VFs that will be used for LXDKERNEL==”0000:05:00.1”, SUBSYSTEM==”pci”, ATTRS{vendor}==”0x15b3”, ATTRS{device}==”0x1015”,ATTR{sriov_drivers_autoprobe}=”1”, ATTR{sriov_numvfs}=”16”`

实际上这意味着，“匹配 PCI 设备 0000:05:00.0，其供应商 ID 是 0x15b3，设备 ID 是 0x1015，对于这个设备不尝试自动分配驱动程序，并创建 4 个 VF”（即保留用于直通）。第二条规则类似，但针对另一个 PF，会分配驱动程序，并创建 16 个 VF（即准备用于容器分配）。

取决于你使用的卡和具体的 Linux 发行版，这些可能不是你所需要的所有属性。例如，如果你使用的是 Fedora，可能需要添加 ENV{NM_UNMANAGED}="1" 以避免 NetworkManager 在启动时控制你的 VF。

与 pciconf 类似， lspci 将为我们提供这些规则的大部分所需内容，包括 PCI 地址、供应商和设备 ID。在这个系统中，我们可以看到 Mellanox ConnectX-4 Lx 网卡。

`lspci -nn | grep ConnectX05:00.0 Ethernet controller [0200]: Mellanox Technologies MT27710 Family [ConnectX-4 Lx] [15b3:1015]05:00.1 Ethernet controller [0200]: Mellanox Technologies MT27710 Family [ConnectX-4 Lx] [15b3:1015]`

由 udev 设置的属性在 /sys/bus/pci/devices/0000:05:00.*/ 中可见，还有许多其他内容。列出该目录的内容是查找要告知 udev 的东西的好地方。

`(linux) $ ls -AC /sys/bus/pci/devices/0000:05:00.0/aer_dev_correctable       device            irq               net           resource0                subsystemaer_dev_fatal             dma_mask_bits     link              numa_node     resource0_wc             subsystem_deviceaer_dev_nonfatal          driver            local_cpulist     pools         revision                 subsystem_vendorari_enabled               driver_override   local_cpus        power         rom                      ueventbroken_parity_status      enable            max_link_speed    power_state   sriov_drivers_autoprobe  vendorclass                     firmware_node     max_link_width    ptp           sriov_numvfs             virtfn0config                    hwmon             mlx5_core.eth.0   remove        sriov_offset             virtfn1consistent_dma_mask_bits  infiniband        mlx5_core.rdma.0  rescan        sriov_stride             virtfn2current_link_speed        infiniband_verbs  modalias         reset         sriov_totalvfs           virtfn3current_link_width        iommu             msi_bus           reset_method  sriov_vf_device          vpdd3cold_allowed            iommu_group       msi_irqs          resource      sriov_vf_total_msix`

在那个列表中，我们看到我们的两个 udev 目标， sriov_drivers_autoprobe 和 sriov_numvfs ，我们希望在启动时设置它们。其他所有内容都是做什么用的？你可能需要你最喜欢的搜索引擎来回答这个问题。

通过 udev，我们已经完成了两个主要步骤中的第一个。它有效地“打开”了硬件的 SR-IOV 功能。我们仍然需要为网络使用配置它，这是第二个主要步骤。这在很大程度上取决于我们用来管理网络的工具。例如，如果你使用 systemd-networkd，你可以像这样操作。

`#/etc/systemd/network/21-wired-sriov-p1.network[Match]Name=enp5s0f1np1[SR-IOV]VirtualFunction=0Trust=true[SR-IOV]VirtualFunction=1Trust=true`

幸运的是，对于 systemd-networkd，文档并不那么糟糕，你可以找到大部分你需要的信息。有了这些，我们重新启动服务，虚拟功能就可以使用了。

但并非所有的文档都很好，除了网络软件本身，像 AppArmor 和 selinux 这样的安全覆盖层可能会创建难以检测的阻碍，尽管它们在技术上做了它们应该做的事情，但会让系统感觉不太正常。

作为挫折的一个具体例子，最近我正在使用 Fedora 39 运行几个 LXD 容器。我发现需要在 udev 中设置 ENV{NM_UNMANAGED}="1" 规则，这样 LXD 就可以管理我的 VFs 了。一切都很正常，直到我重启容器几次后。突然间，LXD 开始抱怨说没有 VFs 了。

结果发现，尽管 udev 规则阻止了 NetworkManager 在启动时管理 VFs，但在容器重新启动时，NetworkManager 却在运行时拦截它们并接管了它们的管理。我意识到有些奇怪的事情正在发生，因为 VF 设备名称在重新启动容器后发生了变化。例如，最初的 enp5s0f0np0 可能变成了重新启动后的 physZqHm0g 。

最终，我找到了一种方法告诉 NetworkManager 不要这样做。我必须创建一个关键的配置文件来阻止 LXD 和 NetworkManager 之间的冲突，以防你感兴趣的话，下面是关键的配置文件。

`[keyfile]unmanaged-devices=interface-name:enp5s0f1*,interface-name:phys*`

这只是一个例子。认为一切都运行正常，只能在几天后发现事情实际上正在慢慢自我毁灭，这并不是一个好经历。总的来说，我发现所有的挫折都有相同的根本原因：在 Linux 生态系统中没有现有或新兴的标准配置 SR-IOV 的方法。一旦您克服了一些不那么明显的设置障碍，Linux 中用于网络的 SR-IOV 就可以正常工作。

## 结论

在 FreeBSD 中，SR-IOV 是一等公民。在本文中提到的所有内容，您都可以在操作系统提供的手册页中找到。从一个简单的 apropos(1)查询开始。

`(host) $ apropos “SR-IOV”iovctl(8) - PCI SR-IOV configuration utility`

iovctl 手册将帮助您入门，驱动程序页面将为您的硬件提供具体信息。当事物显而易见且易于找到时，系统管理就不会感觉像一项艰巨的任务。

Linux 发行版在能力上同样强大，但在 SR-IOV 的协调性和系统内文档方面却欠缺。虽然我依赖 Linux 做各种事情，但我真心欣赏 FreeBSD 中配置的组织结构。很容易回到一年前未曾接触过的系统，并迅速理解我所做的。我更喜欢这种方式，而不是通过详细记录与某些圣人发布的使某事情起作用的方式的观点网站上的评论。

与任何事物一样，根据您的需要做出自己的明智选择。

MARK MCBRIDE 在西雅图华盛顿州从事 CAR-T 细胞疗法工作，在个性化医疗的全新领域整合供应链、制造和患者运营解决方案。在空闲时间，他喜欢过度设计他的车库家庭实验室，并为所有当地西雅图体育队加油助威。您可以在 Libera IRC 服务器的 #freebsd 频道中与他联系，也可以通过他个人网站 markmcb.com 上列出的其他方式与他联系，用户名为 @markmcb。
