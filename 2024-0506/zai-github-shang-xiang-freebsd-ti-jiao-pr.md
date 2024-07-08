# 在 GitHub 上向 FreeBSD 提交 PR


 由 WARNER LOSH 撰写

自由 BSD 项目最近开始支持 GitHub 拉取请求（PRs）以便更容易贡献。我们发现通过我们的缺陷跟踪器 Bugzilla 接受补丁导致许多有用的贡献被忽视并变得陈旧，因此贡献者应该更倾向于使用 GitHub PRs 进行更改，将漏洞留在 Bugzilla 中。虽然 Phabricator 对开发人员很有效，但我们也发现很容易在那里丢失外部贡献者的更改。除非您直接与告诉您使用 Phabricator 的 FreeBSD 开发人员一起工作，请改用 GitHub。GitHub PRs 更容易跟踪，更容易处理，并且对更广泛的开源社区更为熟悉。我们希望更快的决策，更少的丢弃更改，并为所有人提供更好的体验。

由于 FreeBSD 的志愿者时间有限，本项目制定了标准、规范和政策，以有效利用他们的时间。您需要了解这些内容才能提交一个好的 PR。我们有一些自动化程序帮助提交者修复常见错误，从而使志愿者可以审查几乎就绪的提交。请理解我们只能接受最有用的贡献，有些贡献是无法被接受的。

接下来，我将介绍如何将您的更改转换为 Git 分支，如何完善它们以符合 FreeBSD 项目的标准和规范，如何从您的分支创建 PR，以及如何预期审查过程。然后，我将介绍志愿者如何评估 PR，并提供完善 PR 的技巧。

本文关注于基础系统的提交，而不是文档或ports树。这些团队仍在修订这些存储库的详细信息。

## 项目标准

该项目对系统的各个方面有详细的标准。这些标准在 FreeBSD 开发者手册和 FreeBSD 提交者指南中进行了描述。编码标准在 FreeBSD 手册页中记录。根据惯例，手册页被分为多个部分。所有样式手册页出于历史原因都在第 9 部分。对手册页的引用通常呈现为页面名称，后跟其部分编号在括号中，例如 style(9)或 cat(1)。这些文档可以在任何 FreeBSD 系统上使用 man 命令获取，也可以在线查看。

该项目致力于创建一个文档齐全的集成系统，涵盖了控制机器的内核以及常见 Unix 工具的用户空间实现。贡献应写得清晰，并包含相关评论。当行为发生变化时，应更新相关手册页。例如，当您向命令添加一个标志时，也应将其添加到手册页中。当库中添加新功能时，应为这些功能添加新的 man 页面。最后，该项目认为源代码控制系统中的元数据是系统的一部分，因此提交消息应符合项目的标准。

该项目对 C 和 C++代码的标准在 style(9)中描述。这种风格通常被称为“内核规范形式”，并采用了 Kernighan 和 Ritchie 的《C 程序设计语言》中使用的风格。这是研究 unix 使用的标准，后来在 Berkeley 的 CSRG 中继续使用，并产生了 BSD 版本。FreeBSD 项目在这些实践的基础上进行了现代化。这种风格是贡献代码的首选风格，并描述了系统中大多数代码使用的风格。更改这些代码的贡献应遵循这种风格，但有一些文件有自己独特的风格。Lua 和 Makefiles 也有自己的标准，分别在 style.lua(9)和 style.Makefile(9)中找到。

