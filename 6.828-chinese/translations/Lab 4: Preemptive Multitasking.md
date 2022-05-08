# Lab 4: Preemptive Multitasking（可抢占式多任务）

## 介绍

在本实验室中，你将在多个同时活跃的用户模式环境中实现抢占式多任务处理。

在 A 部分中，你将为 JOS 添加多处理器支持，实现循环调度，并添加基本的环境管理系统调用(创建和销毁环境的调用，以及分配/映射内存的调用)。

在 B 部分，你将实现一个类 unix 的 fork()，它允许用户模式环境创建自身的副本。

最后，在 C 部分中，你将添加对进程间通信(IPC)的支持，允许不同的用户环境显式地相互通信和同步。你还将添加对硬件时钟中断和抢占的支持。

### 开始

[omit]

### 实验要求

该课程被分为三部分，A，B 和 C。我们每周完成一个部分。

和以前一样，你需要做所有在实验室中描述的常规练习和至少一个挑战性问题。(你不需要每个部分做一个挑战题，整个实验室只做一个。)另外，你需要对你所实现的挑战问题写一个简短的描述。如果你实现了多个挑战问题，你只需要在报告中描述其中一个，当然也欢迎你做更多。在提交作业之前，将作业记录放在实验室目录的顶层名为 answers-lab4.txt 的文件中。

## Part A: 多处理器支持和协同多任务处理

在本实验的第一部分中，你将首先扩展 JOS 以在多处理器系统上运行，然后实现一些新的 JOS 内核系统调用，以允许用户级环境创建额外的新环境。你还将实现协作轮询调度，允许内核在当前环境自愿放弃 CPU(或退出)时从一个环境切换到另一个环境。在后面的 C 部分中，你将实现抢占式调度，它允许内核在经过一段时间后重新从环境中获得对 CPU 的控制，那怕该环境不合作。

### 多处理器支持

我们打算让 JOS 支持“对称多处理”(SMP)，这是一种多处理器模型，在这种模型中，所有 cpu 对系统资源(如内存和 I/O 总线)都有同等的访问权限。虽然在 SMP 中，所有 cpu 的功能都是相同的，但在引导过程中，它们可以分为两类:引导处理器(bootstrap processor, BSP)负责初始化系统和引导操作系统;只有在操作系统启动并运行之后，BSP 才会激活应用程序处理器(ap)。哪个处理器是 BSP 由硬件和 BIOS 决定。到目前为止，所有现有的 JOS 代码都在 BSP 上运行。

在 SMP 系统中，每个 CPU 都有一个相应的本地 APIC (LAPIC)单元。LAPIC 单元负责在整个系统中传递中断。LAPIC 还为其连接的 CPU 提供一个惟一标识符。在本实验室中，我们使用了以下 LAPIC 单元的基本功能(在 kern/ LAPIC .c 中):

-   读取 LAPIC 标识符(APIC ID)来告诉我们的代码当前运行在哪个 CPU 上(参见 cpunum())。
-   从 BSP 向 ap 发送 STARTUP interprocessor interrupt (IPI)来启动其他 cpu(参见 lapic_startap())。
-   在 C 部分，我们编写了 LAPIC 的内置定时器来触发时钟中断，以支持抢占式多任务(参见 apic_init())。

处理器使用内存映射 I/O (MMIO)访问它的 LAPIC。在 MMIO 中，物理内存的一部分硬连接到一些 I/O 设备的寄存器，因此通常用于访问内存的加载/存储指令也可以用于访问设备寄存器。你已经看到物理地址 0xA0000 上有一个 IO 孔(我们使用它来写入 VGA 显示缓冲区)。LAPIC 位于一个从物理地址 0xFE000000 (32MB 差 4GB)开始的漏洞中，因此我们无法在 KERNBASE 上使用通常的直接映射来访问它。JOS 虚拟内存映射在 MMIOBASE 中留下了 4MB 的空白，所以我们有一个地方可以像这样映射设备。由于后面的实验引入了更多的 MMIO 区域，因此你将编写一个简单的函数来从该区域分配空间并将设备内存映射到该区域。

练习 1。在 kern/map.c 中实现 mmio_map_region。要了解如何使用它，请查看 kern/lapic.c 中 lapic_init 的开头。在运行 mmio_map_region 的测试之前，你也必须进行下一个练习。

#### Application Processor Bootstrap（AP 处理器的启动）

在启动 ap 之前，BSP 应该首先收集多处理器系统的信息，如 cpu 总数、APIC id 和 LAPIC 单元的 MMIO 地址。kern/mpconfig.c 中的 mp_init()函数通过读取驻留在 BIOS 内存区域中的 MP 配置表来获取这些信息。

boot_aps()函数(在 kern/init.c 中)驱动 AP 引导进程。ap 在真实模式下启动，就像引导加载程序在引导/引导中启动一样。，因此 boot_aps()将 AP 入口代码(kern/mpentry.S)复制到一个在实际模式下可寻址的内存位置。与引导加载器不同的是，我们对 AP 从哪里开始执行代码有一些控制;我们将入口代码复制到 0x7000 (MPENTRY_PADDR)，但是任何未使用的、页面对齐的、低于 640KB 的物理地址都可以工作。

在此之后，boot_aps()通过向相应 AP 的 LAPIC 单元发送 STARTUP IPIs 以及初始 CS:IP 地址依次激活 AP, AP 应该在该 IP 地址开始运行它的入口代码(在本例中为 MPENTRY_PADDR)。kern/mpentry.S 中的入口代码。与 boot/boot.S 非常相似。经过一些简短的设置后，它将 AP 放入启用分页的保护模式，然后调用 C 设置例程 mp_main()(也在 kern/init.c 中)。boot_aps()等待 AP 在其结构体 CpuInfo 的 cpu_status 字段中发出 CPU_STARTED 标志，然后继续唤醒下一个 AP。

