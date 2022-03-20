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
