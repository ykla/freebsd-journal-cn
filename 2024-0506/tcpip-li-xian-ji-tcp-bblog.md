# TCP/IP 历险记：TCP BBLog

## FreeBSD 中 TCP 日志记录的演变

4.2 BSD 发布于 1983 年，并包括了 BSD 中的第一个 TCP 实现。该版本还增加了对调试 TCP 实现的支持设施。内核部分由内核选项 TCP_DEBUG （默认禁用）控制，提供了一个全局环形缓冲区，有 TCP_NDEBUG （默认为 100）个元素，并提供了将 TCP 段发送或接收时，TCP 定时器过期时，或处理 TCP 相关协议用户请求时，向环形缓冲区添加条目的例程。这些事件仅添加到启用了 SOL_SOCKET 级套接字选项 SO_DEBUG 的套接字中。4.2 BSD 还提供了命令行实用程序 trpt （音译协议跟踪），可以从活动系统或核心文件中读取环形缓冲区并打印出来。它不仅打印发送和接收的 TCP 段的 TCP 头，还在发送或接收 TCP 段、TCP 定时器到期或处理 TCP 相关协议用户请求时，打印 TCP 端点的最重要参数。需要注意的是，在发生恐慌的情况下，环形缓冲区的内容可能提供足够的信息来弄清楚系统为何陷入糟糕状态。然而，由于这个设施不再符合今天对 TCP 的使用方式，它在 FreeBSD 14 中被移除。在早期版本的 FreeBSD 中，构建一个具有非默认配置的内核是必需的。

在 2010 年， siftr （用于 TCP 研究的统计信息）内核模块被添加到 FreeBSD 中。不需要更改 FreeBSD 内核，只需加载模块即可使用。 siftr 仅通过 sysctl -变量进行控制。启用后，由 sysctl -变量 net.inet.siftr.enabled 控制， siftr 将其输出写入文件，由 sysctl 变量 net.inet.siftr.logfile 控制（默认 /var/log/siftr.log ）。条目，除了第一个和最后一个，对应于发送或接收的 TCP 段，并提供有关方向、IP 地址和 TCPport号码以及内部 TCP 状态的信息。由于它设计用于与像 tcpdump 这样的数据包捕获工具结合使用，因此不存储有关 TCP 段（例如 TCP 标头）的其他信息。每个第 n 个 TCP 段将被记录每个 TCP 连接的发送和接收方向。n 由 sysctl -变量 net.inet.siftr.ppl 控制。可以通过 sysctl-variable net.inet.siftr.port_filter 控制的 TCPport过滤器应用于专注于特定的 TCP 连接。所有信息都以 ASCII 格式存储，因此不需要额外的用户空间工具来访问信息。在默认配置中，仅支持 TCP/IPv4。添加对 TCP/IPv6 的支持需要重新编译 siftr 内核模块。

在 2015 年，内核添加了一个设施，由内核选项 TCP_PCAP 控制（默认情况下禁用）。如果在非默认内核上启用，每个 TCP 端点都包含两个环形缓冲区：一个用于发送的 TCP 段，一个用于接收的 TCP 段。值得注意的是，没有存储任何附加信息，甚至没有存储 TCP 段发送或接收的时间。每个环形缓冲区中的 TCP 段的最大数量由 IPPROTO_TCP 级套接字选项 TCP_PCAP_OUT 和 TCP_PCAP_IN 控制。默认值由 sysctl 变量 net.inet.tcp.tcp_pcap_packets 控制。由于没有用户空间实用程序来提取环形缓冲区的内容，因此此功能的使用仅限于分析核心文件。还应注意，也记录了 TCP 有效负载，这可能会因隐私方面的原因使得包含此类信息的核心文件难以共享。计划在即将发布的 FreeBSD 15 版本中删除对此设施的支持。

最新的 TCP 日志记录设施，TCP BBLog（TCP 黑匣子日志记录）于 2018 年添加。最初称为 TCP BBR（黑匣子记录器），但为了避免与 TCP 拥塞控制称为 BBR（瓶颈带宽和往返传播时间）的混淆，现在称为 BBLog。BBLog 在所有 64 位平台的所有生产版 FreeBSD 上均已启用。它结合了 TCP_DEBUG 和 TCP_PCAP 的优点，却不具备它们的缺点。因此，它旨在取代它们两者。BBLog 可以通过后续列中描述的 sysctl 接口和套接字 API 进行控制。