练习 2。读入 kern/init.c 中的 boot_aps()和 mp_main()，读入 kern/mpentry.S 中的汇编代码。确保你理解 ap 引导过程中的控制流传输。然后修改 kern/pmap.c 中的 page_init()实现，以避免将 MPENTRY_PADDR 上的页面添加到空闲列表中，这样我们就可以安全地复制并运行该物理地址上的 AP 引导代码。你的代码应该通过更新的 check_page_free_list()测试(但可能无法通过更新的 check_kern_pgdir()测试，我们将很快修复它)。

问题

1. 一行一行地对比 kern/mpentry.S 和 boot/boot.S。记住 kern/mpentry.S 被编译和链接，然后运行在 KERNBASE 之上，就像内核的其他部分一样，MPBOOTPHYS 宏的目的是什么？为什么在 kern/mpentry.S 里而不是 boot/boot.S 里？换言之，如果把它在 kern/mpentry.S 里去掉会发生什么？
   提示：重谈我们在实验一里讨论过的链接地址和加载地址的区别。

#### Per-CPU State and Initialization（每个 CPU 的状态及其初始化）

在编写多处理器操作系统时，区分每个处理器私有的每 cpu 状态和整个系统共享的全局状态是很重要的。kern/cpu.h 定义了每个 cpu 的大部分状态，包括 struct CpuInfo，它存储每个 cpu 的变量。cpunum()总是返回调用它的 CPU 的 ID，它可以用作 CPU 等数组的索引。或者说，宏 thiscpu 是当前 CPU 结构体 CpuInfo 的简写。

这里是你需要注意的每个 CPU 的状态：

-   每个 CPU 的内核栈（kernel stack）
    因为多 CPU 可以同时陷入内核，所以我们需要为每一个 CPU 弄一个内核栈，来防止它们相互干涉对方的执行。数组 percpu_kstacks[NCPU][kstksize]为 NCPU 保留了内核栈的空间。

    In Lab 2, you mapped the physical memory that bootstack refers to as the BSP's kernel stack just below KSTACKTOP. Similarly, in this lab, you will map each CPU's kernel stack into this region with guard pages acting as a buffer between them. CPU 0's stack will still grow down from KSTACKTOP; CPU 1's stack will start KSTKGAP bytes below the bottom of CPU 0's stack, and so on. inc/memlayout.h shows the mapping layout.

-   每个 CPU 的 TSS 和 TSS 描述符
    为了指定每一个 CPU 的内核栈都存在于哪里，需要每个 CPU 的任务状态段（TSS）。CPU i 的 TSS 存放在 cpu[i].cpu_ts 中，对应的 TSS 描述符定义在 GDT 项 gdt[(GD_TSS0 >> 3) + i]里。全局 ts 变量定义在 kern/trap.c 不会再有用处。

-   每个 CPU 当前运行的环境的指针
    因为每个 CPU 可以同时运行用户进程，所以我们重新定义符号 curenv，让它指向 cpus[cpunum()].cpu_env（或者是 thiscpu->cpu_env），也就是指向当前 CPU 正在执行的环境（当前代码执行在哪个 CPU 上）。

-   每个 CPU 的系统寄存器
    所有寄存器，包括系统寄存器，都是 CPU 私有的。因此，用来初始化这些寄存器的指令，比如说 lcr3(), ltr(), lgdt(), lidt()等，必须在每个 CPU 上都执行一次。函数 env_init_percpu()和 trap_init_percpu()是为这种目的而设定的。

    除此之外，如果你在你的解决方案中添加了任何额外的每 CPU 状态或执行了任何额外的特定于 CPU 的初始化（比如，在 CPU 寄存器中设置新位）以挑战早期实验室中的问题，请确保每个 CPU 上都复制了它们。

练习 3。修改 kern/pmap.c 里的 mem_init_mp()，使其映射每个 CPU 栈到 KSTACKTOP 的初始位置，正如 inc/memlayout.h 里展示的那样。每个堆栈的大小是 KSTKSIZE 字节加上未映射保护页的 KSTKGAP 字节。你的代码应该通过新测试 check_kern_pgdir()。

练习 4。trap_init_percpu()（位于 kern/trap.c）里的代码为 BSP 初始化 TSS 和 TSS 描述符。这个函数可以在实验三里正常运行，但是它在其他 CPU 上运行就会出错。修改代码，让它可以在所有的 CPU 上运行。（注意：你的新代码不应该再使用全局 ts 变量。）

当你完成了以上练习，在 QEMU 中以用 4 个 CPU 来运行 JOS，`make qemu CPUS=4`或者`make qemu-nox CPUS=4`，你应该可以看到这样的输出：

```
...
Physical memory: 66556K available, base = 640K, extended = 65532K
check_page_alloc() succeeded!
check_page() succeeded!
check_kern_pgdir() succeeded!
check_page_installed_pgdir() succeeded!
SMP: CPU 0 found 4 CPU(s)
enabled interrupts: 1 2
SMP: CPU 1 starting
SMP: CPU 2 starting
SMP: CPU 3 starting
```

#### Locking（锁）

我们当前的代码 mp_main()在初始化 AP 之后就开始自旋。在进一步之前，我们需要弄清楚多个 CPU 运行内核代码时的竞争条件。最简单的方式是使用 big kernel lock。big kernel lock 是单个全局的锁，当一个环境进入内核态的时候获取，当该环境退出内核态的时候释放。在这个模型中，用户态代码可以同时运行在多个 CPU 上，但是同一时刻不能超过一个用户环境运行在内核态上；任何想要进入内核态的环境都被迫等待。

