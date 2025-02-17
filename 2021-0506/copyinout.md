# copyinout 框架

——FreeBSD 和 CheriBSD 中的 ABI 独立、类型感知、能力感知、copyin 和 copyout API

- 原文链接：[The copyinout Framework](https://freebsdfoundation.org/wp-content/uploads/2021/07/The-copyinout-Framework.pdf)
- 作者：** KONRAD WITASZCZYK**

在用户空间和内核空间之间的内存复制是系统调用中的一个关键操作。它用于复制系统调用的参数以及系统调用的结果。目前的复制函数原型是与类型无关的，复制一定数量的字节从任意缓冲区。当内核将其内存复制到用户空间时，必须确保不会泄露任何可能包含机密数据的内核数据。本文描述了 FreeBSD 和 CheriBSD 中当前复制函数的局限性，并提出了一个框架，可以提高系统调用处理程序的安全性和代码质量。

所描述的 copyinout 框架是作为题为《能力感知内存复制在地址空间之间》的硕士论文的一部分实现的，该论文在哥本哈根大学的 Ken Friis Larsen 和微软研究院的 David Chisnall 的指导下完成。类型感知的 copyin 和 copyout API 的最初构想是由 David Chisnall 提出的。

## FreeBSD 中的内存复制函数

FreeBSD 包含两个主要函数，用于在地址空间之间复制内存：copyin() 和 copyout()（见列表 1）。这两个函数接受三个参数：源地址、目标地址和要复制的字节数。copyin() 将 len 字节从用户空间地址 udaddr 复制到内核空间地址 kaddr。copyout() 方向相反，将 len 字节从内核空间地址 kaddr 复制到用户空间地址 udaddr。这些函数在成功时返回 0，如果传递了无效地址，则返回 EFAULT。

```c
int copyin(const void * __restrict udaddr, void * _Nonnull __restrict kaddr, size_t len);
int copyout(const void * _Nonnull __restrict kaddr, void * __restrict udaddr, size_t len);
```

**列表 1.** FreeBSD 13.0-RELEASE 中的 copyin() 和 copyout() 函数原型

从函数原型中，我们可以识别出一个潜在的安全问题。复制函数操作的是任意缓冲区。如果缓冲区包含一个结构体对象，而该结构体字段之间有填充，那么填充部分也会被复制。包含敏感信息的填充泄露通常被称为内核内存泄露或内核内存泄漏。这类漏洞可能导致特权提升。它们不仅仅是 FreeBSD 特有的，许多操作系统中都可以发现类似问题 [3] [4] [5]，并且它们已成为广泛的检测 [6] 和缓解 [7] [8] [9] 技术研究的主题。

复制函数被系统调用处理程序使用，用于将系统调用参数从用户空间复制到内核空间，并将系统调用结果从内核空间复制到用户空间。由于系统调用非常频繁地执行，内核开发人员必须提供最低开销的复制函数实现。这可以通过提供与机器相关的汇编语言实现来实现。根据 CPU 架构，复制函数还可以利用 CPU 模型提供的安全功能。例如，针对 amd64（见 amd64/amd64/support.S）的实现支持监督模式访问防护（SMAP）。

## ABI 支持

FreeBSD 支持多种 ABI。具体而言：

* 针对与内核编译目标相同的程序的本地 ABI；
* 针对内核编译的架构的 32 位版本的 32 位 ABI；
* 针对 Linux 用户空间程序的 Linux ABI

这些 ABI 作为兼容层实现。每个兼容层提供其系统调用处理程序，这些处理程序实现了在内核进入或返回内核例程时，必须执行的额外逻辑。这包括在地址空间之间复制和转换对象。例如，对于 32 位 ABI，系统调用参数或系统调用结果中的指针必须从 32 位指针转换或转换为 32 位指针。

## 系统调用处理程序

每当用户空间程序发出系统调用时，内核会调用一个特定的系统调用处理程序，并将已复制的系统调用参数传入。对于每个支持的 ABI，内核保持一个 sysentvec 结构对象（见列表 2），描述 ABI 特定的属性和在程序执行时内核将使用的函数。该结构包括包含系统调用处理程序函数指针的 sv_table 数组，这些指针存储在由相应的系统调用编号指定的位置，以及指向一个架构特定函数的 sv_fetch_syscall_args 函数指针，该函数用于复制系统调用参数。

作为示例，我们来考虑 jail(2) 系统调用。这个系统调用有一个参数：指向 jail 结构对象的指针（见列表 3）。jail 结构包含描述监狱的参数（见列表 4）。一旦用户空间程序执行 jail 系统调用并进入特权模式，内核将调用 jail 系统调用处理程序——sys_jail() 函数（见列表 5），并将已经复制进来的 jail 系统调用参数——jail_args 结构对象传入。系统调用处理程序复制一个 jail 结构，并调用实现 jail 系统调用逻辑的 kern_jail 内核例程。

```c
struct sysentvec {
 (...)
 struct sysent *sv_table;
 (...)
 int (*sv_fetch_syscall_args)(struct thread *);
 (...)
}
```

列表 2. 描述 ABI 特定属性和函数的 sysentvec 结构。

```c
struct jail_args {
 char jail_l_[PADL_(struct jail *)];
 struct jail * jail;
 char jail_r_[PADR_(struct jail *)];
};
```

列表 3. 本地 ABI 的 jail 系统调用参数结构。

```c
struct jail {
 uint32_t version;
 char *path;
 char *hostname;
 char *jailname;
 uint32_t ip4s;
 uint32_t ip6s;
 struct in_addr *ip4;
 struct in6_addr *ip6;
};

```

列表 4. 本地 ABI 的 jail 结构。

```c
int
sys_jail(struct thread *td, struct jail_args *uap)
{
 (...)
 int error;
 struct jail j;
 (...)
 error = copyin(uap->jail, &j, sizeof(struct jail));
 if (error)
 return (error);
 (...)
 return (kern_jail(td, &j));
}
```

列表 5. 本地 ABI 的 jail 系统调用处理程序。

## 兼容层

兼容层为非本地 ABI 提供系统调用实现。特别地，32 位 ABI 被实现为 freebsd32 兼容层。让我们考虑 32 位 ABI 下相同的 jail 系统调用。32 位版本的 jail 参数结构称为 freebsd32_jail_args（见列表 6），它包括指向 32 位版本的 jail 结构（称为 jail32，见列表 7）的指针。jail 和 jail32 结构之间的唯一区别是架构无关的字段类型。每个指针都被替换为 32 位无符号整数。由于指针在 32 位架构中是 32 位无符号整数，这一变化保证了为 64 位内核编译的 jail32 结构与为 32 位内核编译的 jail 结构具有相同的布局。

32 位 ABI 的 jail 系统调用处理程序实现为 freebsd32_jail 函数（见列表 8）。该函数复制一个 32 位的 jail 对象，使用宏（见列表 9）为其本地 ABI 版本翻译每个字段，并调用与本地 ABI 情况下相同的 kern_jail 内核例程。这意味着本地 ABI 和 32 位 ABI 在 jail 系统调用处理程序中的唯一区别是将用户空间的 jail 对象转换为可以被内核使用的本地 ABI 版本。

```c
struct freebsd32_jail_args {
 char jail_l_[PADL_(struct jail32 *)];
 struct jail32 * jail;
 char jail_r_[PADR_(struct jail32 *)];
};
```

列表 6. 32 位 ABI 的 jail 系统调用参数结构。

```c
struct jail32 {
 uint32_t version;
 uint32_t path;
 uint32_t hostname;
 uint32_t jailname;
 uint32_t ip4s;
 uint32_t ip6s;
 uint32_t ip4;
 uint32_t ip6;
};
```

列表 7. 32 位 ABI 的 jail 结构。

```c
int
freebsd32_jail(struct thread *td, struct freebsd32_jail_args *uap)
{
 (...)
 int error;
 struct jail j;
 (...)
 struct jail32 j32;
 error = copyin(uap->jail, &j32, sizeof(struct jail32));
 if (error)
 return (error);
 CP(j32, j, version);
 PTRIN_CP(j32, j, path);
 PTRIN_CP(j32, j, hostname);
 PTRIN_CP(j32, j, jailname);
 CP(j32, j, ip4s);
 CP(j32, j, ip6s);
 PTRIN_CP(j32, j, ip4);
 PTRIN_CP(j32, j, ip6);
 (...)
 return (kern_jail(td, &j));
}
```

列表 8. 32 位 ABI 的 jail 系统调用处理程序。

```c
#define CP(src, dst, fld) do { \
 (dst).fld = (src).fld; \
} while (0)
#define PTRIN(v) (void *)(uintptr_t)(v)
#define PTRIN_CP(src, dst, fld) do { \
 (dst).fld = PTRIN((src).fld); \
} while (0)
```

列表 9. 32 位 ABI 的 jail 系统调用处理程序使用的辅助宏

## copyinout API

前面描述的内核内存泄漏和代码重复问题是由于 copy 函数缺乏类型意识引起的。如果 copyin() 和 copyout() 函数能理解存储在复制缓冲区中的底层对象的结构，它们就能复制对象的字段并在用户进程 ABI 与内核 ABI 不同的情况下进行转换。

为了解决这些问题，我们引入了 copyinout API。对于每个在用户空间和内核空间之间复制的类型 foo，我们引入了类型感知的 copy 函数变体（见列表 10），它们会复制类型 foo 的每个字段，并根据源 ABI 转换为目标 ABI，例如，将 32 位指针转换为 64 位指针，适用于 64 位架构上的 32 位进程。此外，copy 函数还可以根据 CPU 型号执行额外的操作，例如，针对 CHERI CPU 编译的内核可以创建 CHERI 能力来为字段设置边界或权限。与原始 copy 函数不同，类型感知的 copy 函数只需要两个参数：源地址和目标地址。copyin_foo() 将存储在 uaddr 用户空间地址的对象复制到存储在 kaddr 内核空间地址的对象。copyout_foo() 以相反的方式工作，将存储在 kaddr 内核空间地址的对象复制到存储在 uaddr 用户空间地址的对象。例如，前面描述的 jail 结构可以使用以下函数调用进行复制：

```c
copyin_jail(uap->jail, &j);
```

```c
int copyin_foo(const void *uaddr, struct foo *kaddr);
int copyout_foo(const struct foo *kaddr, const void *uaddr);
```

列表 10. 类型 foo 的 copyin() 和 copyout() 函数变体

## copyinout 框架

类型感知的 copy 函数的实现与 copyinout API 本身是独立的。为了提供这些实现，我们引入了 copyinout 框架，该框架由以下部分组成：

* 描述应该为某个类型生成哪些 copy 函数的类型注解；
* 每个 ABI 的 copy 函数指针表；
* 一个内核接口，用于动态注册 copy 函数并调用它们；
* 一个代码生成工具，用于根据类型注解生成 copy 函数

内核定义了类型注解，指示应该为某个类型生成什么样的 copy 函数：

* `__copyin` 生成 copyin();
* `__copyout` 生成 copyout();
* `__copyinout` 生成 copyin() 和 copyout()。

