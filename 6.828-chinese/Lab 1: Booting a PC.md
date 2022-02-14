# Lab 1: Booting a PC

本实验的目标：

1、熟悉汇编语言和 C 语言。
2、熟悉开发工具如 QEMU, GCC, GDB 等。
3、理解操作系统的启动过程。

PS: 可以在这个实验上多花的时间，学一学 C, Makefile, gcc, nasm, ld 等开发工具（如果不熟悉的话）。

### Exercises

#### 1 熟悉 x86 汇编语言

阅读 [PC Assembly Language](https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf)，写一些简单的汇编代码, 比如说add/sub/multi/echo 函数。

理解 x86 寄存器。

对汇编语言理解地越深入越有利于学习操作系统，因为汇编语言有助于你理解程序如何控制硬件，而操作系统实际上就是一组和硬件打交道的程序。

#### 2 用 GDB 看 ROM BIOS 做了什么

#### 3 用 GDB 看 Boot Loader 运行过程

1、在什么地方开始执行 32bit 代码的？究竟是什么导致了 16bit 到 32bit 模式的转变？

2、Boot Loader 最后一条指令是什么？它加载内核的第一条指令是什么？

3、内核的第一条指令是什么？

4、Boot Loader 如何决定它必须读取多少个扇区才能从磁盘中获取整个内核？它在哪里找到这个信息？

#### 4 熟悉 C 语言指针

阅读 C 语言经典书籍 K&R，尤其是指针的部分。

#### 5 修改代码再运行

#### 6 用 GDB 的 x 指令去查看内存