kern/spinlock.h 声明了 big kernel lock，命名为 kernel_lock。它也提供了 lock_kernel()和 unlock_kernel()函数来获取和释放锁。你应该在这四个地方使用 big kernel lock：

-   i386_init()，在 BSP 唤醒其他 CPU 的之前获得锁。
-   mp_main()，初始化 AP 之后获取锁，接着调用 sched_yield()让 AP 开始运行环境。
-   trap()，用户态陷入内核态时获取锁。为了决定该陷入发生在用户态还是内核态，查看 tf_cs 的低位。
-   env_run()，切换到用户态时释放锁。不要太早也不要太晚做这件事，否则你会遇到竞争和死锁。

练习 5。应用上述的 big kernel lock，在适当的地方调用 lock_kernel() 和 unlock_kernel()。

如何测试你的锁是否之正常工作？此时还不行！但是你在下一个练习里可以实现调度器。

问题

1. 看似使用 big kernel lock 可以保证某一时刻只有一个 CPU 可以运行内核代码，那为什么我们需要给每个 CPU 都弄一个内核栈呢？在什么场景下，共享的内核栈会出错，哪怕是在 big kernel lock 保护下？

挑战！big kernel lock 简单易用。尽管如此，它消除了内核态的并行。大多数现代操作系统使用不同的锁来保护不同的部分，一个方法是 fine-grained locking。fine-grained locking 可以显著增加性能，但是它更难以实现，也很容易出错。如果你足够勇敢，丢弃 big kernel lock，让 JOS 拥抱并行。

由你决定锁的粒度（该锁要保护多少数据）。作为提示，你可能要考虑使用自旋锁来保证 JOS 内核里共享组件的排他访问：

-   The page allocator.
-   The console driver.
-   The scheduler.
-   The inter-process communication (IPC) state that you will implement in the part C.

### Round-Robin Scheduling（循环调度）

你的下一个任务是去改变 JO 内核，以便于它可以循环切换多环境。循环调度的运行如下：

-   kern/sched.c 里的 sched_yield()负责选取一个新的环境来运行。它按照环形顺序来搜索 envs[]数组，从刚刚运行的环境开始搜索起（或者刚刚如果没有运行的环境，那就从最开头开始开始搜索），选取第一个它发现的 ENV_RUNNABLE 状态的环境，然后调用 env_run()来跳进去执行这个环境。
-   sched_yield()永远不要在两个 CPU 上运行同一个环境。看环境的状态为 ENV_RUNNING 就知道它当前运行在一些 CPU 上（很有可能就是当前 CPU）。
-   我们已经为你实现了一个新的系统调用 sys_yield()，用户环境可以调用这个来调用内核的 sched_yield()函数，这样就自愿地放弃 CPU，让给其他环境。

练习 6。在 sched_yield()里实现如上所述的循环调度。不要忘记修改 syscall()来分发 sys_yield()系统调用。

确保在 mp_main 里调用了 sched_yield()。

修改 kern/init.c 来创建三个（甚至更多）的环境，让他们都运行程序 user/yield.c。

运行`make qemu`。你应该看到环境切换来切换去，五次之后终止运行，如下所示。

测试多 CPU：`make qemu CPUS=2`.

```
...
Hello, I am environment 00001000.
Hello, I am environment 00001001.
Hello, I am environment 00001002.
Back in environment 00001000, iteration 0.
Back in environment 00001001, iteration 0.
Back in environment 00001002, iteration 0.
Back in environment 00001000, iteration 1.
Back in environment 00001001, iteration 1.
Back in environment 00001002, iteration 1.
...
```

在 yield 程序退出后，系统里就没有可运行的环境了，调度器应该触发 JOS 内核监控器。如果没有发生，那就修改你的代码，之后再进行下一步。

问题

1. 在你的 env_run()实现中，你应该调用了 lcr3()。在调用 lcr3()前后，你的代码对变量 e 做了引用，也就是 env_run 的参数。在加载%cr3 寄存器之后，MMU 使用的上下文一下子被切换了。但是一个虚拟地址和地址上下文有联系——地址上下文指出虚拟地址要映射到哪个物理地址上。为什么指针 e 依旧可以在地址切换前后进行取值？
2. 不管内核从一个环境切换到另外一个环境，它必须确保老环境寄存器被保存起来了，这样它们就可以正确地恢复。为什么？这个在哪里发生的？

挑战!向内核添加一个不那么简单的调度策略，比如一个固定优先级的调度器，它允许为每个环境分配一个优先级，并确保总是选择高优先级的环境，而不是低优先级的环境。如果你真的想冒险，尝试实现一个 unix 风格的可调优先级调度程序，或者甚至是一个彩票或跨步调度程序。(在谷歌中查找"lottery scheduling"和"stride scheduling"。)

编写一两个测试程序来验证你的调度算法是否正确工作(即，正确的环境以正确的顺序运行)。在本实验室的 B 部分和 C 部分实现了 fork()和 IPC 之后，编写这些测试程序可能会更容易。

挑战!JOS 内核目前不允许应用程序使用 x86 处理器的 x87 浮点单元(FPU)、MMX 指令或流 SIMD 扩展(SSE)。扩展 Env 结构，为处理器的浮点状态提供一个保存区域，并扩展上下文切换代码，以便在从一个环境切换到另一个环境时正确地保存和恢复这个状态。FXSAVE 和 FXRSTOR 指令可能是有用的，但请注意，旧的 i386 用户手册中没有这些指令，因为它们是在最新的处理器中引入的。编写一个用户级的测试程序，用浮点做一些很酷的事情。

