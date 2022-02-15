# Lab 1: Booting a PC

本实验的目标：

1、熟悉汇编语言和 C 语言。
2、熟悉开发工具如 QEMU, GCC, GDB 等。
3、理解操作系统的启动过程。

PS: 可以在这个实验上多花的时间，最好顺带着学一学 C, Makefile, gcc, nasm, ld 等开发工具（如果不熟悉的话）。如果时间充裕，可以把 _xv6-book_ 给看一遍，这本书也有中文翻译的。注意，_xv6-book_ 有两种版本，旧的是 x86 版本，新的是 risc 版本。

在开始实验之前，可以弄一个虚拟机配置好做实验的环境，我使用的 是在 VMware Fusion 下运行的 Ubuntu Desktop 20.04 LTS。推荐使用 Linux，当然你也可以使用其他的操作系统，但是准备开发环境相当费时，还不如直接 Linux ～

### Exercises

#### 1 熟悉 x86 汇编语言

阅读 [PC Assembly Language](https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf)，写一些简单的汇编代码, 比如说 add/sub/multi/echo 函数。

理解 x86 寄存器，主要是 x86 cpu 上的寄存器，寄存器的演变过程，如何从 8bit 的一直到 64bit 的，以及不同长度寄存器的关系，各个寄存器的作用等。

对汇编语言理解地越深入越有利于学习操作系统，因为汇编语言有助于你理解程序如何控制硬件，而操作系统实际上就是一组和硬件打交道的程序。

#### 2 用 GDB 看 ROM BIOS 做了什么

-   The IBM PC starts executing at physical address 0x000ffff0, which is at the very top of the 64KB area reserved for the ROM BIOS.
-   The PC starts executing with CS = 0xf000 and IP = 0xfff0.
-   The first instruction to be executed is a jmp instruction, which jumps to the segmented address CS = 0xf000 and IP = 0xe05b.

可以查看到第一条执行的指令是：

```shell
[f000:fff0]    0xffff0:	ljmp   $0x3630,$0xf000e05b
```

#### 3 用 GDB 看 Boot Loader 运行过程

Set a breakpoint at address 0x7c00, which is where the boot sector will be loaded. Continue execution until that breakpoint. Trace through the code in boot/boot.S, using the source code and the disassembly file obj/boot/boot.asm to keep track of where you are. Also use the x/i command in GDB to disassemble sequences of instructions in the boot loader, and compare the original boot loader source code with both the disassembly in obj/boot/boot.asm and GDB.

要回答几个问题：

1、在什么地方开始执行 32bit 代码的？究竟是什么导致了 16bit 到 32bit 模式的转变？（实模式到保护模式到转变）

答：模式的转变其实就是改变 cr0 寄存器的 PE 位，把它从 0 改成 1。顺着代码一路往下走很快就会发现这个指令。可以查看`seta20.2`符号所在的地址，具体的代码就是在这里执行的。

关于CR0寄存器的维基百科：[Control register](https://en.wikipedia.org/wiki/Control_register)

2、Boot Loader 最后一条指令是什么？它加载内核的第一条指令是什么？

答：

3、内核的第一条指令是什么？

答：

4、Boot Loader 如何决定它必须读取多少个扇区才能从磁盘中获取整个内核？它在哪里找到这个信息？

答：/boot/main.c 里定义好的。

```c
#define SECTSIZE	512
#define ELFHDR		((struct Elf *) 0x10000) // scratch space

void readsect(void*, uint32_t);
void readseg(uint32_t, uint32_t, uint32_t);
```

#### 4 熟悉 C 语言指针

阅读 C 语言经典书籍 K&R，尤其是指针的部分。K&R 是非常经典的书籍，只有两百多页，强烈建议找一本来看看。

#### 5 修改初始地址 0x7c00 为其他地址再尝试编译运行

修改`/boot/Makefrag`里的`-Ttext 0x7C00`为`-Ttext 0x7C02`再编译运行。你会发现 QEMU 运行失败。

运行完后别忘了改回去。

#### 6 用 GDB 的 x 指令去查看内存

Reset the machine (exit QEMU/GDB and start them again). Examine the 8 words of memory at 0x00100000 at the point the BIOS enters the boot loader, and then again at the point the boot loader enters the kernel. Why are they different? What is there at the second breakpoint? (You do not really need to use QEMU to answer this question. Just think.)

#### 7 Examine memory at 0x00100000 and at 0xf0100000.

#### 8 We have omitted a small fragment of code - the code necessary to print octal numbers using patterns of the form "%o". Find and fill in this code fragment.

Read through kern/printf.c, lib/printfmt.c, and kern/console.c, and make sure you understand their relationship.

在留空白的代码上实现八进制的格式化 print 功能。

#### 9 Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?

#### 10 To become familiar with the C calling conventions on the x86, find the address of the test_backtrace function in obj/kern/kernel.asm, set a breakpoint there, and examine what happens each time it gets called after the kernel starts. How many 32-bit words does each recursive nesting level of test_backtrace push on the stack, and what are those words?

熟悉 C 语言和汇编语言的相互调用约定，然后查看 test_backtrace 函数，理解这个函数做了什么。

调用约定可以看 [PC Assembly Language]()，里面说得很详细了。

backtrack功能主要是关注 ebp 寄存器的值，因为按照调用约定，每次调用函数的时候都会把栈指针 esp 保存到 ebp 里，通过检查 ebp 栈就能回溯调用的函数。（原理非常简单）