## BBLog 介绍

BBLog 由内核选项 TCP_BLACKBOX （在所有 64 位平台上默认启用）控制，内核源代码位于 sys/netinet/tcp_log_buf.c 及其对应的头文件 sys/netinet/tcp_log_buf.h 中。在启用 BBLog 的内核上，有一个设备（ /dev/tcp_log ）用于向用户态工具提供 BBLog 信息，每个 TCP 端点包含 BBLog 事件列表。

每个事件包含一组标准的重要 TCP 状态信息，以及（可选地）一块特定事件数据。这些事件被收集到一定限制，当达到限制时，这些事件可能会被发送到一个 /dev/tcp_log ，如果打开，则将信息中继给用于记录的读取进程(s)。请注意，如果没有进程打开设备，则数据将被丢弃。

tcplog_dumper ，来自 FreeBSDports集合，可用于从/ dev/tcp_log 中读取，如下所述。

所有 FreeBSD TCP 堆栈都已经配置，至少包含以下事件类型：

* TCP_LOG_IN — 当 TCP 段到达时生成。
* TCP_LOG_OUT — 当 TCP 段发送时生成。
* TCP_RTO — 定时器过期时生成。
* TCP_LOG_PRU — 当 PRU 事件被调用到堆栈中时生成。

TCP RACK 和 BBR 栈生成许多其他日志；当前定义了 72 种事件类型。这些日志记录了各种条件，TCP BBR 和 RACK 栈甚至在调试堆栈时有详细模式。这些详细选项通过特定于堆栈的变量 sysctl 和 net.inet.tcp.rack.misc.verbose 进行设置。

每个 TCP 端点可以处于以下 BBLog 状态之一：

* TCP_LOG_STATE_OFF (0) — BBLog 已禁用。
* TCP_LOG_STATE_TAIL (1) — 仅记录连接上的最后事件。每个连接被分配了有限数量（默认 5000）的日志条目。当达到最后一条条目时，重新使用第一条条目并覆盖它。
* TCP_LOG_STATE_HEAD (2) — 仅记录在连接上处理的前几个事件，直到达到限制为止。
* TCP_LOG_STATE_HEAD_AUTO (3) — 记录连接上处理的前几个事件，并且当达到限制时，将数据转存到日志转存系统以便收集。
* TCP_LOG_STATE_CONTINUAL (4) — 记录所有事件，当达到最大收集事件数时，将数据发送到日志转存系统，并开始分配新事件。
* TCP_LOG_STATE_TAIL_AUTO (5) — 在连接尾部记录所有事件，当达到限制时将数据发送到日志转储系统。

通常用于一般调试的 BBLog 状态 TCP_LOG_STATE_CONTINUAL  注意。然而，在某些特定情况下（调试崩溃时），最好使用 BBLog 状态 TCP_LOG_STATE_TAIL ，以便在崩溃转储中记录最后的 BBLog 事件。

BBLog 状态可以在建立 TCP 连接时或通过套接字 API 设置。除此之外，当 TCP 连接满足特定条件时也可以设置 BBLog 状态。这称为跟踪点，针对特定的 TCP 堆栈指定，通过数字进行标识。一个跟踪点的示例是当 TCP 堆栈调用 IP 输出例程时获取 ENOBUF 。

每个事件的内容由三个部分组成：

1. 一个包含 TCP 连接的 IP 地址和 TCP port 号码、事件时间、标识符、原因和标签的 BBLog 标头。
2. 包括 TCP 连接状态和各种序列号变量在内的 TCP 连接的一组强制状态变量。
3. 一组可选数据，如发送和接收缓冲区占用情况、TCP 头信息和进一步的事件特定信息。

请注意，任何 BBLog 事件中不包含 TCP 负载信息，但每个 BBLog 事件中都包含关于 IP 地址和 TCP port编号的信息。

## BBLog 配置

