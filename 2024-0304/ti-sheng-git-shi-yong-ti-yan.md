# 提升 Git 使用体验

- 作者：**BENEDICT REUSCHLING**
- 原文链接：<https://freebsdfoundation.org/our-work/journal/browser-based-edition/development-workflow-and-ci/enhance-your-git-experience/>

版本控制历史悠久。掌握一种检索某个特定版本文件的方法，是我们旅程的开端。毕竟，这仅是备份软件的功能。有些人将版本控制系统（VCS）用于该用途，但这仅是皮毛。VCS 真正有价值的地方，是能够审查文件的历史变更。与他人共用，可使重要项目文件与他们保持同步。在这种情况下，blame、diff、分支和合并等功能变得更为耦合。这些功能使得值得在群体内共享的东西（文本、源代码、配置文件等）上进行协作成为可能。许多将大量上述文件纳入版本控制的大型项目，如果没有基于 VCS 构建的复杂工具，将失去可维护性。版本控制系统来了又去，无论是商业供应商还是开源提供商。其中大都实现了用户所确立的基本功能，引入一套全新术语和复杂工具往往会被视为推广障碍。

在 FreeBSD 项目漫长的历史中，每一次的代码更改都附有一个说明文本，即提交信息。对某些人来说，这似乎很麻烦。但事实上，在溯源过去的问题时，很有帮助。为何在 15 年前做出了这项改变？是谁做出了这个修改，当次提交还包含了哪些别的文件，以及实际受影响的代码行是哪些？这些不仅是版本控制系统生成的 diff 相关联的问题，而且提交信息对于掌握如今代码中正进行着的开发是一种宝贵的帮助。这通常贯通了过去与未来，因为 Unix 系统已经存在了很久，而且如今运行的硬件也是过去进行变更的人无法想象的。

由于 FreeBSD 存在了相当长时间，保留这段丰富的历史极为重要。尤其是当必须转换到其他版本控制系统时。随着时间的推移，FreeBSD 曾经使用过 CVS（我可以肯定，在早期也用过 RCS），然后迁移到了 Subversion，现在则使用着 git。每当进行这样的迁移，一个主要的目标就是保留每一次的变更，及提交信息。

有时候被迫切换版本控制系统，因为供应商破产了，或者有比当前解决方案更吸引人的东西出现。最好的往往带有恶意。Git 如今已经成为版本控制的事实标准，即使它有些粗糙和学习曲线也十分陡峭。在我看来，作为 FreeBSD 项目的用户/消费者以及 committer，重新熟悉 git，并让 git 成为为我工作（而不非其他）的工具，让我花了一些时间。我可能仍然不能完全掌握所有的内部工作方式，但幸运的是，我不必这样做。为 src 和 ports 工作的开发人员还会使用更多像分支、合并、Tag、cherry-pick、rebase 以及其他某些在文档存储库中不需要的功能。尽管如此，有时候在 ports 说明信息或改正手册页中的拼写错误时，还是需要像我这样，对这些概念至少有基本了解的人才能做事。

问题也在于使用频率。如果我在一年内某功能只用过一两次，就可能需要再次查阅，因为距上一次使用已经有一段时间了。克隆新树、拉取更新、进行更改、提交和推送这些基础操作，如果你经常进行，就会非常熟悉。我把它比作基本的 vi 使用。我可以打开文档，进行修改，保存并退出。但像 vi 这样强大的编辑器和其他类似工具提供的其他功能，可能会永远隐藏在我看不见的地方。同样，对于 git，它有更多功能隐藏于幕后，用户并不了解或者压根不需要。然而，一般对于这些高级功能有点知识和配置，即使对开发者，也极有帮助的，对用户亦如此。

FreeBSD 14 弃用了工具 portsnap，许多人用它来获取和更新 ports。手册和其他地方的说明现在要求用户直接从 git 检出 ports。当用户需要 src 时，亦如此。比如他们可能需要重新编译 VirtualBox 的内核模块。这些都是很好的使用案例，但有一个问题：由于上述悠久且丰富的历史，克隆 src 和 ports 需要很长时间。让我们看看，是否可以通过一些配置和工具来加速该过程。我们还将了解 git 提供的其他精彩功能，无论对普通用户张三还是开发者李四来说，都一样。

