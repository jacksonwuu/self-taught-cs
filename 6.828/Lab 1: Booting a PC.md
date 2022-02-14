# Lab 1: Booting a PC

Target of this lab：

1、Get familiar with assembly language and C language.
2、Get familiar with dev tools like QEMU, GCC, GDB, etc.
3、Understand how PC boot.

PS: You should spend as time as possible on this lab, because there are many things to learn: C, Makefile, gcc, nasm, ld, etc.

### Exercises

#### 1 Get familiar with x86 Assembly language.

Read Book [PC Assembly Language](https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf)，and write some simple assembly code, like add/sub/multi/echo function. 

Understand x86 registers and what they do.

#### 2 Use GDB to see what ROM BIOS do.

#### 3 Use GDB to see how Boot Loader work.

1、在什么地方开始执行 32bit 代码的？究竟是什么导致了 16bit 到 32bit 模式的转变？

2、Boot Loader 最后一条指令是什么？它加载内核的第一条指令是什么？

3、内核的第一条指令是什么？

4、Boot Loader 如何决定它必须读取多少个扇区才能从磁盘中获取整个内核？它在哪里找到这个信息？

#### 4 Get familiar with C language pointer.

Read K&R，especially the pointer part.

#### 5 Modify code and run.

#### 6 Use GDB's x command to see how memeory changed.
