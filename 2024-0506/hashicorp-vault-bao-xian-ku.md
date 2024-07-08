# Hashicorp Vault（保险库）

 由戴夫·科特尔胡伯撰写

在家工作已经成为新常态。但从安全的角度来看，事情变得更加复杂。安全的办公室已经不复存在，那些精心修锚定的安全边界和全天候的物理安全也已经消失。

安全专业人员谈论我们关心的风险类别，称我们为“威胁格局”和“安全姿态”。

这是一种花哨的说法，即我们可以下定决心，对如 GCSB（政府通信安全局）、克格勃（苏联国家安全委员会）、CIA（美国中央情报局）和摩萨德（以色列情报和特殊使命局）和其他政府资助的攻击者漠不关心，但我们确实关心在火车上忘记笔记本电脑，或者有人的办公室被闯入，窃取高价值物品。我们关心密码和凭证。很多。

## 但是要把秘密放在哪呢？

特别值得注意的是，攻击者和我们作为系统管理员和开发人员的共同兴趣是管理和定期轮换机要。

你见过有多少环境，那里的数据库凭据从未改变过呢？长时间运行的批处理作业，跨多个系统使用相同的密码？或者改变其中一个都可能导致意外停机，如火势一样在公司内蔓延，并最终威胁你的饭碗？

显然，把便笺和凭据存放在 git 存储库，既不具备功能性，也不安全。我们需要一些相对通用的解决方案，同时适用于大部分用例。

那么明智的 DevOps 工程师该怎么做呢？

解决方案是传统的密码存储，例如 BitWarden 和 1Password，并将这些工具扩展到团队中的各种命令行工具中。这些工具只能解决小部分问题，并不能完全解决问题。

它们主要是面向用户的工具，并不容易与复杂的 git 存储库、puppet 和 ansible 部署管道连接起来，确保凭据定期轮换，并且仅对适当的用户和系统可访问。

## 机要管理平台

有时被称为密钥管理系统，这些通常集成到云供应商中。Azure、Amazon、Google、Oracle 都为其平台提供了紧密耦合且集成良好的工具。

但是，如果你在阅读 FreeBSD 杂志，你极有可能对把所有秘密交给商业公司的想法感到极度不安。理想情况下，我们希望自己来管理秘密，而不依赖于外部第三方。

## 折衷选择

这些系统专注于以高度安全和受控的方式分发和管理机要，使运维团队能够进行操作。

它们旨在解决在现代复杂环境中安全管理和编排秘密的挑战，在这些环境中，应用程序、系统和用户需要访问非常敏感的信息。

例如，它们可以集成更改秘密、触发使用该秘密的容器换代到新版本等功能。

他们可以使批处理作业能够短暂解密金融信息，然后安全加密后返回更新后的信息。

或者它可以提供一次性令牌，能让新配置的服务连接到指定数据库，确保令牌既未在传输过程中被劫持，也不能被多个实例使用。

当新实例被配置或部署时，它将获得新的一次性令牌，绑定实例和时间。

这种类型的功能非常适合将系统部署和相关机密信息的访问分离开来。这种掩人耳目的方式通常被称为小隔间部署 — — 一个仅限一次使用的令牌，紧密绑定到单个部署，在部署过程中由自动化工具集注入。此令牌在实例启动时使用，用于获取专用于该实例的运行时机密，可能是通过 IP 地址或其他注意事项绑定。

## 入门指南

本文主要介绍 Hashicorp Vault，它有一个功能等价的开源分支叫做 OpenBao，类似于 Terraform / OpenTofu 许可分支。

有其他工具能用，但 Vault 已经移植到了 FreeBSD，我相信 OpenBao 也很快就会被移植。

## Vault 的甜点

Vault 和其他 KMS 不是存储 Webstorm IDE 激活码和护照扫描件的好地方。它对终端用户并不友好，绝对不能在移动设备上使用。

但如果你正在管理服务器、数据库和网络，那它非常棒。它可以轻松与 Terraform、Chef、Puppet、Ansible 以及几乎任何具有和使用命令行及终端界面的东西集成。

## 内部结构

Vault 将所有密钥和值加密存储在磁盘上。因此，在启动时需要一个主密钥来解锁所有其他密钥。为了避免单一轻量化密钥，Vault 使用 SSS 和 Shamir 的秘密分享将一个大复杂密钥分割成单独的秘密，这些秘密可以重新组合以解锁保险库。在支持 WASM 的现代网页浏览器中尝试 Shamir 演示。

