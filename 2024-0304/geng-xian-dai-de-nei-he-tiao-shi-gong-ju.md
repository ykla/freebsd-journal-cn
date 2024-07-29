# 更现代的内核调试工具

恐慌是一个真正美妙的词！它简洁地描述了一个极其复杂的情感事件。我们可以说“士兵们恐慌了”，我们就知道战斗是怎么进行的。我们可以用它来强调一个小小的疏忽，它解释了当我们踏上飞机并发现对烤箱当前状态存在疑虑时，我们的思绪究竟是如何的。

当然我把它关掉了。

这可能是我最喜欢在书封面上看到的术语之一：“恐慌！Unix 系统崩溃转储分析” ——哇！我一定要拥有这本书。任何使用这样标题的作者，即使它是一本旧的 SunOS/Solaris 技术手册，也肯定写得很棒。

优秀的书名比技术专著更具时间性，而第一个会过时的是现有系统的供应商级文档。到了 2024 年，我发现很难找到最近关于操作系统调试方法的任何作品。看看已出版的作品，如果你认为我们在 2004 年就已经完美化操作系统，那也不算冤枉。我从未使用过 SunOS 或 Solaris，但《恐慌！》为我提供了我一直想要的崩溃分析入门。

我得承认，我一直想知道为什么人们如此渴望核心转储——我常常会想，除了堆栈跟踪，我们还能得到什么呢。但通过“Panic！”，我能够迈出实际进行崩溃分析的第一步，它让我尝试使用 FreeBSD 上的最新内核调试工具。让我向你展示我学到的东西。别担心，我们不必在柠檬浸泡的纸餐巾纸旁等待。

## 获取内核转储

我要坦诚，我知道你不会尝试进行内核转储分析，直到你真正需要它。当你遇到死机的机器时。在那条道路上只有 printf 调试的生活。

获得一个内核进行调试并不难 — 您的系统需要设置以接收崩溃转储（参见 dumpon(8)），然后进入调试器。通常情况下，系统会通过内核恐慌来帮助您，协助您进入调试器的旅程。即使在没有发生任何问题的情况下，FreeBSD 也会乐意地让内核发生恐慌。在测试系统上将 debug.kdb.panic sysctl 设置为 1 将会使您陷入调试提示符：

`# sysctl debug.kdb.panic=1`

如果您在桌面电脑或云中的重要工作机器上运行了这些操作，您可能会陷入麻烦（如果这是您的桌面电脑，本文可能已经消失）。我建议您在虚拟机上学习内核调试，或者至少使用不会因持续崩溃而造成太多麻烦的系统。

另外，FreeBSD 虚拟机镜像默认配置为在启动时运行 savecore 并保存崩溃转储文件。

一旦设置了 debug.kdb.panic，系统将会陷入 ddb(4) 提示符。ddb 是一个完整的实时系统调试器 —— 它可以成为一个很好的分析工具，但今天不是我们想要的。

从 ddb 中，我们可以使用 dump 命令来转储运行中的内核。

`ddb> dumpDumping 925 out of 16047 MB:..2%..11%..21%..32%..42%..51%..61%..71%..82%..92%`

系统发生 panic 后将无法使用，因此您需要重新启动才能继续。

`ddb> reboot`

当您的虚拟机重新启动时，会收到来自 savecore 的关于提取和保存核心文件的消息。

核心文件将放置在 /var/crash 中，还有一些其他文件。

`$ ls /var/crashbounds core.txt.0 info.0 info.last minfreevmcore.0 vmcore.last`

我们测试的核心文件是 vmcore.0 ，它带有匹配的 info.0 和 core.txt.0 。信息文件是主机和转储的摘要，而 core.txt 则是转储文件的摘要，消息缓冲区的任何未读部分，以及如果存在，则是恐慌字符串和堆栈跟踪的摘要。

`Dump header from device: /dev/nvd0p3Architecture: amd64Architecture Version: 2Dump Length: 970199040Blocksize: 512Compression: noneDumptime: 2023-05-17 14:07:58 +0100Hostname: displacementactivityMagic: FreeBSD Kernel DumpVersion String: FreeBSD 14.0-CURRENT #2 main-n261806-d3a49f62a284: Mon Mar 27 16:15:25 UTC 2023tj@displacementactivity:/usr/obj/usr/src/amd64.amd64/sys/GENERICPanic String: Duplicate free of 0xfffff80339ef3000 from zone 0xfffffe001ec2ea00(malloc-2048) slab 0xfffff80325789168(0)Dump Parity: 3958266970Bounds: 0Dump Status: good`

