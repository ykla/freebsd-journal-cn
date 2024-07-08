# 现在用 Webhooks 触发我

由戴夫·科特赫伯（DAVE COTTLEHUBER）

## 什么是 Webhook 以及我为什么想要一个？

Webhook 是一种基于事件驱动的远程回调协议，通过 HTTP 允许脚本和任务可以轻松地从几乎任何编程语言或工具调用。Webhook 的优点在于它们的普遍性和简单性。通过一个简单的 HTTP URL，您可以请求远程服务器调暗灯光，部署代码，或代表您执行任意命令。

最简单的 Webhook 可能只是智能手机网页浏览器中的一个书签链接，或者更复杂的版本可能需要强身份验证和授权。

尽管存在更大的自动化工具集，如 Ansible 和 Puppet，但有时候简单的东西就足够了。Webhooks 就是这样一种东西，它允许您在请求时在远程计算机上运行任务，安全地。调用一个 Webhook 可以被视为“踢”，因此文章标题就这么说。

## 集成

尽管还没有官方标准，通常，Webhooks 通过使用 TLS 加密的 POST 发送 JSON 对象主体，经常使用签名来防止篡改、网络伪造和重放攻击。

常见的集成包括诸如 Mattermost、Slack 和 IRC 等聊天服务，如 Github 和 Gitlab 等软件锻造，如 Zapier 或 IFTT 等通用托管服务，以及许多家庭自动化套件，如 Home Assistant 等。天空是限制，因为 Webhook 几乎在各处发送和接收。

虽然您可以在一个小时内编写一个最小的 Webhook 客户端或服务器，但今天几乎每种编程语言中已有许多选项可用。聊天软件通常提供内置的 Webhook 触发器，用户可以调用该触发器，通常通过 /command 样式语法。IRC 服务器也没有被遗忘，通常通过守护程序或插件。

Webhook 的一个不那么明显的优势是能够界定安全性和特权。低特权用户可以在远程系统上调用 Webhook。远程 Webhook 服务也可以以低特权运行，管理验证和基本语法检查，然后在满足要求后调用更高特权的任务。也许最终的任务有权访问一个特权令牌，该令牌允许重新启动服务、部署新代码，或让孩子们再享受一小时的屏幕时间。

像 GitHub、GitLab 和自托管选项这样的常见软件锻造厂也提供了这些功能，使其能够包括分支名称、提交和进行更改的用户。

这使得可以相对容易地构建工具，以更新站点、重启系统或根据需要触发更复杂的工具链。

## 架构

典型的安排包括一个服务器监听传入的请求和一个客户端，客户端提交请求和一些参数，可能包括一些身份验证和授权。

## 服务器

首先，让我们讨论服务器端。通常，这将是一个守护程序，监听匹配某些条件以便进行处理的 HTTP 请求。如果这些条件不被满足，请求将被拒绝，并返回适当的 HTTP 状态码；对于成功的提交，可以从批准的请求中提取参数，然后根据需要调用自定义操作。

## 客户端

由于服务器使用 HTTP，几乎任何客户端都可以使用。 cURL 是一个非常流行的选择，但我们将使用一个稍微更加愉快的名为 gurl 的选择，它内置支持 HMAC 签名。

## 消息

消息通常是一个 JSON 对象。对于关心重放攻击或定时攻击的人，应在消息体中包含时间戳，并在进一步处理之前验证它。如果您的 Webhook 工具包可以签名和验证特定的头部，那也是一个选项，但大多数工具包不支持。

## 安全性

HTTP 请求的主体可以使用共享密钥进行签名，然后将生成的签名作为消息头提供。这不仅提供了身份验证的手段，还证明请求在传输过程中没有被篡改。它依赖于共享密钥，使得两端可以独立验证消息签名，使用附加的 HTTP 头部与消息主体。

最常见的签名方法是 HMAC-SHA256。这是两种密码算法的组合 —— 熟悉的 SHA256 哈希算法，它可以为更大的消息（在我们的例子中为 HTTP 正文）提供安全摘要，以及 HMAC 方法，该方法使用一个秘密密钥并将其与消息混合以生成一个唯一的代码，类似于数字签名。

这些功能结合在一起，以生成一个高完整性检查，以确定消息是否被篡改。这就像是对内容的数字封印，并确认消息必须是从知道共享秘密的一方发送来的。

