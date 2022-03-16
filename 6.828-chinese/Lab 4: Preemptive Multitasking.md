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

#### 用户环境的 normal stack and exception stack

在正常执行的过程中，JOS 的用户环境将会在正常的用户堆栈上运行，它的 ESP 寄存器开始指向 USTACKTOP，它入栈的数据所在的页处于 USTACKTOP-PGSIZE 和 USTACKTOP-1 之内。

JOS 的用户异常栈也是一页大小。当发送页错误的时候，页错误处理程序可以利用异常栈，来处理页错误。

#### 触发用户页错误处理

你需要改变`kern/trap.c`里的页错误处理代码去处理来自用户态的页错误。我们会把错误处理时的用户环境的状态叫做 trap-time 状态。

如果没有页错误处理注册，JOS 内核会 destroy 掉用户环境，并附上一个 message。否则，内核会在异常栈里创建一个 trap frame，看起来就像是`inc/trap.h`里的 `UTrapframe` 结构体。

                    <-- UXSTACKTOP

trap-time esp
trap-time eflags
trap-time eip
trap-time eax start of struct PushRegs
trap-time ecx
trap-time edx
trap-time ebx
trap-time esp
trap-time ebp
trap-time esi
trap-time edi end of struct PushRegs
tf_err (error code)
fault_va <-- %esp when handler is run

随着页异常处理程序运行在异常栈上，内核会整理用户环境以使得其恢复执行，你必须弄清楚这个过程是如何发生的。`fault_va`是产生页错误的虚拟地址。

如果在执行页错误处理代码的时候，它本身也发生了错误，在这种情况下，你应该在当前的 esp 下创建一个栈帧，你应该推入一个空白的 32 位，再推入一个 UTrapframe 结构。

如果要测试`tf->tf_esp`是否在用户异常栈，只要测试它是否处于 `UXSTACKTOP-PGSIZE` 之间 `UXSTACKTOP-1` 即可。

练习 9：实现`kern/trap.c`里的函数`page_fault_handler`。如果用户环境异常栈没有空间了怎么办？记得做好处理措施。

#### 用户态页错误入口点

接下来，你需要实现调用 C 处理程序和恢复程序的汇编例程，这个汇编例程会通过 `sys_env_set_pgfault_upcall()`注册到内核。

练习 10：实现`lib/pfentry.S`里的`_pgfault_upcall`例程。有趣的部分在于返回出现页异常的用户态代码处。你可以直接返回，不需要通过内核。难点在于同时切换栈和重新加载 eip。

最终，你需要实现用户级别的页错误处理机制，这是一个用户库。

练习 11：完成`lib/pgfault.c`里的`set_pgfault_handler()`。

#### 实现 Copy-on-Write Fork

好了，现在我们有足够的基础功能来实现`copy-on-write fork()`了。

我们提供了 fork()函数的骨架，在这个函数里，应该创建一个用户环境，然后扫描父用户环境的整个地址空间，然后给子用户环境设置相应的内存映射。dumbfork()和 fork()最关键的不同是前者拷贝整个内存而后者只是拷贝内存映射。fork()只有当某个子用户环境尝试写入的时候。

fork()的流程如下：

1. parent 调用 set_pgfault_handler()去设置好页错误处理函数。
2. parent 调用 sys_exofork()去创建 child 。
3. parent 调用 duppage()来为 child 建立内存映射。标记 PTE_COW。
4. parent 为 child 设置页错误入口点函数。
5. child 现在可以准备运行了，所以 parent 将其标记为 runable。

每当一个 child 想要写入一个页时，发生页错误，下面是页错误的处理过程：

1. 内核传递页错误到`_pgfault_upcall`，它会调用 fork()'s pgfault() handler。
2. pgfault()检查页错误是 write，并且检查目标页的 PTE 被标记为 PTE_COW。
3. pgfault()接着申请一个新页，然后把把 child 想要修改的地址的虚拟地址映射到这个页上，PTE 标记为合适的权限。

练习 12：Implement fork, duppage and pgfault in lib/fork.c.

