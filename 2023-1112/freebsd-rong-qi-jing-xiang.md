# FreeBSD 容器镜像

由 Doug Rabson 著

OCI 容器引擎，如 containerd 或 podman，需要镜像。一个容器镜像是一个只读的目录树，通常包含一个带有支持文件和库的应用程序。在容器引擎上运行此镜像会创建镜像的可写克隆，并在某种隔离环境（如一个 jail）中执行应用程序。

镜像通过注册表进行分发，注册表存储镜像数据并提供简单的 REST API 来访问镜像及其元数据。Open Container Initiative 标准化了注册表 API、镜像格式和元数据，这在很大程度上取代了早期的 Docker 格式。

## OCI 镜像

图像表示为一系列图层，每个图层都存储为压缩的 tar 文件。为了解压图像，我们从一个空目录开始，然后按顺序解压每个图层，允许后续图层添加文件或更改来自早期图层的文件。通常，这个过程的结果由容器引擎缓存。

除了图层数据，还使用了两个附加的元数据对象。清单列出了图层，并可以包含注释以描述图像。图像配置描述目标操作系统和体系结构，并允许使用默认命令来运行图像。

所有这些都存储在一个“内容寻址”结构中，其中组件的哈希用于命名它。例如，我用于静态链接应用程序的小型基本图像看起来像这样：

`$ ls -lRtotal 6drwxr-xr-x 3 root dfr 3 Sep 8 10:36 blobs-rw-r--r-- 1 root dfr 275 Sep 8 10:36 index.json-rw-r--r-- 1 root dfr 31 Sep 8 10:36 oci-layout./blobs:total 25drwxr-xr-x 2 root dfr 6 Sep 8 10:36 sha256./blobs/sha256:total 950-rw-r--r-- 1 root dfr 1143 Sep 8 10:36190e4f8bf39f4cc03bf0f723607e58ac40e916a1c15bd212486b6bb0a8c30676-rw-r--r-- 1 root dfr 496 Sep 8 10:365657eb844c0c0142aa262395125099ae065e791157eaa1e1d9f5516531f4fe30-rw-r--r-- 1 root dfr 34916 Sep 8 10:365af368a2a6078dc912135caed94a6375229a5a952355f5fea60dad1daf516f78-rw-r--r-- 1 root dfr 911102 Sep 8 10:36fdb4ee0a131a70df2aae5c022b677c5afbacb5ec19aa24480f9b9f5e8f30fd18`

此包中的所有元数据文件均采用 json 格式描述如下。顶层 index.json 文件使用其哈希链接到清单：

`$ cat index.json | jq{  "schemaVersion": 2,  "manifests": [    {      "mediaType": "application/vnd.oci.image.manifest.v1+json",      "digest": "sha256:190e4f8bf39f4cc03bf0f723607e58ac40e916a1c15bd212486b6bb0a8c30676",      …    }  ]}`

此清单描述了两个数据层，一个仅具有 FreeBSD 标准目录结构，另一个包含最少的支持文件，如 /etc/passwd 和 ssl 证书。它还链接到包含目标操作系统和架构的配置文件。

使用这样的内容可寻址格式可以更轻松地共享存储空间，并在使用从相同基础派生的多个映像时减少下载的数据量。

OCI 图像规范还允许多架构图像，这些图像只是清单列表：

`{  "schemaVersion": 2,  "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",      "manifests": [          {              "mediaType": "application/vnd.oci.image.manifest.v1+json",              "size": 1116,              "digest":"sha256:598b927b8ddc9155e6d64f88ef9f9d657067a5204d3d480a1b1484da154e7c4",              "platform": {                 "architecture": "amd64",                 "os": "freebsd"              }          },          {              "mediaType": "application/vnd.oci.image.manifest.v1+json",              "size": 1118,              "digest":"sha256:ac732db0f4788d5282a8d16fefbea360d937049749c83891367abd02801b582",              "platform": {                 "architecture": "arm64",                 "os": "freebsd"              }         }    ]}`

## FreeBSD 基础镜像

为了更容易地在 FreeBSD 上使用容器，需要合适的基础镜像。传统的 FreeBSD 发布过程生成了少量用于在物理或虚拟主机上安装完整功能的 FreeBSD 操作系统的软件包。我们可以使用 base.txz 软件包构建我们的基础镜像，但这会导致一个占用一千兆字节的镜像，其中超过 90%对大多数应用程序来说是不需要的。大多数 Linux 发行版提供更小的基础镜像 —— 比如，官方的 Ubuntu 镜像约为 80MB。