巧妙地，SSS 有可配置的冗余度 — 比如，需要 5 把钥匙中的 3 把来解锁保险柜。因此，您的 3 位主系统管理员可以解锁它，但如果都不在场，您可以请求您的律师及会计借用他们的钥匙（如果需要的话），以达到您的 3 位法定人数。每位管理员在本地提交其解锁钥匙，并使用 API 挑战来防止任一管理员获取主密钥的所有部分。

一旦保险柜解锁，在用户视角下，它在很大程度上像任何其他可通过 HTTP 访问的键值存储。我们可以存储诸如 ssh 私钥、TLS 证书、常规密码，甚至让保险柜生成有限时间和有限目的的临时密码。

## 入门指南

Vault 支持复杂部署，使用共识协议和多个服务器，但我发现一个小型高度可靠的物理服务器，配备热备份和 ZFS 复制备份，已经足够。当然，我依赖 Tarsnap 进行完全脱机备份——对于像我们的所有秘密这样关键的东西，这是绝对必需的！

## 安装和配置

通常必须以 root 身份执行以下命令：

`# pkg install -r FreeBSD security/vault# mkdir -p /var/{db,log}/vault /usr/local/etc/vault# chown root:vault /var/{db,log}/vault /usr/local/etc/vault# chmod 0750 /usr/local/etc/vault# chmod 0770 /var/{db,log}/vault`

在 rc.conf 设置中，您可以使用 sysrc(8) ，或者您首选的操作工具包。

`# /etc/rc.conf.d/vault or where-ever you prefervault_enable=YESvault_config=/usr/local/etc/vault/vault.hcl`

还有 vault 的配置文件。 当然有许多选项，大部分是自解释的。对于我们的测试部署，我们将禁用 TLS 并使用回环 IP。

`# /usr/local/etc/vault/vault.hcldefault_lease_ttl = “72h”max_lease_ttl = “168h”ui = truedisable_mlock = falselistener “tcp” {address = “127.0.0.1:8200”tls_disable = 1tls_min_version = “tls12”tls_key_file = “/usr/local/etc/vault/vault.key”tls_cert_file = “/usr/local/etc/vault/vault.all”}storage “file” {path = “/var/db/vault”}`

现在以前台模式运行守护进程：

`$ vault server -config /usr/local/etc/vault/vault.hcl==> Vault server configuration:Administrative Namespace            Api Address: http://127.0.0.1:8200...`

在一个新的终端中，让我们检查状态：

`$ export VAULT_ADDR=http://localhost:8200/$ vault statusvault statusKey                 Value---                -----Seal Type          shamirInitialized        falseSealed             trueTotal Shares       0Threshold          0Unseal Progress    0/0Unseal Nonce       n/aVersion             1.14.1Build Date         2023-11-04T05:16:56ZStorage Type       fileHA Enabled         false`

注意，保险库尚未初始化，并且仍然处于密封状态。让我们修复这个问题：

`$ vault operator init --key-shares=3 --key-threshold=2Unseal Key 1: jjcVgHTjWw3j4BsyDhugvS9we5t5qMAhJL8bSWzySjbGUnseal Key 2: WfMeZPA7ixleQAMeeAqyey+gwrxDn9WNfSvdKzdLMaeAUnseal Key 3: V9cd1eVBH6mstyoS2pbD6S80R7NJVz7jPvlPOcLOUVlwInitial Root Token: hvs.RAeqzETRhOXOImMPw7xrXbAl$ export VAULT_TOKEN=hvs.RAeqzETRhOXOImMPw7xrXbAl$ vault statusKey                Value---                -----Seal Type          shamirInitialized        trueSealed              trueTotal Shares       3Threshold          2Unseal Progress    0/2Unseal Nonce       n/aVersion            1.14.1Build Date         2023-11-04T05:16:56ZStorage Type       fileHA Enabled         false`

注意，保险库现在已初始化，但仍然处于密封状态。接下来让我们使用新生成的密钥碎片来解决这个问题：

`$ vault operator unsealUnseal Key (will be hidden):Key Value--- -----Seal Type          shamirInitialized        trueSealed             trueTotal Shares       3Threshold          2Unseal Progress    1/2Unseal Nonce       6ce4351d-012b-df3f-a176-34d266f00795Version            1.14.1Build Date         2023-11-04T05:16:56ZStorage Type       fileHA Enabled         false`

用不同的密钥重复解封，直到 sealed 更改为 false 。最后一步是启用审计，因为安全人员喜欢日志。

`$ vault audit enable file path=/var/log/vault/audit.logSuccess! Enabled the file audit device at: file/`

随时查看，这里从未存储过任何秘密，所以它只是请求的审计日志。

### Shamir 秘密环

现在你已经打开了保险库，将你的秘密通过加密的鸟类信使分发给你选择的秘密保管者。需要进行某种适当的仪式，并确保这些秘密得到充分保护，既要防止无能或其他问题，也要防范摩萨德和朝鲜特工。