`<p>The bounds file lets the dumper know the next coredump will be called<span> </span><code>vmcore.1</code><span> </span>and right now bounds on this machine:</p>``# cat /var/crash/bounds1`

最终，vmcore.last 是指向最近的核心转储文件的链接，以防您有一个有趣的星期并且迷失了最近的崩溃。

## 符号

我们需要处理核心转储的第二件事是内核符号。 发行版的内核符号可从 kernel-dbg 软件包获得，并安装到 /usr/lib/debug/ 中，或者可以从内核构建目录中提取。

## 使用 gdb 查看核心转储文件（首次）

首先，让我们快速查看一个核心转储文件，使用 kgdb 作为对比点，以便了解 lldb 崩溃转储调试的进展情况。

`$ kgdb kernel.debug vmcore.0Unread portion of the kernel message buffer:panic: Assertion !tcp_in_hpts(tp) failed at /usr/src/sys/netinet/tcp_subr.c:2432cpuid = 2time = 1706644478KDB: stack backtrace:db_trace_self_wrapper() at db_trace_self_wrapper+0x2b/frame 0xfffffe0047d2f480vpanic() at vpanic+0x132/frame 0xfffffe0047d2f5b0panic() at panic+0x43/frame 0xfffffe0047d2f610tcp_discardcb() at tcp_discardcb+0x25b/frame 0xfffffe0047d2f660tcp_usr_detach() at tcp_usr_detach+0x51/frame 0xfffffe0047d2f680sorele_locked() at sorele_locked+0xf7/frame 0xfffffe0047d2f6b0tcp_close() at tcp_close+0x155/frame 0xfffffe0047d2f6e0rack_check_data_after_close() at rack_check_data_after_close+0x8a/frame 0xfffffe0047d2f720rack_do_fin_wait_1() at rack_do_fin_wait_1+0x141/frame 0xfffffe0047d2f7a0rack_do_segment_nounlock() at rack_do_segment_nounlock+0x243b/frame 0xfffffe0047d2f9a0rack_do_segment() at rack_do_segment+0xda/frame 0xfffffe0047d2fa00tcp_input_with_port() at tcp_input_with_port+0x1157/frame 0xfffffe0047d2fb50tcp_input() at tcp_input+0xb/frame 0xfffffe0047d2fb60ip_input() at ip_input+0x2ab/frame 0xfffffe0047d2fbc0netisr_dispatch_src() at netisr_dispatch_src+0xad/frame 0xfffffe0047d2fc20ether_demux() at ether_demux+0x17a/frame 0xfffffe0047d2fc50ether_nh_input() at ether_nh_input+0x39f/frame 0xfffffe0047d2fca0netisr_dispatch_src() at netisr_dispatch_src+0xad/frame 0xfffffe0047d2fd00ether_input() at ether_input+0xd9/frame 0xfffffe0047d2fd60vtnet_rxq_eof() at vtnet_rxq_eof+0x73e/frame 0xfffffe0047d2fe20vtnet_rx_vq_process() at vtnet_rx_vq_process+0x9c/frame 0xfffffe0047d2fe60ithread_loop() at ithread_loop+0x266/frame 0xfffffe0047d2fef0fork_exit() at fork_exit+0x82/frame 0xfffffe0047d2ff30fork_trampoline() at fork_trampoline+0xe/frame 0xfffffe0047d2ff30--- trap 0, rip = 0, rsp = 0, rbp = 0 ---KDB: enter: panicReading symbols from /boot/kernel/zfs.ko...Reading symbols from /usr/lib/debug//boot/kernel/zfs.ko.debug...Reading symbols from /boot/kernel/tcp_rack.ko...Reading symbols from /usr/lib/debug//boot/kernel/tcp_rack.ko.debug...Reading symbols from /boot/kernel/tcphpts.ko...Reading symbols from /usr/lib/debug//boot/kernel/tcphpts.ko.debug...__curthread () at /usr/src/sys/amd64/include/pcpu_aux.h:5757 __asm(“movq %%gs:%P1,%0” : “=r” (td) : “n” (offsetof(struct pcpu,(kgdb)`

