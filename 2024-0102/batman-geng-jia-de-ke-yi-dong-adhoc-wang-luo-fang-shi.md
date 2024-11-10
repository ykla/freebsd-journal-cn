# BATMAN：更优的可移动热点网络方式

- 原文链接：[BATMAN: the Better Approach to Mobile Ad-hoc Networks](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/batman-the-better-approach-to-mobile-ad-hoc-networks/)
- 作者：Aymeric Wibo

在广阔的网络协议领域，有一个协议脱颖而出，作为一个多功能且坚韧的竞争者：BATMAN，即 Better Approach to Mobile Ad-hoc Networks。通过在大城市的无线电波中，BATMAN 允许设备在网状网络中无缝通信，而无需任何一台设备了解整个网络拓扑。

在这个夏天，我参加了谷歌编程之夏 (GSoC)，并将 `batman-adv` 内核模块（提供对 Linux 上 BATMAN 协议的支持）移植到 FreeBSD 上。谷歌编程之夏是一个学生在暑期参与开源项目的计划，谷歌为学生提供津贴，并由导师监督。在我的情况下，我的导师是那位唯一无二的 Mahdi（或者 Mehdi，取决于你问他是星期几）Mokhtari，aka mmokhi@。他是个非常有风度的人，我非常感激在 GSoC 的旅程中有他的指导！

目前，使用 `batman_adv`（FreeBSD 上等同于 `batman-adv`）可以创建并参与网状网络，并通过以太网发送/接收数据包。所有这些都能在 Linuxulator 中正常工作。移植的工作主要集中在 LinuxKPI 上（特别是将 `struct net_device` 与 `struct ifnet` 兼容，`mbuf` 与 `sk_buff` 兼容），希望这能在未来简化将其他与网络相关的驱动程序从 Linux 移植到 FreeBSD 的过程。

## 移植过程是什么样的？

虽然我远不是内核组件移植方面的专家（这是我第一次移植像 BATMAN 这样的大型项目），但下面是我对这个过程的一个高层次概述——希望从我的小冒险中可以获得一些洞见。

第一步——自然而然——是将 Linux 上的 `batman-adv` 代码拉入 `sys/contrib/dev/batman_adv`，并创建一个 Makefile 来构建它，Makefile 中列出了所有源文件，并设置了编译选项，例如告诉它包括 LinuxKPI 的内容。正如预期的那样，第一次尝试并没有成功；`batman-adv` 调用了很多函数，并使用了许多仅在 Linux 内核上下文中存在或才有意义的结构体。这正是 LinuxKPI 存在的原因；它提供了一个兼容层，通过调用 FreeBSD 内核中等效（或者有时并不完全等效）的函数来实现 Linux 内核的子集。

因此，下一步自然就是通过为 LinuxKPI 中所有缺失的函数和结构体编写存根（stub）来使其能够编译，填充缺失的字段（我们不能直接复制 Linux 中的结构体，因为 Linux 是 GPL 许可证的）。这些存根只是包含调试打印语句，以便我们知道它们何时被调用。

在一切编译通过后，我可以加载内核模块，但这立即导致了内核崩溃。接下来的过程就是逐一检查所有调用的存根，查找并理解它们在 Linux 中的实现，并在 LinuxKPI 中实现一个等效的版本，直到内核不再崩溃为止。这大概占了工作量的 70%。

一旦每个操作（加载模块、创建接口、发送数据等）都能正常工作且不再崩溃，就该拉上窗帘，在第二台显示器上打开 Wireshark，锁在我的 kot（= = 比利时宿舍房间）里一周，确保 a) 所有设备在网状网络中能够互相识别，b) 数据能够从设备 A 通过设备 B 正常传输到设备 C，不会在过程中丢失或被破坏。大部分时间花在了重新审视之前步骤中（有时不充分的）实现上。这可能占了 30% 的工作量，但感觉像是 90%，大部分时间都在盯着 Wireshark，Wireshark 也在盯着我。

最后，BATMAN 在 FreeBSD 上工作了，剩下的就是让用户态工具支持操作 BATMAN 接口，写几行文档，然后打开窗帘。

## 上游合并 `batman_adv`

去年在 EuroBSDCon 上，我与 core@ 的一些成员讨论了将 `batman_adv` 上游合并的可能性，他们表示不太愿意这样做，因为 BATMAN 是 GPL 许可证的。所以它可能会永远作为一个端口存在，因为 BATMAN 在 FreeBSD 系统中并不是必需的；如果你需要使用 BATMAN 来做某些事情，你很可能处于一个可以轻松自行获取并构建端口的环境中（与 NIC 驱动程序等情况不同）。

## 还有什么需要做的？

目前 BATMAN 在 FreeBSD 上的最大限制是，它不能参与无线网络。我完全打算在接下来的一年内让无线网络也能够正常工作，因为那才是 BATMAN 的主要使用场景。希望能在 BSDCan 2024 之前完成这项工作 ??

我强烈推荐符合申请条件的任何人参加谷歌编程之夏。我从一开始对 FreeBSD 的网络栈和内核了解不多，到现在，在大局上仍然了解不多，但肯定比我刚开始时知道的要多得多。它尤其帮助我更加自如地在源代码树中导航，并调试内核崩溃，即使这些崩溃与网络代码无关。

## 进一步阅读

这是我的 GSoC 项目 Wiki 页面，包含了所有的具体内容、代码和一个小的演示视频：[https://wiki.freebsd.org/SummerOfCode2023Projects/CallingTheBatmanFreeNetworksOnFreeBSD](https://wiki.freebsd.org/SummerOfCode2023Projects/CallingTheBatmanFreeNetworksOnFreeBSD)

此外，你可能会对这个链接感兴趣，它详细介绍了 BATMAN 在 Linux 上的实现（因此也包括在 FreeBSD 上的实现）：[https://www.open-mesh.org/projects/batman-adv/wiki/Wiki](https://www.open-mesh.org/projects/batman-adv/wiki/Wiki)

---


**Aymeric Wibo** 是比利时法语鲁汶大学的计算机科学学生，自高中起就一直使用并开发基于 FreeBSD 的项目。他的主要兴趣在于图形学和网络。
