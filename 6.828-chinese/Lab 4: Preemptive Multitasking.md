# Lab 4: Preemptive Multitasking

本实验分为 ABC 三个部分。

在这个实验中，要实现用户环境的切换，也就是实现多任务。

A：支持多核 CPU，实现进程调度（round-robin scheduling）。

B：实现 fork()。

C：实现进程间通信，实现另外一种进程调度（ preemptive scheduling）。

这个实验有点难度，所以我把课程的一些重要信息翻译为中文贴在下面，来促进自己对这个实验的理解。

很好的材料：

-   [MP spec](https://pdos.csail.mit.edu/6.828/2011/readings/ia32/MPspec.pdf)

### A

让 JOS 可以支持对称多处理器（英语：Symmetric multiprocessing，缩写为 SMP），在 SMP 中，每一个 CPU 的地位都是平等的，对资源的使用权限相同。在计算机启动过程，这些 CPU 内核会被分为两类：the bootstrap processor (BSP) 负责启动和初始化操作系统; the application processors (APs) 会在操作系统启动和运行之后才会被激活。哪一个是 BSP 是是由硬件和 BIOS 决定的。

在一个 SMP 系统里，每一个 CPU 都有一个伴随 local APIC(LAPIC)组件。LAPIC 组件负责传递中断给系统，以及提供一个与之连接的 CPU 的唯一标识。关于 LAPIC 组件的代码在`kern/lapic.c`里。

-   通过 cpunum()来获取 CPU 的唯一标识 APIC ID。
-   lapic_startup()在 BSP 里向 APs 发送 the STARTUP interprocessor interrupt (IPI)来启动 APs。
-   In part C, we program LAPIC's built-in timer to trigger clock interrupts to support preemptive multitasking (see apic_init()).

处理器通过 MMIO（memory-mapped I/O）来与 LAPIC 交互。在 MMIO 里，一部分物理内存会与设备寄存器硬接线。所以，load/store 这样的内存读写指令也可以用来对设备寄存器进行读写。在物理内存 0xA0000 有一个”I/O 洞“，我们通过这一块的物理内存来向 VGA 显示器缓冲写入。

**练习 1**：实现 kern/pmap.c 里的 mmio_map_region。

#### Application Processor 的启动

在启动 APs 之前，需要先用 BSP 收集多处理器系统的信息，比如说 CPUs 的数量，它们的 APIC ID 以及 LAPIC 单元的 MMIO 地址。

`kern/mpconfig.c` 里的`mp_init()`函数通过查询 BIOS 的内存区域里的 MP 配置表来获取的。

`kern/init.c`里的`boot_aps()` 驱动了 AP 的启动。APs 在实模式下启动，我们可以控制 AP 的启动代码。在 JOS 里，我们拷贝入口函数到 0x7000(MPENTRY_PADDR)处。

之后，`boot_aps()`通过向 AP 的 LAPIC 单元发送 STARTUP IPIs，伴随着初始 CS:IP 地址（AP 应该在此运行它的入口代码，在 JOS 里就是 MPENTRY_PADDR）一个一个地激活 AP。在启动之后，给 AP 开启保护模式，开启分页，以及调用 C setup routine `mp_main()`。`boot_aps()` 会等待 AP 发出一个 CPU_STARTED 标志信号到 struct CpuInfo 的 cpu_status 字段，然后`boot_aps()`才会继续激活下一个 AP。

**练习 2**：Then modify your implementation of page_init() in kern/pmap.c to avoid adding the page at MPENTRY_PADDR to the free list, so that we can safely copy and run AP bootstrap code at that physical address. Your code should pass the updated check_page_free_list() test (but might fail the updated check_kern_pgdir() test, which we will fix soon).

#### CPU 各自的状态及其初始化

每个 CPU 都有自己的状态，操作系统也会维护一个数据结构来存放 CPU 们共享的状态。每当调用`cpunum()`的时候总会范围当前 CPU 的 ID，这个 ID 可以作为 cpus 的索引。另外，thiscpu 也可以获取当前的 CPU 的 CpuInfo。

下面是一些各自 CPU 有的状态：

-   Per-CPU kernel stack：这些 CPU 可以同时陷入内核，所以每个内核都需要一个内核栈，以免相互干扰内核代码的执行。`percpu_kstacks[NCPU][KSTKSIZE]`为这些内核栈保留了内存空间。
-   Per-CPU tss and tss descriptor：`cpus[i].cpu_ts`
-   Per-CPU current environment pointer：这些 CPU 可以同时运行用户进程，所以要记录当前 CPU 执行的用户环境。curenv to refer to cpus[cpunum()].cpu_env (or thiscpu->cpu_env)
-   Per-CPU system registers：每一个 CPU 都有一些私有的寄存器，所以一些寄存器初始化函数要在每一个 CPU 上都运行一边，比如说 lcr3()，ltr()，lgdt()等，env_init_percpu() 和 trap_init_percpu() 都是为了这个目的而设计的。

**练习 3**：修改`kern/pmap.c`里的`mem_init_mp()`，把每个 CPU 的 stack 映射到`KSTACKTOP`，修改完后需要通过`check_kern_pgdir()`。

**练习 4**：trap_init_percpu() (kern/trap.c)中的代码为 BSP 初始化 TSS 和 TSS 描述符。它在 Lab 3 中正确，但在其他 cpu 上运行时不正确。更改代码，使其能够在所有 cpu 上工作。(注意:您的新代码不应该再使用全局 ts 变量。)

#### 锁

目前的代码是在 AP 启动之后开始自旋。在开始下一步之前，我们要先解决多个 CPU 同时竞争内核代码的问题，最简单的办法就是`big kernel lock`。big kernel lock 是一个全局锁，当用户态代码进入内核态时获取这个锁，当回到用户态时再释放掉。这样的话，用户态进程可以同时在多个 CPU 里运行，但是某一时刻只有一个 CPU 会运行内核代码，任何想要进入内核态的进程都需要等待。

`kern/spinlock.h`里定义了 big kernel lock，叫做`kernel_lock`。然后也提供了一个获得锁和释放锁的功能，分别是`lock_kernel()`和`lock_kernel()`。我们需要在一下几个地方应用该锁：

-   `i386_init()`
-   `mp_main()`
-   `trap()`
-   `env_run()`

**练习 5**：在上述几个地方应用上这个 big kernel lock。

目前还无法测试你是否写对了，但是等之后实现了调度器之后就可以测试了。

**问题**：为什么我们有了一个全局锁之外，还要把每个 CPU 的内核栈给分开？在哪种场景下，即使有 big kernel lock，共享内核栈却会出问题？
**答**：查看代码可以发现，在执行代码和获取 lock 之间是会执行一些代码的，如果某个 CPU 在执行这些代码的时候，其他 CPU 正在执行内核代码，那么会导致内核栈被污染。

#### Round-Robin 调度

下一个任务是实现“round-robin”来调度多个用户进程：

-   `kern/sched.c`里的`sched_yield()`函数负责选择一个进程（用户环境）去运行。直接遍历 env 数组，从上一个运行环境的地方开始遍历，选取第一个找到的 ENV_RUNNABLE 状态的用户环境。然后调用 env_call 去运行这个环境。
-   保证`sched_yield()`永远不会让同一个用户环境在两个 CPU 里同时运行。
-   课程提供了一个系统调用`sys_yield()`，用户进程可以调用这个系统调用来触发内核的`sched_yield()`来主动放弃 CPU 给其他用户环境。

**练习 6**：实现这个`sched_yield()`函数。

**问题**：在`env_run()`的实现中，你会调用`lcr3()`，在调用这个函数的前后，你的代码还是引用了变量 e，也就是传入的参数。但是当你改变 cr3 寄存器后，地址上下文也会改变。为什么指针 e 均可以在地址上下文（页目录）切换的前后进行取消引用呢？
**答**：e 对于每一个用户环境来说，映射到内存上相同的地方，所以切换页目录是不会改变 e 的值的。

**问题**：当内核从一个用户环境切换到另外一个，它必须保证老的用户环境的寄存器被合理存储以便之后恢复，为什么呢？这个过程什么时候发生的？
**答**：保存了用户环境的寄存器之后，才能正常恢复该用户环境的运行。 `env_run()`中的这个函数`env_pop_tf()`用来保存老用户环境的寄存器。这个函数把当前寄存器保存到传入的 Trapframe 里，然后再调用`iret`指令返回到用户态代码中继续执行。

#### 创建用户环境的系统调用

虽然我们已经实现了用户环境之间的切换，但是功能还非常有限。我们接下来要实现一个系统调用，允许用户环境下的代码来创建一个新的用户环境。

在 Unix 里，通过 fork()来实现这个功能。fork()把父进程的整个地址空间拷贝下来，用来创建子进程，它们唯一不通的就是它们的进程 ID 和父进程 ID。

JOS 将会提供一系列原始的系统调用，通过这些系统调用，我们可以实现 Unix 里 fork()的效果：

-   sys_exofork: 创建一个新的用户环境，在父用户环境里，这个系统调用会返回子用户环境的 id，在被创建的子用户环境里，会返回 0。
-   sys_env_set_status: 标记一个用户环境的状态，当地址空间和寄存器都初始化之后，这个系统调用可以用来把一个用户环境标记为可运行状态。
-   sys_page_alloc: 申请一个物理内存页，并用户环境的虚拟地址映射上去。
-   sys_page_map: 把一个用户环境的页映射拷贝到另外一个用户环境，这样的话，两个用户环境就可以共享同一个物理内存页。
-   sys_page_unmap: 给出一个虚拟地址，把某个映射在某个用户环境的内存页给取消映射。

### B Copy-on-Write Fork

xv6 Unix 的 fork()是把父进程的所有内存页拷贝到新的页里给子进程来使用，这个拷贝内存的过程是 `fork()` 中最费时的一步。但是很多时候，子进程并不会写入内存然后就直接退出来，这样就会浪费大量的内存。

出于这个原因，我们需要实现 copy-on-write，所以要在 fork()中拷贝父进程的映射给子进程，而不是内存页里的内容。当某个一个进程尝试去写入内存时，会产生一个页异常错误，这个时候触发中断，内核才拷贝真正的内存页给子进程来使用。

#### 用户级页错误处理

在实现 copy-on-write 之前，需要先实现页错误的处理程序。

练习 8：实现 sys_env_set_pgfault_upcall 系统调用。