kgdb 启动时显示其许可证（已移除），然后打印消息缓冲区的最终部分，这是从 kgdb 添加的便利功能。消息缓冲区的最终部分告诉我们恐慌消息、信息和堆栈跟踪。

使用 kgdb bt （回溯）命令，我们可以获取堆栈跟踪，并使用 frames 命令在堆栈中移动，查看发生 panic 时的情况。

`(kgdb) bt...#10 0xffffffff80b51233 in vpanic (fmt=0xffffffff811f87ca “Assertion %s failed at %s:%d”, ap=ap@entry=0xfffffe0047d2f5f0) at /usr/src/sys/kern/kern_shutdown.c:953#11 0xffffffff80b51013 in panic (fmt=0xffffffff81980420 <cnputs_mtx> “\371\023\025\201\377\377\377\377”) at /usr/src/sys/kern/kern_shutdown.c:889#12 0xffffffff80d5483b in tcp_discardcb (tp=tp@entry=0xfffff80008584a80) at /usr/src/sys/netinet/tcp_subr.c:2432#13 0xffffffff80d60f71 in tcp_usr_detach (so=0xfffff800100b6b40) at /usr/src/sys/netinet/tcp_usrreq.c:215#14 0xffffffff80c01357 in sofree (so=0xfffff800100b6b40) at /usr/src/sys/kern/uipc_socket.c:1209#15 sorele_locked (so=so@entry=0xfffff800100b6b40) at /usr/src/sys/kern/uipc_socket.c:1236#16 0xffffffff80d545b5 in tcp_close (tp=<optimized out>) at /usr/src/sys/netinet/tcp_subr.c:2539#17 0xffffffff82e37e0a in tcp_tv_to_usectick (sv=0xfffffe0047d2f698) at /usr/src/sys/netinet/tcp_hpts.h:177#18 tcp_get_usecs (tv=0xfffffe0047d2f698) at /usr/src/sys/netinet/tcp_hpts.h:232...(kgdb) frame 12#12 0xffffffff80d5483b in tcp_discardcb (tp=tp@entry=0xfffff80008584a80) at /usr/src/sys/netinet/tcp_subr.c:2432warning: Source file is more recent than executable.2432(kgdb) list2427 #endif24282429 CC_ALGO(tp) = NULL;2430 if ((m = STAILQ_FIRST(&tp->t_inqueue)) != NULL) {2431 struct mbuf *prev;24322433 STAILQ_INIT(&tp->t_inqueue);2434 STAILQ_FOREACH_FROM_SAFE(m, &tp->t_inqueue, m_stailqpkt, prev)2435 m_freem(m);2436 }`

回顾一下，我列出了导致 panic 的回溯，确定了在第 11 号帧附近调用 panic，要求 kgdb 移动到第 12 号帧（导致 panic 的代码本身），然后列出了那里的代码。从这里进一步调查将有助于确定导致此次崩溃转储中的 panic 的原因。

这些是内核调试的基本步骤，查看正在进行的操作并查询崩溃转储以找出变量持有的值。lldb 需要能够在内核上下文中执行这些任务，以便有用。

## lldb

FreeBSD 过去十年来一直在转向更自由许可的 llvm/clang 工具链。有一段时间缺失的一环是调试，但到了 2024 年，FreeBSD 内核调试已经可以使用 lldb。

lldb 能够将 FreeBSD 内核转储文件导入为核心文件，并能浏览堆栈帧。

lldb 是由 Apple 开发的。我记得当他们将 gdb 中的默认调试器更改为 lldb 时，我遭受了严重的文化冲击。所有我从晦涩无文档的 gnu 世界中赢得的调试命令都消失了，被其他奇怪的命令取代了。

lldb 并不真正意味着要作为命令行界面使用，而是意味着通过 API 由软件驱动。这在许多命令的冗长中表现出来。幸运的是，lldb 在其命令行中增加了对更多类似 gdb 的命令的支持，这意味着更多的命令接口匹配。诸如打印之类的基本命令现在具有兼容的语法，但许多其他选项是不同的，要么更好，要么更糟。

## 使用 lldb 进行探索

lldb 不需要特殊配置来分析内核转储。在 lldb 中加载崩溃转储与 kgdb 相同，只是参数位置有些变化：

`$ lldb --core <corefile> path/to/kernel/symbols`

对于这些示例而言：

