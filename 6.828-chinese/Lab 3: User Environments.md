# Lab 3: User Environments

## 介绍

在本实验室中，您将实现运行受保护的用户模式环境(即“进程”)所需的基本内核设施。您将增强 JOS 内核，以设置数据结构来跟踪用户环境、创建单个用户环境、将程序映像加载到其中并启动它。您还将使 JOS 内核能够处理用户环境发出的任何系统调用，并处理它引起的任何其他异常。

注意：在本实验室中，术语“环境”和“进程”是可以互换的——它们都是指允许您运行程序的抽象。我们引入术语“环境”而不是传统的术语“进程”，是为了强调 JOS 环境和 UNIX 进程提供不同的接口，并且不提供相同的语义。

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
inc/	env.h	Public definitions for user-mode environments
        trap.h	Public definitions for trap handling
        syscall.h	Public definitions for system calls from user environments to the kernel
        lib.h	Public definitions for the user-mode support library
kern/	env.h	Kernel-private definitions for user-mode environments
        env.c	Kernel code implementing user-mode environments
        trap.h	Kernel-private trap handling definitions
        trap.c	Trap handling code
        trapentry.S	Assembly-language trap handler entry-points
        syscall.h	Kernel-private definitions for system call handling
        syscall.c	System call implementation code
lib/	Makefrag	Makefile fragment to build user-mode library, obj/lib/libjos.a
        entry.S	Assembly-language entry-point for user environments
        libmain.c	User-mode library setup code called from entry.S
        syscall.c	User-mode system call stub functions
        console.c	User-mode implementations of putchar and getchar, providing console I/O
        exit.c	User-mode implementation of exit
        panic.c	User-mode implementation of panic
user/	*	Various test programs to check kernel lab 3 code
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