### C Preemptive Multitasking and Inter-Process communication (IPC)

#### 时钟中断和预先制止

运行 `user/spin` 测试程序。这个测试程序 fork 了一个子用户环境，这个子用户环境一旦获得了 CPU 的控制权，它就通过一个简单的循环来自旋。用户环境和内核都不能重新控制 CPU。这显然在保护系统方面有一些问题，因为一个子用户环境可以通过无限的循环来控制住 CPU，导致整个系统执行中断。所以我们必须要实现时钟中断切换用户环境的功能。

##### 中断规则

外部中断是和 IRQ 们联系在一起的。总共有 16 个 IRQ，0 号到 15 号。IRQ 号码和 IDT 项的映射不是固定的。 `picirq.c` 里的 `pic_initmaps` 对其建立了相应的线性映射。

在`inc/trap.h`里，IRQ_OFFSET 被设置为十进制的 32。因此，IDT 32-47 相应地映射了 IRQ 0-15。例如，时钟中断是 IRQ 0。因此，IDT[IRQ_OFFSET+0]为内核里时钟中断处理例程的地址。选择这个 IRQ_OFFSET 是为了使设备中断不会与处理器异常重叠，否则很容易引起混淆。(事实上，在运行 MS-DOS 的早期 pc 中，IRQ_OFFSET 实际上是零，这确实在处理硬件中断和处理处理器异常之间造成了巨大的混淆!)

在 JOS 与 xv6 Unix 相比做了一些关键的简化。在内核中总是禁用外部设备中断(和 xv6 一样，在用户空间中启用)。外部中断由%eflags 寄存器的 FL_IF 标志位控制(参见 inc/mmu.h)。设置此位时，将启用外部中断。虽然可以通过几种方式修改该位，但我们做了简化，我们将在进入和离开用户态时仅通过保存和恢复%eflags 寄存器来处理它。

你必须确保在用户环境中运行时设置了 FL_IF 标志，以便当一个中断到达时，它被传递到处理器并由你的中断代码处理。否则，中断将被屏蔽或忽略，直到重新启用中断。我们用引导加载程序的第一个指令屏蔽了中断，到目前为止，我们还没有重新启用它。

练习 13：修改`kern/trapentry.S`和`kern/trap.c`去初始化合适的 IDT 条目，然后位 IRQ 0-15 提供处理程序。然后修改`kern/env.c`里的`env_alloc()`来保证用户环境运行的时候总是把中断打开了。

另外，在 sched_halt()中取消注释 sti 指令，以便空闲的 cpu 可以解除屏蔽中断。

当调用硬件中断处理程序时，处理器从不推入错误代码。此时，您可能需要重新阅读 80386 参考手册 9.2 节，或 IA-32 Intel 架构软件开发人员手册第 3 卷 5.8 节。

在做了这个练习之后，如果你用任何运行了一段时间的测试程序运行你的内核(例如 spin)，你应该会看到内核打印硬件中断的陷阱帧。虽然现在在处理器中启用了中断，但 JOS 还没有处理它们，所以您应该看到它将每个中断错误地归为当前运行的用户环境并销毁该用户环境，最终它将把所有的用户环境都销毁掉。

##### 处理时钟中断

在`user/spin`程序里，子用户环境启动后，它开始自旋。我们让硬件定时发送时钟中断，然后把 CPU 的控制权转交给内核，内核再去执行调度任务。

我们已经为你写好了`lapic_init`和`pic_init` (from i386_init in init.c),调用这两个函数来创建时钟和中断管理器来生成中断。你现在需要写处理这些中断的代码。

练习 14：Modify the kernel's trap_dispatch() function so that it calls sched_yield() to find and run a different environment whenever a clock interrupt takes place.

You should now be able to get the user/spin test to work: the parent environment should fork off the child, sys_yield() to it a couple times but in each case regain control of the CPU after one time slice, and finally kill the child environment and terminate gracefully.

#### IPC

(技术上来说 JOS 里的应该叫做“用户环境内通信”，但是大家都叫做进程间通信，那么我们就用这个标准术语。)

