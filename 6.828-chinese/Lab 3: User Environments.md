# Lab 3: User Environments

## 介绍

在本实验室中，你将实现运行受保护的用户模式环境(即“进程”)所需的基本内核设施。你将增强 JOS 内核，以设置数据结构来跟踪用户环境、创建单个用户环境、将程序映像加载到其中并启动它。你还将使 JOS 内核能够处理用户环境发出的任何系统调用，并处理它引起的任何其他异常。

注意：在本实验室中，术语“环境”和“进程”是可以互换的——它们都是指允许你运行程序的抽象。我们引入术语“环境”而不是传统的术语“进程”，是为了强调 JOS 环境和 UNIX 进程提供不同的接口，并且不提供相同的语义。

### 开始

Use Git to commit your changes after your Lab 2 submission (if any), fetch the latest version of the course repository, and then create a local branch called lab3 based on our lab3 branch, origin/lab3:

```
athena% cd ~/6.828/lab
athena% add git
athena% git commit -am 'changes to lab2 after handin'
Created commit 734fab7: changes to lab2 after handin
 4 files changed, 42 insertions(+), 9 deletions(-)
athena% git pull
Already up-to-date.
athena% git checkout -b lab3 origin/lab3
Branch lab3 set up to track remote branch refs/remotes/origin/lab3.
Switched to a new branch "lab3"
athena% git merge lab2
Merge made by recursive.
 kern/pmap.c |   42 +++++++++++++++++++
 1 files changed, 42 insertions(+), 0 deletions(-)
athena%
```

Lab 3 contains a number of new source files, which you should browse:

```
inc/	env.h	    Public definitions for user-mode environments
        trap.h	    Public definitions for trap handling
        syscall.h	Public definitions for system calls from user environments to the kernel
        lib.h	    Public definitions for the user-mode support library
kern/	env.h	    Kernel-private definitions for user-mode environments
        env.c	    Kernel code implementing user-mode environments
        trap.h	    Kernel-private trap handling definitions
        trap.c	    Trap handling code
        trapentry.S	Assembly-language trap handler entry-points
        syscall.h	Kernel-private definitions for system call handling
        syscall.c	System call implementation code
lib/	Makefrag	Makefile fragment to build user-mode library, obj/lib/libjos.a
        entry.S	    Assembly-language entry-point for user environments
        libmain.c	User-mode library setup code called from entry.S
        syscall.c	User-mode system call stub functions
        console.c	User-mode implementations of putchar and getchar, providing console I/O
        exit.c	    User-mode implementation of exit
        panic.c	    User-mode implementation of panic
user/	*	        Various test programs to check kernel lab 3 code
```

In addition, a number of the source files we handed out for lab2 are modified in lab3. To see the differences, you can type:

```
$ git diff lab2
```

You may also want to take another look at the lab tools guide, as it includes information on debugging user code that becomes relevant in this lab.

### 实验要求

本实验课分为 A 和 B 两部分，A 部分在指定实验课一周后交；你应该在 A 部分的截止日期之前提交你的修改和交上你的实验室，确保你的代码通过了所有的 A 部分的测试(如果你的代码没有通过 B 部分的测试也没关系)。你只需要在第二周的截止日期前通过 B 部分的测试。

和实验 2 一样，你需要完成实验中所描述的所有常规练习和至少一个挑战性的问题(针对整个实验，而不是每个部分)。在你的实验室目录的最上面的一个名为 answers-lab3.txt 的文件中，写下你在实验中提出的问题的简短答案，并用一到两段文字描述你是如何解决你所选择的挑战性问题的。(如果你实施了多个挑战问题，你只需要在书面报告中描述一个即可。)不要忘记使用 `git add answers-lab3.txt` 将答案文件包含在你的提交中

### 内联汇编