`$ lldb --core ../gdb/coredump/vmcore.0 ../gdb/coredump/kernel-debug/kernel.debug(lldb) target create “../gdb/coredump/kernel-debug/kernel.debug” --core “../gdb/coredump/vmcore.0”Core file '/home/tj/code/scripts/gdb/coredump/vmcore.0' (x86_64) was loaded.(lldb)`

这比起起始 kgdb 要安静得多，这很好，但它也错过了我们崩溃转储中的一些重要上下文。究竟是什么导致了这个转储？

kgdb 不能执行任何魔术（如果可以的话，它会有一个与“break”命令相匹配的“fix”命令）。 它所做的只是在崩溃转储中查找各种已知符号，并在启动时打印出来供我们使用。

我们可以自己做。

首先，内核中的恐慌消息存储在字符串 panicstr 中，并由 vpanic （在 kern/kern_shutdown.c 中）设置。 我们可以轻松从 lldb 转储中提取这个：

`(lldb) p panicstr(const char *) 0xffffffff819c1a00 “Assertion !tcp_in_hpts(tp) failed at /usr/src/sys/netinet/tcp_subr.c:2432”`

这可能足够让某人开始调试。我也喜欢可以通过 lldb 中的 bt 得到的堆栈跟踪：

`(lldb) bt* thread #1, name = '(pid 1025) tcplog_dumper'* frame #0: 0xffffffff80b83d2a kernel.debug`sched_switch(td=0xfffff800174be740, flags=259) at sched_ule.c:2297:26
frame #1: 0xffffffff80b5e9e3 kernel.debug`mi_switch(flags=259) at kern_synch.c:546:2frame #2: 0xffffffff80bb0dc4 kernel.debug`sleepq_switch(wchan=0xffffffff817e1448, pri=0) at subr_sleepqueue.c:607:2
frame #3: 0xffffffff80bb11a6 kernel.debug`sleepq_catch_signals(wchan=0xffffffff817e1448, pri=0) at subr_sleepqueue.c:523:3frame #4: 0xffffffff80bb0ef9 kernel.debug`sleepq_wait_sig(wchan=, pri=帧 #3: 0xffffffff80bb11a6 kernel.debug sleepq_catch_signals(wchan=0xffffffff817e1448, pri=0) at subr_sleepqueue.c:523:3frame #4: 0xffffffff80bb0ef9 kernel.debug sleepq_wait_sig(wchan=, pri=) at subr_sleepqueue.c:670:11
frame #5: 0xffffffff80b5df3c kernel.debug`_sleep(ident=0xffffffff817e1448, lock=0xffffffff817e1428, priority=256, wmesg=”tcplogdev”, sbt=0, pr=0, flags=256) at kern_synch.c:219:10frame #6: 0xffffffff8091190e kernel.debug`tcp_log_dev_read(dev=帧 #5: 0xffffffff80b5df3c kernel.debug _sleep(ident=0xffffffff817e1448, lock=0xffffffff817e1428, priority=256, wmesg=”tcplogdev”, sbt=0, pr=0, flags=256) at kern_synch.c:219:10frame #6: 0xffffffff8091190e kernel.debug tcp_log_dev_read(dev=, uio=0xfffffe0079b4ada0, flags=0) at tcp_log_dev.c:303:9
frame #7: 0xffffffff809d99ce kernel.debug`devfs_read_f(fp=0xfffff80012857870, uio=0xfffffe0079b4ada0, cred=<unavailable>, flags=0, td=0xfffff800174be740) at devfs_vnops.c:1413:10frame #8: 0xffffffff80bc9bc6 kernel.debug`dofileread [inlined] fo_read(fp=0xfffff80012857870, uio=0xfffffe0079b4ada0, active_cred=, flags=帧 #7: 0xffffffff809d99ce kernel.debug devfs_read_f(fp=0xfffff80012857870, uio=0xfffffe0079b4ada0, cred=<unavailable>, flags=0, td=0xfffff800174be740) at devfs_vnops.c:1413:10frame #8: 0xffffffff80bc9bc6 kernel.debug dofileread [inlined] fo_read(fp=0xfffff80012857870, uio=0xfffffe0079b4ada0, active_cred=, flags=, td=0xfffff800174be740) at file.h:340:10
框架 #9: 0xffffffff80bc9bb4 kernel.debug dofileread(td=0xfffff800174be740, fd=3, fp=0xfffff80012857870, auio=0xfffffe0079b4ada0, offset=-1, flags=0) at sys_generic.c:365:15frame #10: 0xffffffff80bc9712 kernel.debug sys_read [inlined] kern_readv(td=0xfffff800174be740, fd=3, auio=0xfffffe0079b4ada0) at sys_generic.c:286:10
框架 #11: 0xffffffff80bc96dc kernel.debug sys_read(td=0xfffff800174be740, uap=<unavailable>) at sys_generic.c:202:10frame #12: 0xffffffff810556a3 kernel.debug amd64_syscall [inlined] syscallenter(td=0xfffff800174be740) at subr_syscall.c:186:11
框架 #13: 0xffffffff81055581 kernel.debug amd64_syscall(td=0xfffff800174be740, traced=0) at trap.c:1192:2frame #14: 0xffffffff8102781b kernel.debug fast_syscall_common at exception.S:578