请注意，同时使用 TLS 加密和签名可以提供所包含消息的机密性和完整性，但不能提供可用性。一个处于有利位置的攻击者可以中断或淹没中间网络，消息将在没有通知的情况下丢失。

常见做法是在 webhook 的正文中包含时间戳，由于这由 HMAC 签名保护，因此可以减轻时间和重播攻击。

请注意，非时间戳正文将始终具有相同的签名。这可能很有用。例如，这允许预先计算 HMAC 签名，并使用不变的 HTTP 请求触发远程操作，而无需在发出 webhook 请求的系统上公开 HMAC 密钥。

## 把所有内容整合在一起

我们将安装几个软件包来帮助，一个 Webhook 服务器，一个众所周知的工具 curl，最后是 gurl，一个简化签署 Webhook 的工具。

`$ sudo pkg install -r FreeBSD www/webhook ftp/curl www/gurl`

让我们启动并运行我们的服务器，使用这个最小示例，将其保存为 webhooks.yaml。

它将使用 logger(1) 命令，将成功 Webhook 的 HTTP 用户代理标头写入 /var/log/messages 中的短条目。

请注意，存在一个触发规则键，确保 HTTP 查询参数 secret 与单词 squirrel 匹配。

目前我们没有 TLS 安全性，也没有 HMAC 签名，因此这不是一个非常安全的系统。

`---- id: logger  execute-command: /usr/bin/logger  pass-arguments-to-command:  - source: string    name: '-t'  - source: string    name: 'webhook'  - source: string    name: 'invoked with HTTP User Agent:'  - source: header    name: 'user-agent'  response-message: |    webhook executed  trigger-rule-mismatch-http-response-code: 400&nbs;&nbs;trigger-rule:    match:      type: value      value: squirrel    &nbs; parameter:       source: url       name: secret`

在终端中运行 webhook -debug -hotreload -hooks webhook.yaml 命令。使用的标志应该是不言自明的。

另外一个终端窗口中运行 tail -qF /var/log/messages | grep webhook，以便我们实时查看结果。

最后，使用 curl 来触发 webhook，首先不带查询参数，然后再次带上它：

$ curl -4v ‘http://localhost:9000/hooks/logger’

* 尝试 127.0.0.1:9000...
* 已连接到 localhost (127.0.0.1) port 9000 › GET /hooks/logger HTTP/1.1 › Host: localhost:9000 › User-Agent: curl/8.3.0 › Accept: / › ‹ HTTP/1.1 400 Bad Request ‹ 日期: 2023 年 10 月 20 日 星期五 12:50:35 GMT| ‹ 内容长度: 30 ‹ 内容类型: text/plain; charset=utf-8 ‹
* 到主机 localhost 的连接 #0 保持完整 Hook 规则未满足。

Note how the failed request is rejected using the HTTP status specified in the webhooks.yaml config file and the returned HTTP body explains why.

Providing the required query and secret paramter:

`$ curl -4v 'http://localhost:9000/hooks/logger?secret=squirrel'* Trying 127.0.0.1:9000...* Connected to localhost (127.0.0.1) port 9000› GET /hooks/logger?secret=squirrel HTTP/1.1› Host: localhost:9000› User-Agent: curl/8.3.0› Accept: */*›‹ HTTP/1.1 200 OK‹ Date: Fri, 20 Oct 2023 12:50:39 GMT‹ Content-Length: 17‹ Content-Type: text/plain; charset=utf-8‹webhook executed* Connection #0 to host localhost left intact`

The hook is executed and we can see the result in syslog output.

`Oct 20 12:50:39 akai webhook[67758]: invoked with HTTP User Agent: curl/8.3.0`

## 使用 HMAC 来保护 Webhooks

之前描述的 HMAC 签名，当应用于 HTTP 主体并作为签名发送时，是防篡改的，提供身份验证和完整性保护，但仅适用于主体，不适用于头部。让我们实施这一点。我们的第一步是生成一个简短的密钥，并修改 webhook.yaml 以要求验证。

`$ export HMAC_SECRET=$(head /dev/random | sha256)`

对于本文，我们将使用一个更容易记忆的密钥“n0decaf”，但您应该使用一个强壮的密钥。

用这个文件替换 webhook.yml 文件，该文件将从有效载荷中提取两个 JSON 值（已签名，因此受信任），并将它们传递给我们的命令以执行。

