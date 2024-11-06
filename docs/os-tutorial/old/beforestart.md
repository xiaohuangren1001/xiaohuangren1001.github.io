### -1.写在开始之前
虽然网上此类教程云集，虽然此类书籍很多，但是！

这些书籍有很多地方讲得**不够细致**（主要是代码有缺漏），有些对代码的更改甚至在书中**了无痕迹**。

而这才是我开启这篇教程的原因。

这篇教程之中，只要照着所有的操作做了一遍，应当能够写出完整的**操作系统**！

本教程默认各位读者是会汇编的，或者说，至少应该能看懂汇编。

与其说这篇文章是个教程，倒不如说是一个学习笔记和我自身编程经验的记录。

### 0.开发环境配置
如果您使用的是 `Linux`，我们只需要输入下面一行命令即可完成开发环境的配置：

`sudo apt-get install nasm build-essential qemu-system-x86`

如果您使用的 `Linux` 中不含有 `apt` 系列包管理器，请使用您系统中的包管理器。

如果您使用的是 `Linux`，但您的系统内没有包管理器，那么您可以去 nasm 官网、 gcc 官网和 qemu 官网下载源码，然后 configure -> make -> sudo make install。

如果您使用的是 `Windows`，请去以下地方获取所需要的工具：

[nasm](http://nasm.us)

[交叉编译的gcc](https://github.com/lordmilko/i686-elf-tools/releases/tag/7.1.0)（请下载i686-elf-tools-windows.zip）

[qemu](https://qemu.weilnetz.de/w32/2022/)

[make](https://sourceforge.net/projects/gnuwin32/files/make/)

[dd](http://www.chrysocome.net/dd)

[bochs](http://bochs.sourceforge.io)（其实我们只需要其中的 bximage.exe ）（如果是 32 位电脑请下载 2.5 以前的版本）

[edimg](https://share.weiyun.com/Q1yUyjRp)（这个就是《30天自制操作系统》的写盘工具）

如果您使用的是 `macOS`，那么请注意，系统内置的 `gcc` 会把文件编译成 `Mach-O` 格式，请通过 `Homebrew` 下载交叉编译器：

`brew install i386-elf-binutils`

`brew install i386-elf-gcc`

然后我们还需要去往 [nasm 官网](http://nasm.us)获取可执行文件，并执行：

`brew install qemu`

以获取 qemu。

在安装完之后，如果您使用的是 `Windows`，请**确保它们的路径位于 PATH 下！**

除此之外便没什么重点了，不过，对于下文给出的工具名称**默认以 `Windows` 为准**，若您使用 `Linux`，请去掉工具前缀，若您使用 `macOS`，请将工具前缀中的 `i686` 改为 `i386`！！！

对了，如果您使用的是 `Linux` 或 `macOS`，请确保您在 `dd` 命令的后面**加入 `conv=notrunc`** ！！！

那么，开发环境配置正式结束，征程开始！