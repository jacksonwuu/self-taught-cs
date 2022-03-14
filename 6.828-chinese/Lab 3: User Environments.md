# Lab 3: User Environments

在这个实验中，需要为操作系统实现用户环境，这里的用户环境可以理解为现代操作系统中的线程（或进程）的概念。

这个实验的难度还是挺大的，需要阅读很多手册，理解一些底层概念，比如说 ELF 文件格式、中断、中断描述符、CPU 权限级别、一些 CPU 指令、[gcc inline assembly](http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html) 等。还要理解程序在 CPU 级别是如何运行的，什么组成了进程（或者这个实验里说的用户环境）。所以做这个实验需要慢慢来，多花一些时间学习配套知识。

这个实验分为 AB 两个部分。

### A

这个部分要实现用户环境、关于用户环境的基本函数、以及创建用户环境来执行 ELF 文件的功能。

先理解`Env`这个 struct 的含义。

```c
struct Env {
	struct Trapframe env_tf;	// Saved registers
	struct Env *env_link;		// Next free Env
	envid_t env_id;			// Unique environment identifier
	envid_t env_parent_id;		// env_id of this env's parent
	enum EnvType env_type;		// Indicates special system environments
	unsigned env_status;		// Status of the environment
	uint32_t env_runs;		// Number of times environment has run

	// Address space
	pde_t *env_pgdir;		// Kernel virtual address of page dir
};
```

这里的`Env`类似于 xv6 里的`proc`。

#### 练习 1

在 mem_init()里为 envs 数组分配内存空间。

#### 练习 2

完成以下函数

```c
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

### B

处理页错误（如何处理页错误导致的中断？）

处理断点异常（中断向量 3 就是断点异常中断）

理解系统调用（什么是系统调用？如何实现系统调用？）

理解内存保护（进程之间如何隔离内存？内核的内存如何不被用户代码破坏？）
