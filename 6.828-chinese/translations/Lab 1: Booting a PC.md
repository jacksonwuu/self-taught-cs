# Lab 1: Booting a PC

## 介绍

这个实验室分成三个部分。第一部分主要介绍如何熟悉 x86 汇编语言、QEMU x86 仿真器和 PC 的开机引导程序。第二部分检查 6.828 内核的引导加载程序，它位于实验 boot 目录中。最后，第三部分研究了 6.828 内核本身的初始模板，名为 JOS，它位于 kernel 目录中。

## Part 1: PC Bootstrap

第一个练习的目的是向您介绍 x86 汇编语言和 PC 引导过程，并让您开始 QEMU 和 QEMU/GDB 调试。您不需要为实验室的这一部分编写任何代码，但您应该根据自己的理解仔细阅读它，并准备好回答下面提出的问题。

### 从 x86 汇编开始

如果你没有太熟悉 x86 汇编语言，你会在这个课程里很快对它熟悉！[PC Assembly Language Book](https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf)是个绝佳的开始点。这个书包含了新旧材料。

警告：非常不幸的是书里的例子都是用 NASM 汇编器，但是我们会用 GNU 汇编器。NASM 使用所谓的 Intel 语法，但是 GNU 使用 AT&T 语法。然而语法上都是等价的，用不同语法写的汇编从表面上看会非常不一样。幸运的是这两种语法非常简单，这本书有讲到：[Brennan's Guide to Inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html)。

练习 1：熟悉[the 6.828 reference page](https://pdos.csail.mit.edu/6.828/2018/reference.html)里的汇编语言材料。你现在不需要都读，但是在读写 x86 汇编程序集时，你几乎肯定会想要参考其中一些资料。

我们确实推荐阅读[Brennan's Guide to Inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html)的"The Syntax"部分。它很好地描述了 AT&T 汇编语法，我们会在 JOS 里用到的。

当然，x86 汇编语言编程的权威参考是 Intel 的指令集架构参考，你可以在[the 6.828 reference page](https://pdos.csail.mit.edu/6.828/2018/reference.html)中找到两种风格：旧的[80386 Programmer's Reference Manual](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm)的 HTML 版本，比最近的手册更短，更容易浏览，但描述了我们将在 6.828 中使用的所有 x86 处理器特性；和完整的、最新的和最好的[IA-32 Intel Architecture Software Developer's Manuals](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)从英特尔，涵盖所有最新的处理器的特点，我们不需要在课堂上使用，但你可能会对感兴趣学习。同样的(通常更友好的)一套手册[available from AMD](http://developer.amd.com/resources/developer-guides-manuals/)。将 Intel/AMD 架构手册保存起来，以备以后使用，或者当你想要查找特定处理器特性或指令的明确解释时，将其作为参考。

### 模拟 x86

我们不是在真实的、实际的个人计算机(PC)上开发操作系统，而是使用一个程序来忠实地模拟一个完整的 PC：您为模拟器编写的代码也将在真实的 PC 上启动。使用模拟器简化了调试；例如，您可以在模拟的 x86 中设置断点，这在 x86 的硅版中很难做到。

在 6.828 中，我们将使用[QEMU 模拟器](http://www.qemu.org/)，这是一个现代且相对快速的模拟器。虽然 QEMU 的内置监视器只提供有限的调试支持，但 QEMU 可以充当[GNU 调试器(GDB)](http://www.gnu.org/software/gdb/)的远程调试目标，我们将在本实验中使用它来逐步完成早期引导过程。

首先，将 Lab 1 文件解压缩到 Athena 上您自己的目录中，如“软件安装”中所述，然后在 Lab 目录中输入 make(或 BSD 系统上的 gmake)，以构建最小的 6.828 引导加载程序和内核。(把我们在这里运行的代码称为“内核”有点慷慨，但我们将在整个学期充实它。)

```
athena% cd lab
athena% make
+ as kern/entry.S
+ cc kern/entrypgdir.c
+ cc kern/init.c
+ cc kern/console.c
+ cc kern/monitor.c
+ cc kern/printf.c
+ cc kern/kdebug.c
+ cc lib/printfmt.c
+ cc lib/readline.c
+ cc lib/string.c
+ ld obj/kern/kernel
+ as boot/boot.S
+ cc -Os boot/main.c
+ ld boot/boot
boot block is 380 bytes (max 510)
+ mk obj/kern/kernel.img
```

(如果你得到像“undefined reference to '\_\_udivdi3'”这样的错误，你可能没有 32 位的 gcc multilib。如果你正在 Debian 或 Ubuntu 上运行，尝试安装 gcc-multilib 包。)

现在您已经准备好运行 QEMU，并提供文件 obj/kern/kernel.img，创建在上面，作为模拟 PC 的“虚拟硬盘”的内容。这个硬盘映像包含我们的引导加载程序(obj/boot/boot)和我们的内核(obj/kernel)。

```
athena% make qemu
or
athena% make qemu-nox
```

这将执行 QEMU，其中包含设置硬盘和直接串口输出到终端所需的选项。一些文本应该出现在 QEMU 窗口：

```
Booting from Hard Disk...
6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K>
```

‘Booting from Hard Disk...’之后的一切都是我们的 JOS 内核打印的；K> 是由我们在内核中包含的小监视器或交互式控制程序打印的提示符。如果执行 make qemu，内核打印的这些行将出现在运行 qemu 的常规 shell 窗口和 qemu 显示窗口中。这是因为测试和打分的目的，我们设立了 JOS 内核写它的控制台输出不仅在虚拟 VGA 显示(见 QEMU 窗口)，而且模拟电脑的虚拟串口，QEMU 反过来输出自己的标准输出。类似地，JOS 内核将从键盘和串口接受输入，因此您可以在 VGA 显示窗口或运行 QEMU 的终端中向它提供命令。或者，您可以通过运行 make qemu-nox 在没有虚拟 VGA 的情况下使用串行控制台。这可能是方便的，如果你是 SSH 进入一个 Athena 拨号。要退出 qemu，输入 Ctrl+a x。

内核监视器只有两个命令可以用，help 和 kerninfo。

```
K> help
help - display this list of commands
kerninfo - display information about the kernel
K> kerninfo
Special kernel symbols:
  entry  f010000c (virt)  0010000c (phys)
  etext  f0101a75 (virt)  00101a75 (phys)
  edata  f0112300 (virt)  00112300 (phys)
  end    f0112960 (virt)  00112960 (phys)
Kernel executable memory footprint: 75KB
K>
```

help 命令很明显，我们将很快讨论 kerninfo 命令输出的含义。虽然很简单，但需要注意的是，这个内核监控器“直接”运行在模拟 PC 的“原始(虚拟)硬件”上。这意味着你应该能够复制 obj/kern/kernel 的内容。将硬盘插入真正的 PC，打开它，并在 PC 的真实屏幕上看到与您在上面的 QEMU 窗口中所做的完全相同的事情。(但是，我们不建议您在硬盘上有有用信息的真正机器上这样做，因为复制 kernel.img 添加到其硬盘的开始部分将回收主引导记录和第一个分区的开始部分，从而有效地导致硬盘上之前的所有内容丢失！)

### PC 的物理地址空间

现在我们将更详细地介绍一下 PC 是如何启动的。PC 机的物理地址空间是硬连接的，有以下总体布局：

```

+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```

第一代 PC 基于 16 位 Intel 8088 处理器，只能寻址 1MB 的物理内存。因此，早期 PC 的物理地址空间将从 0x00000000 开始，到 0x000FFFFF 结束，而不是 0xFFFFFFFF。标记为“低内存”的 640KB 区域是早期 PC 能够使用的唯一随机访问内存(RAM)；事实上，最早的电脑只能配置 16KB、32KB 或 64KB 的内存!

从 0x000A0000 到 0x000FFFFF 的 384KB 区域由硬件预留，用于特殊用途，如视频显示缓冲区和非易失性内存中的固件。这个保留区域中最重要的部分是 Basic Input/Output System (BIOS)，它占用了从 0x000F0000 到 0x000FFFFF 的 64KB 区域。在早期的 pc 中，BIOS 保存在真正的只读存储器(ROM)中，但现在的 PC 将 BIOS 存储在可更新的闪存中。BIOS 主要负责对系统进行基本的初始化操作，如激活显卡、检查内存总量等。执行这个初始化之后，BIOS 从一些适当的位置(如软盘、硬盘、CD-ROM 或网络)加载操作系统，并将机器的控制权传递给操作系统。

当 Intel 最终用 80286 和 80386 处理器“突破了 1MB 的障碍”，它们分别支持 16MB 和 4GB 的物理地址空间，PC 架构师仍然保留了原始的 1MB 物理地址空间布局，以确保与现有软件的向后兼容性。因此，现代 PC 在物理内存中有一个从 0x000A0000 到 0x00100000 的“洞”，将 RAM 划分为“低内存”或“常规内存”(前 640KB)和“扩展内存”(其他一切)。此外，PC 的 32 位物理地址空间(尤其是物理 RAM)顶部的一些空间现在通常由 BIOS 保留，供 32 位 PCI 设备使用。

最近的 x86 处理器可以支持超过 4GB 的物理 RAM，所以 RAM 可以扩展到 0xFFFFFFFF 以上。在这种情况下，BIOS 必须安排在系统 RAM 的 32 位可寻址区域的顶部留下第二个洞，为这些 32 位设备的映射留下空间。由于设计限制，JOS 将只使用 PC 的物理内存的前 256MB，所以现在我们假设所有 PC 都“只有”一个 32 位的物理地址空间。但是处理复杂的物理地址空间和硬件组织的其他方面是操作系统开发的一个重要的实际挑战。

### The ROM BIOS

在本部分中，您将使用 QEMU 的调试工具来研究 IA-32 兼容的计算机是如何引导的。

打开两个终端窗口并将两个 shell cd 到您的实验室目录中。其中，输入 make qemu-gdb(或 make qemu-nox-gdb)。这将启动 QEMU，但是 QEMU 在处理器执行第一个指令之前停止，并等待来自 GDB 的调试连接。在第二个终端中，在运行 make 的同一个目录运行 make gdb。你应该看到这样的东西，

```
athena% make gdb
GNU gdb (GDB) 6.8-debian
Copyright (C) 2008 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i486-linux-gnu".
+ target remote localhost:26000
The target architecture is assumed to be i8086
[f000:fff0] 0xffff0:	ljmp   $0xf000,$0xe05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
(gdb)
```

我们提供了一个.gdbinit 文件，该文件设置 GDB 来调试早期引导期间使用的 16 位代码，并将其附加到侦听的 QEMU 上。(如果它不能工作，你可能必须在你的主目录的.gdbinit 中添加一个 add-auto-load-safe-path，来让 GDB 处理我们提供的.gdbinit。GDB 会告诉您是否必须这样做。)

以下行:

```
[f000:fff0] 0xffff0:	ljmp   $0xf000,$0xe05b
```

是 GDB 反汇编第一行执行的指令。从输出端你可以得到一些结论：

-   IBM PC 在物理地址 0x000ffff0 处开始执行，也就是位 BIOS ROM 预留的 64KB 的最顶端。
-   PC 开始执行时 CS = 0xf000，IP = 0xfff0。
-   第一个执行的指令是 jmp 指令，它会跳转到 CS = 0xf000 和 IP = 0xe05b 处。

QEMU 为什么会这样开始呢？英特尔就是这样设计 8088 处理器的，IBM 在他们最初的个人电脑中使用了这种处理器。因为在 PC BIOS 被“硬接线”到物理地址范围 0x000f0000-0x000fffff 处，这种设计可以确保机器的 BIOS 总是在开启或系统重启时第一个获得控制权——这是至关重要的，因为在 RAM 里没有软件可以执行。QEMU 模拟器自己自带的 BIOS，它将 BIOS 放置在处理器模拟物理地址空间的这个位置。处理器复位时，(模拟)处理器进入实模式，并将 CS 设置为 0xf000, IP 设置为 0xfff0，因此执行从那个(CS:IP)段地址开始。分段地址 0xf000:fff0 如何变成物理地址？

为了回答这个问题，我们需要了解一些关于实际模式寻址的知识。在实模式下(PC 启动的模式)，地址转换按如下公式进行：物理地址= 16\*段+偏移量。因此，当 PC 将 CS 设置为 0xf000, IP 设置为 0xfff0 时，所引用的物理地址为:

```
   16 * 0xf000 + 0xfff0   # in hex multiplication by 16 is
   = 0xf0000 + 0xfff0     # easy--just append a 0.
   = 0xffff0
```

0xffff0 是 BIOS 结尾（0x100000）之前的 16 字节。因此不要惊讶做的第一件 BIOS 做的事情是 jmp 跳转回 BIOS 更早之前的位置；毕竟，16 个字节能完成多少任务?

练习 2。使用 GDB 的 si（Step Instruction）命令来追踪 ROM BIOS 更多的指令，然后试着猜猜它在做什么。你可能想要看一下[Phil Storrs I/O Ports Description](http://web.archive.org/web/20040404164813/members.iweb.net.au/~pstorr/pcbook/book2/book2.htm)，以及其他材料[6.828 reference materials page](https://pdos.csail.mit.edu/6.828/2018/reference.html)。不需要弄清楚所有的细节——只要先弄懂 BIOS 的大概运作就行了。

当 BIOS 运行时，它设置一个中断描述符表并初始化各种设备，例如 VGA 显示器。这就是你在 QEMU 窗口看到的 "Starting SeaBIOS" 消息来自的地方。

在 BIOS 初始化它所知道的 PCI bus 和所有重要的设备之后，它搜索一个可启动设备，比如软盘、硬盘或者 CD-ROM。最终，当它找到一个可启动的磁盘时，BIOS 读取磁盘里的 boot loader，然后把控制权转交给它。

## Part 2: The Boot Loader

软盘和硬盘被分成很多 512 比特的区域，这些区域被叫做扇区。一个扇区是磁盘最小的传输粒度：每次读写操作必须是一个或多个扇区的大小，并在扇区边界对齐。如果磁盘是可启动的，第一个扇区被叫做启动扇区，因为这是 boot loader 代码所处的地方。当 BIOS 找到一个可启动的软盘或硬盘时，它加载 512 比特的 boot 扇区到物理地址 0x7c00-0x7dff 处，然后使用一个 jmp 指令设置 CS:IP 为 0000:7c00，传递控制权给 boot loader。就像 BIOS 加载地址一样，这些地址是相当随意的，但对于 PC 来说是固定和标准化的。(译者注：因为历史原因，BIOS 的加载地址被约定为 0x7c00，不用想为什么是这个地址，没有什么直接的原因。)

从 CD-ROM 启动的能力是在 PC 进化过程中很久以后才出现的，因此，PC 架构师利用这个机会稍微重新思考了一下引导过程。因此，现代 BIOS 从 CD-ROM 启动的方式有点复杂(也更强大)。CD-ROM 使用一个 2048 比特的扇区大小而不是 512 比特，BIOS 可以加载一个更大的来自磁盘的 boot 镜像进入内存（不仅只是一个扇区），然后在转交控制权。更多信息可以看["El Torito" Bootable CD-ROM Format Specification](https://pdos.csail.mit.edu/6.828/2018/readings/boot-cdrom.pdf)。

然而，对于 6.828 来说，我们会使用传统的硬盘 boot 机制，也就意味着我们的 boot loader 必须填入一个 512 比特的磁盘区域。这个 boot loader 包含了一个汇编语言的源文件，boot/boot.S 和一个 C 源文件，boot/main.c，细心地把这些源文件通读一遍，确保你理解了运作原理。boot loader 必须执行两个主要的功能：

1. 首先，这个 boot loader 把处理器从实模式切换到 32 位保护模式，因为在保护模式下，软件才可以访问处理器物理地址空间的所有 1MB 以上的内存。[PC Assembly Language](https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf)的 1.2.7 和 1.2.8 小结对保护模式作了简单的描述，详细的细节可以参考 Intel 架构手册。此刻你只需要理解分段地址在保护模式中转化为物理地址的方式不一样，转换的 offset 是 32 位而不是 16 位的。
2. 第二，这个 boot loader 从磁盘上读取内核，通过使用 x86 特殊的 I/O 指令来直接访问 IDE 磁盘寄存器来做到这一点。如果你想要更好地理解特别的 I/O 指令的含义，查看[the 6.828 reference page](https://pdos.csail.mit.edu/6.828/2018/reference.html)的"IDE hard drive controller"小结。你在这门课程里不必学习太多特殊设备的编程：写设备驱动程序在操作系统开发里是非常重要的部分，但是从概念和架构的角度来看，这也是最无聊的部分。

在你理解了 boot loader 的源代码之后，查看 obj/boot/boot.asm 文件。这个文件是我们的 GUNMakefile 创建 boot loader 之后对其进行反汇编。这个反汇编文件可以很容易地看到这个 boot loader 的代码会驻留在物理内存的哪些确切位置，然后就很容易在 GDB 里追踪 boot loader 发生了什么事情。同样的，obj/kern/kernel.asm 包含了一个对 JOS 内核的反汇编，对于 debug 来说很有用。

你可以在 GDB 里用 b 命令来设置地址断点。例如，b \*0x7c00 在地址 0x7c00 处设置了一个断点。一旦到某个断点，你可以进用 c 和 si 命令继续执行：c 让 QEMU 继续执行直到下一个断点（或者直到你按下 Ctrl-C），si N 可以一次性步进 N 条指令。

为了查看内存里的指令（除了即将执行的下一个，GDB 会自动打印），你可以使用 x/i 指令。这个命令的语法是 x/Ni ADDR，N 是连续反汇编指令的数目，ADDR 是开始反汇编的内存地址。

练习 3。看一眼[lab tools guide](https://pdos.csail.mit.edu/6.828/2018/labguide.html)，尤其是 GDB 命令部分。哪怕你熟悉 GDB，这包括一些对于操作系统工作很有用的深奥的 GDB 命令。

在地址 0x7c00 处设置一个断点，也就是 boot 扇区会被加载到的地方。继续执行直到那个断点。追踪整个 boot/boot.S 的代码，用反汇编文件 obj/boot/boot.asm 里的源代码来追踪你所处的代码位置。再用 GDB 的 x/i 指令来反汇编 boot laoder，然后对比一下原始的 boot loader 源代码和反汇编文件里的代码、以及 GDB 产生的反汇编代码。

追踪 boot/main.c 的 bootmain()，然后再追踪 readsect()。确定与 readsect()中的每个语句对应的确切的程序集指令。追踪 readsect()剩余的代码，然后回到 bootmain()，并标识从磁盘读取内核剩余扇区的 for 循环的开始和结束。找到什么代码会在 loop 循环结束的时候运行，在那里设置断点，然后继续到那个断点。然后步进查看 boot loader 的所有剩余代码。

你要能回答以下的问题：

-   在什么时候处理器开始执行 32 位代码？是什么导致了 16 位到 32 位到模式的转变？
-   boot loader 最后执行的指令是什么，kernel 刚加载时执行的第一个指令是什么？
-   kernel 的第一个指令在哪儿？
-   boot loader 如何决定要读取多少扇区才能加载整个内核？它是在哪里找到这个信息的？

### Loading the Kernel（加载内核）

我们现在将进一步详细研究 boot loader 的 C 语言部分，boot/main.c。在开始之前，最好可以先好好回顾一下 C 编程的基础。

练习 4。阅读 C 语言里的指针部分。C 语言最好的材料就是 The C Programming Language by Brian Kernighan and Dennis Ritchie（也被叫做 K&R）。我们推荐学生买这本书(here is an [Amazon Link](http://www.amazon.com/C-Programming-Language-2nd/dp/0131103628/sr=8-1/qid=1157812738/ref=pd_bbs_1/104-1502762-1803102?ie=UTF8&s=books))，或者[MIT's 7 copies](http://library.mit.edu/F/AI9Y4SJ2L5ELEE2TAQUAAR44XV5RTTQHE47P9MKP5GQDLR9A8X-10422?func=item-global&doc_library=MIT01&doc_number=000355242&year=&volume=&sub_library=)。

阅读 K&R 的 5.1（指针与地址）至 5.5（字符指针和函数）部分。然后下载[pointers.c](https://pdos.csail.mit.edu/6.828/2018/labs/lab1/pointers.c)，运行它，确保你理解了所有打印的值都来自哪里。特别是，要确保您理解打印第 1 行和第 6 行中的指针地址来自哪里，打印第 2 行到第 4 行中的所有值是如何到达那里的，以及为什么在第 5 行中打印的值似乎是损坏的。

有很多关于 C 语言指针的材料(e.g., [A tutorial by Ted Jensen](https://pdos.csail.mit.edu/6.828/2018/readings/pointers.pdf)，虽然并没有那么强烈推荐。

警告：除非你已经完全精通 C 语言，否则不要跳过甚至略读这个阅读练习。如果你不能真正理解 C 语言中的指针，你将在随后的实验中经历无数的痛苦和痛苦，然后最终艰难地理解它们。相信我们；你不会想知道什么是"困难的道路"的。

为了理解 boot/main.c，你需要知道 ELF 二进制文件是什么。当你编译和链接一个 C 程序比如说 JOS 内核时，编译器会把每个 C 源文件（'.c'）转化为一个对象（'.o'）文件，它包含了硬件可以接受的汇编指令的编码。链接器然后组合所有这些编译过的对象文件到一整个二进制镜像比如说 obj/kern/kernel 里，在这种情况下是在 ELF 格式里的二进制，ELF 表示"Executable and Linkable Format"（可执行和可链接的格式）。

关于这种格式的完整信息可以在[ELF specification](https://pdos.csail.mit.edu/6.828/2018/readings/elf.pdf)这里找到，但在这门课中，你不需要深入钻研这种格式的细节。 尽管作为一个整体，这种格式非常强大和复杂，但大多数复杂的部分都用于支持共享库的动态加载，但我们在这个课程中不会做这个。[维基百科](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format)里有一个简短的描述。

对于 6.828，您可以把 ELF 可执行文件看作是一个带有加载信息的头文件，后面跟着几个程序段，每个程序段都是一个连续的代码块或数据，打算加载到指定地址的内存中。boot loader 不修改代码或数据；它只是把 ELF 文件加载到内存中并开始执行。

ELF 二进制由一段固定长度的 ELF 头开始，后面是一个可变长度的程序头，列出要加载的每个程序段。这鞋 ELF 头的 C 语言定义在 inc/elf.h 里。我们关心的程序段是：

-   .text：程序的可执行指令们。
-   .rodata：只读数据，比如说 C 编译器产生的 ASCII 字符串常量。（我们不会费心设置硬件来禁止写入。）
-   .data：data 段放置程序段初始化数据，比如声明时初始化的全局变量，比如说 x=5;。

当链接器计算程序的内存布局时，它会为未初始化的全局变量预留空间，比如说 int x;，在一个紧跟着.data 段的叫做.bss 的段。C 要求未初始化的全局变量以零值开始。因此，ELF 二进制的.bss 段里不需要存储内容；相反，链接器只记录.bss 节的地址和大小。加载器或程序本身必须将.bss 节归零。

检查内核可执行文件的名字、大小和链接地址列表，可以输入：

```
athena% objdump -h obj/kern/kernel
```

（如果你自己编译你自己的工具链，你可能要用 i386-jos-elf-objdump）

您将看到比上面列出的更多的部分，但其他部分对我们的目的并不重要。其他的大多数用来保存调试信息，这些信息通常包含在程序的可执行文件中，但不会被程序加载器加载到内存中。

Take particular note of the "VMA" (or link address) and the "LMA" (or load address) of the .text section. The load address of a section is the memory address at which that section should be loaded into memory.

The link address of a section is the memory address from which the section expects to execute. The linker encodes the link address in the binary in various ways, such as when the code needs the address of a global variable, with the result that a binary usually won't work if it is executing from an address that it is not linked for. (It is possible to generate position-independent code that does not contain any such absolute addresses. This is used extensively by modern shared libraries, but it has performance and complexity costs, so we won't be using it in 6.828.)

Typically, the link and load addresses are the same. For example, look at the .text section of the boot loader:

```
athena% objdump -h obj/boot/boot.out
```

The boot loader uses the ELF program headers to decide how to load the sections. The program headers specify which parts of the ELF object to load into memory and the destination address each should occupy. You can inspect the program headers by typing:

```
athena% objdump -x obj/kern/kernel
```

The program headers are then listed under "Program Headers" in the output of objdump. The areas of the ELF object that need to be loaded into memory are those that are marked as "LOAD". Other information for each program header is given, such as the virtual address ("vaddr"), the physical address ("paddr"), and the size of the loaded area ("memsz" and "filesz").

Back in boot/main.c, the ph->p_pa field of each program header contains the segment's destination physical address (in this case, it really is a physical address, though the ELF specification is vague on the actual meaning of this field).

The BIOS loads the boot sector into memory starting at address 0x7c00, so this is the boot sector's load address. This is also where the boot sector executes from, so this is also its link address. We set the link address by passing -Ttext 0x7C00 to the linker in boot/Makefrag, so the linker will produce the correct memory addresses in the generated code.

Exercise 5. Trace through the first few instructions of the boot loader again and identify the first instruction that would "break" or otherwise do the wrong thing if you were to get the boot loader's link address wrong. Then change the link address in boot/Makefrag to something wrong, run make clean, recompile the lab with make, and trace into the boot loader again to see what happens. Don't forget to change the link address back and make clean again afterward!

Look back at the load and link addresses for the kernel. Unlike the boot loader, these two addresses aren't the same: the kernel is telling the boot loader to load it into memory at a low address (1 megabyte), but it expects to execute from a high address. We'll dig in to how we make this work in the next section.

Besides the section information, there is one more field in the ELF header that is important to us, named e_entry. This field holds the link address of the entry point in the program: the memory address in the program's text section at which the program should begin executing. You can see the entry point:

```
athena% objdump -f obj/kern/kernel
```

You should now be able to understand the minimal ELF loader in boot/main.c. It reads each section of the kernel from disk into memory at the section's load address and then jumps to the kernel's entry point.

Exercise 6. We can examine memory using GDB's x command. The [GDB manual](https://sourceware.org/gdb/current/onlinedocs/gdb/Memory.html) has full details, but for now, it is enough to know that the command x/Nx ADDR prints N words of memory at ADDR. (Note that both 'x's in the command are lowercase.) Warning: The size of a word is not a universal standard. In GNU assembly, a word is two bytes (the 'w' in xorw, which stands for word, means 2 bytes).

Reset the machine (exit QEMU/GDB and start them again). Examine the 8 words of memory at 0x00100000 at the point the BIOS enters the boot loader, and then again at the point the boot loader enters the kernel. Why are they different? What is there at the second breakpoint? (You do not really need to use QEMU to answer this question. Just think.)

## Part 3: The Kernel

We will now start to examine the minimal JOS kernel in a bit more detail. (And you will finally get to write some code!). Like the boot loader, the kernel begins with some assembly language code that sets things up so that C language code can execute properly.

### Using virtual memory to work around position dependence

When you inspected the boot loader's link and load addresses above, they matched perfectly, but there was a (rather large) disparity between the kernel's link address (as printed by objdump) and its load address. Go back and check both and make sure you can see what we're talking about. (Linking the kernel is more complicated than the boot loader, so the link and load addresses are at the top of kern/kernel.ld.)

Operating system kernels often like to be linked and run at very high virtual address, such as 0xf0100000, in order to leave the lower part of the processor's virtual address space for user programs to use. The reason for this arrangement will become clearer in the next lab.

Many machines don't have any physical memory at address 0xf0100000, so we can't count on being able to store the kernel there. Instead, we will use the processor's memory management hardware to map virtual address 0xf0100000 (the link address at which the kernel code expects to run) to physical address 0x00100000 (where the boot loader loaded the kernel into physical memory). This way, although the kernel's virtual address is high enough to leave plenty of address space for user processes, it will be loaded in physical memory at the 1MB point in the PC's RAM, just above the BIOS ROM. This approach requires that the PC have at least a few megabytes of physical memory (so that physical address 0x00100000 works), but this is likely to be true of any PC built after about 1990.

In fact, in the next lab, we will map the entire bottom 256MB of the PC's physical address space, from physical addresses 0x00000000 through 0x0fffffff, to virtual addresses 0xf0000000 through 0xffffffff respectively. You should now see why JOS can only use the first 256MB of physical memory.

For now, we'll just map the first 4MB of physical memory, which will be enough to get us up and running. We do this using the hand-written, statically-initialized page directory and page table in kern/entrypgdir.c. For now, you don't have to understand the details of how this works, just the effect that it accomplishes. Up until kern/entry.S sets the CR0_PG flag, memory references are treated as physical addresses (strictly speaking, they're linear addresses, but boot/boot.S set up an identity mapping from linear addresses to physical addresses and we're never going to change that). Once CR0_PG is set, memory references are virtual addresses that get translated by the virtual memory hardware to physical addresses. entry_pgdir translates virtual addresses in the range 0xf0000000 through 0xf0400000 to physical addresses 0x00000000 through 0x00400000, as well as virtual addresses 0x00000000 through 0x00400000 to physical addresses 0x00000000 through 0x00400000. Any virtual address that is not in one of these two ranges will cause a hardware exception which, since we haven't set up interrupt handling yet, will cause QEMU to dump the machine state and exit (or endlessly reboot if you aren't using the 6.828-patched version of QEMU).

Exercise 7. Use QEMU and GDB to trace into the JOS kernel and stop at the movl %eax, %cr0. Examine memory at 0x00100000 and at 0xf0100000. Now, single step over that instruction using the stepi GDB command. Again, examine memory at 0x00100000 and at 0xf0100000. Make sure you understand what just happened.

What is the first instruction after the new mapping is established that would fail to work properly if the mapping weren't in place? Comment out the movl %eax, %cr0 in kern/entry.S, trace into it, and see if you were right.

### Formatted Printing to the Console

Most people take functions like printf() for granted, sometimes even thinking of them as "primitives" of the C language. But in an OS kernel, we have to implement all I/O ourselves.

Read through kern/printf.c, lib/printfmt.c, and kern/console.c, and make sure you understand their relationship. It will become clear in later labs why printfmt.c is located in the separate lib directory.

Exercise 8. We have omitted a small fragment of code - the code necessary to print octal numbers using patterns of the form "%o". Find and fill in this code fragment.

Be able to answer the following questions:

1. Explain the interface between printf.c and console.c. Specifically, what function does console.c export? How is this function used by printf.c?
2. Explain the following from console.c:
    ```
    1      if (crt_pos >= CRT_SIZE) {
    2              int i;
    3              memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
    4              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
    5                      crt_buf[i] = 0x0700 | ' ';
    6              crt_pos -= CRT_COLS;
    7      }
    ```
3. For the following questions you might wish to consult the notes for Lecture 2. These notes cover GCC's calling convention on the x86.
   Trace the execution of the following code step-by-step:
    ```
    int x = 1, y = 3, z = 4;
    cprintf("x %d, y %x, z %d\n", x, y, z);
    ```
    - In the call to cprintf(), to what does fmt point? To what does ap point?
    - List (in order of execution) each call to cons_putc, va_arg, and vcprintf. For cons_putc, list its argument as well. For va_arg, list what ap points to before and after the call. For vcprintf list the values of its two arguments.
4. Run the following code.

    ```
    unsigned int i = 0x00646c72;
    cprintf("H%x Wo%s", 57616, &i);
    ```

    What is the output? Explain how this output is arrived at in the step-by-step manner of the previous exercise. [Here's an ASCII table](http://web.cs.mun.ca/~michael/c/ascii-table.html) that maps bytes to characters.

    The output depends on that fact that the x86 is little-endian. If the x86 were instead big-endian what would you set i to in order to yield the same output? Would you need to change 57616 to a different value?

    [Here's a description of little- and big-endian](http://www.webopedia.com/TERM/b/big_endian.html) and [a more whimsical description](http://www.networksorcery.com/enp/ien/ien137.txt).

5. In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen?
    ```
    cprintf("x=%d y=%d", 3);
    ```
6. Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change cprintf or its interface so that it would still be possible to pass it a variable number of arguments?

Challenge Enhance the console to allow text to be printed in different colors. The traditional way to do this is to make it interpret [ANSI escape sequences](http://rrbrandt.dee.ufcg.edu.br/en/docs/ansi/) embedded in the text strings printed to the console, but you may use any mechanism you like. There is plenty of information on [the 6.828 reference page](https://pdos.csail.mit.edu/6.828/2018/reference.html) and elsewhere on the web on programming the VGA display hardware. If you're feeling really adventurous, you could try switching the VGA hardware into a graphics mode and making the console draw text onto the graphical frame buffer.

### The Stack

In the final exercise of this lab, we will explore in more detail the way the C language uses the stack on the x86, and in the process write a useful new kernel monitor function that prints a backtrace of the stack: a list of the saved Instruction Pointer (IP) values from the nested call instructions that led to the current point of execution.

Exercise 9. Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?

The x86 stack pointer (esp register) points to the lowest location on the stack that is currently in use. Everything below that location in the region reserved for the stack is free. Pushing a value onto the stack involves decreasing the stack pointer and then writing the value to the place the stack pointer points to. Popping a value from the stack involves reading the value the stack pointer points to and then increasing the stack pointer. In 32-bit mode, the stack can only hold 32-bit values, and esp is always divisible by four. Various x86 instructions, such as call, are "hard-wired" to use the stack pointer register.

The ebp (base pointer) register, in contrast, is associated with the stack primarily by software convention. On entry to a C function, the function's prologue code normally saves the previous function's base pointer by pushing it onto the stack, and then copies the current esp value into ebp for the duration of the function. If all the functions in a program obey this convention, then at any given point during the program's execution, it is possible to trace back through the stack by following the chain of saved ebp pointers and determining exactly what nested sequence of function calls caused this particular point in the program to be reached. This capability can be particularly useful, for example, when a particular function causes an assert failure or panic because bad arguments were passed to it, but you aren't sure who passed the bad arguments. A stack backtrace lets you find the offending function.

Exercise 10. To become familiar with the C calling conventions on the x86, find the address of the test_backtrace function in obj/kern/kernel.asm, set a breakpoint there, and examine what happens each time it gets called after the kernel starts. How many 32-bit words does each recursive nesting level of test_backtrace push on the stack, and what are those words?

Note that, for this exercise to work properly, you should be using the patched version of QEMU available on the [tools](https://pdos.csail.mit.edu/6.828/2018/tools.html) page or on Athena. Otherwise, you'll have to manually translate all breakpoint and memory addresses to linear addresses.

The above exercise should give you the information you need to implement a stack backtrace function, which you should call mon_backtrace(). A prototype for this function is already waiting for you in kern/monitor.c. You can do it entirely in C, but you may find the read_ebp() function in inc/x86.h useful. You'll also have to hook this new function into the kernel monitor's command list so that it can be invoked interactively by the user.

The backtrace function should display a listing of function call frames in the following format:

```
Stack backtrace:
  ebp f0109e58  eip f0100a62  args 00000001 f0109e80 f0109e98 f0100ed2 00000031
  ebp f0109ed8  eip f01000d6  args 00000000 00000000 f0100058 f0109f28 00000061
  ...
```

Each line contains an ebp, eip, and args. The ebp value indicates the base pointer into the stack used by that function: i.e., the position of the stack pointer just after the function was entered and the function prologue code set up the base pointer. The listed eip value is the function's return instruction pointer: the instruction address to which control will return when the function returns. The return instruction pointer typically points to the instruction after the call instruction (why?). Finally, the five hex values listed after args are the first five arguments to the function in question, which would have been pushed on the stack just before the function was called. If the function was called with fewer than five arguments, of course, then not all five of these values will be useful. (Why can't the backtrace code detect how many arguments there actually are? How could this limitation be fixed?)

The first line printed reflects the currently executing function, namely mon_backtrace itself, the second line reflects the function that called mon_backtrace, the third line reflects the function that called that one, and so on. You should print all the outstanding stack frames. By studying kern/entry.S you'll find that there is an easy way to tell when to stop.

Here are a few specific points you read about in K&R Chapter 5 that are worth remembering for the following exercise and for future labs.

-   If int _p = (int_)100, then (int)p + 1 and (int)(p + 1) are different numbers: the first is 101 but the second is 104. When adding an integer to a pointer, as in the second case, the integer is implicitly multiplied by the size of the object the pointer points to.
-   p[i] is defined to be the same as \*(p+i), referring to the i'th object in the memory pointed to by p. The above rule for addition helps this definition work when the objects are larger than one byte.
-   &p[i] is the same as (p+i), yielding the address of the i'th object in the memory pointed to by p.

Although most C programs never need to cast between pointers and integers, operating systems frequently do. Whenever you see an addition involving a memory address, ask yourself whether it is an integer addition or pointer addition and make sure the value being added is appropriately multiplied or not.

Exercise 11. Implement the backtrace function as specified above. Use the same format as in the example, since otherwise the grading script will be confused. When you think you have it working right, run make grade to see if its output conforms to what our grading script expects, and fix it if it doesn't. After you have handed in your Lab 1 code, you are welcome to change the output format of the backtrace function any way you like.

If you use read_ebp(), note that GCC may generate "optimized" code that calls read_ebp() before mon_backtrace()'s function prologue, which results in an incomplete stack trace (the stack frame of the most recent function call is missing). While we have tried to disable optimizations that cause this reordering, you may want to examine the assembly of mon_backtrace() and make sure the call to read_ebp() is happening after the function prologue.

At this point, your backtrace function should give you the addresses of the function callers on the stack that lead to mon_backtrace() being executed. However, in practice you often want to know the function names corresponding to those addresses. For instance, you may want to know which functions could contain a bug that's causing your kernel to crash.

To help you implement this functionality, we have provided the function debuginfo_eip(), which looks up eip in the symbol table and returns the debugging information for that address. This function is defined in kern/kdebug.c.

Exercise 12. Modify your stack backtrace function to display, for each eip, the function name, source file name, and line number corresponding to that eip.

In debuginfo*eip, where do \_\_STAB*\* come from? This question has a long answer; to help you to discover the answer, here are some things you might want to do:

-   look in the file kern/kernel.ld for \__STAB_\*
-   run objdump -h obj/kern/kernel
-   run objdump -G obj/kern/kernel
-   run gcc -pipe -nostdinc -O2 -fno-builtin -I. -MD -Wall -Wno-format -DJOS_KERNEL -gstabs -c -S kern/init.c, and look at init.s.
-   see if the bootloader loads the symbol table in memory as part of loading the kernel binary

Complete the implementation of debuginfo_eip by inserting the call to stab_binsearch to find the line number for an address.

Add a backtrace command to the kernel monitor, and extend your implementation of mon_backtrace to call debuginfo_eip and print a line for each stack frame of the form:

```
K> backtrace
Stack backtrace:
  ebp f010ff78  eip f01008ae  args 00000001 f010ff8c 00000000 f0110580 00000000
         kern/monitor.c:143: monitor+106
  ebp f010ffd8  eip f0100193  args 00000000 00001aac 00000660 00000000 00000000
         kern/init.c:49: i386_init+59
  ebp f010fff8  eip f010003d  args 00000000 00000000 0000ffff 10cf9a00 0000ffff
         kern/entry.S:70: <unknown>+0
K>
```

Each line gives the file name and line within that file of the stack frame's eip, followed by the name of the function and the offset of the eip from the first instruction of the function (e.g., monitor+106 means the return eip is 106 bytes past the beginning of monitor).

Be sure to print the file and function names on a separate line, to avoid confusing the grading script.

Tip: printf format strings provide an easy, albeit obscure, way to print non-null-terminated strings like those in STABS tables. printf("%.\*s", length, string) prints at most length characters of string. Take a look at the printf man page to find out why this works.

You may find that some functions are missing from the backtrace. For example, you will probably see a call to monitor() but not to runcmd(). This is because the compiler in-lines some function calls. Other optimizations may cause you to see unexpected line numbers. If you get rid of the -O2 from GNUMakefile, the backtraces may make more sense (but your kernel will run more slowly).

Each line gives the file name and line within that file of the stack frame's eip, followed by the name of the function and the offset of the eip from the first instruction of the function (e.g., monitor+106 means the return eip is 106 bytes past the beginning of monitor).

Be sure to print the file and function names on a separate line, to avoid confusing the grading script.

Tip: printf format strings provide an easy, albeit obscure, way to print non-null-terminated strings like those in STABS tables. printf("%.\*s", length, string) prints at most length characters of string. Take a look at the printf man page to find out why this works.

You may find that some functions are missing from the backtrace. For example, you will probably see a call to monitor() but not to runcmd(). This is because the compiler in-lines some function calls. Other optimizations may cause you to see unexpected line numbers. If you get rid of the -O2 from GNUMakefile, the backtraces may make more sense (but your kernel will run more slowly).
