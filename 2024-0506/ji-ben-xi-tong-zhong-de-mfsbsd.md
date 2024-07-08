# 基本系统中的 mfsBSD

 由 SOOBIN RHO

mfsBSD 是一个内存中的 FreeBSD。

mfsBSD 的不同之处在于它完全在内存中运行 FreeBSD 的实例 — 因此有 mfs（内存文件系统）之称。当然，这意味着我们可以在使用 mfsBSD 时，完全不会影响到现有的驱动器。例如，我们可以将其用于云端或本地服务器的故障排除¹。在邮件列表中搜索 mfsBSD，看看人们是如何解决各种问题的，实在是非常有趣。我个人最喜欢的应用场景之一是在仅有单个可用驱动器的系统中安装 FreeBSD，如果因某种原因无法使用 FreeBSD 安装媒体，我会首先制作一个 mfsBSD 镜像；将镜像安装到驱动器上；启动 mfsBSD；然后运行 bsdinstall。这里有一个例子：

 首先，构建 mfsBSD：

`# The patch set that integrates mfsBSD into base is under review and is available at:# https://reviews.freebsd.org/D41705cd /usr/src/releasemake mfsbsd-se.img# se here refers to mfsBSD special edition, which comes packed with# dist files - base.txz and kernel.txz - which are required for bsdinstall.cd /usr/obj/usr/src/${ARCH}/release/ls -lh`

然后，将 mfsBSD 写入驱动器：

`# Install the mfsBSD image to your target drive.# Replace ada0 with your target drive.dd if=./mfsbsd-se.img of=/dev/ada0 bs=1Mreboot`

启动到 mfsBSD，然后执行 bsdinstall ：

`# Copy the special edition dist files so that bsdinstall can use them for installation.mkdir /mnt/distmount /dev/ada0p3 /mnt/distmkdir /usr/freebsd-distcp /mnt/dist/<version>/*.txz /usr/freebsd-dist/bsdinstall`

mfsBSD 能够将整个 FreeBSD 系统加载到内存中。加载完成后，可以完全修改原始磁盘，因为所有 mfsBSD 文件现在都在内存中运行。正如 Matuška 在他 2009 年的白皮书中描述的那样，“mfsBSD 是一个工具集，用于创建基于 mfsroot 的 FreeBSD 小型但功能齐全的分发，它将所有文件存储在内存中。”²

## mfsBSD 简要历史

Martin Matuška 编写了 mfsBSD。回顾该存储库的提交历史，他在 2007 年 11 月 11 日进行了首次提交，大约是 FreeBSD 7.0 BETA 发布的时候。“这个项目[mfsBSD]基于 Depenguinator 项目的理念”，Depenguinator 是 Colin Percival 在 2003 年创建的一个项目，旨在为仅提供 Linux 发行版的专用服务器远程安装 FreeBSD³。Matuška 希望为 FreeBSD 6.x 提供 Depenguinator 的功能，这就是 mfsBSD 的起源。

自此以来，Matuška 维护 https://mfsbsd.vx.sk/ 来分发 mfsBSD 的镜像，随着其流行度的增加，已经在 GitHub 上维护了十七年，一路修复了无数个 bug，并添加了对 zfsinstall 的支持， /usr 的压缩包等。

在 2023 年 5 月，这篇文章写作前一年，一个 Google 暑期代码项目开始将 mfsBSD 集成到基本系统。

## Google 暑期代码

如何一切开始？我正在阅读黑客新闻（可能当时在拖延我的大学作业）。这是我第一次了解到 Google 夏季编程活动（GSoC）。那里的一条热门评论说 FreeBSD 是参与的组织之一，我发现有趣的是评论中没有提到其他任何内容，仿佛其他一切都不言而喻。

我立刻被吸引住了。关于 FreeBSD 最令我惊讶的是 macOS 是 FreeBSD 的衍生版本，而且 Netflix 也在其 CDN 中使用 FreeBSD。GSoC 的申请过程包括提交项目提案。强烈建议申请者从各组织的项目创意列表中找到一个项目主意（除非您有自己想要追求的想法）。我看了看列表，mfsBSD 项目对我来说最有趣，因为其他项目创意似乎比我能接受的内核开发更接近。

给我的导师们发了封电子邮件后，我收到了 Joseph Mingrone 和 Juraj Lutter 的回复；进行了简短的 Zoom 通话；几周后我收到了 GSoC 的录取通知。之后，我们有了所谓的社区绑定期，在这期间，所有贡献者和导师们聚集在一起，进行了半小时的虚拟会议介绍自己。那是 2023 年 5 月 12 日，星期五。

## 基本系统的 mfsBSD

三个月零二十二天后，将 mfsBSD 集成到基本系统的项目终于完成了，经过了很多来回调试（很多 Bug 是通过谷歌搜索和翻阅所有过去的 GitHub 问题来解决的），以及测试（使用我的两台笔记本电脑 Thinkpad T440 和 P17 运行 shell 脚本），以及我向导师提出太多问题。一套三个补丁被提交到 Phabricator。

基本上，第一个提交“mfsBSD: 供应商导入 mfsBSD”将 mfsBSD 导入为 contrib/mfsbsd 。主要提交“release: 将 mfsBSD 镜像构建目标集成到发行工具集中”在 release/Makefile 中添加了 mfsbsd-se.img 和 mfsbsd-se.iso 目标作为 release/Makefile.mfsbsd 。最后一次提交“release(7): 为新的 mfsBSD 构建目标添加条目”在 share/man/man7/release.7 上添加了相应的条目。这意味着我们现在可以在构建所有 FreeBSD 安装媒体（如 cdrom、dvdrom、memstick 和 mini-memstick）时使用的相同发行 Makefile 中构建 mfsBSD 作为 make release WITH_MFSBSD=1 。

现在，补丁集正在审核中。mfsBSD 之前存在于 FreeBSD 发布工具链之外，仅生成发布版本。我设想的是将这个补丁集作为基本系统的一部分提供 mfsBSD 镜像，并且通过调用 cd /usr/src/release && make release WITH_MFSBSD=1 来构建定制的 mfsBSD 镜像，然后在 /usr/obj/usr/src/${ARCH}/release/ 处创建 mfsbsd-se.img 和 mfsbsd-se.iso 。

### 参考文献

1. Matuška, Martin. (2022). FreeBSD 在 Hetzner 专用服务器上 — VX Weblog。[在线] 可获取于: https://blog.vx.sk/archives/353
2. 马丁·马图什卡（2009 年）。 mfsBSD 工具集，用于创建基于内存文件系统的 FreeBSD 发行版。[在线] 可在以下网址找到：https://people.freebsd.org/~mm/mfsbsd/mfsbsd.pdf
3. Colin Percival（2003 年）。Depenguinator — FreeBSD 远程安装。[在线] 可在以下网址找到：http://www.daemonology.net/depenguinator/
4. 马丁·马图什卡（2024 年）。mmatuska/mfsbsd。[在线] GitHub。 可在以下网址找到：https://github.com/mmatuska/mfsbsd
5. [https://reviews.freebsd.org/D41705](https://reviews.freebsd.org/D41705)

SOOBIN RHO 是南达科他州奥古斯塔纳大学的大四学生。他出生于韩国，但在迪拜长大，最终选择在美国上大学，并且现在是班科普银行的网络安全部门的兼职员工。毕业后，他将成为信息安全分析师。自从 2023 年 Google 暑期代码活动以来，他一直是 FreeBSD 的贡献者。