## 加速克隆

首先，在 FreeBSD 上安装 git ，请运行 `pkg install git`。由于我们还没有 ports，况且现在没有 `portsnap` 了。这可能会在无法访问 FreeBSD 镜像站的系统上，造成先有鸡，还是先有蛋的悖论。在你的局域网中，poudriere 机器可能是解决方案，或者在不同的机器上下载文件，复制二进制文件，然后用 `pkg install ./<软件包名>` 进行安装。

请注意，`git port` 本身包含了大量可配置的选项。由于我们刚开始，可以暂时忽略这些选项，并在经验丰富时重新审视它们。如果你发现某些有用的东西，请考虑写成博客或为 FreeBSD 杂志写篇文章。毕竟，一个好汉三个帮，我不能自己盲干。

如 git 可用，我们可以返回上面的使用案例：下载 ports。FreeBSD 手册第 4 章告诉我们，我们可以获取 HEAD 分支（最新最全的）或者稍微保守一点，获取季度分支。无论我们选择哪一个，我们都会看到一个非常熟悉的命令 `git clone`。这都很好，但是下载全部的这些文件和目录需要很长时间。让我们看看写这篇文章时的季度分支。我将使用 time(1) 命令来测量下载时间。

```
$ time git clone https://git.FreeBSD.org/ports.git -b 2024Q1 /usr/ports
Cloning into '/usr/ports'...
remote: Enumerating objects: 6125935, done.
remote: Counting objects: 100% (960/960), done.
remote: Compressing objects: 100% (142/142), done.
Receiving objects: 100% (6125935/6125935), 1.20 GiB | 36.28 MiB/s, done.
remote: Total 6125935 (delta 925), reused 833 (delta 818), pack-reused 6124975
Resolving deltas: 100% (3700108/3700108), done.
Updating files: 100% (158490/158490), done.
git clone https://git.FreeBSD.org/ports.git /usr/ports
0.00s user 0.03s system 0% cpu 3:34.48 total
```

除带宽外，对我而言，3 分 34 秒太长了。我们可以让他更快。由于我们不需要所有的历史，仅需最新版本的文件，若使用 `--depth-1` 进行浅克隆会快得多。

```
time git clone --depth=1 https://git.FreeBSD.org/ports.git /usr/ports
Cloning into '/usr/ports'...
remote: Enumerating objects: 194509, done.
remote: Counting objects: 100% (194509/194509), done.
remote: Compressing objects: 100% (182218/182218), done.
remote: Total 194509 (delta 11904), reused 120301 (delta 5787), pack-reused 0
Receiving objects: 100% (194509/194509), 85.40 MiB | 10.48 MiB/s, done.
Resolving deltas: 100% (11904/11904), done.
Updating files: 100% (158490/158490), done.
git clone --depth=1 https://git.FreeBSD.org/ports.git /usr/ports
0.01s user 0.01s system 0% cpu 28.709 total
```

确实快得多（29 秒），我得到了我想要的东西。如果我需要完整的历史记录：因为我在解决一个 bug，那么我可以使用一个过滤函数首先获取整个提交历史，但不包括历史记录。后者可能作为一个单独的步骤出现。我希望减少下载时间，因此让我们尝试这个：

```
time git clone --filter=blob:none https://git.FreeBSD.org/ports.git /usr/ports
Cloning into '/usr/ports'...
remote: Enumerating objects: 3706789, done.
remote: Counting objects: 100% (794/794), done.
remote: Compressing objects: 100% (82/82), done.
remote: Total 3706789 (delta 771), reused 721 (delta 712), pack-reused 3705995
Receiving objects: 100% (3706789/3706789), 704.87 MiB | 48.79 MiB/s, done.
Resolving deltas: 100% (2043361/2043361), done.
remote: Enumerating objects: 152073, done.
remote: Counting objects: 100% (63494/63494), done.
remote: Compressing objects: 100% (61224/61224), done.
remote: Total 152073 (delta 7810), reused 2276 (delta 2270), pack-reused 88579
Receiving objects: 100% (152073/152073), 78.98 MiB | 10.93 MiB/s, done.
Resolving deltas: 100% (11301/11301), done.
Updating files: 100% (158490/158490), done.
git clone --filter=blob:none https://git.FreeBSD.org/ports.git /usr/ports
0.00s user 0.03s system 0% cpu 1:51.29 total
```

