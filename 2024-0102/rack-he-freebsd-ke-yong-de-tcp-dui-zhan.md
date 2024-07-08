# RACK 和 FreeBSD 可用的 TCP 堆栈

由 RANDALL STEWART 和 MICHAEL TÜXEN

2017 年，在 FreeBSD 中对 TCP 栈进行了更改，允许多个 TCP 栈共存。这种方式可以保持现有 TCP 栈的不变，允许在一定的函数调用数量限制下进行创新。所有 TCP 栈仍然共享一些功能：包括 SYN-Cache 的实现，包括 SYN-Cookies 的处理以及处理传入 TCP 段的初始步骤，如校验和验证并基于数字和 IP 地址查找 TCP 端点。在任何给定时间，TCP 连接由一个 TCP 栈处理，但此 TCP 栈在 TCP 连接的生命周期内可以更改。

这是 TCP RACK 栈开始作为原始 TCP 栈的完全重写的地方，从调用到函数 tcp_do_segment() 和许多其他模块化的子功能开始。最初的目标是添加对一种称为最近确认（RACK）的丢失检测方法的支持。RACK 在一个 Internet 草案中描述，后来于 2021 年成为 RFC 8985。这就是这个 TCP 栈——RACK——名称的由来。但是 TCP RACK 栈已远远超出了仅仅添加对 RFC 8985 支持的范畴。重写的一部分包括完全不同的选择性确认（SACK）信息处理方式。在 TCP RACK 栈中，维护了所有已发送用户数据的完整映射，这允许更好地处理用户数据的重传，以及增加 RFC 8985 中描述的 RACK 丢失检测。许多额外的功能都源于此重写，并在本文中进行了描述。

## 如何使用 TCP RACK 堆栈

RACK 堆栈在 FreeBSD CURRENT 和 FreeBSD 14.0 中都可用。如何使其可用取决于 FreeBSD 版本。

对于 FreeBSD 14.0，需要向内核配置文件中添加以下两行

`option TCPHPTSmakeoptions WITH_EXTRA_TCP_STACKS=1`

重建内核。第一行导致将 TCP 高精度计时器系统（HPTS）编译到内核中。第二行导致生成用于 TCP RACK 堆栈（tcp_rack.ko）的内核可加载模块。要使用 TCP RACK 堆栈，必须加载内核模块。可以通过在每次重新启动时添加以下行来执行此操作

`tcp_rack_load=”YES”`

 到文件 /boot/loader.conf.

在 FreeBSD CURRENT 中，TCP RACK 和 HPTS 都默认构建为内核模块。由于 tcphpts.ko 自动加载为 tcp_rack.ko 的依赖项，只需使用 kldload 加载后者。要在每次重新启动时加载 TCP RACK 堆栈，需要将以下两行添加到文件 /boot/loader.conf:

`tcphpts_load=”YES”tcp_rack_load=”YES”`

将 TCP RACK 堆栈编译到 FreeBSD CURRENT 的内核中也是可能的，方法是将以下两行添加到内核配置文件中

`option TCPHPTSoption TCP_RACK`

然后重新构建内核。

请注意，TCP Blackbox 日志记录（选项 TCP_BLACKBOX ）现在在 FreeBSD 14.0 及更高版本以及所有 64 位平台的 FreeBSD CURRENT 中默认构建，因为这是 TCP 传输开发人员仪表化和调试各种 TCP 堆栈的标准方式。

上述描述了如何在 FreeBSD 系统上使用 TCP RACK 堆栈。通过运行命令显示所有可用的 TCP 堆栈列表

`sysctl net.inet.tcp.functions_available`

 在shell中。

在即将发布的版本——FreeBSD 14.1 及更高版本中，使用 TCP RACK 堆栈的方法与上述 FreeBSD CURRENT 版本中描述的方法相同。

实际上使用 TCP RACK 堆栈的不同方式有一些涉及应用程序源代码更改，有一些只涉及配置更改。

