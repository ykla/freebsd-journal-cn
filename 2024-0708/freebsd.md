# 嵌入式 FreeBSD：打造自己的镜像

- 原文链接：[Rolling Your Own Images](https://freebsdfoundation.org/our-work/journal/browser-based-edition/storage-and-filesystems/embedded-freebsd-rolling-your-own-images/)
- 作者：Christopher R. Bowman

在上一列中，我稍微介绍了我使用的开发板，Digilent 的 [ARTYZ7](https://digilent.com/shop/zedboard-zynq-7000-arm-fpga-soc-development-board/)。在这一列中，我将讨论如何制作自己的镜像。到某个时候，你可能会需要一个与现有镜像略有不同的镜像，因此你将希望从源代码开始构建并创建一个镜像写入 SD 卡。

首先，让我们了解一下 [ARTYZ7](https://digilent.com/shop/zedboard-zynq-7000-arm-fpga-soc-development-board/) 如何启动。 [Zynq-7000 SoC 技术参考手册](https://docs.xilinx.com/v/u/en-US/ug585-Zynq-7000-TRM) 是一个宝贵的技术信息来源，第 6 章讲述了芯片及所有 Zynq 板的启动过程。这里有很多技术细节，但根据默认配置的跳线设置，ARTYZ7 是从 SD 卡启动的。你可能已经下载并查看了我在上一列中提供的镜像。如果查看过，你会发现它是以 MBR 模式格式化的，首先是一个 FAT 分区，第二个 MBR 分区上是一个带有 UFS 的 FreeBSD 切片。处理器将查找 MBR，并寻找 FAT16 或 FAT32 分区。它会在 FAT 分区中查找一个名为 `boot.bin` 的文件。如果找到，它会将其加载到内存中并开始执行。如果你想要直接编程到裸机，可以将你的应用程序写入并命名为 `boot.bin`。对我来说这有些复杂。相反，当启动 FreeBSD 时，我们使用了一个第一阶段引导加载程序，像许多嵌入式板一样，我们使用 [Das U-Boot](https://en.wikipedia.org/wiki/Das_U-Boot)。U-Boot 是一个由社区维护的开源引导加载程序。在我们的用例中，U-Boot 使用了两次。首先，U-Boot 被编译为最小的第一阶段引导加载程序（FSBL），它设置硬件并查找第二阶段引导加载程序，第二阶段引导加载程序也是 U-Boot，但功能更加丰富。在我们的使用中，第二阶段的 U-Boot 位于 FAT 分区中的名为 `U-boot.img` 的文件中。这个 U-Boot 第二阶段加载程序会从 FAT 分区中的 `EFI/BOOT/bootarm.efi` 文件加载 lua 加载器。然后，lua 加载器会将内核等文件加载到内存中并执行。

所以，如果我们要构建镜像，我们需要创建分区并将 U-Boot 和 FreeBSD 安装到 SD 卡上。

构建 U-Boot 相对直接。虽然有许多 U-Boot 的移植版本用于各种板子，但 ARTYZ7 并没有现成的移植版本。我已经创建了一个移植版本，你可以在 [这里](http://www.chrisbowman.com/crb/ArtyZ7/u-boot_ports/patches.html) 找到。虽然我还没有把它纳入 FreeBSD 的 ports 树中，但你可以直接将其放入一个最新的 ports 树中的 `/usr/ports/sysutils` 目录。 [FreeBSD 手册第 4.5 章](https://docs.freebsd.org/en/books/handbook/ports/#ports-using) 提供了非常详细的安装 ports 和构建 ports 的说明。一旦你将该 port 添加到你的 ports 树中，简单地在 `sysinstall/u-boot-artyz7` 目录下运行 `make`，应该就能自动下载并构建 U-Boot。运行 `make install` 后，`boot.bin` 和 `U-boot.img` 文件应该会出现在 `/usr/local/share/U-boot/U-boot-artyz7` 目录下。注意：我曾经可以使用较高的 “`-j`” 值来通过 `make` 使用多个核心进行构建，但在最新的 ports 树（2024Q3）中，这似乎无法正常工作。

从源代码构建也在 FreeBSD 手册中有很好的文档。在 [第 26.6 章：从源代码更新 FreeBSD](https://docs.freebsd.org/en/books/handbook/cutting-edge/#makeworld) 中，你将找到有关下载 FreeBSD 源代码并从中构建的详细信息。如果你已经在 `/usr/src` 安装了 FreeBSD 源代码，你可以直接进入该目录并运行以下命令：

```sh
# make buildworld
# make buildkernel KERNCONF=ARTYZ7
```

虽然这些命令应该能在 ARTYZ7 板上正常工作，毕竟它有完整的 FreeBSD 安装，但你应该准备好它会花费很长时间。由于 PC 硬件变得非常强大且价格低廉，我使用一台 AMD64 系统来托管所有文件、开发环境，并进行所有构建。FreeBSD 对交叉编译和构建提供了内置支持。在我的 PC 上，我使用以下命令让我的 PC 为基于 ARM 的 ARTYZ7 板进行源代码构建：

```sh
# make buildworld TARGET=arm TARGET_ARCH=armv7 -j32
# make buildkernel KERNCONF=ARTYZ7 TARGET=arm \ TARGET_ARCH=armv7 -j32
```

这将从 AMD64 构建一个交叉编译器到 ARMv7，并使用这个编译器来构建所有内容。

对于内核配置文件，我已将 `src/sys/arm/conf/ZEDBOARD` 中的 ZEDBOARD 配置文件复制到 `src/sys/arm/conf/ARTYZ7` 并在其中修改了名称以匹配。`-j32` 开关使得编译过程可以使用最多 32 个进程来进行构建。在一台配备快速 SSD 的 AMD 5950x PC 上，构建世界大约需要 10 分钟，构建内核大约需要 60 秒。我无法想象在 ARTYZ7 上需要多少天来完成这个过程。


一旦你从源代码构建了所有内容，过程可能会有所不同，具体有以下几种方式：

1. 你可以将 SD 卡挂载到主机开发系统，并直接从主机安装到 SD 卡上。
2. 你可以使用 [`mdconfig`](https://man.freebsd.org/cgi/man.cgi?query=mdconfig&apropos=0&sektion=0&manpath=FreeBSD+13.2-RELEASE+and+Ports&arch=default&format=html) 命令创建一个基于文件的内存设备。这使你可以将一个文件当作块设备来处理。你可以使用所有标准的 FreeBSD 工具来分区设备，并将分区挂载到文件系统中。从那里，你可以像操作物理设备一样进行安装，最终得到一个适合使用 [`dd`](https://man.freebsd.org/cgi/man.cgi?query=dd&apropos=0&sektion=0&manpath=FreeBSD+13.2-RELEASE+and+Ports&arch=default&format=html) 复制到 SD 卡或提供给其他人的文件。
3. 最后一种方法，也是我正在使用并将在这里讨论的方法，是先在主机 PC 上的某个目录中进行安装，然后使用 [`mkimg`](https://man.freebsd.org/cgi/man.cgi?query=mkimg&apropos=0&sektion=0&manpath=FreeBSD+13.2-RELEASE+and+Ports&arch=default&format=html) 和 [`makefs`](https://man.freebsd.org/cgi/man.cgi?query=makefs&apropos=0&sektion=0&manpath=FreeBSD+13.2-RELEASE+and+Ports&arch=default&format=html) 从主机目录构建一个镜像，该镜像同样适合使用 [`dd`](https://man.freebsd.org/cgi/man.cgi?query=dd&apropos=0&sektion=0&manpath=FreeBSD+13.2-RELEASE+and+Ports&arch=default&format=html) 复制到 SD 卡上。

接下来，我们更详细地看看方法 3。我通常会创建一个 `msdos` 目录和一个 `ufs` 目录，来表示我需要在 SD 卡上的两个分区：

```sh
# mkdir msdos ufs
```

接下来，我在 `ufs` 目录上进行安装：

```sh
# make installworld installkernel TARGET=arm \
TARGET_ARCH=armv7 -j32 DESTDIR=ufs
```

你还需要运行分发目标，该目标会在 `/etc` 中创建所有默认的配置文件。当你对一个工作系统进行源代码升级时，通常不会运行这个命令，因为它会覆盖你的配置文件，但在从零开始构建系统时，你确实需要默认的配置文件：

```sh
# make distribution TARGET=arm \
TARGET_ARCH=armv7 DESTDIR=ufs -j32
```

此时，我已经在 `ufs` 目录下完成了完整的安装，并可以对镜像进行任何自定义。例如，我可以定制 `ufs/etc/rc.conf`，以便自动启动以 DHCP 获取 IP 地址的以太网接口 `cgem0`。我可以将 `ssh` 密钥安装到 `ufs/etc/ssh` 中，这样系统每次启动时都会使用相同的 `ssh` 密钥，而不是在第一次启动时生成新的密钥。由于我使用主机系统来进行所有构建并托管我的文件，我还喜欢配置 `/etc/fstab` 以通过 NFS 挂载我的主目录。

我发现创建一个用户账户非常方便，这样我可以立即登录：

```sh
# echo 'xxxx' | pw -R ${ufs} useradd -n crb -m -u 1001 \
-d /homes/crb -g crb -G 1001,wheel,operator\
-c “Christopher R. Bowman” -s /bin/tcsh -H 0
```

`pw` 命令的 `-R` 选项使得 `pw` 会编辑 `ufs/etc` 中的密码文件，而不是我主机系统中的文件。`-H0` 选项允许我使用 `echo` 将密码通过管道传输给 `pw`，而不需要交互式输入（你需要使用你主机系统中的编码密码替代 `xxxx`）。你也可以觉得修改 root 账户的密码更加方便，避免没有密码的情况。

现在，既然我已经按照希望的方式定制了 `ufs` 目录，我们就来关注一下 FAT 分区。

我需要安装 `boot.bin` 和 `U-boot.img`：

```sh
# cp /usr/local/share/U-boot/U-boot-artyz7/boot.bin msdos
# cp /usr/local/share/U-boot/U-boot-artyz7/U-boot.img msdos
```

`boot.bin` 会加载 `U-boot.img`，而 `U-boot.img` 模拟了 FreeBSD 加载器的 EFI 固件。EFI 系统会在 FAT 分区上查找一个名为 `EFI/BOOT/bootarm.efi` 的文件，因此我们需要将 FreeBSD 的 lua 加载器复制到该位置：

```sh
# mkdir -p msdos/ EFI/BOOT
# cp ufs/boot/loader_lua.efi msdos/EFI/BOOT/bootarm.efi
```

注意，我复制的是我们构建的 ARM 版本，而不是主机版本（后者是 AMD64 代码）。

现在，我们已经有了两个目录 `msdos` 和 `ufs`，它们包含了我们想要写入 SD 卡的文件。接下来我们只需要创建镜像文件。这是一个 4 步的过程：

```sh
makefs -t msdos \
  -o fat_type=16 \
  -o sectors_per_cluster=1 \
  -o volume_label=EFISYS \
  -s 32m \
  efi.part msdos

makefs -B little \
  -o label=rootfs \
  -o version=2 \
  -o softupdates=1 \
  -s 3g \
  rootfs.ufs ufs

mkimg -s bsd \
-p freebsd-ufs:=rootfs.ufs \
  -p freebsd-swap::1G \
  -o freebsd.part

mkimg -s mbr -f raw -a 1\
-p fat16b:=efi.part \
  -p freebsd:=freebsd.part \
  -o selfbuilt.img
```

第一个命令从 `msdos` 目录构建文件系统镜像文件 (`efi.part`)。第二个命令从 `ufs` 目录构建文件系统镜像文件 (`rootfs.ufs`)。第三个命令将 `rootfs.ufs` 文件组合成一个包含 1GB 交换分区的 FreeBSD 切片。最后一个命令将我们的 `efi.part` 和 `freebsd.part` 文件打包成一个单一的镜像文件 (`selfbuilt.img`)。

如果我将 SD 卡插入主机，我会看到一个 `/dev/da0` 设备，并使用简单的 [`dd`](https://man.freebsd.org/cgi/man.cgi?query=dd&apropos=0&sektion=0&manpath=FreeBSD+13.2-RELEASE+and+Ports&arch=default&format=html) 命令将镜像复制到 SD 卡：

```sh
# dd if=selfbuilt.img of=/dev/da0 bs=1m status=progress
```

到此为止，剩下的工作就是将 SD 卡插入 ARTYZ7 板卡，按下重置按钮，观察精彩的启动过程。由于我在 `ufs` 分区中的配置，我能够在启动完成后立即通过 `ssh` 登录到我的板卡，并且我的主目录已经通过 NFS 挂载。此时，世界征服就可以开始了！或者喝一杯啤酒——总是该喝啤酒的时候。

---

**Christopher R. Bowman** 自 1989 年开始使用 BSD，当时他在约翰·霍普金斯大学应用物理实验室的地下二层工作。后来，在90年代中期，他在马里兰大学使用 FreeBSD 设计了自己的第一个 2 微米 CMOS 芯片。从那时起，他一直是 FreeBSD 用户，并对硬件设计及其驱动软件非常感兴趣。他在过去 20 年里一直从事半导体设计自动化行业工作。