Git 将工作分为两部分：首先，所有的 blob（这里是文件）被过滤掉，并获取历史。在第二步中，获取 blob。快于完全克隆，但慢于浅拷贝。检索的大小也有所不同，这有助于加快速度。在常规克隆中，我们下载了 1.20 GB。无 blob 克隆的两步过程让 git 先下载了 704.87 MB 的历史，然后是 78.98 MB。但这种好处也伴随着一个缺点：当我找到 bug 并想知道这是什么时引入的时候， `git blame` 操作需要首先从服务器获取这些修订版版。如果我在路上，没有网络访问，我就有点倒霉了。完全克隆能够给我这些信息，因为它已检索了所有的历史。再次强调，对于对获取文件本身感兴趣的非开发人员来说，这无所谓，好处在于下载时间更短。

## 更进一步

想象一下，如果你正在处理位于源代码中的 man 页面。下载整个内核、用户空间、工具和其中的所有内容对于初次克隆来说是相当多的工作。如果你只偶尔处理这些 man 页面会怎样呢？毫无疑问，他人会对其进行更改，我们需要知晓这些更改。如果我们的系统能够帮我们获取这些更改，使得我们本地的副本不会距离源代码树顶端太远那不是很好吗？git 中的 scalar 工具解决了这个问题：快速下载一个大型存储库并定期从上游检索更改。这将把本地克隆置于维护模式，这个功能的另一个花俏的说法就是这个。以下是如何使用它的方法：用 scalar 替换 `git` ，命令的其余部分则保持不变。

```
time scalar clone https://git.FreeBSD.org/src.git /usr/src
Initialized empty Git repository in /usr/src/src/src/.git/
remote: Enumerating objects: 2386494, done.
remote: Counting objects: 100% (258756/258756), done.
remote: Compressing objects: 100% (16493/16493), done.
remote: Total 2386494 (delta 253705), reused 244654 (delta 242263), pack-reused 2127738
warning: fetch normally indicates which branches had a forced update,
but that check has been disabled; to re-enable, use '--show-forced-updates'
flag or run 'git config fetch.showForcedUpdates true'
warning: fetch normally indicates which branches had a forced update,
but that check has been disabled; to re-enable, use '--show-forced-updates'
flag or run 'git config fetch.showForcedUpdates true'
remote: Enumerating objects: 20, done.
remote: Counting objects: 100% (17/17), done.
remote: Compressing objects: 100% (17/17), done.
remote: Total 20 (delta 0), reused 0 (delta 0), pack-reused 3
Receiving objects: 100% (20/20), 196.11 KiB | 16.34 MiB/s, done.
warning: fetch normally indicates which branches had a forced update,
but that check has been disabled; to re-enable, use '--show-forced-updates'
flag or run 'git config fetch.showForcedUpdates true'
branch 'main' set up to track 'origin/main'.
Switched to a new branch 'main'
Your branch is up to date with 'origin/main'.
crontab: no crontab for root
scalar clone https://git.FreeBSD.org/src.git
0.01s user 0.00s system 0% cpu 31.971 total
```

现在先无视那些警告，进程仍会完成。这里有关于定期获取更新的 crontab(1) 的一些内容。要将现有存储库转换为使用 scalar，无需再次克隆：在存储库的根目录中运行 `scalar register`，它将会将本地副本转换为使用它。不错！scalar 命令将设置一个 crontab 条目。如果你没有用户特定的 crontab（就像我这里为 root 用户设置的那样），那么运行 `crontab -e` 来设置它。如果一切顺利，`git` 将添加一个 scalar 运行的条目：

```
# BEGIN GIT MAINTENANCE SCHEDULE
# The following schedule was created by Git
# Any edits made in this region might be
# replaced in the future by a Git command.

29 1-23 * * * “/usr/local/libexec/git-core/git” --exec-path=”/usr/local/libexec/git-core” for-each-repo --config=maintenance.repo maintenance run --schedule=hourly
29 0 * * 1-6 “/usr/local/libexec/git-core/git” --exec-path=”/usr/local/libexec/git-core” for-each-repo --config=maintenance.repo maintenance run --schedule=daily
29 0 * * 0 “/usr/local/libexec/git-core/git” --exec-path=”/usr/local/libexec/git-core” for-each-repo --config=maintenance.repo maintenance run --schedule=weekly
# END GIT MAINTENANCE SCHEDULE
```