`---- id: echo  execute-command: /bin/echo  include-command-output-in-response: true  trigger-rule-mismatch-http-response-code: 400  trigger-rule:    and:    # ensures payload is secure -- headers are not trusted    - match:        type: payload-hmac-sha256        secret: n0decaf        parameter:          source: header          name: x-hmac-sig  pass-arguments-to-command:  - source: ‘payload’    name: ‘os’  - source: ‘payload’    name: ‘town’`

并使用 openssl dgst 计算主体上的签名

`$ echo -n ‘{“os”:”freebsd”,”town”:”vienna”}’ \<br/>    | openssl dgst -sha256 -hmac n0decafSHA2-256(stdin)= f8cb13e906bcb2592a13f5d4b80d521a894e0f422a9e697bc68bc34554394032`

有了主体和签名，现在让我们发出第一个已签名请求:

`$ curl -v http://localhost:9000/hooks/echo \<br/>    --json {“os”:”freebsd”,”town”:”vienna”} \<br/>    -Hx-hmac-sig:sha256=f8cb13e906bcb2592a13f5d4b80d521a894e0f422a9e697bc68bc34554394032*  Trying [::1]:9000...* Connected to localhost (::1) port 9000› POST /hooks/echo HTTP/1.1› Host: localhost:9000› User-Agent: curl/8.3.0› x-hmac-sig:sha256=f8cb13e906bcb2592a13f5d4b80d521a894e0f422a9e697bc68bc34554394032› Content-Type: application/json› Accept: application/json› Content-Length: 32›‹ HTTP/1.1 200 OK‹ Date: Sat, 21 Oct 2023 00:41:57 GMT‹ Content-Length: 15‹ Content-Type: text/plain; charset=utf-8‹freebsd vienna* Connection #0 to host localhost left intact`

在服务器端以-debug 模式运行

`[webhook] 2023/10/21 00:41:57 [9d5040] incoming HTTP POST request from [::1]:11747[webhook] 2023/10/21 00:41:57 [9d5040] echo got matched[webhook] 2023/10/21 00:41:57 [9d5040] echo hook triggered successfully[webhook] 2023/10/21 00:41:57 [9d5040] executing /bin/echo (/bin/echo) with arguments [“/bin/echo” “freebsd” “vienna”] and environment [] using as cwd[webhook] 2023/10/21 00:41:57 [9d5040] command output: freebsd vienna[webhook] 2023/10/21 00:41:57 [9d5040] finished handling echo‹ [9d5040] 0‹ [9d5040]‹ [9d5040] freebsd vienna[webhook] 2023/10/21 00:41:57 [9d5040] 200 | 15 B | 1.277959ms | localhost:9000 | POST /hooks/echo`

每次单独计算签名都容易出错。gurl 是一个早期项目的分支，添加了自动 HMAC 生成以及一些关于处理和处理 JSON 的便利功能。

签名类型和签名头名称被放置在密钥前面，并通过:连接。这被导出为一个环境变量，以便在历史记录中不直接可见。

`$ export HMAC_SECRET=sha256:x-hmac-sig:n0decaf$ gurl -json=true -hmac HMAC_SECRET \<br/>  POST http://localhost:9000/hooks/echo \<br/>  os=freebsd town=otutahiPOST /hooks/echo HTTP/1.1Host: localhost:9000Accept: application/jsonAccept-Encoding: gzip, deflateContent-Type: application/jsonUser-Agent: gurl/0.2.3X-Hmac-Sig: sha256=f634363faff03deed8fbcef8b10952592d43c8abbb6b4a540ef16af0acaff172{“os”:”freebsd”,”town”:”otutahi”}`

正如我们在上面所看到的，签名已经为我们生成，添加 JSON 键值对非常简单，无需引号和转义。

返回的响应如下，已经漂亮地打印出来：服务器已验证 HMAC，两个键的值被提取并作为参数传递给我们的回显命令，并捕获并返回到 HTTP 响应体中。

`HTTP/1.1 200 OKDate : Sat, 21 Oct 2023 00:50:25 GMTContent-Length : 16Content-Type : text/plain; charset=utf-8freebsd otutahi`

更复杂的示例可以在 port 的示例 webhook.yaml 或广泛的文档中找到。

## 保护 Webhook 内容

虽然使用 HMAC 可以防止消息体被篡改，但它仍然以明文形式对那些可恶的黑客可见。

让我们为本地主机上的 Webhook 服务器添加一些传输层安全性，使用自签名的 TLS 密钥和证书，并重新启动 Webhook 服务器：