幸运的是，pkgbase 项目一直在努力将细粒度软件包集合化，将传统的 base.txz 压缩包细分为数百个更小的软件包。目前，这包括许多用于个别库和实用程序的软件包，还有两个较大的软件包，FreeBSD-runtime 包含 shell 以及一些核心实用程序，FreeBSD-utilities 则有更大量常用实用程序的集合。

早期，我使用 pkgbase 创建了一个“极简”映像，其中包括 FreeBSD-runtime，加上 SSL 证书和 pkg。这个映像约为 80MB，包含足够简单 shell 脚本功能以及安装软件包的能力。这与类似的 Linux 映像相比表现出色，尽管与基于 busybox 的 alpine 映像的 7.5MB 相比仍有差距。从那时起，我制作了一系列小型映像，部分受到 distroless 项目的启发：

* “static” 只包含 SSL 证书和时区数据。这可用作静态链接应用程序的基础。
* "基础”，通过添加一系列共享库来扩展“静态”，以支持各种动态链接的应用程序。
* "最小”，与之前一样添加了 FreeBSD 运行时软件包和软件包管理。
* "小型”，添加了 FreeBSD 实用程序，以支持更广泛的shell-based 应用程序。

为了支持各种版本的 FreeBSD，我将版本嵌入到镜像名称中，例如，“freebsd13.2-minimal:latest” 包含来自 releng/13.2 分支最新版本的软件包，而“freebsd13-minimal:latest”则是从 stable/13 构建的。我为所有这些镜像构建了对 amd64 和 arm64 架构的支持，容器引擎将自动从清单列表中选择正确的镜像。

## 安全性

在将容器映像传输到容器引擎时，很重要的一点是能够验证容器映像的受信任来源，并且在传输过程中没有遭到篡改。

镜像的清单通常包含镜像数据层的 SHA256 哈希以及相应镜像配置的哈希。这意味着可以使用清单的哈希来唯一标识镜像。这可用于验证镜像，例如通过在可信位置列出受信任的镜像哈希。

或者，可以使用哈希来创建一个签名，该签名可以证明镜像由某个公钥的所有者信任。目前有两种常见的机制用于此目的——podman 使用 PGP 创建镜像签名并提供一种将一组镜像与签名存储区关联的机制，该存储区可以是本地目录或受信任的网站。在拉取镜像时可以使用此功能来验证镜像是否匹配签名。sigstore 的替代方案是 cosign，它将签名存储在镜像存储库中的镜像旁边。

## 限制和未来工作

虽然这些镜像很有用，但它们包含了安装的软件包的相当随意选择。最初，我包括了支持 alpha.pkgbase.live 软件包仓库的功能，这简化了通过安装额外软件包来扩展镜像的过程。不幸的是，这个项目失去了资金支持，一段时间内没有公开可用的 pkgbase 仓库。幸运的是，现在可以从标准的 FreeBSD 软件包仓库获取到 pkgbase 软件包。

构建镜像的当前机制使用 pkg 将 pkgbase 软件包安装到镜像层中。这很方便并且记录了安装到镜像中的内容。不幸的是，pkg 元数据存储在一个 sqlite 数据库中，这不支持可重现的构建。sqlite 数据库包含了安装软件包的时间戳 —— 这可以被覆盖为某个合适的固定时间，但即便如此，sqlite 数据库也不是可重现的。更大的问题是可信度 —— 我将这些镜像托管在我的个人仓库 docker.io 和 quay.io 中，但从潜在用户的角度来看，没有理由相信这些镜像是可信的。即使我可以使用来自 FreeBSD 软件包仓库的软件包构建镜像，这些镜像也没有经过 FreeBSD 项目的签名或支持。

在我看来，这对于 FreeBSD 容器引擎的潜在用户来说是一个重大障碍，阻碍了这些项目从当前的‘实验性’状态转变为可以考虑用于生产的状态。最近关于支持 FreeBSD 作为构建和使用镜像的开源项目平台的几次对话已经证实了这一点。

理想情况下，除了托管 pkgbase 软件包集，FreeBSD 项目还应构建 FreeBSD 容器镜像，可以通过托管镜像注册表或在公共仓库（如 docker.io）上提供这些镜像。我计划在发布构建基础设施中进行原型添加，将容器镜像构建整合到现有的 pkgbase 框架中，这可能有助于推动这一进展。

DOUG RABSON 是一名软件工程师，拥有超过 30 年的经验，从上世纪 80 年代的 8 位文本冒险游戏到 2020 年代的每秒 TB 级的分布式长聚合系统。自 1994 年以来，他一直是 FreeBSD 项目成员和提交者，目前致力于改善 FreeBSD 对现代容器编排系统（如 podman 和 kubernetes）的支持。

——FreeBSD 杂志，2023 年 11 月/12 月