调整这些条目以适应你自己的需求，或保持不变。如果机器设置了正确的邮件设置，当 cron 作业运行时，你将收到包含获取的修订版本的消息。scalar 克隆和相关的维护作业所做的另一件事是将一个条目添加到你的 git 配置文件中，恰当地命名为 `.gitconfig`。

```
[scalar]
repo = /usr/src
[maintenance]
repo = /usr/src
```

这将直接将我们带入 git 的配置文件中。

## 创建你的每日提交历史。

随时间变化，你可能会在多个存储库上进行工作：一个用于工作，另一个用于私人项目——为你喜爱的开源项目做贡献。这些克隆存储库的配置可能不同。例如，你可能在提交时使用你的企业电子邮件来标识自己，但在提交到私人项目时可能不合适甚至不允许。因此，我们可以在存储库的一部分设有特定于项目的 `git/config` 和一个适用于你正在工作的所有存储库的全局 `git/config`。

全局 `.gitconfig` 位于你的主目录中。你可以直接编辑该文件（如果你知道你在做什么），或者使用 git 来管理文件的内容并设置适当的值。后者使用这种语法：

```
git config --global NAME VALUE
````

例如，要注册我的姓名以进行提交，我运行：

```
git config --global user.name Benedict Reuschling
```

这会在 .gitconfig 中出现这样的条目：

```
[user]
        name = Benedict Reuschling
```

你可以看到，像用户（电子邮件在其中，你最好也设置它）这样的条目有括号中的类别。其他的是 commit，diff 和 branch 。

### 更改提交行为

放下火把和草叉，我不喜欢用 nano 来写我的提交消息。要定义你自己的编辑器，执行这个命令：

```
git config --global core.editor nvim
```

你几乎总是希望将其设置为全局选项。在项目的 `.git/config` 中引入这个配置并将其提交，将导致生产效率急剧下降，因为其他贡献者将同你开始另一场神圣的编辑器之战，导致仓库中充斥着试图将其更改为他们自己个人喜欢的编辑器的修改。在同一天，讽刺你的老板永远地关上了你身后的公司大门，你会记住他最后说的话是什么——“干得漂亮”。

还有其他配置设置会（更积极地）影响你的提交体验。这里列出了较多选项，但默认设置很好。稍调一些选项可以在一定程度上改善你的 git 体验，并可能消除一些个人困扰（请参阅下文）。有关详细信息，请参阅 git-config(1)。

如何为复杂和常用但繁琐的命令定义别名？这就是别名部分派上用场的地方。我已经定义了这些别名：

```
[alias]
    last = last -1 HEAD
    g = log --graph --all --pretty=format:'%Cred%h -%C(yellow)%d%Creset %s %Cgreen(%ci) %C(bold blue)<%an (%ae)>'