`$ openssl req -newkey rsa:2048 -keyout hooks.key \<br/>  -x509 -days 365 -nodes -subj ‘/CN=localhost’ -out hooks.crt$ webhook -debug -hotreload \<br/>  -secure -cert hooks.crt -key hooks.key \<br/>  -hooks webhook.yaml`

Curl 命令将需要一个额外的 -k 参数来忽略我们的自签名证书，但除此之外，一切照旧：

`curl -4vk https://localhost:9000/hooks/logger?secret=squirrel'* Trying 127.0.0.1:9000...* Connected to localhost (127.0.0.1) port 9000* ALPN: curl offers h2,http/1.1* TLSv1.3 (OUT), TLS handshake, Client hello (1):* TLSv1.3 (IN), TLS handshake, Server hello (2):* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):* TLSv1.3 (OUT), TLS handshake, Client hello (1):* TLSv1.3 (IN), TLS handshake, Server hello (2):* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):* TLSv1.3 (IN), TLS handshake, Certificate (11):* TLSv1.3 (IN), TLS handshake, CERT verify (15):* TLSv1.3 (IN), TLS handshake, Finished (20):* TLSv1.3 (OUT), TLS handshake, Finished (20):* SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256* ALPN: server accepted http/1.1* Server certificate:* subject: CN=localhost* start date: Oct 20 13:05:09 2023 GMT* expire date: Oct 19 13:05:09 2024 GMT* issuer: CN=localhost* SSL certificate verify result: self-signed certificate (18), continuing anyway.* using HTTP/1.1› GET /hooks/logger?secret=squirrel HTTP/1.1› Host: localhost:9000› User-Agent: curl/8.3.0› Accept: */*›* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):› HTTP/1.1 200 OK› Date: Fri, 20 Oct 2023 13:12:07 GMT› Content-Length: 17› Content-Type: text/plain; charset=utf-8›webhook executed* Connection #0 to host localhost left intact`

gurl 没有这样的选项，希望您能正确地完成任务。对于生产使用，更好的方式是使用反向代理，如 nginx 或 haproxy 提供强大的 TLS 终止，并允许使用公共 TLS 证书，通过 Let's Encrypt 和类似的服务。

## 使用 Github 和 Webhooks 更新网站

要使此过程成功，您需要拥有自己的域和一个用于托管守护程序的小型服务器或虚拟机。虽然本文无法涵盖设置您自己的网站、TLS 加密证书和 DNS 的全部细节，但下面的步骤基本上对于任何软件锻造都是类似的。

您需要设置代理服务器，例如 Caddy、nginx、haproxy 或类似的服务器，并确保 TLS 正常工作。一个很好的选择是使用 ACME 协议，通过 Let's Encrypt 来为您维护它。

您需要调整代理服务器以将适当的请求路由到 Webhook 守护程序。请考虑限制可以访问它的 IP 地址，并限制 HTTP 方法。Github 的 API 有一个 /meta 终点来检索他们的 IP 地址，但是您需要及时更新这些信息。

启用 Webhook 服务，使用我们之前使用的相同选项，并通过 sudo service webhook start 启动您的守护进程。

`# /etc/rc.conf.d/webhookwebhook_enable=YESwebhook_facility=daemonwebhook_user=wwwwebhook_conf=/usr/local/etc/webhook/webhooks.ymlwebhook_options=” \<br/>  -verbose \<br/>  -hotreload \<br/>  -nopanic \<br/>  -ip 127.0.0.1 \<br/>  -http-methods POST \<br/>  -port 1999 \<br/>  -logfile /var/log/webhooks.log \<br/>  “`

您需要从外部验证 URL 和 Webhook 守护程序是否可访问。

在您的软件 Forge 端，创建一个新的 JSON 格式 Webhook，并使用共享的 HMAC 密钥，在每次推送到您的代码仓库时调用它。

例如，使用 Github，您必须提供一个:

* 负载 URL，指向用于代理内部 webhook 守护程序的外部 URL
* 内容类型为 application/json
* 共享密钥，例如示例中的 n0decaf

![](https://freebsdfoundation.org/wp-content/uploads/2024/01/02_add_webhook.png)

一旦在 GitHub 端创建了 Webhook，您应该能够确认已成功接收到事件，在下次推送代码时，您可以在 GitHub 网站上查看 GitHub 发送的请求以及您的守护程序提供的响应

![](https://freebsdfoundation.org/wp-content/uploads/2024/01/03_webhook_req.png)

——FreeBSD Journal，2023 年 11 月/12 月
