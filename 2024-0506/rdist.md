# rdist

作者：CY SCHUBERT

## 什么是 RDIST？

引用手册的话说，“rdist 是在多个主机之间实现文件同步的程序。” rdist 是一款可用于多种用途的通用工具，比如在网络上确保文件的同步，类似于 rsync 和 unison。又比如作为分布式配置管理工具，如 cfengine、ansible、puppet、salt 和其他配置管理工具。

## 为什么使用 RDIST？

要理解为什么要使用 rdist，了解点历史能让我们更好地了解它因何产生。

Ralph Campbell 在1983-1984 年间的 UCB 编写了它。rdist 首次出现在 4.3BSD（1986 年），是第一个解决分布式软件管理问题的软件。从 20 世纪 80 年代末到 90 年代初，它几乎随所有商业 UNIX 一同分发。当时它成为了远程平台管理的标准。

rdist 早于其它同类软件。甚至早于 rsync。Rsync 是一款备份工具，用于备份和克隆目录树，而 rdist 通常用作网络文件分发工具。它们都根据其设计目的进行了定制。

有了这么多选择，为什么要用 rdist 呢？

* rdist 比后辈配置分发应用程序（如 ansible 等）更轻量化。
* rdist 容易集成到 shell 脚本和 Makefiles 中。
* rsync 不能像 rdist 一样并行分发到多个主机。rsync 也不能使用配置文件来同步文件，但是 rdist 可以。Rsync 和 rdist 设计用于不同目的，rsync 用于备份和文件克隆，而 rdist 更适合用作配置管理工具。

另一方面，为什么有人想使用另外的工具呢？与诸如 cfengine 和 ansible 之类的工具相比，rdist 更轻量化，其配置远程节点的能力仅限于分发文件和执行简单的分发后任务。而更重量级的工具可以用来执行分发前任务，通过简单的 shell 脚本和 Makefile 就能解决此问题。以我自己为例，我使用一款叫 ipfmeta 的配置工具来管理我的 ipfilter 防火墙规则，该工具用规则文件和对象文件生成防火墙配置文件，然后使用 Makefile 将生成的文件分发到写在 rdist 的 Distfile 中远程防火墙上。Distfile 与 rdist 的关系有点类似于 Makefile 与 make。与 rsync 不同，rdist 使用在其 Distfile 中具体的规则来分发文件。

## RDIST 是如何工作的？

就像 make（1）解析其 Makefile 来构建程序一样，rdist 解析其 Distfile，获得要分发的文件和目录以及要在分发后执行哪些任务。最初，rdist 使用不安全的 rcmd（3）接口进行网络通信。rcmd（）会连接到远程 rshd（8）。当连接建立时，它会生成 rdistd（8）远程文件分发服务器，在远程服务器上执行分发功能。这类似于 ssh 的 sftp，ssh 给 sftp 提供了远程功能。

伯克利的 “r” 命令（如 rsh）是不安全的。今天的 rdist 传输实现可基于 ssh，而不用 rsh。使用 ssh 可以使用 ssh 密钥和 GSSAPI（kerberos）身份验证。与 ansible 不同，在 ansible 中连接是使用您自己的账户进行的，并且通过“become”进行特权升级，rdist 必须登录到目标服务器上的 root。为了便于实现这一点，可以将 sshd_config 中的 PermitRootLogin 设置为 prohibit-password，从而强制使用 ssh 密钥和 Kerberos 凭据。

rdist 本身不进行身份验证。它依赖于传输机制进行身份验证。与 ansible 相比，它还依赖于 ssh 传输机制进行身份验证，并依赖于 su(1)、sudo(1)或 ksu(1)进行权限提升。

rdist 可用于管理服务帐户中的应用程序文件，例如 mysql、oracle 或其他应用程序帐户。用所需的帐户名替换 root。

rdistd(8) 必须在目标服务器的用户搜索路径($PATH)中。

rdist 协商协议版本。systuils/rdist6 port/package 使用 RDIST 版本 6 协议，而 sysutils/rdist7（alpha）使用 RDIST 版本 7 协议。

## 安装 RDIST

要安装 rdist，只需，

`pkg install rdist6`

 或者

`pkg install rdist7`

或者使用 ports，以 rdist7 为例，

`cd /usr/ports/sysutils/rdist7make install clean`

## 使用 RDIST

類似於主機 make 使用其配置文件，如前所述，rdist 使用類似於其配置文件的配置文件。我們必須構建我們的 Distfile。

有三種 Distfile 聲明類型。

### Distfile

像 make 一样，rdist 寻找一个名为 Distfile 或 distfile 的文件。就像我们可以覆盖 make 使用的 Makefile 的名称一样，我们也可以覆盖 rdist Distfile 的名称。

Distfiles 包含一系列条目，指定要分发（复制）的文件，要将这些文件复制到哪些节点，以及在分发文件后要执行的后续操作。

### 变量

可以使用以下格式将一个或多个项目分配给变量。

`<variable name> '=' <name list>`

 例如，

`HOSTS = ( matisse root@arpa )`

此将字符串 matisse 和 root@arpa 定义为变量 HOSTS。

另一个示例将三个目录名分配给名为 FILES 的变量。

`FILES = ( /bin /lib /usr/bin /usr/games )`

### 将文件分发到其他主机

第二种语句类型告诉 rdist 将文件分发到其他主机。其格式是，

`[ label: ] <source list> '->' <destination list> <command list>`

是文件或变量的名称。是要将文件复制到的主机列表。在复制操作中，是要应用的一组 rdist 指令。