到现在，你应该已经准备好存储秘密了。

### 存储秘密

Vault 具有引擎的概念 - 包括简单的键值存储，还有用于 ssh 证书、AWS 和 Google Cloud 集成、RabbitMQ、PostgreSQL 等的引擎。每个引擎都需要单独启用。

`$ vault secrets enable -version=2 kvSuccess! Enabled the kv secrets engine at: kv/`

从这里开始，我们需要指定引擎类型和其挂载路径。可以将数据检索为 JSON 或 yaml 格式，甚至可以直接存储文件。

`$ vault kv put -mount=kv blackadder scarlet_pimpernel=”we do not know”=== Secret Path ===kv/data/blackadder======= Metadata =======Key Value--- -----created_time 2024-05-12T23:04:50.283028044Zcustom_metadata <nil>deletion_time n/adestroyed falseversion 1$ vault kv get -mount=kv -format=json blackadder{“request_id”: “48141452-8f8f-b497-9c53-1af71e24e2a5”,“lease_id”: “”,“lease_duration”: 0,“renewable”: false,“data”: {“data”: {“scarlet_pimpernel”: “we do not know”},“metadata”: {“created_time”: “2024-05-12T23:04:50.283028044Z”,“custom_metadata”: null,“deletion_time”: “”,“destroyed”: false,“version”: 1}},“warnings”: null}$ vault kv put -mount=kv blackadder scarlet_pimpernel=”comte de frou frou”=== Secret Path ===kv/data/blackadder======= Metadata =======Key Value--- -----created_time 2024-05-12T23:08:22.369551931Zcustom_metadata <nil>deletion_time n/adestroyed falseversion 2$ vault kv get -mount=kv -format=yaml blackadderdata:data:scarlet_pimpernel: comte de frou froumetadata:created_time: “2024-05-12T23:08:22.369551931Z”custom_metadata: nulldeletion_time: “”destroyed: falseversion: 2lease_duration: 0lease_id: “”renewable: falserequest_id: 686965d9-811f-8689-d75f-a02f7dded9a7warnings: null$ vault kv put kv/blackadder scarlet_pimpernel=@/etc/motd.template`

## 基于角色的访问控制

Vault 可以配置为要求 GitHub 认证，并将角色和认证委托给除 LDAP 外的其他系统。你们中的许多人会为这个消息而高兴。使用 GitHub 认证可以强制所有用户使用双因素认证（2FA），因此对小团队来说这是一个合理的权衡。

`$ vault auth enable githubvault auth enable githubSuccess! Enabled github auth method at: github/$ vault write auth/github/config organization=skunkwerksSuccess! Data written to: auth/github/config$ vault write auth/github/map/teams/admin value=adminsSuccess! Data written to: auth/github/map/teams/admin`

将这个小策略文件放置在 /usr/local/etc/vault/admins.hcl 中

`# grant members of github admins group all rights in the kv/ mountpath “kv/*” {capabilities = [“create”, “read”, “update”, “delete”, “list”]}`

然后在 Vault 中启用它：

`$ vault policy write admins /usr/local/etc/vault/admins.hclSuccess! Uploaded policy: admins`

您当然可以为各种群体制定更为严格的策略，例如在权限、路径或选择的挂载点上，比如一个部署机器人。

最后，每个希望使用 github 认证来访问保险箱的用户，必须转到 https://github.com/settings/tokens，并添加一个新的个人令牌，权限为 admin— read:org 。

现在可以使用此令牌生成您的保险箱登录令牌。

`$ vault login -method=github token=$GITHUBSuccess! You are now authenticated. The token information displayed belowis already stored in the token helper. You do NOT need to run “vault login”again. Future Vault requests will automatically use this token.Key Value--- -----token hvs....token_accessor ...token_duration 72htoken_renewable truetoken_policies [“admins” “default”]identity_policies []policies [“admins” “default”]token_meta_org skunkwerkstoken_meta_username dch$ vault kv get -mount=kv -format=yaml blackadder...`

## 自动化中的保险库

诸如 Chef、Puppet、Ansible 等自动化工具可以使用保险库存储密钥，以便在部署时解密，或者在某些情况下，仅在运行时解密。

让我们看看第一种情况，即在部署时解密密钥。这实际上扩展了自动化工具的模板化能力，并依赖于能够在推送新的和更新的密钥之后触发服务重启。

我们可以以 4 种方式使用保险库：

