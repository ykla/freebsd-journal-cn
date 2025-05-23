# FreeBSD 13.0 中有新加载器吗？

- 原文链接：[Is There a New Loader in FreeBSD 13.0?](https://freebsdfoundation.org/wp-content/uploads/2021/05/Is-There-a-New-Loader-in-FreeBSD-13.0.pdf)
- 作者：**TOOMAS SOOME**


简短的回答是“不是，仍然是那个老旧的加载程序。”但我们正在努力使它更友好并支持更多的功能。

我最初开始研究的启动加载程序并不是 FreeBSD 的，而是 illumos 的。那时，illumos 使用的是旧版 grub 0.96，只支持 BIOS 启动。UEFI 系统在那时已经出现，而 illumos 迫切需要支持 UEFI 系统。我的最初工作是研究更新版本的 grub。它功能丰富、广泛使用，但管理起来很复杂，而且当你需要支持如 zfs 之类的文件系统特定功能，或是添加操作系统特定功能时，它的许可协议并不友好。这促使我转向 FreeBSD 启动加载程序，并开始为其开发做出贡献。

当我开始将 FreeBSD 启动加载程序移植到 illumos 时，illumos 只支持串口控制台和 VGA 文本模式控制台。为了支持 UEFI，我必须为 illumos 内核实现对基于 UEFI 帧缓存的控制台的支持。实现后，下一步就是为加载程序添加相同的功能。当可以在 UEFI 帧缓存上绘制控制台时，重新使用相同的代码在 Vesa BIOS 扩展（VBE）线性帧缓存上绘制，就成为了开发中的另一个逻辑步骤。完成后，我们就得到了像这样的公开发布：<https://omnios.org/setup/fb>

![](https://github.com/user-attachments/assets/a0acca66-6058-4471-9a53-c279eefdee7d)


图 1 启动加载程序与 ASCII 图

![](https://github.com/user-attachments/assets/2521f172-692c-4a26-a5ac-098d791ba829)

图 2 启动加载程序与图像。

## 回到 FreeBSD 启动加载程序

说了这么多 illumos 的内容，让我们回到 FreeBSD，看看我们在这里做了什么。请注意，我所做的大部分工作涉及从 FreeBSD 到 illumos 或反向的迁移。

## OpenZFS

由于 FreeBSD 13.0 现在使用 OpenZFS，我们支持大多数用于启动的 OpenZFS 特性。不过，加密数据集和 draid 仍然在待办事项清单上。

## 控制台

最显著的变化无疑是启动加载程序的图形控制台。虽然当前的实现还不完美，并且可以改进，但我希望大多数用户会喜欢更新后的外观。

加载程序的控制台终端模拟器是从内核树中提取的。在这里，我们有两个可以设置的第一个调优参数：

```sh
teken.fg_color
teken.bg_color
```

接受的值是 ANSI 颜色名称或数字值 0 – 7。

UEFI 启动加载程序默认使用帧缓冲控制台，除非配置了串行控制台。

BIOS 启动加载程序当前默认使用文本模式。对于 BIOS 启动加载程序，可以通过设置以下参数来控制控制台：

```sh
screen.textmode="0"
```

这将导致加载程序设置为使用 VBE 帧缓冲模式，并在有 EDID 信息时使用显示首选分辨率。默认的回退分辨率为 `800x600`。

UEFI 和 BIOS 启动加载程序都允许通过以下调优参数设置屏幕分辨率：

```c
efi_max_resolution
vbe_max_resolution
```

设置 `vbe_max_resolution` 也会导致加载程序切换到使用 VBE 帧缓冲模式。

帧缓冲模式也可以通过平台特定的命令来设置、查询和列出支持的模式。

命令 `gop` 能让用户在 UEFI 加载程序中获取、设置和列出分辨率。命令 `gop off` 会将加载程序从绘制控制台切换到 UEFI 内建的终端输出方法——简单文本输出协议。

命令 `vbe` 能让用户在 BIOS 加载程序中获取、设置和列出分辨率。命令 `vbe on` 会将加载程序切换到使用 VBE 帧缓冲，而命令 `vbe off` 会将加载程序切换到使用 VGA 文本模式。

如果用户已将 BIOS 加载程序切换为使用 VBE 帧缓冲，但正在启动一个不提供 VT vbefb 驱动程序的旧内核，那么加载程序会在启动加载的内核之前将控制台切换为 VGA 文本模式。

## 字体

目前，我们提供了 Terminus 家族的控制台字体，安装在目录 `/boot/fonts` 中。加载程序内置了一个 8x16 字体，但为了节省空间，内置字体仅提供 ASCII 集。

在加载程序启动并初始化之后，它获得对磁盘的访问权限并确定了启动设备和启动文件系统，加载程序将搜索 `/boot/fonts` 目录和文件 `INDEX.fonts` 。如果存在，加载程序将获取可用字体的列表，并构建一个内部的、索引化的可用字体列表。使用 `INDEX.fonts` 文件是因为我们需要支持 tftp 文件传输，但 tftp 协议不支持读取目录列表。

若我们有了可用字体的列表，我们将根据控制台显示的分辨率加载首选字体。如果默认选择对用户不起作用，可以通过两种方法来更改默认设置：

首先，环境变量 `screen.font` 出现，并允许更改使用的字体。尝试使用无效值或取消该值将导致当前可用字体的列表在控制台上打印。

其次，命令 `loadfont` 允许用户加载自定义字体文件，例如 `/boot/fonts/gallant.fnt`。字体需要使用 vtfontcvt(8) 工具进行准备。

加载程序中的字体索引假定字体大小唯一。当已经注册了一个 8x16 字体文件时，尝试加载提供相同字体大小的不同字体文件将导致之前加载的文件被新文件替换。

加载程序当前使用的字体将传递给加载的内核。通过这种方式，我们保留了外观和感觉，并实现了从启动加载程序到正在运行的操作系统的一致过渡。

![](https://github.com/user-attachments/assets/e24b69fe-0b6b-4b8e-a6a2-7e96a6e4aff3)


图 3 FreeBSD 13.0 启动加载程序

## 图像

为了构建更美观的屏幕，启动加载程序支持显示 PNG 格式的图像文件。目前，我们要求图像为带有 alpha 通道的 TrueColor，并且支持图像缩放。可以在 `drawer.lua` 和 `gfx-orb.lua` 文件中找到如何在启动加载程序菜单中使用图像作为徽标或品牌组件的示例。

## 缺点

某些系统在使用帧缓冲控制台时会遇到控制台变慢的情况。使用 VBE 帧缓冲时，一种可能的解决方法是使用较小的颜色深度——默认是 32 位颜色。对于 VBE 和 UEFI，可能可以使用较小的分辨率，并在 KMS 驱动程序运行后配置更好的分辨率。当然，在某些情况下，唯一合理的选项可能是使用文本控制台。

## 总结

所以，再次回答，FreeBSD 并没有新的启动加载程序，但我们正在努力确保它能完成应该做的事情——即支持加载和启动 FreeBSD 操作系统，提供人们可能需要的功能，并且看起来相当不错。

---

**TOOMAS SOOME**  出生于爱沙尼亚。Toomas 自 1993 年起一直从事 UNIX 管理工作，主要使用 Solaris。他还是一名基础架构架构师、illumos 开发者和 FreeBSD 源代码提交者。Toomas 不怕启动加载程序，并且能读写 Forth。