sysctl 变量 net.inet.tcp.functions_default 用于指定在使用 socket(2) 系统调用创建新的 TCP 端点时使用的默认 TCP 堆栈。执行

`sysctl net.inet.tcp.functions_default=rack`

设置默认堆栈为 TCP RACK 堆栈。通过添加该行

`net.inet.tcp.functions_default=rack`

在重启系统后，TCP RACK 堆栈将成为默认的 TCP 堆栈。当通过侦听器创建 TCP 端点时，TCP 堆栈会从侦听器继承，或根据 sysctl 变量 net.inet.tcp.functions_inherit_listen_socket_stack 是非零还是零来确定使用默认 TCP 堆栈。该变量的默认值为一。

也可以使用 tcpsso(8) 命令行工具来更改单个 TCP 连接的 TCP 堆栈，如工具的 man 页面所述。

如果可以修改源代码，可以使用名为 TCP_FUNCTION_BLK 的 IPPROTO_TCP-level 套接字选项来将套接字使用的 TCP 堆栈切换到 TCP RACK 堆栈。选项值的类型为 struct tcp_function_set 。例如，以下代码执行此操作：

`struct tcp_function_set tfs;strncpy(tfs.function_set_name, “rack”, TCP_FUNCTION_NAME_LEN_MAX);tfs.pcbcnt = 0;setsockopt(fd, IPPROTO_TCP, TCP_FUNCTION_BLK, &tfs, sizeof(tfs));`

使用 TCP RACK 栈允许使用许多默认 TCP 栈目前不支持的功能。这些功能大多可以通过 IPPROTO_TCP-level 套接字选项或 net.inet.tcp.rack 下的 sysctl 变量来控制。

## TCP RACK 栈的特性

以下部分描述了 TCP RACK 栈提供的最重要特性。

### RACK/TLP

最近的确认（RACK）和尾部丢失探测（TLP）是 TCP RACK 堆栈中集成的两个功能。最近的确认改变了丢包检测和重传触发的方式。实现在 FreeBSD 基本堆栈中的丢包检测，根据 RFC 5681 的规定，需要三个重复的确认或具有 SACK 的确认到达才能让 TCP 堆栈发送重传。在某些情况下，例如发送的数据包少于四个，这将导致 TCP 堆栈只在发生重传超时后才发送重传。RACK 对此进行了更改，当有一个 SACK 到达时，如果自丢失数据包发送以来已经过了足够长的时间，则立即进行重传。如果时间不够长（通常比当前往返时间大一点），那么会启动一个小的 RACK 计时器，当此计时器到期时，将发送重传。这将解决许多情况，但不是全部，其中必须通过重传超时才能强制发送数据的情况。最后一种情况由 TLP 解决。每当 TCP RACK 堆栈发送数据时，它会启动一个 TLP 计时器而不是重传计时器。如果 TLP 计时器到期，则 TCP RACK 堆栈会发送新段或上次发送的最后一段。通过这个 TLP 发送的段，希望发送方要么收到确认，指示已接收到所有数据（最后一个确认丢失的情况），要么 TLP 会引发 SACK，这将允许正常的快速恢复机制接管，而不会触发重传超时，从而将拥塞窗口折叠到 1 MSS。

TCP RACK 堆栈的用户只需启用堆栈，就可以自动获得 RACK 和 TLP 的益处。上层不需要任何套接字选项或配置。

### 比例速率减少 (PRR)

比例速率减少 (PRR) 是 TCP RACK 协议栈的另一个自动内置功能，由 RFC 6937 规定，并正在由 IETF 更新。PRR 改进了在快速恢复期间发送数据的方式。当使用 RFC 5681 规定的 TCP 拥塞控制时，进入快速恢复时拥塞窗口会减半。这导致在快速恢复期间发送新数据的停顿。基本上，发送方必须等待半数的未确认数据被确认，然后发送方可以开始发送新数据（以及我们的重传）。这导致数据流在发送方和接收方之间“停顿”。PRR 的设计目的是改进这一点，使得在快速恢复期间，大约每隔一个确认可以发送一个新的数据段。这样可以防止数据“停顿”，保持数据持续流动，从而保持 RTT 和其他传输指标活跃并更新。