通常有两种配置 BBLog 的方式。一般配置通过 sysctl -界面完成，TCP 连接特定配置通过套接字 API 完成。

### 通过 sysctl-界面进行通用配置

这是 BBog 相关的变量列表，这些变量都在 net.inet.tcp.bb 下：

`[rrs]$ sysctl net.inet.tcp.bbnet.inet.tcp.bb.pcb_ids_tot: 0net.inet.tcp.bb.pcb_ids_cur: 0net.inet.tcp.bb.log_auto_all: 1net.inet.tcp.bb.log_auto_mode: 4net.inet.tcp.bb.log_auto_ratio: 1net.inet.tcp.bb.disable_all: 0net.inet.tcp.bb.log_version: 9net.inet.tcp.bb.log_id_tcpcb_entries: 0net.inet.tcp.bb.log_id_tcpcb_limit: 0net.inet.tcp.bb.log_id_entries: 0net.inet.tcp.bb.log_id_limit: 0net.inet.tcp.bb.log_global_entries: 5016net.inet.tcp.bb.log_global_limit: 5000000net.inet.tcp.bb.log_session_limit: 5000net.inet.tcp.bb.log_verbose: 0net.inet.tcp.bb.tp.count: 0net.inet.tcp.bb.tp.bbmode: 4net.inet.tcp.bb.tp.number: 0`

使用 sysctl -界面，可以为 TCP 连接启用 BBLog。这个的关键 sysctl -变量是 net.inet.tcp.bb.log_auto_all ， net.inet.tcp.bb.log_auto_mode 和 net.inet.tcp.bb.log_auto_ratio 。

要考虑的第一个 sysctl -变量是 net.inet.tcp.bb.log_auto_all 。如果此变量设置为 1，则所有连接都将被视为 BBLog 比率。如果该值设置为零，则只有已设置 TCP_LOGID 的连接才会被应用 BBLog 比率。 在大多数情况下，使用 sysctl -方法启用 BBLog 时，应用程序可能并未设置 TCP_LOGID ，因此将 net.inet.tcp.bb.log_auto_all 设置为 1 可以确保每个连接都会被考虑。

要设置的下一个 sysctl -变量是 net.inet.tcp.bb.log_auto_ratio 。该值决定每 n 个连接（n 是通过设置 net.inet.tcp.bb.log_auto_ratio 提供的值）将应用 BBLog 启用。因此，例如，如果 net.inet.tcp.bb.log_auto_ratio 设置为 100，则每 100 个连接中就会有一个连接启用 BBLog。 如果需要为每个连接启用 BBLog，则 net.inet.tcp.bb.log_auto_ratio 需要设置为 1。

要考虑的最后一个 sysctl -变量是 net.inet.tcp.bb.log_auto_mode 。对于 TCP 开发，默认值可以设置为 4，以便 TCP_LOG_STATE_CONTINUAL 记录由任何连接生成的每个事件，用于调试目的。

sysctl -变量中的其他一些项目也可能很有用， net.inet.tcp.bb.log_session_limit 控制连接在必须处理数据之前可以收集多少个 BBLog 事件，即将其发送到收集系统或回收（覆盖）事件。 net.inet.tcp.bb.log_global_limit 强制执行全局系统限制，限制操作系统允许分配的总 BBLog 事件数。

最后三个 sysctl -变量与跟踪点相关。 net.inet.tcp.bb.tp.bbmode 指定在触发跟踪点时要使用的 BBLog 状态。 net.inet.tcp.bb.tp.count 是允许触发指定跟踪点的连接数。例如，如果设置为 4，则 4 个连接可以触发该跟踪点，之后不会触发该特定点（这是为了限制生成的 BBLog 事件数量）。 net.inet.tcp.bb.tp.number 指定要启用的跟踪点。

### 通过 Socket API 实现 TCP 连接特定配置

可用于控制单个连接上的 BBLog 的以下 IPPROTO_TCP -级别套接字选项：