### System Calls for Environment Creation（进行环境创建的系统调用）

尽管你的内核现在能够在多个用户级环境之间运行和切换，但它仍然局限于内核最初设置的运行环境。现在，你将实现必要的 JOS 系统调用，以允许用户环境创建和启动其他新用户环境。

Unix 提供 fork()系统调用作为它的进程创建原语。Unix fork()复制调用进程(父进程)的整个地址空间来创建一个新进程(子进程)。用户空间中两个可观察对象之间的唯一区别是它们的进程 id 和父进程 id(由 getpid 和 getppid 返回)。在父进程中，fork()返回子进程的进程 ID，而在子进程中，fork()返回 0。默认情况下，每个进程都有自己的私有地址空间，两个进程对内存的修改都不可见。

你将提供一组不同的、更原始的 JOS 系统调用来创建新的用户模式环境。通过这些系统调用，你将能够完全在用户空间中实现类 unix 的 fork()，以及其他类型的环境创建。你将为 JOS 编写的新系统调用如下:

-   sys_exofork：
    这个系统创建一个新的环境，几乎空白的环境：没有地址空间的映射，不可运行。当调用该系统调用时，新环境会和父环境拥有相同的寄存器状态。在父环境中，sys_exofork 将返回新创建的环境的 envid_t(如果环境分配失败，则返回负的错误代码)。然而，在子进程中，它将返回 0。(由于子进程一开始被标记为不可运行，sys_exofork 实际上不会在子进程中返回，直到父进程使用....显式地将子进程标记为可运行。)
-   sys_env_set_status：
    设置指定环境的状态为 ENV_RUNNABLE 或 ENV_NOT_RUNNABLE。这个系统调用通常用于标记一个新环境准备运行，一旦它的地址空间和寄存器状态已经完全初始化。
-   sys_page_alloc:
    分配一页物理内存，并将其映射到给定环境的地址空间中的给定虚拟地址。
-   sys_page_map:
    将一个页面映射(而不是页面的内容!)从一个环境的地址空间复制到另一个环境，保留一个内存共享安排，以便新映射和旧映射都引用物理内存的同一页。
-   sys_page_unmap:
    取消在给定环境中给定虚拟地址映射的页的映射。

对于以上接受环境 id 的所有系统调用，JOS 内核支持这样的约定:值 0 表示“当前环境”。这个约定由 kern/env.c 中的 envid2env()实现。

我们已经在测试程序 user/dumbfork.c 中提供了一个类 unix 的 fork()的非常原始的实现。这个测试程序使用上面的系统调用来创建和运行带有自己地址空间副本的子环境。然后，这两个环境使用 sys_yield 在前面的练习中来回切换。父进程在迭代 10 次后退出，而子进程在迭代 20 次后退出。

练习 7。实现 kern/syscall.c 中描述的系统调用，并确保 syscall()调用它们。您将需要使用 kern/pmap.c 和 kern/env.c 中的各种函数，特别是 envid2env()。现在，无论何时调用 envid2env()，都要在 checkperm 参数中传递 1。确保您检查了任何无效的系统调用参数，在这种情况下返回-E_INVAL。使用 user/dumbfork 测试你的 JOS 内核，确保它能正常工作。

挑战!添加必要的附加系统调用，以读取现有环境的所有重要状态并设置它。然后实现一个用户模式程序，它分叉一个子环境，运行它一段时间(例如，sys_yield()的几个迭代)，然后获取子环境的完整快照或检查点，运行子环境一段时间，最后将子环境恢复到它在检查点时的状态，并从那里继续它。因此，您可以有效地从中间状态“重放”子环境的执行。使孩子环境执行一些与用户的交互使用 sys_cgetc()或 readline(),以便用户可以查看和改变其内部状态,与你的检查站,并验证/重启可以给孩子环境的选择性失忆,这使得“忘记”,超过某特定点上发生的一切。

这完成了实验室的 A 部分;当你运行 make grade 时，确保它通过所有的 A 部分测试，然后像往常一样使用 make handin 交上来。如果您试图找出为什么某个特定的测试用例失败，运行./grade-lab4 -v，它将显示内核构建的输出，并为每个测试运行 QEMU，直到一个测试失败。当测试失败时，脚本将停止，然后您可以检查 jos。出来看看内核实际打印了什么。

## Part B: Copy-on-Write Fork（写时复制 Fork）

如前所述，Unix 提供 fork()系统调用作为它的主要进程创建原语。fork()系统调用复制调用进程(父进程)的地址空间，以创建一个新进程(子进程)。

xv6 Unix 通过将所有来自父节点的数据复制到分配给子节点的新页面来实现 fork()。这基本上与 dumbfork()采用的方法相同。将父节点的地址空间复制到子节点是 fork()操作中开销最大的部分。

然而，在调用 fork()之后，经常会立即在子进程中调用 exec()，这将用一个新程序替换子进程的内存。例如，这就是 shell 通常做的事情。在这种情况下，花在复制父进程地址空间上的时间基本上是浪费的，因为子进程在调用 exec()之前只会使用很少的内存。

由于这个原因，Unix 的后续版本利用虚拟内存硬件，允许父进程和子进程共享映射到各自地址空间的内存，直到其中一个进程真正修改它。这种技术称为写时复制。要做到这一点，在 fork()上，内核会将地址空间映射从父节点复制到子节点，而不是映射页面的内容，同时将现在共享的页面标记为只读。当两个进程中的一个试图写入这些共享页面时，该进程将出现页面错误。在这一点上，Unix 内核意识到页面实际上是一个“虚拟”或“写时复制”副本，因此它为出现故障的进程创建了一个新的、私有的、可写的页面副本。通过这种方式，在实际写入各个页面之前，不会实际复制各个页面的内容。这种优化使得子进程中 fork()后跟 exec()的代价更低:子进程在调用 exec()之前可能只需要复制一页(堆栈的当前页)。

