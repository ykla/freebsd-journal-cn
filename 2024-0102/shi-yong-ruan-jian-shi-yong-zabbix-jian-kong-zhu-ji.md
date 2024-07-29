# 实用软件：使用 Zabbix 监控主机

 由本尼迪克特·鲁斯林撰写

我想了解正在发生的事情，尤其是我负责的服务器和机器上的情况。对这些系统进行监控已成为我的良好实践。每天检查的系统太多了，大多数时间都很平静。但一旦出现问题，无论我是在睡觉还是离开终端时发生的，我都想知道。监控还让我可以了解到我的机群概况。有时我会发现某个主机已经完成任务但尚未被回收。甚至有过这样的情况，我可以将新服务迁移到一个利用率较低的机器上，而不是再启动另一台主机或 jail。

图表在展示机器在一段时间内的运行情况方面帮助很大。上周的系统负载异常吗？还是大学实验组开始了？那块磁盘正在慢慢填满，我最好处理一下。这类问题经常出现，而通过收集的机器指标的图表可以帮助解答。

任何优秀的系统管理员都会因为他们的系统正常运行而睡得更香。监控软件可以告诉你它们正在运行，你甚至可以为每台机器制作一份运行时间报告，以填充 PowerPoint 幻灯片。

监控系统可以做所有这些事情，称为 Zabbix。这是一个开源解决方案，用于监控您的 IT 基础设施，由一家公司开发，如果需要，甚至可以提供专业支持。我使用的功能齐全的解决方案正在长期支持周期上运行，我发现安装文档写得很好。一个中央服务器从运行代理的系统收集数据。服务器提供具有仪表板、图形、有关特定事件的警报等功能的 Web UI。对我来说，另一个好处是 FreeBSD 不是一个陌生的操作系统，而且 Zabbix 有机器模板可用于监视重要指标，如 CPU、RAM，甚至是 ZFS。

我从jail上运行 Zabbix，没有遇到任何重大问题。基于 PHP 的解决方案需要一个数据库来存储指标。可以使用 Postgres，MySQL/MariaDB，SQLite，甚至商业的 IBM 和 Oracle 数据库服务器。监控本身通过 SNMP/IPMI、ssh 或简单的 ping 来检查可用性。对于主动监控(收集实时机器指标)，需要在这些主机上安装代理。甚至有功能可以监视整个子网以获取新主机，并在它们出现时将其添加到监视中。甚至有功能可以监视整个子网，以获取新主机，并在它们首次出现时自动将其添加到监视中。可以通过 Web 界面配置对某些主机的触发器(例如，磁盘已满)。有关这些事件的警报可以通过电子邮件、Jabber、短信或自定义脚本操作发送。

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/reuschling_fig1.jpg)

## Zabbix 设置

让我们看看如何安装这个监控解决方案。我从一个全新的 FreeBSD 14.0 jail 开始，它连接到我正在监控的网络，并从网络或单独的 Poudriere 服务器下载所需的软件包。请注意，我通过在 net/mgmt/zabbix64-server ports 目录中运行 “make config” 来配置 port 从默认的 MySQL 使用 PostgreSQL。

这些是我们需要的软件包：

`# pkg install zabbix64-server zabbix64-frontend-php82 postgresql15-server nginx`

在早期的尝试中，我使用了 zabbix6-server 和 frontend，因为我认为版本 6.4 最终会成为版本 6.5。但 Zabbix 的发布计划有点不同。长期支持版本是主要版本（在这种情况下是 6），而次要发布版本（6.4）有较短的支持周期。我切换到长期支持版本，因为我不想在监控的最前沿运行，可以在一段时间内使用当前的功能集。稳定性是我的主要目标。当你依赖于监控你的关键系统时，你不想每次新版本发布时都修复你的监控。

当监控主机（或jail）启动时，使用 sysrc 激活服务：

`# sysrc zabbix_server_enable=yes# sysrc zabbix_agentd_enable=yes# sysrc postgresql_enable=yes# sysrc nginx_enable=yes# sysrc php_fpm_enable=yes`

