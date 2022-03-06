# Lab2: Memory Management

本实验的目标：

1、实现操作系统的内存管理模块。

内存管理有两个模块：

-   physical memory allocator

-   virtual memory

PS：在开始做实验之前，建议先充分理解 CPU 对分页的支持（MMU），以及操作系统管理内存管理的知识，推荐阅读 Linux 0.11 内存管理的代码。

主要是实现以下几个函数，使其按照预期工作即可：

```c
static void * boot_alloc(uint32_t n)

void page_init(void)

struct PageInfo * page_alloc(int alloc_flags)

// 用于释放一个页的内存，把页数组中对应的项清零，再把物理页的内存全部清零。
void page_free(struct PageInfo *pp)

void page_decref(struct PageInfo* pp)

pte_t * pgdir_walk(pde_t *pgdir, const void *va, int create)

static void boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)

int page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)

struct PageInfo * page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)

void page_remove(pde_t *pgdir, void *va)
```

实际上，只要理解了用来表示内存的数据结构，那么这些函数很快就可以写出来。

#### physical memory allocator

It keeps track of which pages are free with a linked list of struct PageInfo objects (which, unlike xv6, are not embedded in the free pages themselves), each corresponding to a physical page.

#### virtual memory

先查看 Intel 80386 Reference Manual 来学习 CPU 的分页和分段功能。

Look at chapters 5 and 6 of the Intel 80386 Reference Manual, if you haven't done so already. Read the sections about page translation and page-based protection closely (5.2 and 6.4). We recommend that you also skim the sections about segmentation; while JOS uses the paging hardware for virtual memory and protection, segment translation and segment-based protection cannot be disabled on the x86, so you will need a basic understanding of it.

通过查询 CMOS RAM 硬件来获取该计算机有多少物理内存。一般来说 BIOS 会提供一些函数来查询硬件的基本信息（通过扫描硬件）。

-   inc/memlayout.h
-   kern/pmap.c
-   kern/pmap.h
-   kern/kclock.h
-   kern/kclock.c

Now you'll write a set of routines to manage page tables: to insert and remove linear-to-physical mappings, and to create page table pages when needed.

#### Kernel Address Space（内核地址空间）

#### 挑战一

Each user-level environment maps the kernel. Change JOS so that the kernel has its own page table and so that a user-level environment runs with a minimal number of kernel pages mapped. That is, each user-level environment maps just enough pages mapped so that the user-level environment can enter and leave the kernel correctly. You also have to come up with a plan for the kernel to read/write arguments to system calls.

给内核也分配页表。

#### 挑战二

Write up an outline of how a kernel could be designed to allow user environments unrestricted use of the full 4GB virtual and linear address space. Hint: do the previous challenge exercise first, which reduces the kernel to a few mappings in a user environment. Hint: the technique is sometimes known as "follow the bouncing kernel." In your design, be sure to address exactly what has to happen when the processor transitions between kernel and user modes, and how the kernel would accomplish such transitions. Also describe how the kernel would access physical memory and I/O devices in this scheme, and how the kernel would access a user environment's virtual address space during system calls and the like. Finally, think about and describe the advantages and disadvantages of such a scheme in terms of flexibility, performance, kernel complexity, and other factors you can think of.

#### 挑战三

Since our JOS kernel's memory management system only allocates and frees memory on page granularity, we do not have anything comparable to a general-purpose malloc/free facility that we can use within the kernel. This could be a problem if we want to support certain types of I/O devices that require physically contiguous buffers larger than 4KB in size, or if we want user-level environments, and not just the kernel, to be able to allocate and map 4MB superpages for maximum processor efficiency. (See the earlier challenge problem about PTE_PS.)

Generalize the kernel's memory allocation system to support pages of a variety of power-of-two allocation unit sizes from 4KB up to some reasonable maximum of your choice. Be sure you have some way to divide larger allocation units into smaller ones on demand, and to coalesce multiple small allocation units back into larger units when possible. Think about the issues that might arise in such a system.

支持各种大小的内存。