```

类似地，当查看日志时，我希望至少看到这些字段：提交、作者、作者进行更改的日期，以及提交及其日期。通过查看 git-log(1) 并找出这个选项被称为更全面实现：

```
git config --global format.pretty fuller
```

在我的代码库中，我可以运行 `git last` 等同于运行命令 `git last -1 HEAD`。想想现在只需敲几下键盘，就能看到彩色等花哨的效果，我就更喜欢 `git lg` 命令了。你自己试试看，回头再谢我吧。

当我进行实际提交时，我希望看到提交了什么。Git 在提交消息的文本区域下方显示头修订版本与我的更改之间的差异，使用此设置：

```
git config --global commit.verbose true
```

### 请签署你的签名

谈到提交，为什么你不签署你的提交？"嗯，GPG/PGP 设置太复杂可能是一个回答。有一个解决方法：改用 SSH。在 GitHub 等服务或你的公司（或私人）的 GitLab 实例上，你已经上传了一个用于通过 SSH 拉取存储库的公钥。使用相同的密钥对提交进行签名为你的更改增加了额外的信誉。在这些平台上提交旁边经常有一个“已签名（signed）”图标或标签表示信用。设置非常简单，我想知道为什么到现在还不是默认设置。下面是具体方法：

```
git config --global gpg.format ssh
git config --global user.signingKey ‘ssh-ed25519 AAAAC3(...)34rve user@host’
```

是否应在所有地方使用相同的 SSH 密钥，目前存在争议。如果不希望，请从上面的最后一行中删除参数 `--global` ，并为每个存储库单独更改其密钥。

现在可以这样提交：

```
git commit -S
```

要始终签名，请将其设为默认：

```
git config --global commit.gpgsign true
```

但这是什么呢？当运行 `git show --show-signature` 时，不显示我们的签名，而是显示错误消息。不酷！幸运的是，消息还告诉我们需要更改的内容：` gpg.ssh.allowedSignersFile` 就是我们需要改变的选项。

Git 报错因为 SSH 没有建立一个由其他人签署密钥的信任网络。故，我们需要告诉 git 我们信任哪些密钥。一个单独的文件包含所有信任的 SSH 签名。因为我们是有条理的人，让我们把这个文件放在 `~/.config/git/allowed_signers` 下（如果路径不存在则创建它们）。

`allowed_signers` 的内容如下：

```
email ssh-ed25519 ssh_public_key comment
```

目光敏锐的人会认出它与 ssh-keygen(1) 使用的格式相同。我们至少需要信任自己的 SSH 密钥，所以将其放在这里。重复此步骤，适用于你圈子中的所有其他为仓库做贡献并签署其提交的人。为了教会 git 如何处理这个文件，添加另一个配置选项（正是之前错误消息抱怨的那一个）。

```
git config --global gpg.ssh.allowedSignersFile “~/.config/git/allowed_signers”
```

重试命令 `git show --show-signature`（并为其创建一个别名），看看错误消息是否被 git 签名替换。

### 修复小问题

更新本地副本建议运行 `git pull --ff-only`，这是默认行为。如果经常忘记添加参数，则可以将其设置为默认拉取行为，如下所示：

```
git config --global pull.ff only
```

当然，你也可以为其创建别名。这是一项“一次性”的设置。

在查看差异时，我一直在想为什么 git 使用 a/ 和 b/ 来区分文件。我不需要这些，文件名已经可以代表它们自己。我发现可以通过以下选项禁用此行为：

```
git config --global diff.noprefix true
```

说到差异，我想至少看到我更改周围的 5 行内容。 这是一个个人偏好，但任何人都可以使用此选项设置自己喜欢的方式：

```
git config --global diff.context 5
```

在像 FreeBSD 这样的国际项目上工作教会我日期可以有多种书写方式。git 使用的默认显示方式是 `Fri Mar 01 12:34:56 2024`。 这并没有什么，但我习惯于以下方式：`2024-03-01 12:34:56`。 这个选项将日期设置为我喜欢的样子：

```
git config --global log.date iso
```

我发现，另外一件奇怪的事情是 git 在运行 `git branch` 时列出分支的顺序。 我希望最近提交的分支显示在顶部，而不是其他随机的顺序。 要更改这个设置，我的 `.gitconfig` 包含这个内容：

```
git config --global branch.sort -committerdate
```

现在我可以准确地看到哪个分支接收了最近的更改。是时候合并了！

## 结论

随着时间的推移，我的配置可能会不断增加，因为我大概发现了其他有用的选项。 Git 配置十分灵活。默认设置对大多数人来说都很好，而且也很容易修改。这篇文章应该可以帮助你开始编写自己的配置，可在使用 git 时减少某些磨合问题。

 参考资料：

[https://blog.gitbutler.com/git-tips-and-tricks/](https://blog.gitbutler.com/git-tips-and-tricks/)

[https://jvns.ca/blog/2024/02/16/popular-git-config-options/](https://jvns.ca/blog/2024/02/16/popular-git-config-options/)

[https://blog.dbrgn.ch/2021/11/16/git-ssh-signatures/](https://blog.dbrgn.ch/2021/11/16/git-ssh-signatures/)

---

**BENEDICT REUSCHLING** 是 FreeBSD 项目的文档提交者和文档工程团队成员。他曾任两届 FreeBSD 核心团队成员。他在德国达姆施塔特应用技术大学管理着一个大数据集群。他还为本科生教授“Unix 开发者”这门课。Benedict 是 bsdnow.tv 每周播客的主持人。