我在我的设置中不使用 SNMP（将来可能会改变），但 pkg-message 中有关于如何启用 SNMP 守护程序的详细信息。安装包后的第一个任务是设置 zabbix 数据库。我喜欢使用 Postgres，在这里我使用它作为我的设置。如前所述，支持其他数据库，因此在这里选择您喜欢的数据存储解决方案。

### 数据库设置

首先，我切换到作为软件包的一部分安装的 postgres 用户。

`# su postgres`

切换到/var/db/postgres（即 postgres 主目录）后，我运行 initdb 命令来初始化数据库集群。然后，我使用 pg_ctl 命令启动数据库，因为我们需要运行的数据库来导入 Zabbix 基础表。

`$ cd /var/db/postgres$ initdb data$ pg_ctl -D ./data start`

一旦数据库正在运行，我切换到

`/usr/local/share/zabbix64/server/database/postgresql/`

数据库模板用于表、触发器以及其他一切的存放位置

`$ cd /usr/local/share/zabbix64/server/database/postgresql/`

然后我运行 PostgreSQL 的交互式 shell，在那里，我为 zabbix 创建一个新数据库，创建一个同名的用户并设置密码。在为该用户授予 zabbix 数据库权限之后，我需要从 psql 注销以切换到我们刚刚创建的 zabbix 用户。

`psql -d template1psql> create database zabbix;psql> CREATE USER zabbix WITH password 'yourZabbixPassword’;psql> GRANT ALL PRIVILEGES ON DATABASE zabbix to zabbix;psql> exit`

再次使用 psql 以新创建的 zabbix 用户身份登录空的 zabbix 数据库。按顺序加载包含 zabbix 表定义（以及其他一些数据库对象）的三个文件： schema.sql ， images.sql ，最后是来自我们之前切换到的本地目录的 data.sql 。完成后，我们可以再次注销数据库。

`$ psql -U zabbix zabbixpsql> \i schema.sqlpsql> \i images.sqlpsql> \i data.sqlpsql> exit`

这就是数据库所需的一切。让我们继续配置 Zabbix 本身。

### Zabbix 配置

Zabbix 需要知道要使用哪个数据库来存储指标、监视的机器以及大量的其他信息。主要的 Zabbix 配置位于 /usr/local/etc/zabbix64/zabbix_server.conf 中，是一个直观的键值文件。注释描述了这些值的作用，大多数值都是被注释掉的。在我进行必要的更改后，我的文件看起来像这样：

`SourceIP=IP.address.of.monitoring-hostLogFile=/var/log/zabbix/zabbix_server.logDBHost=DBName=zabbixDBUser=zabbixDBPassword=yourZabbixPasswordTimeout=4LogSlowQueries=3000StatsAllowedIP=127.0.0.1`

中央 Zabbix 服务器收集指标的 SourceIP 地址是我定义的第一件事情。请注意，前端（Web UI）不必驻留在同一主机上，但我在这里保持集中化。我定义 Zabbix 在运行时发生的任何事件的日志文件路径（LogFile）。如果尚不存在，请创建该路径和文件，并将所有者设置为 Zabbix 用户（这是随软件包安装的用户）。

不，我没有忘记为 DBHost 参数设置值。在使用 Postgres 时，此参数需要为空。有一天，当您必须定义自己的配置文件时，您可能会回想起这一点，并可能会采取不同的方法。对我来说输入更少，记住，其他数据库可能需要在此处设置值，因此如果您不使用 Postgres，请调整您的设置。

DBName、DBUser 和 DBPassword（明文）是我们在数据库设置中之前创建的。这将 Zabbix 连接到数据库实例，以便在监控过程中存储和检索值。 Timeout 和 LogSlowQueries 都是我保留的默认值。我还没有需要调整它们，但我可以想象资源限制可能是这样做的一个很好的理由。Zabbix 本身的内部统计数据也会被收集， StatsAllowedIP 参数定义了从哪里接受这些数据。对于我的用例来说，本地主机完全可以。另一个 Zabbix 实例可能也希望访问这些数据，因此如果您有多个 Zabbix 服务器相互监控，可以在此定义多个地址。

这就是服务器配置文件的全部内容，至少是基础部分。我同时为客户端和服务器本身使用 Agent 监控。当使用 SNMP 时，可能需要额外的值。查看 Zabbix 的文档以获取详细信息。