* TCP_LOG — 此选项在连接上设置 BBLog 状态。对此套接字选项的任何使用将覆盖以前的设置。
* TCP_LOGID — 此选项接收一个字符串，设置生成的 tcplog_dumper 文件名时使用的名称。它将字符串关联为与连接关联的“ID”。请注意，多个连接可以使用相同的“ID”字符串。这是因为 tcplog_dumper 还将 IP 地址和ports包含在生成的文件名中。
* TCP_LOGBUF — 此套接字选项可用于从当前连接的日志缓冲区中读取数据。通常不使用此选项，而是由诸如 /dev/tcp_log （读取和存储 BBLogs 的通用工具）的工具读取。但是，这作为一种替代方案，允许用户进程收集多个日志。
* TCP_LOGDUMP — 此套接字选项指示 BBLog 系统将连接队列中的任何记录转储到 /dev/tcp_log 。如果未给出转储原因或 ID，则在转储文件内部使用正在进行的日志类型的系统默认值作为任何“原因”字段。
* TCP_LOGDUMPID — 此套接字选项，像 TCP_LOGDUMP, directs the BBLog system to dump out any records to<span> </span><code>/dev/tcp_log</code>, but in addition it specifies a specific user given “reason” for the output which will be included in the BBlog “reason” field. <li><p><code>TCP_LOG_TAG</code><span> </span>— This option associates an additional “tag” in the form of a string with all BBLog records for this connection.</p></li>

例如，如果可以访问使用 TCP 连接的程序的源代码，则可以使用 TCP_LOG 套接字选项将连接的 BBLog 状态设置为 TCP_LOG_STATE_CONTINUAL ：

`#include <netinet tcp_log_buf.h=""></netinet>`

`int err;int log_state = TCP_LOG_STATE_CONTINUAL;err = setsockopt(sd, IPPROTO_TCP, TCP_LOG, &log_state, sizeof(int));`

此代码也可用于以前提到的任何其他 BBLog 状态。

如果没有访问源代码的权限，可以使用 root 权限

`tcpsso -i id TCP_LOG 4`

其中 id 是 inp_gencnt ，可以通过运行 sockstat -iPtcp. 4 确定为 TCP_LOG_STATE_CONTINUAL 的数值。

## 生成 BBLog 文件

在将 BBLog 在特定 TCP 连接上启用之前，首先需确保 BBLogs 的收集正在进行。FreeBSD 有一个专为此目的设计的工具，称为 tcplog_dumper  ，可在ports树中获取（ net/tcplog_dumper ）。可以通过以 root 权限运行以下命令来安装：

`pkg install tcplog_dumper`

 加入

`tcplog_dumper_enable=”YES”`

将 /etc/rc.conf 添加到文件中会使守护进程在下次重启后自动启动。也可以通过以 root 权限手动运行以下命令来启动守护进程：

`tcplog_dumper -d`

默认情况下， tcplog_dumper 将在目录 /var/log/tcplog_dumps 收集 BBLog 的日志。还支持几种其他选项，包括：

* -J  — 此选项将导致 tcplog_dumper 输出带有 xz 的压缩文件。
* -D directory path — 将收集的文件存储在指定的目录路径中，而非默认位置。这也可以通过 rc.conf 变量 tcplog_dumper_basedir 来控制。

tcplog_dumper 将输出 pcapng（pcap 下一代）文件。 pcapng 支持存储元信息以及数据包信息。 对于 TCP_LOG_IN 和 TCP_LOG_OUT 事件， tcplog_dumper 从事件生成 IP 头部（因此除了源 IP 地址和目的 IP 地址之外，IP 头部中的字段可能与原始数据包中的字段不同），使用事件中的 TCP 头部（这意味着段与原始数据包中的段相同），并添加正确长度的虚拟负载。 对于每个 TCP 连接， tcplog_dumper 创建一系列文件，并且将大约 5000 个 BBLog 事件放入每个文件中，文件按顺序编号为 0、1、2 等等。 以下是为单个 TCP 连接创建的一系列 7 个文件的示例::

`[rrs]$ ls /var/log/tcplog_dumps/UNKNOWN_18262_10.1.1.1_9999.0.pcapng UNKNOWN_18262_10.1.1.1_9999.4.pcapngUNKNOWN_18262_10.1.1.1_9999.1.pcapng UNKNOWN_18262_10.1.1.1_9999.5.pcapngUNKNOWN_18262_10.1.1.1_9999.2.pcapng UNKNOWN_18262_10.1.1.1_9999.6.pcapngUNKNOWN_18262_10.1.1.1_9999.3.pcapng records`

