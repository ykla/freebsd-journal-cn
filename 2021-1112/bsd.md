# WPT/CFT：OccamBSD

- 原文链接：[WPT/CFT：OccamBSD](https://freebsdfoundation.org/wp-content/uploads/2022/01/Importing-a-ZFS-ZIL-via-iSCSI.pdf)
- 作者：**BENEDICT REUSCHLING**

---

本栏目涵盖了 FreeBSD 的 Ports 和软件包，这些 Port 和软件包在某些方面有用、独特或值得了解。 Ports 扩展了基本操作系统的功能，确保你能完成任务，或者简单地让你开心。一起加入吧，也许你会发现一些新东西。

---

我已经在虚拟机上用 FreeBSD 运行我们计算机科学系的 PostgreSQL 服务器多年，并且取得了很好的效果。该服务器主要用于数据库课程和需要数据库后端的项目。我在 vBSDcon 2019 上做过关于该服务器的演讲，你可以在 YouTube 上找到。

最近，托管该虚拟化服务器的部门将其基础存储更换为 Ceph。这通过在校园的三个不同建筑之间同步 I/O，增加了更多的容量和冗余。与此同时，数据库教授们设计了新的实验练习，让学生们熟悉大规模数据集。其中一个练习是创建大数据并将其插入到数据库表中，比较有无表索引的执行时间。一切看似正常，但在该实验开始后不久，教授和学生们开始抱怨性能差。在一些情况下，学生笔记本上的本地 PostgreSQL 安装比我们的服务器更快，尽管我们的服务器有更多的 CPU 和内存。例如，执行 “SELECT COUNT(*) from bigtable;” 查询，大约有 1000 万行，服务器平均需要 2 分钟 5 秒，而本地笔记本只需大约 1 秒。第二次运行相同的查询，服务器只需要 1 秒，这证明了数据是从更快的主内存缓存中提供的。

我开始调查 PostgreSQL，调整了一些 postgresql.conf 文件中的参数并重启了服务器。这仅取得了边际性的成功，大家仍然抱怨插入和查询时间过长。既然有证据表明 PostgreSQL 的默认设置性能更好，那么问题一定是存储或 I/O 相关的。当虚拟机创建时，它的 Ceph 存储部分被转换成了 ZFS 池，进而提供了大部分存储作为 PostgreSQL 数据库的数据集。由于很多学生插入相同的数据并随后查询，ZFS ARC 会直接从内存中提供这些数据。然而，并非所有数据都能适应 ARC，或者被其他查询驱逐出 ARC。一旦我们在磁盘上进行写操作，随着用户生成的大数据量，性能下降变得显而易见。

为了确认我们怀疑的底层存储是问题所在，我从我们的大数据集群中挑选了一台拥有 64 个 CPU、384 GB 内存、4 个 512 GB NVMe 的服务器，并在其上安装了 FreeBSD。然后，我使用 “zfs send” 将托管 PostgreSQL 服务器的数据集复制到这台新服务器上。在启动 PostgreSQL 服务后，我得到了一个完整的服务器副本，可以在更强大的硬件上进行实验。通过在新服务器上运行相同的 COUNT(*) 查询，证明它们和学生的笔记本一样快（如果不是更快的话），即使他们的笔记本使用的是 SSD。显然，我们的虚拟服务器性能较差，才是问题的根源。然而，解决这个问题并不容易，因为我们的 IT 部门不能简单地为这个虚拟机连接一个 SSD 或 NVMe 来加速它。购买并安装到服务器中（这意味着停机时间）需要的时间将超过本学期剩余的时间。

我的想法是通过 iSCSI 将我们刚才测试的服务器上的一个 NVMe 硬盘导出到虚拟机，以创建一个表空间。表空间允许数据库管理员定义数据库对象在文件系统中存储的位置。使用 iSCSI，服务器上的存储（称为目标）可以通过网络发送到另一台计算机（称为发起者），后者会导入它。与网络共享不同，iSCSI 协议使得存储在导入机器上以本地块存储的形式出现，这是一个重要的区别。这种新存储可以像其他存储一样进行分区并格式化，使用新文件系统，就像本地附加的设备一样。

FreeBSD 默认内置了 iSCSI，只需在配置文件中做一些简单的修改即可设置。以下是导出 NVMe 的服务器上的配置：

首先，我在其中一个 NVMe 硬盘上创建了一个 200 GB 的卷，名为 iscsi_export：

```sh
# zfs create -V 200g nvme/iscsi_export
```

接下来，我编辑了 `/etc/ctl.conf` 文件，添加了以下内容：

```sql
portal-group pg0 {
    discovery-auth-group no-authentication
    listen ip.address.of.initiator
}

target iqn.dns-name-of-initiator:nvme {
    portal-group pg0
    chap postgres verysecurepasswordgoeshere
    lun 0 {
        path /dev/nvme/iscsi_export
        size 200G
    }
}
```


我将此文件的所有权和权限更改为 root，因为它包含明文密码。

在服务器重启后，iSCSI 启动器应该重新启动，因此我在 /etc/rc.conf 中添加了 `ctld_enable="YES"`：

```sh
# sysrc ctld_enable=yes
```

为了激活启动器，我启动了服务：

```sh
# service ctld start
```

这大致遵循了 FreeBSD 手册中 iSCSI 部分的描述。在导入存储磁盘的虚拟机上，我在 /etc/iscsi.conf 中添加了以下内容：

```ini
TargetAddress = ip.address.of.initiator
TargetName = iqn.dns-name-of-initiator:nvme
AuthMethod = CHAP
chapIName = postgres
chapSecret = verysecurepasswordgoeshere
}
```

由于 postgres 用户通过 SSH 登录此服务器以运行 PostgreSQL 的命令行工具 psql，因此保护此文件中的密码不被窥视非常重要。执行 `chmod 0700` 并将所有者和组更改为 root 和 wheel 后就能解决此问题。为了在重启后启动存储导入，需要在 /etc/rc.conf 中添加一项：

```sh
# sysctl iscsid_enable=yes
```

接下来，我们可以通过启动服务来导入磁盘：

```sh
# service iscsid start
```

导入成功后，一个新的设备（可能是 da0 或类似名称）将出现在 /dev 中。然后在该设备上创建一个单独的 ZFS 池：

```sh
# zpool create nvme_ts /dev/da0
```



是的，这并不是冗余的，但对于我们的基准测试目的，它已经足够了。在 PostgreSQL 端，作为数据库超级用户登录到 psql 后，使用以下语句定义表空间（有关详细信息，请参见 https://www.postgresql.org/docs/current/manage-ag-tablespaces.html）：

```sql
psql#>CREATE TABLESPACE nvme LOCATION '/nvme';
```

再次检查访问权限，但在命令完成后，PostgreSQL 数据库用户可以使用该表空间，并将数据库对象（如表）放置在其中。可以通过明确指定数据存储的位置：

```sql
psql#>CREATE TABLE nvme_powered_table(i int) TABLESPACE nvme_ts;
```

或者将表空间设置为默认表空间：

```sql
psql#>SET default_tablespace = nvme_ts;
```

使用此新配置（首先清除缓存），并重新加载一批新的 10 GB 数据到 nvme_powered_table，虚拟机上的数据库插入性能提升至 7 秒（从原来的 2 分钟多）。拥有一个 NVMe 表空间当然很好，但我们还进一步优化了。就在这个时候，麻烦开始了……

## 没有考虑周全

我们决定将导出的存储用作 ZIL，以加速 Ceph 支持池上的较慢写入。ZIL 会确认应用程序（数据库）已经将写入操作保存到稳定存储中，并稍后将其写入较慢的磁盘，同时数据库继续工作。ZIL 通常不需要很大，因为其中的数据会很快被驱逐。我们减少了 iSCSI 启动器中导出的磁盘空间，并重新导入了数据库虚拟机中的磁盘。然后，我们使用以下命令将 iSCSI 磁盘配置为 ZIL：

```sh
# zpool add pgpool log da0
```



设备立即显示并正常工作。现在池上的 I/O 操作被迅速确认“已写入”，数据库可以继续运行，而不需要等待。ZIL 将写入请求传递到较慢的 Ceph 存储。这大大提高了数据库性能，我们进入了生产阶段。

## 别在工作中尝试这个

当时我没有意识到的是，这种做法与启动过程的整合有多糟糕。当 FreeBSD 启动一个仅包含 ZFS 文件系统的系统时，它会尝试检测池中包含的所有存储设备。在启动过程中，这时网络尚未完全配置，因此没有 iSCSI 服务可用来导入外部设备。当涉及到 ZIL 设备时，事实证明，ZFS 需要这个设备来正常启动，并且会抱怨池中缺少磁盘。即使池的主 vdev 可用（但 ZIL 不可用），启动过程也会在此早期阶段停止。你可以想象，这在生产服务器上是不行的，只有服务器本身的管理控制台揭示了发生了什么。

请注意，这种情况可以通过两种方式发生：要么 iSCSI 目标（导出存储的服务器）发生故障或失去连接，要么是启动器（导入设备的客户端）。经验丰富的系统管理员知道，在典型的工作日里，这种中断可能会发生，通常是没有预告和意外的。只不过是时间问题，迟早会发生这种情况，而现在它发生了，我们需要一种方法来快速修复它。

接下来的步骤是用 FreeBSD ISO 镜像重新启动服务器，并选择安装程序中的 Live-CD 选项。在 Live-CD 的 shell 环境中，我们可以这样挂载缺失 ZIL 设备的池到 /mnt：

```sh
# zpool import -R /mnt -m pgpool
```

导入完成后，我们可以检查池中的剩余设备：

```sh
# zpool status
```

输出显示了缺失的缓存设备及其长的唯一数字标识符。下一步是从池中移除 ZIL 设备：

```sh
# zpool remove pgpool <verylongnumericidentifier>
```

输入长标识符而不是更短的设备名称，提醒我们以后避免发生这种情况。一旦完成，并且 zpool status 的输出确认移除，池就再次被导出。这通常在重启时完成，但我们不想冒险。

```sh
# zpool export pgpool
```

机器重新启动后，我们很高兴看到它这次完成了启动，并给我们返回了熟悉的登录提示。灾难得以避免，但底层的性能问题仍然存在。



## 幸福结局

显然，iSCSI 导出太过冒险，可能再次失败。尽管我们以这种方式运行了一个学期，但根据墨菲定律，它一定会在最不合时宜的晚上发生——那时系统管理员应该在睡觉。确实，脚本可以在每次关机时安全地移除 ZIL。但如果两台涉及 iSCSI 导出的机器发生断电或崩溃，这种方法就无法应对了。幸运的是，我们的 IT 部门终于为这台机器提供了一个基于 SSD 的 Ceph 存储替代方案。导入过程类似于 iSCSI，但它更加稳定，不容易崩溃。

Ceph 在 FreeBSD 上是可用的，但导入该设备证明是……有趣的。Ceph 仅通过 geom_gate 支持这种类型的导入，geom_gate 类似于 iSCSI。在安装了 net/ceph14 包之后，rbd-ggate 命令可用（rbd 是 Ceph 的 Rados 块设备）。手册页 rbd-ggate(8) 相当简短，仅列出了几个命令和选项。我开始时有些担心，因为它的发布日期是 2014 年。没有最近的更新，因此支持可能会因为新的 FreeBSD 版本变化而失效。然而，这种担心是没有根据的。我们只需要处理 Linux 和 FreeBSD 在命令行参数处理上的一些差异。在 Linux 上使用 --option，而在 FreeBSD 上使用单个 -option 更为常见。命令最初是这样的：

```sh
# rbd map -t ggate volumes/ssdvolume
```

volumes/ssdvolume 是 IT 部门提供的 SSD Ceph 存储路径，在成功导入后，它会映射为一个 geom gate 设备。命令失败了，因为没有提供执行导入操作的用户的 --id（用户名和密码保护该存储不被其他人未经授权导入）。这时，单破折号和双破折号的混合问题变得棘手，因为基于 Linux 的 rbd 命令拒绝将 --id 与单破折号的 -t 参数混用。我们找到了解决方案，将 ID 作为环境变量提供，如下所示：

```sql
CEPH_ARGS='--id postgresdb' rbd map -t ggate volumes/postgresdb
```

通过这种组合，命令成功运行，并告诉我们：

```sh
ggate0 created
```

通过查看 /dev/ggate0 确认了这一点。这是从 Ceph 导入的设备，我们现在可以在其上创建一个新的 ZFS 池：

```sh
# zpool create ssdpool /dev/ggate0
```

回想起上次学到的经验，我们尝试重新启动机器，看看在启动过程中如何处理这个设备。我们高兴地看到系统顺利重启，并且我们可以使用以下命令重新导入这个新的池：

```sh
# zpool import ssdpool
```

然后，我们可以创建一个启动脚本，该脚本在系统完成启动后执行，自动重新导入这个池并激活其中的 postgres 数据库。postgres 数据库通过快照和 zfs 发送从旧的较慢池克隆，并接收它到更快的 ssdpool 中。这一切运行得相当顺利，性能差异也确实非常明显。当我写下这些时，第一批学生小组已经开始使用它（他们还不知道），而我还没有收到任何投诉。


## 经验教训

衡量性能丧失的地方并隔离瓶颈。使用不同的测试用例来验证关于问题可能所在位置的假设。在将任何东西投入生产之前进行测试。确保解决方案在导出和导入机器重启时依然有效，特别是在处理通过网络传输的存储时。保持一个 FreeBSD Live-CD ISO 镜像，以便在发生灾难时修复问题。记录每一步操作和命令，以便自己和同事在面对用户的需求（特别是在已经投入生产时）时能够迅速找回这些信息。做好实验和尝试新事物的准备。最后，依赖 FreeBSD 作为存储领域的坚实基础，利用它提供的灵活性和不同解决方案组合的选项。

---

**BENEDICT REUSCHLING** 是 FreeBSD 项目的文档提交者，且是文档工程团队的成员。他还担任 FreeBSD 基金会董事会副主席。过去，他曾担任 FreeBSD 核心团队成员两届。他在德国达姆施塔特应用科技大学管理一个大数据集群，同时为本科生教授课程《开发者的 Unix》。Benedict 还是每周播客 bsdnow.tv 的主持人之一。