在本实验的下一部分中，您将实现一个“适当的”类 unix 的带有 copy-on-write 的 fork()，作为一个用户空间库例程。在用户空间中实现 fork()和 copy-on-write 支持的好处是，内核仍然更简单，因此更有可能是正确的。它还允许单独的用户模式程序为 fork()定义它们自己的语义。如果一个程序需要一个稍微不同的实现(例如，像 dumbfork()这样代价昂贵的总是复制版本，或者父和子实际上在之后共享内存的版本)，那么它可以很容易地提供自己的实现。

### User-level page fault handling（用户级页错误处理）

用户级 write-copy-fork()需要知道写保护页面上的页面错误，所以这是您首先要实现的。写时复制只是用户级页面错误处理的许多可能用途之一。

设置一个地址空间是很常见的，这样页面错误就可以指示什么时候需要执行某些操作。例如，大多数 Unix 内核最初只映射一个新进程的堆栈区域中的单个页面，然后随着进程的堆栈消耗增加，“按需”分配和映射额外的堆栈页面，并导致尚未映射的堆栈地址的页面错误。典型的 Unix 内核必须跟踪在进程空间的每个区域发生页面错误时应该采取的操作。例如，堆栈区域中的故障通常会分配和映射物理内存的新页。程序的 BSS 区域中的错误通常会分配一个新页面，用零填充它，并映射它。在具有按需分页可执行文件的系统中，文本区域中的错误将从磁盘中读取相应的二进制文件页，然后映射它。

这是内核需要跟踪的大量信息。您将决定如何处理用户空间中的每个页面错误，而不是采用传统的 Unix 方法，在用户空间中，错误的破坏性较小。这种设计还有一个额外的好处，即允许程序在定义它们的内存区域时具有很大的灵活性;稍后，您将使用用户级页面错误处理来映射和访问基于磁盘的文件系统上的文件。

#### Setting the Page Fault Handler（设置页错误处理器）

为了处理自己的页面错误，一个用户环境需要注册一个页错误处理器端点到内核里。用户环境通过新的 sys_env_set_pgfault_upcall 系统调用来注册这个页错误处理端点。我们已经在 Env 结构体里添加了一个新成员，env_pgfault_upcall 来记录这个信息。

练习 8。实现 sys_env_set_pgfault_upcall 函数调用。确保在查询目标环境 id 的时候做了权限检查，因为这是一个“危险”的系统调用。

#### Normal and Exception Stacks in User Environments（用户环境的正常栈和异常栈）

当正常执行士，JOS 里的一个用户环境会运行在正常的用户栈上：它的 ESP 寄存器开始指向 USTACKTOP，它推送的数据驻留在 USTACKTOP-PGSIZE 和 USTACKTOP-1 之间。 当一个页错误出现在用户态时，然而，内核会重启用户环境去运行一个委托的用户级页错误处理程序在一个不同的栈上，这个栈被叫做用户异常栈。本质上，我们将让 JOS 内核实现代表用户环境的自动“堆栈切换”，就像 x86 处理器在从用户模式转换到内核模式时已经代表 JOS 实现了堆栈切换一样!

JOS 用户异常栈也是只有一页大小，它的顶部定义在虚拟地址的 UXSTACKTOP，所以确定的用户异常栈在 UXSTACKTOP-PGSIZE 到 UXSTACKTOP-1 之间。当运行在这个异常栈时，这个用户级页错误处理程序可以使用 JOS 的常规系统调用来映射新的页或者调整映射来恢复页面错误导致的任何问题。之后这个用户级页错误处理程序通过一个汇编 stub 返回错误码到原始栈上。

每一个想要支持用户级页错误处理的用户环境，都需要为自己的异常栈申请内存，使用 A 部分的 sys_page_alloc()系统调用。

#### Invoking the User Page Fault Handler（触发用户级页错误处理程序）

你现在需要去修改 kern/trap.c 里的页错误处理代码，来处理来自用户态的页错误。我们将用户环境发生错误时的状态称为 trap-time 状态。

如果没有事先注册好的页错误处理程序，JOS 内核就会像之前一样，破坏掉该用户环境，发出一个 message。否则，内核会在异常栈上建立一个 trap frame，看起来像是来自 inc/trap.h 的 UTrapframe 结构体。

```
                    <-- UXSTACKTOP
trap-time esp
trap-time eflags
trap-time eip
trap-time eax       start of struct PushRegs
trap-time ecx
trap-time edx
trap-time ebx
trap-time esp
trap-time ebp
trap-time esi
trap-time edi       end of struct PushRegs
tf_err (error code)
fault_va            <-- %esp when handler is run
```

内核通过构建这个栈帧来帮助恢复用户环境的运行；你必须弄清楚这个是如何发生的。fault_va 是导致页错误时的虚拟地址。

如果当一个错误出现时，用户环境已经运行在用户异常栈了，然后页错误处理程序它本身发送错误了。在这种情况下，你应该开始一个新的栈帧，就在当前的 tf->tf_esp 之下，而不是 UXSTACKTOP。你应该开始推入一个空的 32 位字，然后推入一个 UTrapframe 结构体。

要是想测试是否 tf->tf_esp 已经在用户异常栈上，那就查看它是否在 UXSTACKTOP-PGSIZE 和 UXSTACKTOP-1 之间。