也可以从 lldb 中选择一个有趣的帧来查看：

`(lldb) frame select 12frame #12: 0xffffffff810556a3 kernel.debug`amd64_syscall [inlined] syscallenter(td=0xfffff800174be740) at subr_syscall.c:186:11
183 if (!sy_thr_static)
184 系统调用线程退出（td，se）；
 185 } 否则 {
-> 186 错误 =（se->sy_call）（td，sa->args）；
187 /* 保存最新的错误返回值。 */
188 if (__predict_false((td->td_pflags & TDP_NERRNO) != 0))
189 td->td_pflags &= ~TDP_NERRNO;`

## 获取内核缓冲区

打印东西并在堆栈中移动是内核崩溃转储调试中我们所需要的大部分。 gdb 的启动消息相当不错，显示的是内核消息缓冲区的最后部分，好像直接来自本地控制台一样。

lldb 目前还没有像那样友好的启动命令。幸运的是，“Panic！”给我们一些关于如何提取这些信息的提示。“Panic！”使用一个名为“ msgbuf ”的宏来从 struct msgbuf 打印内核消息缓冲区。

在 FreeBSD 源代码中进行一些探索，我们找到了类似的内容可用：

`(lldb) p *msgbufp(msgbuf) {msg_ptr = 0xfffff8001ffe8000 “---<<BOOT>>---\nCopyright (c) 1992-2023 The FreeBSD Project.\nCopyright (c) 1979, 1980, 1983, 1986, 1988, 1989, 1991, 1992, 1993, 1994\n\tThe Regents of the University of California. All rights reserved.\nFreeBSD is a registered trademark of The FreeBSD Foundation.\nFreeBSD 15.0-CURRENT #0 main-272a40604: Wed Nov 29 13:42:38 UTC 2023\n tj@vpp:/usr/obj/usr/src/amd64.amd64/sys/GENERIC amd64\nFreeBSD clang version 16.0.6 (https://github.com/llvm/llvm-project.git llvmorg-16.0.6-0-g7cbf1a259152)\nWARNING: WITNESS option enabled, expect reduced performance.\nVT: init without driver.\nCPU: 12th Gen Intel(R) Core(TM) i7-1260P (2500.00-MHz K8-class CPU)\n Origin=\”GenuineIntel\” Id=0x906a3 Family=0x6 Model=0x9a Stepping=3\n Features=0x9f83fbff<FPU,VME,DE,PSE,TSC,MSR,PAE,MCE,CX8,APIC,SEP,MTRR,PGE,MCA,CMOV,PAT,PSE36,MMX,FXSR,SSE,SSE2,SS,HTT,PBE>\n Features2=0xfeda7a17<SSE3,PCLMULQDQ,DTES64,DS_CPL,SSSE3,SDBG,FMA,CX16,xTPR,PCID,SSE4.1,SSE4.2,MOVBE,POPCNT,AESNI,XSAVE,OSXSAVE,AVX,F16C,RDRAND,HV>\n AMD Features=0x2c100800<SYSCALL,”...msg_magic = 405602msg_size = 98232msg_wseq = 16777msg_rseq = 15001msg_cksum = 1421737msg_seqmod = 1571712msg_lastpri = -1msg_flags = 0msg_lock = {lock_object = {lo_name = 0xffffffff81230bcc “msgbuf”lo_flags = 196608lo_data = 0lo_witness = NULL}mtx_lock = 0}}`

我们在内核中有一个全局可见的结构 msgbuf ，实现了内核的消息缓冲区。lldb 显示我们缓冲区的起始位置。字段 msg_wseq 和 msg_rseq 告诉我们写入和读取的位置。

读取消息缓冲区中未读部分很容易：

`(lldb) p msgbufp->msg_ptr+msgbufp->msg_rseq(char *) 0xfffff8001ffeba99 “panic: Assertion !tcp_in_hpts(tp) failed at /usr/src/sys/netinet/tcp_subr.c:2432\ncpuid = 2\ntime = 1706644478\nKDB: stack backtrace:\ndb_trace_self_wrapper() at db_trace_self_wrapper+0x2b/frame 0xfffffe0047d2f480\nvpanic() at vpanic+0x132/frame 0xfffffe0047d2f5b0\npanic() at panic+0x43/frame 0xfffffe0047d2f610\ntcp_discardcb() at tcp_discardcb+0x25b/frame 0xfffffe0047d2f660\ntcp_usr_detach() at tcp_usr_detach+0x51/frame 0xfffffe0047d2f680\nsorele_locked() at sorele_locked+0xf7/frame 0xfffffe0047d2f6b0\ntcp_close() at tcp_close+0x155/frame 0xfffffe0047d2f6e0\nrack_check_data_after_close() at rack_check_data_after_close+0x8a/frame 0xfffffe0047d2f720\nrack_do_fin_wait_1() at rack_do_fin_wait_1+0x141/frame 0xfffffe0047d2f7a0\nrack_do_segment_nounlock() at rack_do_segment_nounlock+0x243b/frame 0xfffffe0047d2f9a0\nrack_do_segment() at rack_do_segment+0xda/frame 0xfffffe0047d2fa00\ntcp_input_with_port() at tcp_input_with_port+0x1157/frame 0xfffffe0047d2fb50\ntcp_input() at tcp_input+0xb/frame 0xfffffe0047d2fb60\nip_input() at ip_in”...`

输出的格式不太友好，控制字符只是被打印出来，但我们可以读取内核消息缓冲区。在完整的回溯可用之前截断输出。让我们尝试一些其他命令：

`(lldb) x/b msgbufp->msg_ptr+msgbufp->msg_rseq0xfffff8001ffeba99: “panic: Assertion !tcp_in_hpts(tp) failed at /usr/src/sys/netinet/tcp_subr.c:2432\ncpuid = 2\ntime = 1706644478\nKDB: stack backtrace:\ndb_trace_self_wrapper() at db_trace_self_wrapper+0x2b/frame 0xfffffe0047d2f480\nvpanic() at vpanic+0x132/frame 0xfffffe0047d2f5b0\npanic() at panic+0x43/frame 0xfffffe0047d2f610\ntcp_discardcb() at tcp_discardcb+0x25b/frame 0xfffffe0047d2f660\ntcp_usr_detach() at tcp_usr_detach+0x51/frame 0xfffffe0047d2f680\nsorele_locked() at sorele_locked+0xf7/frame 0xfffffe0047d2f6b0\ntcp_close() at tcp_close+0x155/frame 0xfffffe0047d2f6e0\nrack_check_data_after_close() at rack_check_data_after_close+0x8a/frame 0xfffffe0047d2f720\nrack_do_fin_wait_1() at rack_do_fin_wait_1+0x141/frame 0xfffffe0047d2f7a0\nrack_do_segment_nounlock() at rack_do_segment_nounlock+0x243b/frame 0xfffffe0047d2f9a0\nrack_do_segment() at rack_do_segment+0xda/frame 0xfffffe0047d2fa00\ntcp_input_with_port() at tcp_input_with_port+0x1157/frame 0xfffffe0047d2fb50\ntcp_input() at tcp_input+0xb/frame 0xfffffe0047d2fb60\nip_input() at ip_i”warning: unable to find a NULL terminated string at 0xfffff8001ffeba99. Consider increasing the maximum read length.(lldb) x/2048b msgbufp->msg_ptr+msgbufp->msg_rseqerror: Normally, ‘memory read’ will not read over 1024 bytes of data.error: Please use --force to override this restriction just once.error: or set target.max-memory-read-size if you will often need a larger limit.`

我们达到打印限制了，尽管我尽力了，我也无法说服 lldb 再往前走。是时候使用更高级的工具了。

## 月球上的一些帮助

lldb 还为控制提供了一个脚本界面，这解释了为什么许多命令键入起来非常冗长。目前，lldb 支持使用 C ++、Python 进行脚本编写，并具有 Lua 的实验性支持。FreeBSD 在基础中集成了 Lua，并且 2024 年的 FreeBSD 构建版本默认包含 Lua 支持。

我们可以通过以下方式简单地尝试一下：

`(lldb) script>>> print(“hello esteemed FreeBSD Journal readers!”)hello esteemed FreeBSD Journal readers!>>> quit`

在 >>> 提示符指示下，我们已经进入 Lua 解释器。

从“Panic！”我们知道，adb SunOS/Solaris 调试器有一个方便易懂的宏，用于查找和打印消息缓冲区。

`msgbuf/”magic”16t”size”16t”bufx”16t”bufr”n4X+,(*msgbuf+0t8)-*(msgbuf+0t12)))&80000000$<msgbuf.wrap.+*(msgbuf+0t12),(*(msgbuf+0t8)-*(msfbuf+0t12))/c`

使用 Lua 实现类似的机制应该毫无问题，以此为例子。

lldb lua 接口是从 swig 绑定中生成的，这是一种描述库之间接口的 C++格式。Python 和 Lua 绑定是以相同方式生成的。关于 API 或如何使用它的任何问题，您可以从可从 lldb 项目获得的 Python API 文档中弄清楚。这是一种非常笨拙的做事方式，但是是可行的。

我很快就厌倦了在解释器中输入命令，考虑到其中一些命令的长度，尝试运行它们可能会很烦人。一旦解释器运行一次，lldb 可以从文件加载 Lua 脚本。从一个全新的会话开始：

`$ lldb --core coredump/vmcore.1 coredump/kernel-debug/kernel.debug(lldb) target create “coredump/kernel-debug/kernel.debug” --core “coredump/vmcore.1”Core file ‘/home/tj/code/scripts/gdb/coredump/vmcore.1’ (x86_64) was loaded.(lldb) script>>> print(“hello”)hello>>> quit(lldb) command script import ./hello.luahello from the script hello.lua`

`Assuming the file hello.lua contains:`

`print(“hello from the script hello.lua”)`

lldb Lua 环境提供一个 lldb 变量，其成员使得可以访问目标、调试器、帧、进程和线程。这些对象与 Python API 中描述的对象相对应。

我并不是很喜欢 lldb API，它编写起来可能会相当笨拙，并且如果你在选择函数或者变量在内存中的布局方面遇到问题时，理解起来也会很困难。

一旦你有了一些经验，就会更容易理解它对你的要求。

让我举个例子，用 lldb Lua 绑定来打印崩溃转储中的消息缓冲区。

从 lldb Lua 变量中，我们可以访问转储图像中的文件。当我开始做核心转储分析时，一个大的障碍是理解如何在内存中定位事物。有各种内核全局变量可以作为起点访问，大多数子系统都有一些可供构建的东西。

正如我们之前所看到的， msgbufp 是内核消息缓冲区的全局实例。从 lldb Lua，我们可以通过以下方式访问它：

`msgbuf = lldb.target:FindFirstGlobalVariable(“msgbufp”)`

这给我们一个表示内存中该结构实例的 SBValue 实例，从核心转储中。我们可以使用 GetChildMemberWithName 方法访问结构的子成员，例如 msg_rseq。

lldb.process 对象使我们能够从内核转储中读取内存。有时候，需要做一些调整来获取正确的引用、地址和值，以执行您想要的操作。

通过这些方法，我们可以组装到消息缓冲区开头的点，从核心转储中读取它，并使用 Lua 打印它。我把所有这些都放入一个名为 msgbuf.lua 的脚本中：

`msgbuf = lldb.target:FindFirstGlobalVariable(“msgbufp”)msgbuf_start = msgbuf:GetChildMemberWithName(“msg_rseq”):GetValue()msgbuf_end = msgbuf:GetChildMemberWithName(“msg_wseq”):GetValue()unread_len = msgbuf_end - msgbuf_startmsgbuf_addr = msgbuf:GetChildMemberWithName(“msg_ptr”):Dereference():GetLoadAddress() + msgbuf_startmsgbuf_ptr = lldb.process:ReadMemory(msgbuf_addr, unread_len, lldb.SBError())print(“Unread portion of the kernel message buffer:”)print(msgbuf_ptr)`

如果我们从 lldb 会话中运行这个命令，我们会获得以下输出：

`(lldb) command script import ./msgbuf.luaUnread portion of the kernel message buffer:panic: Assertion !tcp_in_hpts(tp) failed at /usr/src/sys/netinet/tcp_subr.c:2432cpuid = 2time = 1706644478KDB: stack backtrace:db_trace_self_wrapper() at db_trace_self_wrapper+0x2b/frame 0xfffffe0047d2f480vpanic() at vpanic+0x132/frame 0xfffffe0047d2f5b0panic() at panic+0x43/frame 0xfffffe0047d2f610tcp_discardcb() at tcp_discardcb+0x25b/frame 0xfffffe0047d2f660tcp_usr_detach() at tcp_usr_detach+0x51/frame 0xfffffe0047d2f680sorele_locked() at sorele_locked+0xf7/frame 0xfffffe0047d2f6b0tcp_close() at tcp_close+0x155/frame 0xfffffe0047d2f6e0rack_check_data_after_close() at rack_check_data_after_close+0x8a/frame 0xfffffe0047d2f720rack_do_fin_wait_1() at rack_do_fin_wait_1+0x141/frame 0xfffffe0047d2f7a0rack_do_segment_nounlock() at rack_do_segment_nounlock+0x243b/frame 0xfffffe0047d2f9a0rack_do_segment() at rack_do_segment+0xda/frame 0xfffffe0047d2fa00tcp_input_with_port() at tcp_input_with_port+0x1157/frame 0xfffffe0047d2fb50tcp_input() at tcp_input+0xb/frame 0xfffffe0047d2fb60ip_input() at ip_input+0x2ab/frame 0xfffffe0047d2fbc0netisr_dispatch_src() at netisr_dispatch_src+0xad/frame 0xfffffe0047d2fc20ether_demux() at ether_demux+0x17a/frame 0xfffffe0047d2fc50ether_nh_input() at ether_nh_input+0x39f/frame 0xfffffe0047d2fca0netisr_dispatch_src() at netisr_dispatch_src+0xad/frame 0xfffffe0047d2fd00ether_input() at ether_input+0xd9/frame 0xfffffe0047d2fd60vtnet_rxq_eof() at vtnet_rxq_eof+0x73e/frame 0xfffffe0047d2fe20vtnet_rx_vq_process() at vtnet_rx_vq_process+0x9c/frame 0xfffffe0047d2fe60ithread_loop() at ithread_loop+0x266/frame 0xfffffe0047d2fef0fork_exit() at fork_exit+0x82/frame 0xfffffe0047d2ff30fork_trampoline() at fork_trampoline+0xe/frame 0xfffffe0047d2ff30--- trap 0, rip = 0, rsp = 0, rbp = 0 ---KDB: enter: panic`

Lua 友好地为我们扩展了缓冲区中的控制字符，为我们提供了来自消息缓冲区的良好格式化的输出。

## 更好的调试可能性

lldb 是在内核调试方面的新生力量，Lua 环境中仍有许多功能尚未支持，但它已具备足够的功能成为一个有用的工具。老牌的 gdb 用户可能会觉得这些示例的价值不大，毕竟 lldb 添加了更为复杂的语法，可能会认为这种变化只是为了变化。

lldb 及其内置的 Lua 带来的一个重大价值是在发布的 FreeBSD 镜像中进行了集成。lldb Lua 具有自由许可证，并与 FreeBSD 兼容，从 2024 年初开始，它已默认启用在 CURRENT 构建中。这使得内核开发人员和故障排除人员能够在 lldb Lua 中编写脚本，并提供给用户进行分析。

kgdb 长期以来一直支持 gdb 脚本，但这并不是最愉快的脚本语言进行编程。另一方面，Lua 虽然有点奇怪，但在许多环境中被广泛使用，并且是 FreeBSD 引导加载程序的一部分。我编写了一个工具，用于从崩溃的内核映像中提取 TCP 日志文件 - 主要的困难是弄清楚如何获取内存。一旦有了数据，创建并将其写入文件就变得很容易。

内核转储包含其中的所有内容，可能包含敏感信息。一个合理的脚本语言可以使开发人员提供脚本，从内核映像中提取更多的调试信息而无需移动大型核心转储文件，也无需处理担心信任陌生人可能包含敏感信息。

Tom Jones 是一位 FreeBSD 提交者，致力于保持网络堆栈的速度。