Commit messages follow the form favored by the Open Source communities that use git. The first line of the commit message should summarize the entire commit, but do so in 50 or so characters. The rest of the message should describe what changed and why. If what changes is obvious, only explaining why is preferred. The lines should be 72 characters or fewer. It should be written in the present tense, with an imperative tone. It ends with a series of lines that Git calls “trailers” which The Project uses to track additional data about the commits: where they came from, where details about the bug can be found, etc. The Commit Log Message section of the [Committer’s Guide](https://docs.freebsd.org/en/articles/committers-guide/#commit-log-message) covers all the details.

## 不接受的更改

在通过 GitHub 接受更改的几年实验后，该项目不得不设定一些限制，以确保未获得项目仓库写入权限的人员通过 GitHub 提交的更改限制在合理范围内。这些限制确保了验证和应用更改的志愿者能够最有效地利用他们的时间。因此，该项目无法接受以下内容：

* 在 GitHub 上无法审核的更改过大
* 注释中的拼写错误
* 通过对树上运行的静态分析器发现的更改（除非它们包含了对静态分析器发现的错误的新测试用例）。针对与我们的测试工具箱不良互动的系统部分中的“显然正确”修复，可根据具体情况进行例外处理。
* 理论性的变化，但没有具体的错误或可表达的行为缺陷。
* 没有配备前后对比度量以显示改进的性能优化。微小优化很少值得，因为编译器和 CPU 技术通常会使它们在几年内变得过时（甚至更慢）。
* 有争议的变化。这些需要首先在 freebsd-arch@freebsd.org 或最适合的邮件列表上进行社交化。GitHub 为讨论这类问题提供了一个很差的论坛。

PRs 应该在某种用户可见的方式上改进项目。

## 评估标准

* 这个变更是项目正在接受的吗？
* 变更的范围/规模是否合适？
* 是否有合理数量的提交（比如少于 20 个）？
* 每个提交的大小是否适合审查（比如少于 100 行）？
* C 和 C++代码是否符合 style(9)风格（或文件当前的风格）
* 对 lua 的更改是否符合 style.lua(9)规范
* 对 Makefile 的更改是否符合 style.Makefile(9)规范
* 更改 man 页面是否同时通过 mdoc -Tlint 和 igor ？
* 有争议的更改是否在适当的邮件列表中讨论过？
* make tinderbox 是否成功运行？
* 是否修复了特定且明确的问题，或者添加了特定的功能？
* 提交消息是否良好？

在避免以下这些问题的同时：

* 引入新的测试回归吗？
* 引入行为回归吗？
* 引入性能回归吗？

## 进程概述

从高层次来看，贡献到 FreeBSD 是一个简单直接的过程，尽管深入细节可能会使这种简单性变得不那么明显。

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/basicflowchart.png)

1. FreeBSD 开发者直接向 FreeBSD 存储库推送提交，该存储库托管在 FreeBSD.org 集群中。
2. 每 10 分钟，FreeBSD 源代码库会被镜像到 freebsd-src GitHub 存储库。
3. 想要创建 PR 的用户会在他们的 freebsd-src 存储库的分支上创建一个分支。
4. 用户分支上的更改会用来创建 FreeBSD PR。
5. 一个 FreeBSD 开发人员审核 PR，提供反馈，并可能要求用户进行更改。
6. 一个 FreeBSD 开发人员将更改推送到 FreeBSD 源代码库。

## 为提交拉取请求做准备

如果您还没有 GitHub 帐户，您需要创建一个。 此链接将指导您完成创建新 GitHub 帐户的过程。 由于许多人已经因其他原因拥有 GitHub 帐户，我们将跳过深入讨论详细信息。

下一步是将 FreeBSD 的存储库 fork 到您的帐户中。 使用 GitHub Web 界面是创建分支并解释的最简单方法，因为您只需执行一次此操作。 对分支的更改不会影响 FreeBSD 的存储库。 用户可以通过单击“Fork”按钮（如图 1 所示）来分叉存储库。 您将要点击突出显示的“创建新分支”菜单项。 这将打开类似于图 2 的屏幕。 从这里，点击绿色的“创建分支”按钮。

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/Figure1_witharrows.jpg)

图 1：单击“分叉”旁边的向下箭头后，您将看到一个弹出式窗口，显示创建分支对话框。

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/Figure2.png)

创建分支图 2，第 2 部分。

一旦您点击“创建分支”，GUI 将会重定向到新创建的存储库。您可以在通常的位置复制您需要克隆存储库的 URL，如图 3 所示。

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/Figure3.png)

复制 URL 以克隆（我以前用旧存储库名称创建的分支）图 3。

这些是您将在 GitHub Web 界面上执行的步骤。其余的命令将在运行 FreeBSD 的主机上的终端窗口中完成。为简单起见，屏幕截图已更改为命令或命令及其生成的输出。

使用以下命令克隆您新创建的存储库。

`% git cloneCloning into 'freebsd-src'...remote: Enumerating objects: 3287614, done.remote: Counting objects: 100% (993/993), done.remote: Compressing objects: 100% (585/585), done.remote: Total 3287614 (delta 412), reused 815 (delta 397), pack-reused 3286621Receiving objects: 100% (3287614/3287614), 2.44 GiB | 22.06 MiB/s, done.Resolving deltas: 100% (2414925/2414925), done.Updating files: 100% (100972/100972), done.% cd freebsd-src`

请注意，您应该将上述命令中的“user”更改为您的 GitHub 用户名。“-o github”将会将此远程命名为“github”，在下面的示例中将使用它。

PR 工作流通常需要一个分支。我们假设您已经按照类似以下命令操作，尽管有许多使用预先存在的分支的方法超出了本文的范围。

