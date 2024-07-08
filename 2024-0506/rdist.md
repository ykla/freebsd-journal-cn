# rdist

由 CY SCHUBERT 撰写

## 什么是 RDIST？

引用手册的话说，“rdist 是一个用于在多个主机之间保持文件相同副本的程序。” rdist 是一个通用工具，可以用于多种用途，比如在网络上保持文件的一致副本，类似于 rsync 和 unison 的功能，或作为分布式配置管理工具，如 cfengine、ansible、puppet、salt 或其他配置管理工具。

## 为什么使用 RDIST？

要理解为什么要了解 rdist，了解一点它的历史将让我们更好地了解它为何被创建。

Ralph Campbell 在 UCB 于 1983-1984 年编写，rdist 首次出现在 4.3BSD（1986 年），是第一个解决分布式软件管理问题的应用程序之一。在 20 世纪 80 年代末和 90 年代期间，它几乎与每个商业 UNIX 一起分发。当时它成为远程平台管理的标准。

rdist 早于其它同类软件。它也早于 rsync。Rsync 是一个备份工具，用于备份或克隆目录树，而 rdist 通常用作网络文件分发应用程序。每个都根据其设计目的进行了定制。

有了这么多选择，为什么要使用 rdist 呢？

* rdist 比诸如 ansible 等后续配置分发应用程序更轻量级。
* rdist 容易集成到shell脚本和 Makefiles 中。
* rsync 不能像 rdist 一样并行分发到多个主机。rsync 也不能使用配置文件来同步文件，就像 rdist 那样。Rsync 和 rdist 设计用于不同目的，rsync 用于备份和文件克隆，而 rdist 更适用作为配置管理工具。

另一方面，为什么有人想使用另一个工具呢？与诸如 cfengine 和 ansible 之类的工具相比，rdist 更轻量级，其配置远程节点的能力仅限于分发文件和执行简单的分发后任务。而更重量级工具可以执行分发前任务，可以通过简单的shell脚本或 Makefile 解决此问题。举个个人例子，我使用一个配置工具（名为 ipfmeta）来管理我的 ipfilter 防火墙规则，该工具从规则文件和对象文件生成防火墙配置文件，然后使用 Makefile 将生成的文件分发到在 rdist 的 Distfile 中定义的远程防火墙上。可以将 Distfile 看作与 rdist 的关系类似于 Makefile 与 make 的关系。与 rsync 不同，rdist 使用在其 Distfile 中编码的规则来分发文件。

## RDIST 是如何工作的？

就像 make（1）解析其 Makefile 以构建应用程序一样，rdist 解析其 Distfile 以描述要分发的文件或目录以及要执行的任何分发后任务。最初，rdist 使用不安全的 rcmd（3）接口进行网络通信。rcmd（）将连接到远程 rshd（8）。当连接建立时，它会生成一个 rdistd（8）远程文件分发服务器，在远程服务器上执行分发功能。这类似于 ssh 的 sftp 提供远程函数给 sftp。

伯克利的 “r” 命令，例如 rsh 是不安全的。今天的 rdist 实现可以使用 ssh 作为传输，而不是 rsh。使用 ssh 可以使用 ssh 密钥或 GSSAPI（kerberos）身份验证。与 ansible 不同，在 ansible 中连接是使用您自己的帐户进行的，并且通过“become”进行特权升级，rdist 必须直接连接到目标服务器上的 root。为了方便这一点，可以将 sshd_config 中的 PermitRootLogin 设置为 prohibit-password，从而强制使用 ssh 密钥或 Kerberos 票证。

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
