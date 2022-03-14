# Lab 4: Preemptive Multitasking

本实验分为 ABC 三个部分。

在这个实验中，要实现用户环境的切换，也就是实现多任务。

A：支持多核 CPU，实现进程调度（round-robin scheduling）。

B：实现 fork()。

C：实现进程间通信，实现另外一种进程调度（ preemptive scheduling）。

这个实验有点难度，所以我把课程的一些重要信息翻译为中文贴在下面，来促进自己对这个实验的理解。

### A

让 JOS 可以支持对称多处理器（英语：Symmetric multiprocessing，缩写为 SMP），在 SMP 中，每一个 CPU 的地位都是平等的，对资源的使用权限相同。在计算机启动过程，这些 CPU 内核会被分为两类：the bootstrap processor (BSP) 负责启动和初始化操作系统; the application processors (APs) 会在操作系统启动和运行之后才会被激活。哪一个是 BSP 是是由硬件和 BIOS 决定的。

在一个 SMP 系统里，每一个 CPU 都有一个伴随 local APIC(LAPIC)组件。LAPIC 组件负责传递中断给系统，以及提供一个与之连接的 CPU 的唯一标识。关于 LAPIC 组件的代码在`kern/lapic.c`里。

-   通过 cpunum()来获取 CPU 的唯一标识 APIC ID。
-   lapic_startup()在 BSP 里向 APs 发送 the STARTUP interprocessor interrupt (IPI)来启动 APs。
-   In part C, we program LAPIC's built-in timer to trigger clock interrupts to support preemptive multitasking (see apic_init()).

处理器通过 MMIO（memory-mapped I/O）来与 LAPIC 交互。在 MMIO 里，一部分物理内存会与设备寄存器硬接线。所以，load/store 这样的内存读写指令也可以用来对设备寄存器进行读写。在物理内存 0xA0000 有一个”I/O 洞“，我们通过这一块的物理内存来向 VGA 显示器缓冲写入。

**练习一**：实现 kern/pmap.c 里的 mmio_map_region。

#### Application Processor 的启动

在启动 APs 之前，需要先用 BSP 收集多处理器系统的信息，比如说 CPUs 的数量，它们的 APIC ID 以及 LAPIC 单元的 MMIO 地址。

`kern/mpconfig.c` 里的`mp_init()`函数通过查询 BIOS 的内存区域里的 MP 配置表来获取的。

`kern/init.c`里的`boot_aps()` 驱动了 AP 的启动。APs 在实模式下启动，我们可以控制 AP 的启动代码。在 JOS 里，我们拷贝入口函数到 0x7000(MPENTRY_PADDR)处。

之后，`boot_aps()`通过向 AP 的 LAPIC 单元发送 STARTUP IPIs，伴随着初始 CS:IP 地址（AP 应该在此运行它的入口代码，在 JOS 里就是 MPENTRY_PADDR）一个一个地激活 AP。在启动之后，给 AP 开启保护模式，开启分页，以及调用 C setup routine `mp_main()`。`boot_aps()` 会等待 AP 发出一个 CPU_STARTED 标志信号到 struct CpuInfo 的 cpu_status 字段，然后`boot_aps()`才会继续激活下一个 AP。

**练习二**：Then modify your implementation of page_init() in kern/pmap.c to avoid adding the page at MPENTRY_PADDR to the free list, so that we can safely copy and run AP bootstrap code at that physical address. Your code should pass the updated check_page_free_list() test (but might fail the updated check_kern_pgdir() test, which we will fix soon).

#### CPU 各自的状态及其初始化

每个 CPU 都有自己的状态，操作系统也会维护一个数据结构来存放 CPU 们共享的状态。每当调用`cpunum()`的时候总会范围当前 CPU 的 ID，这个 ID 可以作为 cpus 的索引。另外，thiscpu 也可以获取当前的 CPU 的 CpuInfo。

下面是一些各自 CPU 有的状态：

-   kernel stack
-   tss and tss descriptor
-   current environment pointer
-   system registers

#### 锁