### RACK 快速恢复 (RRR)

RACK Rapid Recovery (RRR)是一个有趣的功能，最初是作为一个 bug 开始的。在最初的开发中，TCP RACK 堆栈无意中允许了一种情况，即当到达一个声明多个段丢失的 SACK 并且所有数据的 RACK 计时器均已过期时，TCP RACK 堆栈将发送一个段并启动 RACK 计时器。当 RACK 计时器过期时（设置为 RACK 最小超时值 1 毫秒），TCP RACK 堆栈将发送另一个丢失的段。这将重复，直到所有丢失的段均已发送。这在初始恢复过程中有效地忽略了 PRR，并伴有稍后发送更多 PRR 段的成本。例如，如果 RRR 发送了 3 个段，第一个重传和两个额外的段，则需要大约 6 个额外的确认才能使 PRR 发送一个新段。

当发现并“修复”此 bug 后，用户的体验质量（QoE）下降了。这是因为那些早期的段丢失通常会延迟相当多的数据段的传递。这导致将其作为一个可以关闭的功能，并且可以编程设置时间量，因为有关的时间有效地使 RRR 在其默认设置中以 12Mbps 的速率进行分段。默认情况下，此功能处于打开状态，并且 RRR 恢复速率设置为每毫秒一个段。这导致速率为 12 Mbps，假设最大传输单元（MTU）为 1500 字节。

### SACK 攻击检测

保持完整映像的一个缺点是，在某些情况下，此映像可能会变得非常大。TCP RACK 栈始终试图将映像折叠到尽可能小的尺寸，同时仍然跟踪所有未完成数据的各个阶段。然而，这样做引入了一个可能性，即恶意对等方可以设计攻击 TCP RACK 栈用于 TCP 连接的内存和 CPU 资源，方法是不断将发送映像分割成越来越小的片段，从而使 TCP RACK 栈使用大量内存，并花费过多时间在其中搜索。例如，攻击者可能会每隔一个字节发送 SACKs。这可能构成严重威胁，并以不希望的方式影响机器。

TCP RACK 栈包含一个名为 TCP_SAD_DETECTION 的可选编译功能。SAD 代表 SACK 攻击检测（SAD）。可以通过向内核配置文件添加以下行启用它，然后重新构建内核。

`option TCP_SAD_DETECTION`

to the kernel configuration file and rebuilding the kernel.

一旦添加，它默认启用。 它监视恶意对等体，如果检测到，它将禁用来自对等体的 SACK 的处理。 这会降低该单个对等体的性能，但不会阻止连接继续进行。 实际上，它变成了一个响应好像从未启用 SACK 一样的连接。 这惩罚了丢失恢复，但仍然允许连接继续。

### 突发缓解

内置于 TCP RACK 栈中，无需任何用户干预，具有突发缓解功能。 为了减轻突发，堆栈在发送机会时只会发送一组固定大小（最大突发大小），并启动一个小计时器（以发送更多数据）或依赖返回的确认流来提示发送更多数据。 这有助于减轻可能导致过多丢失的大突发。

### 支持 TCP Blackbox Logging (BBLog)

TCP RACK 堆栈的一个有趣方面是对 TCP Blackbox Logging 的广泛支持，用于调试和一般统计分析和工具。这使得更容易跟踪问题，并获得连接行为的分析。

### 大接收卸载（LRO）集成以进行突发缓解

TCP 大型接收卸载（LRO）是一项功能，通过将多个接收的 TCP 数据段合并成一个单独的数据段，然后再传递到 TCP 协议栈，从而减少接收方所需的 CPU 资源。通常，这会导致有关各个接收数据段的信息丢失，但可以减少所需的 CPU 资源，因为 TCP 协议栈需要处理的数据段更少。