### Zabbix 代理配置

这个代理被称为 Zabbix Agentd（好老的 Unix 守护进程），它应该在服务器jail启动时运行。在/etc/rc.conf 的条目是使用 sysrc 完成的，看起来像这样：

`# sysrc zabbix_agentd_enable=yes`

位于服务器配置文件旁边的是代理的配置文件。该文件名为/usr/local/etc/zabbix64/zabbix_agentd.conf 中的 zabbix_agentd.conf。我的更改使此文件看起来如下所示：

`LogFile=/var/log/zabbix/zabbix_agentd.logSourceIP=IP.address.of.host-to-monitorServer=IP.address.of.monitoring-hostServerActive=127.0.0.1Hostname=Zabbix server`

我们在此文件中找到熟悉的行。 LogFile 应该是一个不同的值，以区分来自服务器和客户端的消息，特别是对于服务器。通常，被监视的机器不运行服务器部分，因此在这些系统上不会混淆它们。该目录现在应该已经存在，但是需要通过 touch(1)手动触发文件的创建。 SourceIP 与监控主机发送其自身指标到自身完全相同。对于其他客户端，请将其设置为相应主机的 IP 或 DNS 地址。由于我们正在使用一个中央系统来收集这些指标， ServerIP 在所有系统上保持不变。如上所述已讨论，Hostname 是 Zabbix 内部用于保持系统区分的标识符。

### Zabbix 前端

Zabbix 前端需要一个运行 PHP 的 Web 服务器，并且需要其自己的配置文件。前端需要知道如何从后端获取指标和其他数据，因此另一个配置文件需要数据库凭据。所讨论的文件位于 /usr/local/www/zabbix64/conf/zabbix.conf.php ，就在构成 Zabbix 的其余网站文件所在的位置。一些注释将指导您填写正确值的行。我的看起来像这样：

`<?php  // Zabbix GUI configuration file.  $DB['TYPE']             = 'POSTGRESQL';  $DB['SERVER']           = 'localhost';  $DB['PORT']             = '0';  $DB['DATABASE']         = 'zabbix';  $DB['USER']             = 'zabbix';  $DB['PASSWORD']         = 'yourZabbixPassword';  $DB['SCHEMA']           = '';`

关于 TLS 加密的其他设置留给读者自行完成。一定要考虑实施这一点，否则有人可能能够从定期向中央服务器发送其指标的主机发送的信息中获取大量信息。我在这里省略了这一点，以便将教程集中在使 Zabbix 首次运行的最重要部分。

我选择的 Web 服务器是 nginx，但其他任何 Web 服务器也可以完全胜任。更改后的主要 nginx.conf 文件包含以下行：

`worker_processes 1;events {  worker_connections 1024;}http {  include             mime.types;  default_type        application/octet-stream;  sendfile            von;  keepalive_timeout   65;  server {    listen            80;    server_name       localhost;    root /usr/local/www/zabbix64;    index index.php index.html index.htm;    location / {      try_files $uri $uri/ =404;    }    location ~ .php$ {      fastcgi_split_path_info ^(.+\.php)(/.+)$;      fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;      fastcgi_param PATH_INFO $fastcgi_path_info;      fastcgi_param REMOTE_USER $remote_user;      fastcgi_pass unix:/var/run/php-fpm.sock;      fastcgi_index index.php;      include fastcgi_params;    }    error_page    500 502 503 504   /50x.html;    location = /50x.html {      root   /usr/local/www/nginx-dist;    }}`

注意，这也忽略了现在网页上普遍存在的 SSL/TLS 设置。我之前使用 SSL 代理进行请求，但 Let's Encrypt 解决方案便宜且易于实施。再次强调我在这里专注于 Zabbix。

对于 PHP 本身，配置离生产使用只有几行之遥。编辑 /usr/local/etc/php-fpm.d/www.conf 并将这些设置更改为所示的值：

`listen = “/var/run/php-fpm.sock”`

取消以下注释：

`listen.ownerlisten.grouplisten.mode`

/usr/local/etc/php.ini 文件应使用生产值，因此请将 /usr/local/etc/php.ini-production 复制并重命名为该文件名。我们还没有完成，因为设置将稍后检查某些 PHP 值，并抱怨较低（或直接错误）的值。更改这些值以获得流畅体验（并将时区更改为您的时区）