练习 9。实现 kern/trap.c 里 page_fault_handler 函数的代码，这个函数在分发页错误的时候被用户级处理程序所需要。在写入异常堆栈时，请确保采取适当的预防措施。（如果用户环境在异常栈上用完了内存空间会怎么样？）

#### User-mode Page Fault Entrypoint（用户态页错误端点）

接着，你需要去实现一个汇编例程，它要负责调用 C 语言写的页错误处理程序以及恢复原始指令的执行。这个汇编例程会用 sys_env_set_pgfault_upcall()把它注册到内核里。

练习 10。采用 实现 lib/pfentry.S 里的\_pgfault_upcall 例程。有趣的部分是返回到原来产生页错误的用户代码处。你会直接回到那儿，不需要经过内核。难点在于同时切换栈和重新加载 EIP。

最终，你需要实现相关的 C 语言用户库。

练习 11。完成 lib/pgfault.c 里的 set_pgfault_handler()。

#### Testing（测试）

运行 user/faultread（make run-faultread）。你应该看到：

```
...
[00000000] new env 00001000
[00001000] user fault va 00000000 ip 0080003a
TRAP frame ...
[00001000] free env 00001000
```

Run user/faultdie. You should see:

```
...
[00000000] new env 00001000
i faulted at va deadbeef, err 6
[00001000] exiting gracefully
[00001000] free env 00001000

```

运行 user/faultalloc。你应该看到：

```
...
[00000000] new env 00001000
fault deadbeef
this string was faulted in at deadbeef
fault cafebffe
fault cafec000
this string was faulted in at cafebffe
[00001000] exiting gracefully
[00001000] free env 00001000
```

如果只看到"this string"这一行，这意味着您没有正确地处理递归页面错误。

运行 user/faultallocbad。你应该看到：

```
...
[00000000] new env 00001000
[00001000] user_mem_check assertion failure for va deadbeef
[00001000] free env 00001000
```

确保你理解了为什么 user/faultalloc 和 user/faultallocbad 的是不同的。

挑战！扩展你的内核，不仅限于页错误，而且还处理所有用户会产生的错误，可以重定向到用户态异常处理程序。写一些用户态的测试程序来测试用户态的不同错误，比如说除零错误，通用保护错误和非法操作码错误。

### Implementing Copy-on-Write Fork（实现写时复制 Fork）

现在，您拥有了完全在用户空间中实现 copy-on-write fork()的内核工具。

我们已经给你提供了一个 fork()的骨架在 lib/fork.c 文件里。像 dumbfork()，fork()应该创建一个新环境，然后扫描父环境的整个地址空间以及创建子环境对应的页映射。关键的不同是，dumbfork()复制页，而 fork()最初只会复制页映射。fork()只有当某个环境尝试写入时才会复制各个页。

fork()的基本控制流如下：

1. 父环境注册 pgfault()为 C 语言级别的页面错误处理程序，就用你之前实现的 set_pgfault_handler()函数来注册。
2. 父环境调用 sys_exofork()来创建一个子环境。
3. 对于它在 UTOP 之下地址空间里每一个可写或写时复制的页，父环境会调用 duppage，应该会映射该页到子环境的地址空间，然后重新映射自己地址空间的该页。【注意：这里的顺序是很重要的（为子环境标记一个页为 COW 要在为父环境标记之前）。你能明白吗？尝试想一个具体的调转顺序会发生问题的情况。】duppage 会在两个过程中都设置 PTE 位，这样每个页都是不可写的，“avail”字段里的 PTE_COW 用来标记写时复制页和真正的只读页。
   然而，异常栈不会通过这种方式重映射。而是，你需要在子环境里为异常栈申请一个新的页。因为页错误处理函数会做真正的复制，页错误处理函数也会运行在异常栈上，异常栈不能写时复制：谁会复制？

    fork()也需要掌管当前的页，但不是可写的或写时复制。

4. 父环境为子环境设置用户页错误处理端点，让子环境看起来这个就像是它自己设置的一样。
5. 子环境现在准备好运行了，所以父环境会标记它为 runnable。

每次某个环境写入一个“写时复制”页时，它会产生一个页错误。如下是用户页错误处理程序的控制流：

1. 内核将页面错误传播到 \_pgfault_upcall，它会调用 fork()的 pgfault()处理函数。
2. pgfault()查看这个错误是由一个写操作（查看错误码是不是 FEC_WR）导致的，以及页的 PTE 被标记为 PTE_COW。如果不是，panic。
3. pgfault()申请一个新的页，这个页映射到一个暂时地址，然后复制错误页的内容进去。然后错误处理程序映射一个新的页到合适的地址，设置好读写权限，以此代替旧的只读映射。