在这个实验中，你可能会发现 GCC 的内联汇编语言特性很有用，尽管不使用它也可以完成这个实验。至少，你需要能够理解我们提供的源代码中已经存在的内联汇编语言片段(“asm”语句)。你可以在课程[参考资料页面](https://pdos.csail.mit.edu/6.828/2018/reference.html)上找到关于 GCC 内联汇编语言的几个信息来源。

## Part A: User Environments and Exception Handling

新的 include 文件 inc/env.h 包含了 JOS 中用户环境的基本定义。现在读它。内核使用 Env 数据结构来跟踪每个用户环境。在本实验室中，你将最初只创建一个环境，但你需要设计 JOS 内核来支持多个环境；Lab 4 将通过允许用户环境 fork（创建）其他环境来利用这个特性。

正如你在 kern/env.c 中看到的，内核维护了三个与环境相关的主要全局变量：

```c
struct Env *envs = NULL;		    // All environments
struct Env *curenv = NULL;		    // The current env
static struct Env *env_free_list;	// Free environment list
```

JOS 启动并运行后，envs 指针指向一个代表系统中所有环境的 Env 结构体的数组。在我们的设计中，JOS 内核将最多支持 NENV 个同时激活的环境，尽管在任何给定的时间内，运行的环境通常要少得多。(NENV 是一个在 inc/env.h 中定义的常量。)一旦它被分配，envs 数组将包含每个可能的 NENV 个环境的 Env 数据结构的单个实例。

JOS 内核将所有停止运行的 Env 结构保存在 env_free_list 中。这种设计使环境的分配和再分配变得容易，因为它们只需要添加到空闲列表中或从空闲列表中删除即可。

内核使用 curenv 符号在任何给定时间跟踪当前执行的环境。启动时，在运行第一个环境之前，curenv 最初被设置为 NULL。

### Environment State

Env 结构体被定义在 inc/env.h 里，如下所示（之后的实验里会添加更多的字段）：

```c
struct Env {
	struct Trapframe env_tf;	// Saved registers
	struct Env *env_link;		// Next free Env
	envid_t env_id;			    // Unique environment identifier
	envid_t env_parent_id;		// env_id of this env's parent
	enum EnvType env_type;		// Indicates special system environments
	unsigned env_status;		// Status of the environment
	uint32_t env_runs;		    // Number of times environment has run

	// Address space
	pde_t *env_pgdir;		    // Kernel virtual address of page dir
};
```

这里是`Env`的每个字段的解释：

```
env_tf:
    在 inc/trap.h 中定义的这个结构体，在该环境不运行时保存该环境的寄存器值，例如当内核或其他环境正在运行时。当从用户模式切换到内核模式时，内核会保存这些信息，以便以后环境可以从停止的地方恢复。
env_link:
    这是一个指向 env_free_list 中的下一个 Env 的指针。env_free_list 指向列表中的第一个的空闲环境。
env_id:
    内核在这里存储一个值，该值唯一标识当前使用这个Env结构的环境(即使用env数组中的这个特定槽位)。在用户环境终止后，内核可能会将相同的Env结构重新分配到不同的环境中——但是新的环境将拥有与旧环境不同的env_id，即使新环境重用了envs数组中的相同槽位。
env_parent_id:
    内核在这里存储创建该环境的环境（类似于父进程）的env_id。通过这种方式，环境可以形成一个“家族树”，这将有助于做出关于哪些环境允许对谁做什么事情的安全决策。
env_type:
    这个是用来区分特殊环境的。大多数环境都是ENV_TYPE_USER类型。在以后的实验中，我们将介绍更多用于特殊系统服务环境的类型。
env_status:
    该变量用来存放以下值之一：
    ENV_FREE:
        表示该Env结构体没有在运行，所以它在env_free_list列表里。
    ENV_RUNNABLE:
        表示该Env结构体为正在等待运行的环境。
    ENV_RUNNING:
        表示该Env结构体为当前的运行环境。
    ENV_NOT_RUNNABLE:
        表示该Env结构体为当前激活的环境，但它目前还没有准备好运行：例如，因为它正在等待来自另一个环境的进程间通信(IPC)。
    ENV_DYING:
        表示该Env结构体为一个僵尸环境。僵尸环境会在下一次陷入内核时被清除掉。我们在 Lab 4 之前都不会用到这个标志。
env_pgdir:
    这个变量保存了这个环境的页目录的内核虚拟地址。
```

像 Unix 进程一样，JOS 环境将“线程”和“地址空间”的概念结合在一起。线程主要由保存的寄存器(env_tf 字段)定义，地址空间由 env_pgdir 指向的页面目录和页面表定义。为了运行一个环境，内核必须使用保存的寄存器和适当的地址空间来设置 CPU。

我们的 struct Env 类似于 xv6 中的 struct proc。这两种结构都以 Trapframe 结构保存环境(即进程)的用户模式寄存器状态。在 JOS 中，各个环境不像 xv6 中的进程那样有自己的内核堆栈。在内核中，一次只能有一个活动的 JOS 环境，因此 JOS 只需要一个内核堆栈。

### 为环境数组申请内存

在实验室 2 中，你在 mem_init()中为 pages[]数组分配了内存，这是一个表，内核使用它来跟踪哪些页面是空闲的，哪些不是。现在需要进一步修改 mem_init()来分配一个类似的 Env 结构数组，称为 envs。

练习 1：修改 kern/pmap.c 中的 mem_init()来分配和映射 envs 数组。这个数组由 NENV 个 Env 结构体组成，分配内存的方式很像你给 pages 数组分配内存的方式。同样如 pages 数组一样，envs 的内存应该被映射到 UENVS (定义在 inc/memlayout.h 中)，并设为用户只读，这样用户进程可以从这个数组中读取数据。

你应该运行代码并确保 check_kern_pgdir()成功。

### 创建及运行环境

现在，你要到 kern/env.c 中编写运行用户环境所必需的代码。因为我们还没有文件系统，所以我们将设置内核来加载内核本身嵌入的静态二进制映像。JOS 将该二进制文件作为 ELF 可执行映像嵌入到内核中。

Lab 3 GNUmakefile 在 obj/user/ 目录中生成许多二进制镜像。如果你查看 kern/Makefrag，你会注意到一些神奇的东西，将这些二进制文件直接“链接”到内核可执行文件中，就好像它们是 .o 文件一样。链接器命令行上的 -b 二进制选项让这些文件被链接为“raw”未解释的二进制文件，而不是编译器生成的常规 .o 文件。(就链接器而言，这些文件根本不必是 ELF 镜像——它们可以是任何文件，例如文本文件或图片!)在构建完内核后，如果你查看 obj/kern/kernel.sym，你会注意到链接器“神奇地”生成了许多有趣的符号，它们的名字晦涩难懂，比如\_binary_obj_user_hello_start、\_binary_obj_user_hello_end 和\_binary_obj_user_hello_size。链接器通过修改二进制文件的文件名来生成这些符号名；这些符号为常规内核代码提供了一种引用嵌入二进制文件的方式。

在 kern/init.c 中的 i386_init()中，你将看到在环境中运行着这些二进制映像之一的代码。然而，创建用户环境的关键功能还不完整；你需要把它们填上。

练习 2：在文件 env.c 里，完成如下函数里的代码：

```
env_init()
    初始化 envs 数组里所有的 Env 结构体，并把它们添加到 env_free_list 上。也要调用 env_initpercpu，用不同特权级别的分段（level 0 为内核态，level 3 为用户态）来配置分段硬件。
env_setup_vm()
    为新用户环境申请一个页目录，并在内核里初始化新用户环境的地址空间。
region_alloc()
    为某个用户环境申请并映射物理内存。
load_icode()
    你需要解析 ELF 二进制镜像文件，就像 boot loader 所做的那样，并加载它的内容到一个新的环境的用户地址空间里。
env_create()
    用 env_alloc() 申请一个环境并调用 load_icode 来加载一个 ELF 二进制文件进去。
env_run()
    在用户模式下运行一个给定的环境。
```

在编写这些函数时，你可能会发现 cprintf 新的动词 %e 很有用——它输出与错误代码对应的描述。例如：

```c
r = -E_NO_MEM;
panic("env_alloc: %e", r);
```

会以消息“env_alloc: out of memory”产生 panic。

下面是到调用用户代码之前的函数调用图。确保你理解了每一步的目的。

-   start (kern/entry.S)
-   i386_init (kern/init.c)
    -   cons_init
    -   mem_init
    -   env_init
    -   trap_init (still incomplete at this point)
    -   env_create
    -   env_run
        -   env_pop_tf

一旦完成，你应该编译内核并在 QEMU 下运行它。如果一切正常，系统应该进入用户空间并执行 hello 二进制文件，直到它使用 int 指令进行系统调用。在这一点上将会有麻烦，因为 JOS 还没有设置硬件来允许从用户空间到内核的任何形式的转换。当 CPU 发现它还没有建立这个系统调用中断处理，它将生成一个通用保护异常，发现它无法处理这种错误，生成一个 double fault exception，然后发现它也无法处理错误，最后以所谓的“三重错误”而放弃错误处理。通常，你会看到 CPU 复位和系统重新启动。虽然这对于遗留应用程序很重要(关于原因的解释，请参阅[这篇博客文章](http://blogs.msdn.com/larryosterman/archive/2005/02/08/369243.aspx))，但这对内核开发来说是一件痛苦的事情，因此使用 QEMU 的 6.828 补丁版，你将看到一个寄存器转储和一个“Triple fault.”消息。

我们将很快定位这个问题，但是现在我们可以使用 debugger 来检查我们是否进入了用户模式。使用 make qemu-gdb 并在 env_pop_tf 上设置一个 GDB 断点，这应该是你实际进入用户模式之前执行的最后一个函数。通过 si 命令来单步调试这个函数；在 iret 指令执行之后，处理器应该进入了用户模式。然后，你应该在用户环境的可执行文件中看到第一个指令，即 lib/entry.S 中标签开始处的 cmpl 指令。现在使用 b \*0x… 命令在 hello 中的 sys_cputs()中 int $0x30 处设置断点(请参见 obj/user/hello.asm 的用户空间地址)。这个 int 是向控制台显示一个字符的系统调用。如果你不能执行到 int，那么你的地址空间设置或程序加载代码有问题；在继续下一步之前先修复好这些问题。

### 处理中断和异常

此时，用户空间中的第一个 int $0x30 系统调用指令是一个死胡同：一旦处理器进入用户模式，就没有办法返回了。现在，你需要实现基本的异常和系统调用处理，以便内核可以从用户模式代码恢复对处理器的控制。你应该做的第一件事是彻底熟悉 x86 中断和异常机制。

练习 3：如果你还没有准备好写代码，那么就读一下[80386 程序员手册](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm)里的[Chapter 9, Exceptions and Interrupts](https://pdos.csail.mit.edu/6.828/2018/readings/i386/c09.htm)(或第五章的[IA-32 开发者手册](https://pdos.csail.mit.edu/6.828/2018/readings/ia32/IA32-3A.pdf))。

在这个实验室中，我们通常遵循 Intel 关于中断、异常等的术语。然而，像 exception，trap，interrupt，fault 和 abort 这样的术语在体系结构或操作系统中没有标准的含义，并且经常在使用时不考虑它们在特定体系结构(如 x86)上的细微差别。当你在这个实验室之外看到这些术语时，它们的含义可能会略有不同。

Exercise 3. Read [Chapter 9, Exceptions and Interrupts](https://pdos.csail.mit.edu/6.828/2018/readings/i386/c09.htm) in the [80386 Programmer's Manual](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm) (or Chapter 5 of the [IA-32 Developer's Manual](https://pdos.csail.mit.edu/6.828/2018/readings/ia32/IA32-3A.pdf)), if you haven't already.

### Basics of Protected Control Transfer（保护控制转移的基础）

异常和中断都是“受保护的控制转移”，这导致处理器从用户模式切换到内核模式(CPL=0)，而不会给用户态代码提供任何干扰内核或其他环境功能的机会。在 Intel 的术语中，中断是一种受保护的控制转移，通常是由处理器外部的异步事件引起的，比如外部设备 I/O 活动的通知。有个例外正好相反，异常是由当前运行的代码同步引起的受保护的控制转移，例如由于除数为 0 或无效的内存访问而导致的异常。

为了确保这些受保护的控制传输实际上受到了保护，处理器的中断/异常机制被设计成这样，当中断或异常发生时，当前运行的代码不会任意选择进入内核的位置或方式。相反，处理器确保只有在严格控制的条件下才能进入内核。在 x86 上，有两种机制共同提供这种保护：

1. 中断描述符表。处理器确保中断和异常导致的进入内核只会从一些特定的、提前设置好的入口（这些入口是由内核自己决定的）进，而不是由产生中断和异常时产生的代码决定的。

    x86 允许多达 256 个不同的中断或异常进入内核，每个都有不同的中断向量。向量是 0 到 255 之间的一个数。中断的向量由中断的来源决定：不同的设备、错误条件和应用程序对内核的请求都会产生不同的中断向量。CPU 将 vector 用作处理器的中断描述符表(IDT)的索引，内核在内核私有内存中设置该表，与 GDT 非常相似。处理器从该表的相应条目中加载:

    The x86 allows up to 256 different interrupt or exception entry points into the kernel, each with a different interrupt vector. A vector is a number between 0 and 255. An interrupt's vector is determined by the source of the interrupt: different devices, error conditions, and application requests to the kernel generate interrupts with different vectors. The CPU uses the vector as an index into the processor's interrupt descriptor table (IDT), which the kernel sets up in kernel-private memory, much like the GDT. From the appropriate entry in this table the processor loads:

    - the value to load into the instruction pointer (EIP) register, pointing to the kernel code designated to handle that type of exception.
    - the value to load into the code segment (CS) register, which includes in bits 0-1 the privilege level at which the exception handler is to run. (In JOS, all exceptions are handled in kernel mode, privilege level 0.)

2. The Task State Segment. The processor needs a place to save the old processor state before the interrupt or exception occurred, such as the original values of EIP and CS before the processor invoked the exception handler, so that the exception handler can later restore that old state and resume the interrupted code from where it left off. But this save area for the old processor state must in turn be protected from unprivileged user-mode code; otherwise buggy or malicious user code could compromise the kernel.

    For this reason, when an x86 processor takes an interrupt or trap that causes a privilege level change from user to kernel mode, it also switches to a stack in the kernel's memory. A structure called the task state segment (TSS) specifies the segment selector and address where this stack lives. The processor pushes (on this new stack) SS, ESP, EFLAGS, CS, EIP, and an optional error code. Then it loads the CS and EIP from the interrupt descriptor, and sets the ESP and SS to refer to the new stack.

    Although the TSS is large and can potentially serve a variety of purposes, JOS only uses it to define the kernel stack that the processor should switch to when it transfers from user to kernel mode. Since "kernel mode" in JOS is privilege level 0 on the x86, the processor uses the ESP0 and SS0 fields of the TSS to define the kernel stack when entering kernel mode. JOS doesn't use any other TSS fields.

### Types of Exceptions and Interrupts

All of the synchronous exceptions that the x86 processor can generate internally use interrupt vectors between 0 and 31, and therefore map to IDT entries 0-31. For example, a page fault always causes an exception through vector 14. Interrupt vectors greater than 31 are only used by software interrupts, which can be generated by the int instruction, or asynchronous hardware interrupts, caused by external devices when they need attention.

In this section we will extend JOS to handle the internally generated x86 exceptions in vectors 0-31. In the next section we will make JOS handle software interrupt vector 48 (0x30), which JOS (fairly arbitrarily) uses as its system call interrupt vector. In Lab 4 we will extend JOS to handle externally generated hardware interrupts such as the clock interrupt.

### An Example

Let's put these pieces together and trace through an example. Let's say the processor is executing code in a user environment and encounters a divide instruction that attempts to divide by zero.

1. The processor switches to the stack defined by the SS0 and ESP0 fields of the TSS, which in JOS will hold the values GD_KD and KSTACKTOP, respectively.
2. The processor pushes the exception parameters on the kernel stack, starting at address KSTACKTOP:

```
                     +--------------------+ KSTACKTOP
                     | 0x00000 | old SS   |     " - 4
                     |      old ESP       |     " - 8
                     |     old EFLAGS     |     " - 12
                     | 0x00000 | old CS   |     " - 16
                     |      old EIP       |     " - 20 <---- ESP
                     +--------------------+
```

3. Because we're handling a divide error, which is interrupt vector 0 on the x86, the processor reads IDT entry 0 and sets CS:EIP to point to the handler function described by the entry.
4. The handler function takes control and handles the exception, for example by terminating the user environment.

For certain types of x86 exceptions, in addition to the "standard" five words above, the processor pushes onto the stack another word containing an error code. The page fault exception, number 14, is an important example. See the 80386 manual to determine for which exception numbers the processor pushes an error code, and what the error code means in that case. When the processor pushes an error code, the stack would look as follows at the beginning of the exception handler when coming in from user mode:

```
                     +--------------------+ KSTACKTOP
                     | 0x00000 | old SS   |     " - 4
                     |      old ESP       |     " - 8
                     |     old EFLAGS     |     " - 12
                     | 0x00000 | old CS   |     " - 16
                     |      old EIP       |     " - 20
                     |     error code     |     " - 24 <---- ESP
                     +--------------------+
```

### Nested Exceptions and Interrupts

The processor can take exceptions and interrupts both from kernel and user mode. It is only when entering the kernel from user mode, however, that the x86 processor automatically switches stacks before pushing its old register state onto the stack and invoking the appropriate exception handler through the IDT. If the processor is already in kernel mode when the interrupt or exception occurs (the low 2 bits of the CS register are already zero), then the CPU just pushes more values on the same kernel stack. In this way, the kernel can gracefully handle nested exceptions caused by code within the kernel itself. This capability is an important tool in implementing protection, as we will see later in the section on system calls.

If the processor is already in kernel mode and takes a nested exception, since it does not need to switch stacks, it does not save the old SS or ESP registers. For exception types that do not push an error code, the kernel stack therefore looks like the following on entry to the exception handler:

```
                     +--------------------+ <---- old ESP
                     |     old EFLAGS     |     " - 4
                     | 0x00000 | old CS   |     " - 8
                     |      old EIP       |     " - 12
                     +--------------------+
```

For exception types that push an error code, the processor pushes the error code immediately after the old EIP, as before.

There is one important caveat to the processor's nested exception capability. If the processor takes an exception while already in kernel mode, and cannot push its old state onto the kernel stack for any reason such as lack of stack space, then there is nothing the processor can do to recover, so it simply resets itself. Needless to say, the kernel should be designed so that this can't happen.

### Setting Up the IDT

You should now have the basic information you need in order to set up the IDT and handle exceptions in JOS. For now, you will set up the IDT to handle interrupt vectors 0-31 (the processor exceptions). We'll handle system call interrupts later in this lab and add interrupts 32-47 (the device IRQs) in a later lab.

The header files inc/trap.h and kern/trap.h contain important definitions related to interrupts and exceptions that you will need to become familiar with. The file kern/trap.h contains definitions that are strictly private to the kernel, while inc/trap.h contains definitions that may also be useful to user-level programs and libraries.

Note: Some of the exceptions in the range 0-31 are defined by Intel to be reserved. Since they will never be generated by the processor, it doesn't really matter how you handle them. Do whatever you think is cleanest.

The overall flow of control that you should achieve is depicted below:

```
      IDT                   trapentry.S         trap.c

+----------------+
|   &handler1    |---------> handler1:          trap (struct Trapframe *tf)
|                |             // do stuff      {
|                |             call trap          // handle the exception/interrupt
|                |             // ...           }
+----------------+
|   &handler2    |--------> handler2:
|                |            // do stuff
|                |            call trap
|                |            // ...
+----------------+
       .
       .
       .
+----------------+
|   &handlerX    |--------> handlerX:
|                |             // do stuff
|                |             call trap
|                |             // ...
+----------------+
```

Each exception or interrupt should have its own handler in trapentry.S and trap_init() should initialize the IDT with the addresses of these handlers. Each of the handlers should build a struct Trapframe (see inc/trap.h) on the stack and call trap() (in trap.c) with a pointer to the Trapframe. trap() then handles the exception/interrupt or dispatches to a specific handler function.

Exercise 4. Edit trapentry.S and trap.c and implement the features described above. The macros TRAPHANDLER and TRAPHANDLER*NOEC in trapentry.S should help you, as well as the T*\* defines in inc/trap.h. You will need to add an entry point in trapentry.S (using those macros) for each trap defined in inc/trap.h, and you'll have to provide \_alltraps which the TRAPHANDLER macros refer to. You will also need to modify trap_init() to initialize the idt to point to each of these entry points defined in trapentry.S; the SETGATE macro will be helpful here.

Your \_alltraps should:

1. push values to make the stack look like a struct Trapframe
2. load GD_KD into %ds and %es
3. pushl %esp to pass a pointer to the Trapframe as an argument to trap()
4. call trap (can trap ever return?)

Consider using the pushal instruction; it fits nicely with the layout of the struct Trapframe.

Test your trap handling code using some of the test programs in the user directory that cause exceptions before making any system calls, such as user/divzero. You should be able to get make grade to succeed on the divzero, softint, and badsegment tests at this point.