因此该连接未设置 TCP_LOGID ，其中一个 TCP ports 为 18262，另一个 TCP port 为 9999，远程 IPv4 地址为 10.1.1.1。

从核心转储生成 BBLog 文件目前正在进行中。 调试器将用于提取信息并将其提供给 cplog_dumper 以实际编写 BBLog 文件。

## 阅读 BBLog 文件

有两个易于访问的工具可以读取 BBLog 文件。这些是 read_bbrlog 和 wireshark ，都可以作为ports或软件包使用。

### read_bbrlog

read_bbrlog 是一个小程序，用于读取一系列 BBLog 文件并以文本形式显示每个日志条目。它需要输入 BBLog 文件的前缀作为输入源，并查找所有与该 TCP 连接关联的文件，并将每个事件以文本形式打印到 stdout 。请注意，还有一个选项可以将输出重定向到文件（强烈建议，因为会显示大量数据）。以下是运行 read_bbrlog 的示例：

`[rrs]$ read_bbrlog -i UNKNOWN_18262_10.1.1.1_9999 -omy_output_file.txt -e Files:7 Processed 30964 records Saw30964 records from stackid:3 total_missed:0 dups:0`

在这种情况下，使用了三个选项： -i input ，其中输入参数是基本连接 ID，即 ls 减去 .X.pcapng 的显示文本。使用 -o outfile 将输出重定向到输出文件 my_output_file.txt ，最后使用 -e 选项通常用于输出更详细的“扩展”输出。

这里是文件 my_output_file.txt 的一个小片段，以展示数据的 flavor。由于行长度较长，一些显示数据已被截断：

`106565924 0 rack [50] PKT_OUT    Sent(0) 763046978:5  (PUS|ACK fas:0 bas:1) bw:208.00 bps(26)                       avail:5 cw:14480 scw:14480 rw:65535 flt:0 (spo:64 ip:0)106565979 0 rack [55] TCP_HYSTART  -- New round begins round:1 ends:763046983 cwnd:14480 106565982 0 rack [3] BBR_PACING_CALC Old rack burst mitigation len:5 slot:0 trperms:369 106565985 0 rack [3] TIMERSTAR type:TLP(timer:4) srtt:39001 (rttvar:17063 * 4) rttmin:30000 106565986 0 rack [1] USERSEND   avail:5 pending:5 snd_una:763046978 snd_max:763046983 out:5 106565986 0 rack [0] TCP_LOG_PRU pru_method:SEND (9) err:0106607480 0 rack [2] IN         Ack:Normal 5 (PUS|ACK) off:32 out:5 lenin:5 avail:5 cw:14480                                 rw:4000512 una:763046978 ack:763046983`

这显示在时间标记 106565924 发送了一个 5 字节的数据包，并且序列号为 763046978。当时的拥塞窗口大小为 14480 字节，飞行大小（flt）为 0。没有启用任何调节。大约在 41 毫秒（41,494 即 106604046 - 106607480）时接收到了对这些字节的确认。

## Wireshark

wireshark 和 tshark 也可以用来显示 BBLog 文件。它们只在单个文件上操作，而不是像 read_bbrlog 那样对文件系列进行操作。当前不会显示特定事件信息。对于 TCP_LOG_IN 和 TCP_LOG_OUT 事件，BBLog 信息显示在帧信息中。对于所有其他事件，BBLog 信息直接显示。

兰德尔·斯图尔特（rrs@freebsd.org）是一位拥有 40 多年操作系统开发经验的自由 BSD 开发者，自 2006 年以来一直致力于 FreeBSD 开发。他擅长于传输，包括 TCP 和 SCTP，但也涉足操作系统的其他领域。他目前是一名独立顾问。

迈克尔·蒂克森（tuexen@freebsd.org）是明斯特应用科学大学的教授，Netflix 的兼职承包商，自 2009 年以来一直是 FreeBSD 源代码贡献者。他专注于传输协议，比如 SCTP 和 TCP，在 IETF 的标准化以及在 FreeBSD 中的实现。
