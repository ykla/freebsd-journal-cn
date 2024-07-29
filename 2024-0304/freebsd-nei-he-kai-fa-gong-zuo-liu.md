# FreeBSD 内核开发工作流

由 NAVDEEP PARHAR

内核像任何其他软件一样，使用典型的克隆、编辑、构建、测试、调试、提交的工作流进行开发。但与用户空间软件不同，内核开发必然涉及重复的重新启动（和锁定和崩溃），如果没有专用的测试系统，会很不方便。本文描述了一种用于内核开发的实际设置，使用虚拟机进行测试和调试。

基于虚拟机的内核开发设置具有多个优点：

* 隔离性。主机系统不会受虚拟机重新启动或崩溃的影响。
* 速度。虚拟机重新启动比裸金属系统快得多。
* 调试性。虚拟机很容易设置用于现场源级内核调试。
* 灵活性。虚拟机可用于构建一个“网络内部的网络”，用于处理网络编码，而无需真实的物理网络，例如在飞机上的笔记本电脑上。
* 可管理性。虚拟机易于创建、重新配置、克隆和移动。

## 概述

用于构建内核的系统还运行测试虚拟机，所有这些虚拟机都连接到一个内部网桥。主机为桥接配置的 IP 网络提供 DHCP、DNS 和其他服务。源代码树和构建产物都位于主机上的一个自包含工作区内，该工作区被导出到这个内部网络。工作区被挂载到测试虚拟机内部，并从那里安装新的测试内核。虚拟机配置了一个额外的串行端口 port 用于远程内核调试，虚拟机内核中的 gdb stub 通过虚拟空模拟电缆连接到主机系统上的 gdb 客户端。

用最简单的方法设置它的方式是使用单个系统，通常是笔记本电脑或工作站，用于一切。这是所有开发环境中最便携的，但系统必须具有足够的资源（CPU、内存和存储）来构建 FreeBSD 源码树和运行虚拟机。

在工作环境中，通常会在实验室里为开发和测试专门使用专用服务器，与开发人员的工作站分开。服务器级别的系统比桌面系统具有更高的规格，更适合构建源代码和运行虚拟机。它们还提供更广泛的 PCIe 扩展插槽，并适用于 PCIe 设备驱动程序的开发。