* Ansible 和类似工具可以将秘密存储在保险库中，并且只在部署时解密它们，使用查找
* rc.d  框架脚本可以使用应用程序角色在启动时提取它们的凭据，让本地 root 用户拥有仅允许发出小隔间令牌的委派令牌。守护进程本身将仅能检索自己的凭证
* Vault 可以发行时间和 IP 绑定的动态凭证，在过期时会被吊销，适用于守护进程、定时任务和批处理脚本的时间限制
* 我们还可以在运行时使用 Vault 代理来模板化文件

### Ansible

有许多 ansible 的插件，令人困惑的是，有一个内部 ansible“vault”模块，与 Hashicorp Vault 不兼容。

安装插件，并使用典型的 lookup 功能：

`super_secret: “{{lookup('hashivault', 'kv', 'blackadder', version=2)}}”`

### rc.d 应用角色

AppRole 是一种内置的认证方法，专门用于机器和应用程序进行 Vault 认证，随后获取令牌，仅允许获取相关密钥。这通常被称为 cubby-hole 凭据，因为它只允许解包外层，获取内部密钥。

这些可以通过时间限制、有限的使用次数等进行限制。我们的受信根进程生成此受限密钥 ID，并将其和角色 ID 传递给守护进程以获取其自身凭据。受限密钥 ID 的生成可以设置为仅限制此类凭据的发行。

再次启用 approle 挂载点，然后创建我们的应用程序特定凭据，因为这是一种形式的认证。为了方便起见，这个 approle 将重复使用之前使用的 admins 组策略，但它应该有一个更为严格的策略，专门用于这个守护程序/服务。

`$ vault auth enable approleSuccess! Enabled approle auth method at: approle/$ vault write auth/approle/role/beastie \<br/>secret_id_ttl=60m \<br/>token_num_uses=10 \<br/>token_ttl=1h \<br/>token_max_ttl=4h \<br/>secret_id_num_uses=40 \<br/>policies=”default,admins”Success! Data written to: auth/approle/role/beastie$ vault read auth/approle/role/beastie/role-idKey Value--- -----role_id 6caaeac3-d8fa-a0e3-83ba-7d37750603c2$ vault write -f auth/approle/role/beastie/secret-idKey Value--- -----secret_id 8dd54c92-fe54-0d6d-bee6-e433e815aaa1secret_id_accessor cb9bc17c-c756-42b3-c391-b61ebde12bffsecret_id_num_uses 0secret_id_ttl 0s`

如果我们想要使这些秘密在假想的 beastie 守护程序中可用，可以将这两个参数放入一个 /etc/rc.conf.d/beastie 文件中，该文件可以安全地仅由 root 可读取。

beastie_enable=YES beastie_env=” ROLE_ID=6caaeac3-d8fa-a0e3-83ba-7d37750603c2 SECRET_ID=8dd54c92-fe54-0d6d-bee6-e433e815aaa1 SECRET_PATH=kv/beastie VAULT_ADDR=[http://localhost:8200/](http://localhost:8200/) “

 /usr/local/etc/rc.d/beastie 脚本运行一个预命令，以 root 身份获取秘密，并将其注入子环境中。

`start_precmd=${name}_vaultbeastie_vault() {# Authenticate with Vault using the approleVAULT_TOKEN=$(vault write auth/approle/login role_id=”$ROLE_ID” \<br/>secret_id=”$SECRET_ID” \<br/>-format=json | jq -r '.auth.client_token')# Retrieve the secret from Vaultexport BEASTIE_SECRET=$(vault kv get -field=data -format=json ${SECRET_PATH} | jq -r .)}`

### 代理

Vault 还提供了代理模式，可以为您处理大部分凭据管理工作，并支持简单配置文件的模板化。

## 密封 Vault

通常情况下，保险库会保持长达数月之久的开启状态，除非进行补丁和升级。在发生安全事件时，只需停止运行保险库守护程序的服务器，或者选择发出 seal 命令。这会关闭保险库，并卸载主密钥。

`$ vault operator sealSuccess! Vault is sealed.`

DAVE COTTLEHUBER 在过去的 2 十年里一直努力保持在互联网上的恶意行为者至少一步之先，从 OpenBSD 2.8 开始，过去 9 年则使用 FreeBSD 自 9.3 起，这段时间他获得了 ports 提交位，并且倾向于使用 jails，以及与他对分布式系统和极具危险边缘性质的电动工具非常相契的晦涩的函数式编程语言。

* 自 2000 年左右开始，从事专业的牦牛牧羊人，剃 BSD 色的牦牛。
* 自由的ports@ 提交者
* Ansible DevOops 主人
* Elixir 开发人员
* 使用 RabbitMQ 和 Apache CouchDB 构建分布式系统
* 热衷于电销滑雪，并在多种乐器上演奏凯尔特民间音乐