一个有趣的功能交互是对 LRO 代码进行了一系列更改，以更好地支持 TCP RACK 协议栈中的速率控制。当 TCP 连接正在进行突发缓解时，它往往会更频繁地通过发送路径，发送更小的突发数据。因此，对 LRO 代码进行了更改，允许所有关于数据包到达时间的计时数据传递到 TCP RACK 协议栈而不丢失。基本上，在数据包处理期间，LRO 代码会查找数据包是否与允许将数据包直接排队到 TCP RACK 协议栈的连接相关联。如果是，数据包将直接排队到连接，并且根据连接状态的不同，连接可能会被唤醒。在 TCP RACK 协议栈进行突发缓解或速率控制的情况下，该唤醒将被推迟，直到定时器到期并且可以处理传入的确认。这些步骤还绕过了 IP 协议栈处理，从而进一步减少所需的 CPU 资源。

### 一堆替代功能

通过各种套接字选项和 sysctl 变量，TCP RACK 栈支持许多其他功能。当前，TCP RACK 栈支持 58 个套接字选项，这些选项包括节奏控制、突发缓解选项和恢复响应修改等各种功能。除了套接字选项外，还有大约 150 个 sysctl 变量，用于将套接字选项应用于所有连接或修改各种 TCP RACK 栈的默认配置。所有这些功能和配置都可帮助调整 TCP RACK 栈，以更好地符合您的网络条件和要求。

## Netflix 如何演进 TCP RACK 栈

Netflix 目前仅使用 TCP RACK 栈，默认栈为 FreeBSD，默认栈存在但未使用。Netflix 使用 TCP RACK 栈的方式有些新颖，值得注意。Netflix 实际上保留了几代 TCP RACK 栈，以其发布版本命名。始终保留“最新”的 TCP RACK 栈，其中包含由其传输组开发的所有前沿功能。

定期地，当一个发布版本完成时，最新的 TCP RACK 堆栈正在开发中，并根据发布编号进行复制和支持。然后根据 QoE 和 CPU 性能评估此 TCP 堆栈，以与当前使用的默认旧 TCP 堆栈进行比较。当最新的 TCP RACK 堆栈至少与旧的 TCP RACK 堆栈一样好或更好时，在下一个发布版本中将默认切换到更新的 TCP RACK 堆栈。旧的 TCP RACK 堆栈将在几个发布版本中维护，并最终被移除。

TCP RACK 堆栈上的新功能也通过这种方式进行测试，以确定该功能是否增加了价值。在减少网络影响的同时，对 Netflix 用户的 QoE 没有降级是传输团队的主要目标之一，以便 Netflix 既是更好的网络公民，同时也提供良好的整体 QoE。

## 结论和展望

TCP RACK 堆栈为 FreeBSD 基础堆栈提供了强大的替代方案。它添加了更多功能和选项，为应用程序开发人员提供了更丰富的选择，以更好地定制用户的 TCP 体验。

TCP RACK 堆栈已经通过 Netflix 的设置和工作负载进行了广泛测试。但是，在其他设置和工作负载中测试它同样重要。因此，如果用户能在他们的硬件上，使用他们的设置和工作负载测试 TCP RACK 堆栈，将会很好。请将在测试过程中发现的任何问题报告给 net@freebsd.org 或本文的作者。根据反馈和进一步测试，TCP RACK 堆栈可能会成为 FreeBSD 的默认堆栈。

RANDALL STEWART（rrs@freebsd.org）已经从事操作系统开发超过 40 年，并且是 FreeBSD 开发人员超过 10 年。他专注于传输，包括 TCP 和 SCTP，但也涉足操作系统的其他领域。他目前在 Netflix 的传输团队工作，支持 TCP 堆栈，并不断创新以提高用户的体验质量。

MICHAEL TÜXEN（tuexen@freebsd.org）是明斯特应用科技大学的教授，Netflix 的兼职承包商，自 2009 年起是 FreeBSD 源代码提交者。他专注于传输协议，如 SCTP 和 TCP，在 IETF 的标准化及其在 FreeBSD 中的实现。