![](https://freebsdfoundation.org/wp-content/uploads/2024/05/dev_workflow_redrawn.png)

## 配置

本文的其余部分假定有两个系统设置，并使用主机名“desktop”和“builder”来指代它们。源代码的主要副本位于桌面上，用户在那里编辑，然后同步到 builder。其余操作在 builder 上以 root 用户身份进行。builder 在其内部网络中也称为“vmhost”。

`WS=devDWSDIR=~/work/wsWSDIR=/ws`

假定桌面上所有已检出的树都位于共同父目录${DWSDIR}中的它们自己的目录${WS}中。示例使用工作空间‘dev’在‘~user/work/ws’目录中。builder 上的${WSDIR}目录是独立的工作区，包含构建配置文件、源代码树、共享的 obj 目录和用于 gdb 的 sysroot。示例使用 builder 上的位置‘/ws’。

## 桌面设置

### 源树

FreeBSD 源代码可在 https://git.FreeBSD.org/src.git 的 git 存储库中找到。新开发在主分支上进行，并在浸泡期后从主分支合并到最近的稳定分支中。

通过从官方存储库或镜像克隆分支来创建本地工作副本。

`desktop# pkg install gitdesktop$ git ls-remote https://git.freebsd.org/src.git heads/main heads/stable/*desktop$ git clone --single-branch -b main ${REPO} ${DWSDIR}/${WS}desktop$ git clone --single-branch -b main https://git.freebsd.org/src.git ~/work/ws/dev`

### 定制的 KERNCONF 用于开发

每个内核都是从一个纯文本的内核配置文件（KERNCONF）构建的。传统上，内核的标识字符串（即 'uname -i' 的输出）与其配置文件的名称匹配。例如，GENERIC 内核是从名为 GENERIC 的文件构建的。有许多用于调试和诊断的 KERNCONF 选项，在早期开发阶段启用它们是很有用的。主分支中的 GENERIC 配置已经启用了一个合理的子集，适合开发工作。然而，现代编译器似乎会在低优化级别甚至没有优化时，过度优化变量和其他调试信息，有时建议构建一个关闭所有优化的内核。用于此目的的定制 KERNCONF 如下所示，称为 DEBUG0。通常情况下，最简单的方法是包含一个现有的配置，并使用 nooptions/options 和 nomakeoptions/makeoptions 进行调整，而不是从头开始编写一个配置。

DEBUG makeoptions 被添加到内核和模块的编译器标志中。堆栈大小需要增加以容纳未优化代码的较大堆栈占用空间。

`desktop$ cat ${DWSDIR}/${WS}/sys/amd64/conf/DEBUG0desktop$ cat ~/work/ws/dev/sys/amd64/conf/DEBUG0include GENERICident DEBUG0nomakeoptions DEBUGmakeoptions DEBUG=”-g -O0”options KSTACK_PAGES=16`

### 将源代码树传送到构建器

将源代码树复制到构建器，在那里以 root 身份进行构建。记住在桌面上做了更改之后但在构建之前要同步内容。

`desktop# pkg install rsyncdesktop$ rsync -azO --del --no-o --no-g ${DWSDIR}/${DWS} root@builder:${WSDIR}/src/desktop$ rsync -azO --del --no-o --no-g ~/work/ws/dev root@builder:/ws/src/`

## 构建器设置

### 构建配置

在构建器的工作区域创建 make.conf 和 src.conf 文件，而不是修改 /etc 中的全局配置文件。obj 目录也位于工作区域，而不是 /usr/obj。使用元模式进行快速增量重建。元模式需要 filemon。KERNCONF= 列表中的所有内核都将默认构建，第一个将默认安装。始终可以在命令行上提供 KERNCONF 并覆盖默认设置。

`builder# kldload -n filemonbuilder# sysrc kld_list+=”filemon”builder# mkdir -p $WSDIR/src $WSDIR/obj $WSDIR/sysrootbuilder# mkdir -p /ws/src /ws/obj /ws/sysrootbuilder# cat $WSDIR/src/src-env.confbuilder# cat /ws/src/src-env.confMAKEOBJDIRPREFIX?=/ws/objWITH_META_MODE=”YES”`

`builder# cat /ws/src/make.confKERNCONF=DEBUG0 GENERIC-NODEBUGINSTKERNNAME?=dev`

`builder# cat /ws/src/src.confWITHOUT_REPRODUCIBLE_BUILD=”YES”`

### 网络配置

为内部网络找一个未使用的网络/掩码。示例使用 192.168.200.0/24。第一个主机（192.168.200.1）始终是 VM 主机（构建者）。两位数的主机号保留用于已知 VM。三位数 100 的主机号由 DHCP 服务器分配给未知 VM。

1. 创建桥接接口作为连接所有 VM 和 host.  的虚拟交换机。为桥接分配固定的 IP 地址和主机名。 builder# echo ‘192.168.200.1 vmhost’ >> /etc/hostsbuilder# sysrc cloned_interfaces=”bridge0”builder# sysrc ifconfig_bridge0=”inet vmhost/24 up”builder# service netif start bridge0
2. 配置主机执行 IP 转发和 NAT，以使其 VM 可以访问外部网络。这是可选的，只有在 VM 需要访问外部网络时才应执行。在这里的示例中，公共接口是 igb1。 builder# cat /etc/pf.confext_if=”igb1”int_if=”bridge0”set skip on loscrub innat on $ext_if inet from !($ext_if) -> ($ext_if)pass outbuilder# sysrc pf_enable=”YES”builder# sysrc gateway_enable=”YES”
3. 在主机上启动 ntpd。DHCP 服务器将向虚拟机提供自身作为 ntp 服务器。 builder# sysrc ntpd_enable=”YES”builder# service ntpd start
4. DHCP 和 DNS。安装 dnsmasq 并将其配置为内部网络的 DHCP 和 DNS 服务器。 builder# pkg install dnsmasqbuilder# cat /usr/local/etc/dnsmasq.confno-pollinterface=bridge0domain=vmlan,192.168.200.0/24,localhost-record=vmhost,vmhost.vmlan,192.168.200.1synth-domain=vmlan,192.168.200.100,192.168.200.199,anon-vm*dhcp-range=192.168.200.100,192.168.200.199,255.255.255.0dhcp-option=option:domain-search,vmlandhcp-option=option:ntp-server,192.168.200.1dhcp-hostsfile=/ws/vm-dhcp.conf
    将其添加为本地 resolv.conf 中的第一个名称服务器。dnsmasq 解析器将仅为内部网络和构建者的环回接口服务查询。
    `builder# sysrc dnsmasq_enable=”YES”builder# service dnsmasq startbuilder# head /etc/resolv.confsearch corp-net.example.comnameserver 127.0.0.1...`
5. 将整个工作区导出到内部网络。 builder# cat /etc/exportsV4: /ws/ws -ro -mapall=root -network 192.168.200.0/24builder# sysrc nfs_server_enable=”YES”builder# sysrc nfsv4_server_only=”YES”builder# service nfsd start

### vm-bhyve（bhyve 前端）

vm-bhyve 是一个易于使用的 bhyve 前端。

为 VM 识别一个 ZFS 池，并在池中创建一个用于 vm-bhyve 的数据集。在 rc.conf 中指定此池和数据集的名称为 vm_dir。一旦 vm_dir 设置正常，初始化 vm-bhyve。

`builder# kldload -n vmmbuilder# kldload -n nmdmbuilder# sysrc kld_list+=”vmm nmdm”builder# pkg install vm-bhyvebuilder# zfs create rpool/vmbuilder# sysrc vm_dir=”zfs:rpool/vm”builder# vm initbuilder# sysrc vm_enable=”YES”builder# service vm start`

所有的 VM 将使用文本模式下的串行控制台，可通过 tmux 访问。

`builder# pkg install tmuxbuilder# vm set console=tmux`

将先前创建的桥接口添加为 vm-bhyve 交换机。

`builder# vm switch create -t manual -b bridge0 vmlan`

确定新虚拟机的合理默认设置。根据需要编辑 $vm_dir/.templates/default.conf 中的默认模板。至少指定 2 个串行 ports — 一个用于串行控制台，一个用于远程调试。将所有新虚拟机连接到 vmlan 开关。

`builder# vim /rpool/vm/.templates/default.confloader=”uefi”cpu=2memory=2Gcomports=”com1 com2”network0_type=”virtio-net”network0_switch=”vmlan”disk0_size=”20G”disk0_type=”virtio-blk”disk0_name=”disk0.img”`

### 镜像种子

在新虚拟机中运行 FreeBSD 的最简单方法是使用预先安装了它的磁盘映像。虚拟机将启动其默认内核或开发内核，并且其用户空间需要与两者兼容，因此最好在虚拟机中使用与开发树相同版本的 FreeBSD。

FreeBSD.org 提供主要分支和稳定分支的发布和最新快照的磁盘映像。

`# fetch https://download.freebsd.org/releases/VM-IMAGES/14.0-RELEASE/amd64/Latest/FreeBSD-14.0-RELEASE-amd64.raw.xz# fetch https://download.freebsd.org/snapshots/VM-IMAGES/15.0-CURRENT/amd64/Latest/FreeBSD-15.0-CURRENT-amd64.raw.xz# unxz -c FreeBSD-14.0-RELEASE-amd64.raw.xz > seed-14_0.img# unxz -c FreeBSD-15.0-CURRENT-amd64.raw.xz > seed-main.img# du -Ash seed-main.img; du -sh seed-main.img6.0G seed-main.img1.6G seed-main.img`

也可以从源代码树生成磁盘映像。此示例显示如何构建一个带有非调试内核和其他一些节省空间选项的映像。

`# cd /usr/src# make -j1C KERNCONF=GENERIC-NODEBUG buildworld buildkernel# make -j1C -C release WITH_VMIMAGES=1 clean obj# make -j1C -C release WITHOUT_KERNEL_SYMBOLS=1 WITHOUT_DEBUG_FILES=1 \<br/>NOPORTS=1 NOSRC=1 WITH_VMIMAGES=1 VMFORMATS=raw VMSIZE=4g SWAPSIZE=2g \<br/>KERNCONF=GENERIC-NODEBUG vm-image# cp /usr/obj/usr/src/amd64.amd64/release/vm.ufs.raw seed-main.img# du -Ash seed-main.img; du -sh seed-main.img6.0G seed-main.img626M seed-main.img`

修改香草映像，用作内部网络上的测试虚拟机。

从镜像创建内存磁盘，并挂载 UFS 分区。当虚拟机启动时，这将成为预安装操作系统的根分区。

`# mdconfig -af seed-main.imgmd0# gpart show -p md0# mount /dev/md0p4 /mnt`

从 rc.conf 中移除主机名，强制使用 DHCP 服务器提供的主机名。

`# sysrc -R /mnt -x hostname# sysrc -R /mnt -x ifconfig_DEFAULT# sysrc -R /mnt ifconfig_vtnet0=”SYNCDHCP”# sysrc -R /mnt ntpd_enable=”YES”# sysrc -R /mnt ntpd_sync_on_start=”YES”# sysrc -R /mnt kld_list+=”filemon”`

允许 VM 开箱即用的 ssh 访问。注意，这是一个实验室网络内的开发环境，不用担心以 root 身份操作或重用相同的主机密钥。复制主机密钥和 root 的.ssh 到正确的位置。在所有 VM 上使用相同的密钥很方便。更新 sshd 配置以允许 root 登录并启用服务。

`# cp -a .../vm-ssh-hostkeys/ssh_host_*key* /mnt/etc/ssh/# cp -a .../vm-root-dotssh /mnt/root/.ssh# vim /mnt/etc/sshd_configPermitRootLogin yes# sysrc -R /mnt sshd_enable=”YES”`

将第一个串行port配置为潜在控制台，第二个用于远程内核调试。

`# vim /mnt/boot/loader.confkern.msgbuf_show_timestamp=”2”hint.uart.0.flags=”0x10”hint.uart.1.flags=”0x80”`

`Create the mount point for the work area and add an entry in fstab to mount it on boot. /dev/fd and /proc are useful in general.`

`# mkdir -p /mnt/ws# vim /mnt/etc/fstab...fdesc   /dev/fd fdescfs rw      0       0proc    /proc   procfs  rw      0       0vmhost:/ /ws nfs ro,nfsv4 0 0`

完成所有操作。卸载并销毁 MD。

`# umount /mnt# mdconfig -du 0`

seed-main.img 文件已准备就绪。

## 新测试虚拟机

创建一个新的虚拟机并记下其自动生成的 MAC 地址。 更新配置，以便 DHCP 服务向已知的虚拟机提供分配的主机名和 IP 地址。 这些静态分配的地址不得与 dhcp 范围重叠。 本文中的惯例是对已知的虚拟机使用双位主机号，对动态 dhcp 范围使用三位主机号。

创建一个“dhcp-host”条目，其中包含分配给虚拟机的主机名、其 MAC 地址和不属于动态区域的固定 IP。然后重新加载解析器。

`builder# vm create vm0builder# vm info vm0 | grep fixed-mac-addressbuilder# echo ‘vm0,58:9c:fc:03:40:dc,192.168.200.10’ >> /ws/vm-dhcp.confbuilder# service dnsmasq reload`

用镜像的副本替换 disk0.img 文件，并将其大小增加到 VM 所需的磁盘大小。VM 的磁盘镜像可以在 VM 未运行时随时调整大小。首次使用调整大小后的磁盘启动 VM 时运行“service growfs onestart”命令。

`builder# cp seed-main.img /rpool/vm/vm0/disk0.imgbuilder# truncate -s 30G /rpool/vm/vm0/disk0.img`

### 首次启动

在首次启动前审查 VM 的配置。

`builder# vm configure vm0`

使用控制台前台启动 VM，或者在后台启动然后附加到其控制台。控制台只是一个名为 VM 的 tmux 会话。

`builder# vm start -i vm0builder# vm start vm0builder# vm console vm0`

首次启动 VM 时，请验证以下内容：

* VM 的主机名是由 DHCP 服务器分配的。主机名和 tty 可以在登录提示符中看到。 FreeBSD/amd64 (vm0) (ttyu0)login:
* VM 的 uart0 是控制台，uart1 用于远程调试。 vm0# dmesg | grep uart[1.002244] uart0: console (115200,n,8,1)...[1.002252] uart1: debug port (115200,n,8,1)
* 工作区已挂载到预期位置。 vm0# mount | grep nfsvmhost:/ on /ws (nfs, read-only, nfsv4acls)vm0# ls /ws...
* 从主机和桌面（使用 VM 主机作为跳板主机）可以通过 ssh 访问 VM。 builder# ssh root@vm0desktop$ ssh -J root@builder root@vm0

## 在虚拟机中进行 PCIe 设备驱动程序开发

PCI 透传允许主机将 PCIe 设备导出（透传）到虚拟机，使其可以直接访问 PCIe 设备。这使得在虚拟机内部对真实 PCIe 硬件进行设备驱动程序开发成为可能。

设备由主机上的 ppt 驱动程序占用，并在虚拟机内部显示为连接到虚拟机的 PCIe 根复杂。系统上的 PCIe 设备通过 BSF（或 BDF）3 元组进行标识，在虚拟机内部可能会有所不同。

使用 pciconf 或 vm-bhye 获取系统上 PCIe 设备的列表，并记录要传递的设备的 BSF 元组。请注意，pciconf 选择器以冒号分隔的 BSF 结束，而 bhyve/vmm/ppt 使用斜杠分隔的 B/S/F 来标识设备。例如，带有选择器 “none193@pci0:136:0:4” 的 PCIe 设备在 bhyve/ppt 符号中为 “136/0/4”。

`builder# pciconf -llbuilder# vm passthru`

让 ppt 驱动程序声明将要传递的设备。这样可以防止正常的驱动程序连接到设备。

`builder# vim /boot/loader.confpptdevs=”136/0/4 137/0/4”`

重新启动以使 loader.conf 的更改生效，或者尝试在系统运行时将设备从其驱动程序分离并附加到 ppt。

`builder# devctl detach pci0:136:0:4builder# devctl clear driver pci0:136:0:4builder# devctl set driver pci0:136:0:4 ppt(repeat for 137)`

验证 ppt 驱动已连接到设备，并且 vm-bhyve 已准备好使用它们。

`builder# pciconf -ll | grep pptppt0@pci0:136:0:4:      020000   00   00   1425   640d   1425   0000ppt1@pci0:137:0:4:      020000   00   00   1425   640d   1425   0000builder# vm passthru | awk ‘NR == 1 || $3 != “No” {print}’DEVICE     BHYVE ID     READY        DESCRIPTIONppt0       136/0/4      Yes         T62100-CR Unified Wire Ethernet Controllerppt1       137/0/4      Yes          T62100-CR Unified Wire Ethernet Controller`

重新配置测试虚拟机，并列出应通过传递给该虚拟机的设备。

`builder# vm configure vm0passthru0=”136/0/4”passthru1=”137/0/4”`

启动测试虚拟机，并验证 PCIe 设备是否可见。请注意，虚拟机中的 BSF 与主机上实际硬件 BSF 不同。

`vm0# pciconf -ll...none0@pci0:0:6:0:       020000   00   00   1425   640d   1425   0000none1@pci0:0:7:0:       020000   00   00   1425   640d   1425   0000...`

## 主要工作流程循环（编辑、构建、安装、测试、重复）

### 编辑

在桌面上编辑源代码树并发送给构建者。

`desktop$ cd ~/work/ws/devdesktop$ gvim sys/foo/bar.c...desktop$ rsync -azO --del --no-o --no-g ~/work/ws/dev root@builder:/ws/src/`

### 构建

`builder# alias wsmake=’__MAKE_CONF=${WSDIR}/src/make.conf SRC_ENV_CONF=${WSDIR}/src/src-env.conf SRCCONF=${WSDIR}/src/src.conf make -j1C’builder# alias wsmake=’__MAKE_CONF=/ws/src/make.conf SRC_ENV_CONF=/ws/src/src-env.conf SRCCONF=/ws/src/src.conf make -j1C’builder# cd ${WSDIR}/src/${WS}builder# cd /ws/src/devbuilder# wsmake kernel-toolchain (one time)builder# wsmake buildkernel`

### 安装

1. 在虚拟机中安装内核。INSTKERNNAME 在 make.conf 中设置，因此 /boot/${INSTKERNNAME} 中的测试内核不会干扰 /boot/kernel 中的标准内核，后者是出现测试内核问题时的安全回退。也可以在命令行上显式指定。 vm0# alias wsmake=’__MAKE_CONF=${WSDIR}/src/make.conf SRC_ENV_CONF=${WSDIR}/src/src-env.conf SRCCONF=${WSDIR}/src/src.conf make -j1C’vm0# alias wsmake=’__MAKE_CONF=/ws/src/make.conf SRC_ENV_CONF=/ws/src/src-env.conf SRCCONF=/ws/src/src.conf make -j1C’vm0# cd ${WSDIR}/src/${WS}vm0# cd /ws/src/devvm0# wsmake installkernel
2. 如果在构建者上使用 gdb 进行源级调试，则也需将其安装到构建者的系统根目录。使用与虚拟机中相同的 INSTKERNNAME 和 KERNCONF。 builder# cd /ws/src/devbuilder# wsmake installkernel DESTDIR=/ws/sysroot

### 测试

仅为下次重启选择测试内核，或永久选择。

`vm0# nextboot -k ${WS}vm0# nextboot -k devvm0# shutdown -r nowvm0# sysrc -f /boot/loader.conf kernel=”${WS}”vm0# sysrc -f /boot/loader.conf kernel=”dev”vm0# shutdown -r now`

使用调试 KERNCONF（例如，前面显示的自定义 DEBUG0 或主要的 GENERIC）进行初始测试，并在之后切换到发布内核（例如，主要的 GENERIC-NODEBUG）是一个好习惯。

## 调试测试内核

验证测试内核当前是否正在运行。

`vm0# uname -iDEBUG0vm0# sysctl kern.bootfilekern.bootfile: /boot/dev/kernel`

 后端

有两个调试器后端可供选择，并且当前后端可以实时切换。

`vm0# sysctl debug.kdb.availablevm0# sysctl debug.kdb.currentvm0# sysctl debug.kdb.current=ddbvm0# sysctl debug.kdb.current=gdb`

### 进入调试器

1. 在系统崩溃时自动进入调试器，而不是重新启动内核。 vm0# sysctl debug.debugger_on_panic
2. 从虚拟机内部手动进行。 vm0# sysctl debug.kdb.enter=1
3. 从虚拟机主机手动进行。如果虚拟机被锁死且无响应，向其注入 NMI。 builder# bhyvectl --vm=vm0 --inject-nmi

### 使用 gdb 进行源代码级调试

源代码级调试需要源代码、二进制文件和调试文件，这些文件在主机和虚拟机上都可以找到，但位置不同。

### 实时远程调试

确保调试后端设置为 gdb。如果虚拟机已经使用 ddb 后端进入调试器，请交互式切换到 gdb 后端。

`vm0# sysctl debug.kdb.current=gdbdb> gdb`

内核中的远程 gdb 存根在内核进入调试器时是活动的。从主机连接到 gdb 存根。连接通过连接到虚拟机的第二个串口（VM 内部的 uart1）上的虚拟空调制模电缆进行。

`builder# gdb -iex ‘set sysroot ${WSDIR}/sysroot’ -ex ‘target remote /dev/nmdm-${VM}.2B’ ${WSDIR}/sysroot/boot/${INSTKERNNAME}/kernelbuilder# gdb -iex ‘set sysroot /ws/sysroot’ -ex ‘target remote /dev/nmdm-vm0.2B’ /ws/sysroot/boot/dev/kernel`

### 核心转储分析

与实时调试相同，唯一不同之处是目标是一个 vmcore，而不是远程。

`builder# gdb -iex ‘set sysroot ${WSDIR}/sysroot’ -ex ‘target vmcore ${VMCORE}’ ${WSDIR}/sysroot/boot/${INSTKERNNAME}/kernelbuilder# scp root@vm0:/var/crash/vmcore.0 /ws/tmp/builder# gdb -iex ‘set sysroot /ws/sysroot’ -ex ‘target vmcore /ws/tmp/vmcore.0’ /ws/sysroot/boot/dev/kernel`

NAVDEEP PARHAR 已经是 FreeBSD 用户 20 年以上，并自 2009 年起是 FreeBSD 开发人员。他受雇于 Chelsio Communications 公司，负责 Chelsio Terminator 系列 NIC 的 FreeBSD 软件开发。他是 cxgbe(4)驱动程序的作者和维护者，他感兴趣的领域包括网络堆栈、设备驱动程序、通用内核调试和分析。
