## 6.828

### Lab

-   [Lab 1: Booting a PC](./Lab%201:%20Booting%20a%20PC.md)
-   [Lab 2: Memory Management](./Lab%202:%20Memory%20Management.md)
-   [Lab 3: User Environments](./Lab%203:%20User%20Environments.md)
-   [Lab 4: Preemptive Multitasking](./Lab%204:%20Preemptive%20Multitasking.md)
-   [Lab 5: File system, Spawn and Shell](./Lab%205:%20File%20system%2C%20Spawn%20and%20Shell.md)
-   [Lab 6: Network Driver](./Lab%206:%20Network%20Driver.md)
-   [Lab 7: Final JOS project](./Lab%207:%20Final%20JOS%20project.md)

### Homework

-   [boot xv6](./Homework:%20boot%20xv6.md)
-   [shell](./Homework:%20shell.md)
-   [xv6 log]()
-   [xv locking]()
-   [xv CPU alarm]()
-   [mmap()]()
-   [barrier]()
-   [big files]()

### My gains

操作系统主要分为几个部分：

-   启动内核：熟悉硬件，以及计算机的启动过程，写一个 bootloader 来启动程序。
-   内存管理：主要是通过分段、分页机制来实现内存的管理的。这里会引入虚拟地址空间的概念，我们要合理规划虚拟地址空间，以便满足操作系统的功能需要。
-   陷入、中断和驱动：操作系统是靠中断进行驱动的，在这里要熟悉硬件是如何处理中断的
-   进程管理：进程管理是建立在内存管理和中断的基础之上的，每个进程都有自己的地址空间
-   文件系统：要熟悉对磁盘的管理。
-   网络：核心问题是如何解决高速接收数据而不影响操作系统的流畅。Linux 使用的是软中断线程。

-   BIOS 加载机制
-   汇编语言基础知识
-   C 语言和汇编的互相调用约定
-   分段、分页机制
-   中断机制
-   多核处理器运行机制
-   进程锁机制
-   DMA 机制

### Reading material

-   xv6-book
-   80386 Reference Manual
-   IA-32 Intel Architecture Software Developer's Manual
-   https://pdos.csail.mit.edu/6.828/2018/reference.html
-   Linux 内核设计与实现

**如何学习 MIT6.828？**

MIT 6.828 可以说是世界上最好的操作系统课程，非常建议每一个对技术有追求的朋友学习。

在每次做实验之前，最好先把实验的指南以及本次实验配套的代码都通读一遍，尤其是要充分理解这些代码做了什么事情，以及实验需要你做什么事情，不要着急着去写实现，不然一开始写代码就容易两眼发黑。其实，Lab 1 就是帮助学生熟悉现有代码和开发流程的。这是最重要的一点！！！因为实验的任务书篇幅很长，内容很多很细，所以要反复多读几遍任务书。

理解数据结构是理解代码的最佳切入点。

为了让课程学习更加平滑，还有另外一个建议是按照顺序去做作业和实验，作业一般都是为实验做普遍，让你先理解一些概念再做实验。

操作系统开发比其他软件都要复杂，首先不能使用 C 标准库来开发内核，因为这些库程序本身就是建立在内核之上的，你不但不能使用而且你还要自己写这些程序库；另外一个难度在于程序调试的难度，要通过 QEMU 这样的硬件模拟器以及 gdb 这样的调试器来做，同时还要能深刻理解程序运行原理；以及对于内核代码来说是没有内存保护机制的，所以要非常小心程序的编写，不然一个错误就会导致整个系统崩溃；最后一点是要非常注意内核里的同步和并发，尤其是为多核处理器开发操作系统时。鉴于操作系统的开发难度，我们一定会遇到一时解决不了的问题，放平心态，积极与学习就好了。

这个课程已经把许多晦涩的部分给我们实现了，我们只需要关注操作系统中最精华的那部分：中断与中断处理、内存管理、进程管理、文件系统、网络等模块的核心实现细节。所以不一定需要理解每一行代码。对于一些晦涩的代码，只要理解它做了什么事情即可。

### Other things

-   Assembly Language

As a programmer, I found that learning assembly language is pretty much helpful. Learning assembly make me grasp a deeper understanding of what high language code actually do, how registers works and how Operating System works.

MIT 6.828 course strongly recommend a book about x86 assembly, [PC Assembly Language](https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf), I‘ve read through, it's a really a great book!