在这个实验中，您可能会发现 GCC 的内联汇编语言特性很有用，尽管不使用它也可以完成这个实验。至少，您需要能够理解我们提供的源代码中已经存在的内联汇编语言片段(“asm”语句)。您可以在课程[参考资料页面](https://pdos.csail.mit.edu/6.828/2018/reference.html)上找到关于 GCC 内联汇编语言的几个信息来源。

## Part A: User Environments and Exception Handling

新的 include 文件 inc/env.h 包含了 JOS 中用户环境的基本定义。现在读它。内核使用 Env 数据结构来跟踪每个用户环境。在本实验室中，您将最初只创建一个环境，但您需要设计 JOS 内核来支持多个环境；Lab 4 将通过允许用户环境 fork（创建）其他环境来利用这个特性。

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
    This is used to distinguish special environments. For most environments, it will be ENV_TYPE_USER. We'll introduce a few more types for special system service environments in later labs.
env_status:
    This variable holds one of the following values:
    ENV_FREE:
        Indicates that the Env structure is inactive, and therefore on the env_free_list.
    ENV_RUNNABLE:
        Indicates that the Env structure represents an environment that is waiting to run on the processor.
    ENV_RUNNING:
        Indicates that the Env structure represents the currently running environment.
    ENV_NOT_RUNNABLE:
        Indicates that the Env structure represents a currently active environment, but it is not currently ready to run: for example, because it is waiting for an interprocess communication (IPC) from another environment.
    ENV_DYING:
        Indicates that the Env structure represents a zombie environment. A zombie environment will be freed the next time it traps to the kernel. We will not use this flag until Lab 4.
env_pgdir:
    This variable holds the kernel virtual address of this environment's page directory.
```

Like a Unix process, a JOS environment couples the concepts of "thread" and "address space". The thread is defined primarily by the saved registers (the env_tf field), and the address space is defined by the page directory and page tables pointed to by env_pgdir. To run an environment, the kernel must set up the CPU with both the saved registers and the appropriate address space.

Our struct Env is analogous to struct proc in xv6. Both structures hold the environment's (i.e., process's) user-mode register state in a Trapframe structure. In JOS, individual environments do not have their own kernel stacks as processes do in xv6. There can be only one JOS environment active in the kernel at a time, so JOS needs only a single kernel stack.

### Allocating the Environments Array

In lab 2, you allocated memory in mem_init() for the pages[] array, which is a table the kernel uses to keep track of which pages are free and which are not. You will now need to modify mem_init() further to allocate a similar array of Env structures, called envs.

Exercise 1. Modify mem_init() in kern/pmap.c to allocate and map the envs array. This array consists of exactly NENV instances of the Env structure allocated much like how you allocated the pages array. Also like the pages array, the memory backing envs should also be mapped user read-only at UENVS (defined in inc/memlayout.h) so user processes can read from this array.

You should run your code and make sure check_kern_pgdir() succeeds.

### Creating and Running Environments

You will now write the code in kern/env.c necessary to run a user environment. Because we do not yet have a filesystem, we will set up the kernel to load a static binary image that is embedded within the kernel itself. JOS embeds this binary in the kernel as a ELF executable image.

The Lab 3 GNUmakefile generates a number of binary images in the obj/user/ directory. If you look at kern/Makefrag, you will notice some magic that "links" these binaries directly into the kernel executable as if they were .o files. The -b binary option on the linker command line causes these files to be linked in as "raw" uninterpreted binary files rather than as regular .o files produced by the compiler. (As far as the linker is concerned, these files do not have to be ELF images at all - they could be anything, such as text files or pictures!) If you look at obj/kern/kernel.sym after building the kernel, you will notice that the linker has "magically" produced a number of funny symbols with obscure names like \_binary_obj_user_hello_start, \_binary_obj_user_hello_end, and \_binary_obj_user_hello_size. The linker generates these symbol names by mangling the file names of the binary files; the symbols provide the regular kernel code with a way to reference the embedded binary files.

In i386_init() in kern/init.c you'll see code to run one of these binary images in an environment. However, the critical functions to set up user environments are not complete; you will need to fill them in.

Exercise 2. In the file env.c, finish coding the following functions:

```
env_init()
    Initialize all of the Env structures in the envs array and add them to the env_free_list. Also calls env_init_percpu, which configures the segmentation hardware with separate segments for privilege level 0 (kernel) and privilege level 3 (user).
env_setup_vm()
    Allocate a page directory for a new environment and initialize the kernel portion of the new environment's address space.
region_alloc()
    Allocates and maps physical memory for an environment
load_icode()
    You will need to parse an ELF binary image, much like the boot loader already does, and load its contents into the user address space of a new environment.
env_create()
    Allocate an environment with env_alloc and call load_icode to load an ELF binary into it.
env_run()
    Start a given environment running in user mode.
```

As you write these functions, you might find the new cprintf verb %e useful -- it prints a description corresponding to an error code. For example,

```c
r = -E_NO_MEM;
panic("env_alloc: %e", r);
```

will panic with the message "env_alloc: out of memory".

Below is a call graph of the code up to the point where the user code is invoked. Make sure you understand the purpose of each step.

-   start (kern/entry.S)
-   i386_init (kern/init.c)
    -   cons_init
    -   mem_init
    -   env_init
    -   trap_init (still incomplete at this point)
    -   env_create
    -   env_run
        -   env_pop_tf

Once you are done you should compile your kernel and run it under QEMU. If all goes well, your system should enter user space and execute the hello binary until it makes a system call with the int instruction. At that point there will be trouble, since JOS has not set up the hardware to allow any kind of transition from user space into the kernel. When the CPU discovers that it is not set up to handle this system call interrupt, it will generate a general protection exception, find that it can't handle that, generate a double fault exception, find that it can't handle that either, and finally give up with what's known as a "triple fault". Usually, you would then see the CPU reset and the system reboot. While this is important for legacy applications (see this blog post for an explanation of why), it's a pain for kernel development, so with the 6.828 patched QEMU you'll instead see a register dump and a "Triple fault." message.

We'll address this problem shortly, but for now we can use the debugger to check that we're entering user mode. Use make qemu-gdb and set a GDB breakpoint at env_pop_tf, which should be the last function you hit before actually entering user mode. Single step through this function using si; the processor should enter user mode after the iret instruction. You should then see the first instruction in the user environment's executable, which is the cmpl instruction at the label start in lib/entry.S. Now use b \*0x... to set a breakpoint at the int $0x30 in sys_cputs() in hello (see obj/user/hello.asm for the user-space address). This int is the system call to display a character to the console. If you cannot execute as far as the int, then something is wrong with your address space setup or program loading code; go back and fix it before continuing.

### Handling Interrupts and Exceptions

At this point, the first int $0x30 system call instruction in user space is a dead end: once the processor gets into user mode, there is no way to get back out. You will now need to implement basic exception and system call handling, so that it is possible for the kernel to recover control of the processor from user-mode code. The first thing you should do is thoroughly familiarize yourself with the x86 interrupt and exception mechanism.

Exercise 3. Read [Chapter 9, Exceptions and Interrupts](https://pdos.csail.mit.edu/6.828/2018/readings/i386/c09.htm) in the [80386 Programmer's Manual](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm) (or Chapter 5 of the [IA-32 Developer's Manual](https://pdos.csail.mit.edu/6.828/2018/readings/ia32/IA32-3A.pdf)), if you haven't already.

In this lab we generally follow Intel's terminology for interrupts, exceptions, and the like. However, terms such as exception, trap, interrupt, fault and abort have no standard meaning across architectures or operating systems, and are often used without regard to the subtle distinctions between them on a particular architecture such as the x86. When you see these terms outside of this lab, the meanings might be slightly different.

### Basics of Protected Control Transfer

Exceptions and interrupts are both "protected control transfers," which cause the processor to switch from user to kernel mode (CPL=0) without giving the user-mode code any opportunity to interfere with the functioning of the kernel or other environments. In Intel's terminology, an interrupt is a protected control transfer that is caused by an asynchronous event usually external to the processor, such as notification of external device I/O activity. An exception, in contrast, is a protected control transfer caused synchronously by the currently running code, for example due to a divide by zero or an invalid memory access.

In order to ensure that these protected control transfers are actually protected, the processor's interrupt/exception mechanism is designed so that the code currently running when the interrupt or exception occurs does not get to choose arbitrarily where the kernel is entered or how. Instead, the processor ensures that the kernel can be entered only under carefully controlled conditions. On the x86, two mechanisms work together to provide this protection:

1. The Interrupt Descriptor Table. The processor ensures that interrupts and exceptions can only cause the kernel to be entered at a few specific, well-defined entry-points determined by the kernel itself, and not by the code running when the interrupt or exception is taken.

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