我们已经专注于操作系统的“隔离”方面，这看起来就像是每个进程都占有整个机器。另外一个操作系统提供的很重要的服务就是允许进程间通信。这是一个让程序间交互的很强大的功能。Unix 的管道模型就是一个典型的例子。

现在有许多进程间通信模型。即使是现在也有很多关于哪种 IPC 是最好的争议。我们不会去争论这个，而是实现尝试实现了一个简单的 IPC 机制。

##### IPC in JOS

你要实现几个额外的 JOS 内核系统调用，合起来提供简单的进程间通信机制。你会实现两个系统调用`sys_ipc_recv`和`sys_ipc_try_send`。然后你要实现两个库函数`ipc_recv`和`ipc_send`。

在 JOS 的 IPC 机制中，用户环境发送给另外一个用户环境的消息包含两个部分：一个 32 位的值以及可选的单个页的映射。允许用户环境在消息里传递页映射是传递更多数据的一种高效方式，允许用户环境之间共享内存。

##### 发送和接受消息

为了接收消息，一个用户环境调用`sys_ipc_recv`。这个系统调用重新调度当前的用户环境，然后停止运行该用户环境直到一个消息被接收到。当一个环境等着接收消息的时候，其他任何用户环境都可以给它发送消息，不只是一个特定的用户环境，也不只是有父子关系的用户环境。换句话说，我们在 Part A 实现的权限检查并不应用到这个 IPC 上，因为 IPC 系统调用到设计特别小心来保证它是安全的：一个用户环境无法只是通过发送它的消息就导致另外一个用户环境发生故障（除非目标用户环境本身就有 bug）。

为了尝试发送一个值，一个用户环境调用`sys_ipc_try_send`，同时带上目标用户环境的 id 和想要发送的值。如果一个用户环境正在接收（调用`sys_ipc_recv`），那么发送者传递消息并返回 0。否则发送者返回`-E_IPC_NOT_RECV`来表示目标用户环境当前并没有在等待接收值。

给用户提供的库函数`ipc_recv`会调用`sys_ipc_revc`，然后到当前用户环境的 Env 结构中去查找接收到的值。

同样的，库函数`ipc_send`也会重复调用`sys_ipc_try_send`直到发送成功。

##### Transferring Pages

当一个用户环境带着一个可用的 dstva 参数（在 UTOP 之下）调用`sys_ipc_recv`时，表明用户环境将会收到一个页面映射。如果发送者发送了一个页，那么这个页应该被映射到接收者地址空间的 dstva。如果接收者已经有一个页在 dstva 处有映射，那就把之前的映射给取消掉。

当一个用户环境带着一个可用的`srcva`参数（在 UTOP 之下）调用`sys_ipc_try_send`，这意味着发送者想要发送一个当前正映射在 srcva 的页给接收者，带着权限参数。IPC 成功之后，发送者保持它原来的映射在 srcva 处的页在它的地址空间，但是接收者也能获取一个映射，映射到接收者提供的 dstva，在用户的地址空间。

最终，这个页由接收者和发送者共享。

如果发送者和接收者任何一方没有指示一个页需要被转移，那么就没有页会被转移。在任何 IPC 之后，内核会在接收者的 Env 数据结构的`env_ipc_perm`处设置新值，设为接收到的页的权限，如果没有接收到页那就设置为 0。

##### 实现 IPC

练习 15：实现`kern/syscall.c`里到`sys_ipc_recv`和`sys_ipc_try_send`。在实现它们之前，请通读它们的注视，因为这两个是一起工作的。当你在例程里调用 envid2env 时，你应该把 checkperm 位设置为 0，意味着任何用户环境都被允许发送 IPC 消息到任何用户环境，并且内核不做任何特别的权限检查，只是检查目标 envid 是不是正确的。

然后实现`lib/ipc.c`里的`ipc_recv`和`ipc_send`。

使用 user/pingpong 和 user/primes 函数去测试你的 IPC 机制。user/primes 会一直生成质数个用户环境直到 JOS 的用户环境用完。你可能会发现阅读 user/primes.c 很有趣。