`date.timezone = Europe/Berlinpost_max_size = 16Mmax_execution_time = 300max_input_time = 300`

我保存了那些文件，运行了 sysrc nginx_enable=yes，然后启动了 Zabbix 堆栈中的所有服务：

`service postgresql restartservice php-fpm startservice nginx startzabbix_server startzabbix_agentd start`

### 网站问题

那一切都很顺利，我在 Zabbix 的日志文件中看到了几行信息（服务器和代理都有）。我兴奋地打开我选择的浏览器，输入了 Zabbix 的网址。什么也没有发生。一个空白的页面迎接了我。我重新启动了服务，检查了日志，再次检查了配置文件。没有效果，还是同样的空白页。我试了一个不同的浏览器，只是为了排除任何特殊情况。但是空白页面依然在我面前，一成不变。

每当你在 BSD 大会上遇见我（这是一个可能的事件），你会注意到我的头发已经不多了。我可以责怪糟糕的基因或环境因素，但是在 IT 领域工作是我为什么在这种情况下抓狂的主要原因。根据文档，这应该是可以正常工作的，那么为什么网站上什么也看不到呢？

如果运气没有指引我朝正确的方向，这篇文章可能会以一个令人不满的结尾结束。我当时已经打开了 Zabbix 的常见问题解答，看看其他问题，而我的眼睛注意到了这一部分：

`If “opcache” is enabled in the PHP 7.3 configuration, Zabbix frontend may show a blank screen when loaded for the first time. This is a registered PHP bug. To work around this, please set the “opcache.optimization_level” parameter to 0x7FFFBFDF in the PHP configuration (/usr/local/etc/php.ini file). https://www.zabbix.com/documentation/current/en/manual/installation/known_issues`

### 解决方案

好的，所以我回到 /usr/local/etc/php.ini 并像这样设置 opcache.optimization_level：

`opcache.optimization_level=0x7FFFBFDF`

尽管这个设置可能有些晦涩，但在您实施时可能是必需的。我把它留在那里，以避免再次抓狂。再次重启服务后，我被 Zabbix 网页配置界面所迎接。设置后端数据库的 PHP 设置和值是主要的确认步骤。为网页前端设置密码，然后首次登录。

安装新系统后，可能看不到任何主机。要至少看到自己的监控服务器，请点击左侧的“监控”（大眼睛图标），然后选择“主机”。在新页面上，点击右上角的“创建主机”按钮。将会出现一个表单，您可以在其中输入主机的名称、IP 地址以及要使用的监控类型（我们的情况下是 Zabbix 代理）。模板字段包含特定类型主机的预设配置，当输入“FreeBSD”操作系统名称时，我们将在那里找到相应的设置。组将逻辑上结合具有相似特征的主机（您可以根据需要创建多个组）。这使得过滤更容易，如果发生故障，还可以告知您需要注意的其他系统。在“接口”部分，点击“添加”，选择“代理”。新的输入字段将出现，您可以在其中输入 IP 地址和/或 DNS 名称。port 10050 是代理的默认端口，请检查是否有防火墙规则阻止访问。描述是可选的，但随着服务器数量的增长，提醒自己系统用途是很有好处的。点击蓝色的“添加”按钮来创建主机。它将出现在主机列表中，如果一切顺利，代理将开始收集数据，并很快显示一些图表。仪表板将显示在监控期间检测到的任何问题。

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/reuschling_fig2.png)

查看 Zabbix 文档以了解如何监控整个子网、如何使用 SNMP 监控以及如何自动添加主机，而无需逐个点击每个主机。探索 Zabbix UI 的其余部分，发现您以前未注意到但确实想要定期查看以检测异常情况的酷功能。愉快的监控！

BENEDICT REUSCHLING 是 FreeBSD 项目的文档提交者，也是文档工程团队的成员。过去，他曾在 FreeBSD 核心团队任职两届。他在德国达姆斯塔特应用科学大学管理着一个大数据集群。他还为本科生开设了一门“Unix 开发者课程”。Benedict 也是每周 bsdnow.tv 播客的主持人之一。