`% git checkout -b journal-demo% # make changes, test them etc% git commit`

所有您进行的提交都必须将您的真实姓名和电子邮件地址作为提交的“作者”。Git 有两个配置字段用于此目的。user.name 包含您的真实姓名。user.email 包含您的电子邮件地址。您可以这样设置它们：

`% git config --global user.name “Pat Bell”% git config –global user.email “pbell@example.com”`

此外，请阅读我们关于提交日志消息的建议，并在创建提交时遵循它。

大多数通过 PR 方式提交的更改都很小，所以我们将继续提交它们。但是，如果您有较大的更改，请在提交之前阅读下面的评估标准，以获得更顺畅的流程。

## 提交您的 Pull 请求

下一步是将 journal-demo 分支推送到 GitHub（与上述类似，用您的 GitHub 用户名替换下面的“user”：

`% git push githubEnumerating objects: 24, done.Counting objects: 100% (24/24), done.Delta compression using up to 8 threadsCompressing objects: 100% (16/16), done.Writing objects: 100% (16/16), 5.21 KiB | 1.74 MiB/s, done.Total 16 (delta 13), reused 0 (delta 0), pack-reused 0remote: Resolving deltas: 100% (13/13), completed with 8 local objects.remote:remote: Create a pull request for ‘journal-demo’ on GitHub by visiting:remote: https://github.com/user/freebsd-src/pull/new/journal-demoremote:To github.com:user/freebsd-src.git* [new branch] journal-demo -> journal-demo`

`You’ll notice that GitHub helpfully tells you how to create a pull request. When you visit the above URL, you’re presented with a blank form, as shown in Figure 4.`

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/figure4.png)

图 4: 拉取请求提交表单。

在“添加标题”字段中，添加一个简要描述你的工作的内容，以传达更改的本质。保持在约十几个词以内，以便易于阅读。如果分支只有一个提交，可以使用提交消息的第一行来作为此更改的标题。如果有多个提交，则需要总结它们为一个简短的标题。

在“添加描述”字段中，写下您的更改摘要。如果这是一个只有一个提交的分支，请使用提交消息的正文部分。如果有多个提交，则创建一个简短的摘要，简要描述解决的问题。解释您做出了什么更改以及为什么，如果不是显而易见的话。

Figure 4 中的示例试图解决贝尔实验室和伯克利之间著名的历史争端。这是下面概述的一个有争议的提交的很好的例子。这是一个应该得到社会化的有争议的提交的很好的例子。

## 期待什么

在您提交之后，评估过程就开始了。将运行几个自动检查器。这些检查确保您的提交的格式和样式符合我们的指南。它们确保所提出的更改可以编译。它们将提供反馈，指出您在有人查看之前应该进行的更改。其中一些测试需要时间，因此在提交后几个小时再来检查是个好主意。自动测试标记的项目将是我们的志愿者要求您更正的首要事项，因此积极解决这些问题可以节省每个人的时间。

## 回复反馈

一旦收到反馈，通常需要进行代码更改。请进行建议的更改。通常这意味着您将不得不编辑一些您的更改的子集（无论是提交消息还是提交本身）。 GitLab 有一个关于使用 git rebase 机制的很好的教程。

一旦您进行了更改，您将需要将更改推送回您的分支，以便 PR 更新并且反馈循环重新启动：

`% git push github --force-with-lease`

## 供应链攻击

最近，一名恶意行为者攻击了 xz 源代码库，插入了一些代码，从而危害了某些 Linux 系统上的 sshd。由于一定程度的幸运和流程，FreeBSD 没有受到这次攻击的影响。我们的流程经过设计，可以通过多重保护层来抵御此类攻击。我们在允许代码进行测试之前会进行代码审查。只有在提交的代码中没有明显恶意行为时我们才运行自动化测试。在开源项目必须应对日益恶劣的工作环境的情况下，某些看似不必要的问题往往是由此驱使的。

## 结束

无论您是一个偶尔会对 FreeBSD 进行微调的休闲用户，还是一个提交变更如此频繁以至于将获得提交权限的开发者，项目都欢迎您的贡献。本文尝试覆盖这些基础内容，但更适合偏向休闲用户。在线资源将帮助您处理超出基础情况的情形。

WARNER LOSH 自 FreeBSD 项目成立之前就开始贡献开源，甚至在“开源”这个术语正式定义之前。最近，他深入探索 Unix 的早期历史，揭示其丰富而隐秘的遗产。他与妻子和女儿居住在科罗拉多州的一座稻草房屋中，这座房屋由太阳能、一个小锅炉和偶尔的古董计算机加热。
