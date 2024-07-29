# BATMAN：更佳的可移动 ad-hoc 网络方式

 艾梅里克·维波

在网络协议的广阔领域中，有一种多才多艺且坚韧不拔的竞争者：BATMAN，即 Better Approach to Mobile Ad-hoc Networks。在大城市的空中波段上，BATMAN 允许设备在网状网络上无缝通信，而无需任何一个设备了解更广泛的网络拓扑结构。

这个夏天，我参加了 Google 夏季代码（GSoC），在那里我将 BATMAN 协议的 batman-adv 内核模块（该模块在 Linux 上支持 BATMAN 协议）移植到 FreeBSD。GSoC 是一个由 Google 赞助的项目，学生在夏季期间参与开源项目工作，由导师监督。在我的情况下，导师是无人能敌的 Mahdi（或 Mehdi，具体取决于你问他的周几），也就是 mmokhi@。他非常有风度，我很感激在整个 GSoC 旅程中有他的指导！

目前使用 batman_adv （FreeBSD 中类似于 batman-adv 的东西），您可以创建和参与网状网络，并通过以太网发送/接收数据包。这一切也可以在 Linuxulator 内部运行。移植工作主要集中在 LinuxKPI 上（特别是通过 struct net_device 的后备 struct ifnet 和 mbuf 的 sk_buff ），这将有望为将来移植其他与网络相关的 Linux 驱动程序提供便利。

## port过程是怎样的？

虽然我远非内核组件移植的专家（这是我第一次移植像 BATMAN 这样大的东西），但这里是我看过的整个过程的高级概述 — 说不定从我的小冒险中能得到一些见解。

第一步是 — 自然而然地 — 从 Linux 中拉取 batman-adv 的代码到 sys/contrib/dev/batman_adv 中，并创建 Makefile 来构建它，其中包含要使用的所有源文件列表和要设置的编译时选项，例如告诉它包含 LinuxKPI 的内容。预料之中，第一次尝试编译并不成功； batman-adv 调用了许多函数并使用了许多结构，这些函数和结构只在 Linux 内核的上下文中存在或有意义。这就是 LinuxKPI 的理由所在；它提供了一个兼容层，通过调用等效（有时是不那么等效）的函数在 FreeBSD 内核中实现 Linux 内核的子集。

所以，接下来自然要做的就是通过为 LinuxKPI 中所有缺失的函数和结构编写存根来编译，将缺失的字段在使用时填入（我们不能直接复制 Linux 的结构，因为 Linux 是 GPL 许可的）。这些存根只包含调试打印语句，让我们知道它们何时被使用。

编译完所有内容后，我可以加载内核模块，但它立即使内核崩溃。然后就是遍历所有被调用的存根，查找并理解它们在 Linux 中的实现，并在 LinuxKPI 中为 FreeBSD 实现一个等价物，直到内核不再崩溃。这可能占了工作量的 70%。

一旦每个操作（加载模块、创建接口、通过它发送数据等）都能正常工作且不破坏任何东西，就该拉上窗帘，在第二个显示器上打开 Wireshark，把自己锁在宿舍（即比利时宿舍）一周，确保 a) 网状网络中的所有设备相互识别，b) 数据能从设备 A 经过设备 B 传输到设备 C 而不会在过程中被损坏或丢失。很多时间都花在修正前一步（有时是不够的）实现上。这可能占了工作量的 30%，但感觉像 90%，大部分时间都在盯着 Wireshark，而它也在盯着我。

最后，BATMAN 在 FreeBSD 上运行，只剩下让用户态工具支持操纵 BATMAN 接口，写几行文档，并拉开窗帘。

## 向上游提交 batman_adv

去年在 EuroBSDCon 上，我与 core@的一些成员谈到了上游 batman_adv 的可能性，他们希望避免这样做，因为 BATMAN 是 GPL 许可的。所以它可能会永远保持为port，因为没有真正需要 BATMAN 来使 FreeBSD 系统正常工作的用例；如果你需要使用 BATMAN 做某事，你可能会处于一个很容易获取和构建port本身的位置（与 NIC 驱动程序不同）。

## 还有什么要做的？

目前在 FreeBSD 上 BATMAN 的一个重要限制是它无法参与无线网络。我完全打算在接下来的一年内至少让无线网络运行起来，因为那是 BATMAN 的主要用例所在。希望我能及时在 2024 年的 BSDCan 之前完成这项工作：🙂：

我全心全意推荐任何符合申请要求的人参加 GSoC。我从对 FreeBSD 的网络栈和内核不太了解，到现在，在伟大计划中仍然对它不是很了解，但肯定比开始时要了解得多。这尤其帮助我更加熟悉在源代码树中导航和调试内核崩溃，即使与非网络代码相关。

## 进一步阅读

这里有一个链接到我的 GSoC 项目维基页面，其中包含所有的具体信息、代码和一个小型演示视频：https://wiki.freebsd.org/SummerOfCode2023Projects/CallingTheBatmanFreeNetworksOnFreeBSD

而且，你可能会发现这个链接到 batman-adv 概述很有趣，因为它更详细地介绍了在 Linux 上的 BATMAN 实现（因此也适用于 FreeBSD）：https://www.open-mesh.org/projects/batman-adv/wiki/Wiki

AYMERIC WIBO 是比利时 UCLouvain 的计算机科学学生，自高中起便开始使用和开发基于 FreeBSD 的项目。他主要兴趣在于图形和网络。