用户级别的 lib/fork.c 代码必须查看环境的页表来执行上面的几个操作(例如，页面的 PTE 标记为 PTE_COW)。内核在 UVPT 上映射环境的页表正是为了这个目的。它使用了一个[聪明的映射技巧](https://pdos.csail.mit.edu/6.828/2018/labs/lab4/uvpt.html)，使其更容易查找用户代码的 PTE。lib/entry.S 设置 uvpt 和 uvpd，这样你才可以很容易地在 lib/fork.c 里查找页表信息。

Exercise 12. Implement fork, duppage and pgfault in lib/fork.c.

Test your code with the forktree program. It should produce the following messages, with interspersed 'new env', 'free env', and 'exiting gracefully' messages. The messages may not appear in this order, and the environment IDs may be different.

```
	1000: I am ''
	1001: I am '0'
	2000: I am '00'
	2001: I am '000'
	1002: I am '1'
	3000: I am '11'
	3001: I am '10'
	4000: I am '100'
	1003: I am '01'
	5000: I am '010'
	4001: I am '011'
	2002: I am '110'
	1004: I am '001'
	1005: I am '111'
	1006: I am '101'
```

Challenge! Implement a shared-memory fork() called sfork(). This version should have the parent and child share all their memory pages (so writes in one environment appear in the other) except for pages in the stack area, which should be treated in the usual copy-on-write manner. Modify user/forktree.c to use sfork() instead of regular fork(). Also, once you have finished implementing IPC in part C, use your sfork() to run user/pingpongs. You will have to find a new way to provide the functionality of the global thisenv pointer.

Challenge! Your implementation of fork makes a huge number of system calls. On the x86, switching into the kernel using interrupts has non-trivial cost. Augment the system call interface so that it is possible to send a batch of system calls at once. Then change fork to use this interface.

How much faster is your new fork?

You can answer this (roughly) by using analytical arguments to estimate how much of an improvement batching system calls will make to the performance of your fork: How expensive is an int 0x30 instruction? How many times do you execute int 0x30 in your fork? Is accessing the TSS stack switch also expensive? And so on...

Alternatively, you can boot your kernel on real hardware and really benchmark your code. See the RDTSC (read time-stamp counter) instruction, defined in the IA32 manual, which counts the number of clock cycles that have elapsed since the last processor reset. QEMU doesn't emulate this instruction faithfully (it can either count the number of virtual instructions executed or use the host TSC, neither of which reflects the number of cycles a real CPU would require).

This ends part B. Make sure you pass all of the Part B tests when you run make grade. As usual, you can hand in your submission with make handin.

## Part C: Preemptive Multitasking and Inter-Process communication (IPC)

In the final part of lab 4 you will modify the kernel to preempt uncooperative environments and to allow environments to pass messages to each other explicitly.

### Clock Interrupts and Preemption

Run the user/spin test program. This test program forks off a child environment, which simply spins forever in a tight loop once it receives control of the CPU. Neither the parent environment nor the kernel ever regains the CPU. This is obviously not an ideal situation in terms of protecting the system from bugs or malicious code in user-mode environments, because any user-mode environment can bring the whole system to a halt simply by getting into an infinite loop and never giving back the CPU. In order to allow the kernel to preempt a running environment, forcefully retaking control of the CPU from it, we must extend the JOS kernel to support external hardware interrupts from the clock hardware.

#### Interrupt discipline

External interrupts (i.e., device interrupts) are referred to as IRQs. There are 16 possible IRQs, numbered 0 through 15. The mapping from IRQ number to IDT entry is not fixed. pic_init in picirq.c maps IRQs 0-15 to IDT entries IRQ_OFFSET through IRQ_OFFSET+15.

In inc/trap.h, IRQ_OFFSET is defined to be decimal 32. Thus the IDT entries 32-47 correspond to the IRQs 0-15. For example, the clock interrupt is IRQ 0. Thus, IDT[IRQ_OFFSET+0] (i.e., IDT[32]) contains the address of the clock's interrupt handler routine in the kernel. This IRQ_OFFSET is chosen so that the device interrupts do not overlap with the processor exceptions, which could obviously cause confusion. (In fact, in the early days of PCs running MS-DOS, the IRQ_OFFSET effectively was zero, which indeed caused massive confusion between handling hardware interrupts and handling processor exceptions!)

In JOS, we make a key simplification compared to xv6 Unix. External device interrupts are always disabled when in the kernel (and, like xv6, enabled when in user space). External interrupts are controlled by the FL_IF flag bit of the %eflags register (see inc/mmu.h). When this bit is set, external interrupts are enabled. While the bit can be modified in several ways, because of our simplification, we will handle it solely through the process of saving and restoring %eflags register as we enter and leave user mode.

You will have to ensure that the FL_IF flag is set in user environments when they run so that when an interrupt arrives, it gets passed through to the processor and handled by your interrupt code. Otherwise, interrupts are masked, or ignored until interrupts are re-enabled. We masked interrupts with the very first instruction of the bootloader, and so far we have never gotten around to re-enabling them.

Exercise 13. Modify kern/trapentry.S and kern/trap.c to initialize the appropriate entries in the IDT and provide handlers for IRQs 0 through 15. Then modify the code in env_alloc() in kern/env.c to ensure that user environments are always run with interrupts enabled.

Also uncomment the sti instruction in sched_halt() so that idle CPUs unmask interrupts.

The processor never pushes an error code when invoking a hardware interrupt handler. You might want to re-read section 9.2 of the [80386 Reference Manual](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm), or section 5.8 of the [IA-32 Intel Architecture Software Developer's Manual, Volume 3](https://pdos.csail.mit.edu/6.828/2018/readings/ia32/IA32-3A.pdf), at this time.

After doing this exercise, if you run your kernel with any test program that runs for a non-trivial length of time (e.g., spin), you should see the kernel print trap frames for hardware interrupts. While interrupts are now enabled in the processor, JOS isn't yet handling them, so you should see it misattribute each interrupt to the currently running user environment and destroy it. Eventually it should run out of environments to destroy and drop into the monitor.

### Handling Clock Interrupts

In the user/spin program, after the child environment was first run, it just spun in a loop, and the kernel never got control back. We need to program the hardware to generate clock interrupts periodically, which will force control back to the kernel where we can switch control to a different user environment.

The calls to lapic_init and pic_init (from i386_init in init.c), which we have written for you, set up the clock and the interrupt controller to generate interrupts. You now need to write the code to handle these interrupts.

Exercise 14. Modify the kernel's trap_dispatch() function so that it calls sched_yield() to find and run a different environment whenever a clock interrupt takes place.

You should now be able to get the user/spin test to work: the parent environment should fork off the child, sys_yield() to it a couple times but in each case regain control of the CPU after one time slice, and finally kill the child environment and terminate gracefully.

This is a great time to do some regression testing. Make sure that you haven't broken any earlier part of that lab that used to work (e.g. forktree) by enabling interrupts. Also, try running with multiple CPUs using make CPUS=2 target. You should also be able to pass stresssched now. Run make grade to see for sure. You should now get a total score of 65/80 points on this lab.

### Inter-Process communication (IPC)

(Technically in JOS this is "inter-environment communication" or "IEC", but everyone else calls it IPC, so we'll use the standard term.)

We've been focusing on the isolation aspects of the operating system, the ways it provides the illusion that each program has a machine all to itself. Another important service of an operating system is to allow programs to communicate with each other when they want to. It can be quite powerful to let programs interact with other programs. The Unix pipe model is the canonical example.

There are many models for interprocess communication. Even today there are still debates about which models are best. We won't get into that debate. Instead, we'll implement a simple IPC mechanism and then try it out.

#### IPC in JOS

You will implement a few additional JOS kernel system calls that collectively provide a simple interprocess communication mechanism. You will implement two system calls, sys_ipc_recv and sys_ipc_try_send. Then you will implement two library wrappers ipc_recv and ipc_send.

The "messages" that user environments can send to each other using JOS's IPC mechanism consist of two components: a single 32-bit value, and optionally a single page mapping. Allowing environments to pass page mappings in messages provides an efficient way to transfer more data than will fit into a single 32-bit integer, and also allows environments to set up shared memory arrangements easily.

#### Sending and Receiving Messages

To receive a message, an environment calls sys_ipc_recv. This system call de-schedules the current environment and does not run it again until a message has been received. When an environment is waiting to receive a message, any other environment can send it a message - not just a particular environment, and not just environments that have a parent/child arrangement with the receiving environment. In other words, the permission checking that you implemented in Part A will not apply to IPC, because the IPC system calls are carefully designed so as to be "safe": an environment cannot cause another environment to malfunction simply by sending it messages (unless the target environment is also buggy).

To try to send a value, an environment calls sys_ipc_try_send with both the receiver's environment id and the value to be sent. If the named environment is actually receiving (it has called sys_ipc_recv and not gotten a value yet), then the send delivers the message and returns 0. Otherwise the send returns -E_IPC_NOT_RECV to indicate that the target environment is not currently expecting to receive a value.

A library function ipc_recv in user space will take care of calling sys_ipc_recv and then looking up the information about the received values in the current environment's struct Env.

Similarly, a library function ipc_send will take care of repeatedly calling sys_ipc_try_send until the send succeeds.

#### Transferring Pages

When an environment calls sys_ipc_recv with a valid dstva parameter (below UTOP), the environment is stating that it is willing to receive a page mapping. If the sender sends a page, then that page should be mapped at dstva in the receiver's address space. If the receiver already had a page mapped at dstva, then that previous page is unmapped.

When an environment calls sys_ipc_try_send with a valid srcva (below UTOP), it means the sender wants to send the page currently mapped at srcva to the receiver, with permissions perm. After a successful IPC, the sender keeps its original mapping for the page at srcva in its address space, but the receiver also obtains a mapping for this same physical page at the dstva originally specified by the receiver, in the receiver's address space. As a result this page becomes shared between the sender and receiver.

If either the sender or the receiver does not indicate that a page should be transferred, then no page is transferred. After any IPC the kernel sets the new field env_ipc_perm in the receiver's Env structure to the permissions of the page received, or zero if no page was received.

#### Implementing IPC

Exercise 15. Implement sys_ipc_recv and sys_ipc_try_send in kern/syscall.c. Read the comments on both before implementing them, since they have to work together. When you call envid2env in these routines, you should set the checkperm flag to 0, meaning that any environment is allowed to send IPC messages to any other environment, and the kernel does no special permission checking other than verifying that the target envid is valid.

Then implement the ipc_recv and ipc_send functions in lib/ipc.c.

Use the user/pingpong and user/primes functions to test your IPC mechanism. user/primes will generate for each prime number a new environment until JOS runs out of environments. You might find it interesting to read user/primes.c to see all the forking and IPC going on behind the scenes.

Challenge! Why does ipc_send have to loop? Change the system call interface so it doesn't have to. Make sure you can handle multiple environments trying to send to one environment at the same time.

Challenge! The prime sieve is only one neat use of message passing between a large number of concurrent programs. Read C. A. R. Hoare, ``Communicating Sequential Processes,'' Communications of the ACM 21(8) (August 1978), 666-667, and implement the matrix multiplication example.

Challenge! One of the most impressive examples of the power of message passing is Doug McIlroy's power series calculator, described in [M. Douglas McIlroy, ``Squinting at Power Series,'' Software--Practice and Experience, 20(7) (July 1990), 661-683](https://swtch.com/~rsc/thread/squint.pdf). Implement his power series calculator and compute the power series for sin(x+x^3).

Challenge! Make JOS's IPC mechanism more efficient by applying some of the techniques from Liedtke's paper, [Improving IPC by Kernel Design](http://dl.acm.org/citation.cfm?id=168633), or any other tricks you may think of. Feel free to modify the kernel's system call API for this purpose, as long as your code is backwards compatible with what our grading scripts expect.

This ends part C. Make sure you pass all of the make grade tests and don't forget to write up your answers to the questions and a description of your challenge exercise solution in answers-lab4.txt.

Before handing in, use git status and git diff to examine your changes and don't forget to git add answers-lab4.txt. When you're ready, commit your changes with git commit -am 'my solutions to lab 4', then make handin and follow the directions.
