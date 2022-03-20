# Lab 5: File system, Spawn and Shell

在这个实验中，你将要实现 spawn，一个用来加载并运行磁盘可执行文件的库程序。然后你将会充实你的内核和库以达到可以在 console 里允许 shell 的程度。这些特性需要一个文件系统，本实验会介绍一个简单的读/写文件系统。

## 文件系统初步

你要做的文件系统比大部分真实的文件系统都要简单，包括 xv6 Unix 的，但是提供基本的特性就非常不错了：创建、读取、写入以及删除文件，这些文件通过层级结构来组织。

我们(至少在目前)只开发一个单用户操作系统，它提供了足够的保护来捕捉 bug，但不能保护多个相互怀疑的用户。因此我们的文件系统不支持 UNIX 文件所有权或权限的概念。我们的文件系统目前也不支持硬链接、符号链接、时间戳或像大多数 UNIX 文件系统那样的特殊设备文件。

### 磁盘文件系统结构

大多数 UNIX 文件系统把磁盘空间切分为两种主要区域：inode 区域和 data 区域。UNIX 文件系统给文件系统里的每一个文件分配一个 inode；一个文件的 inode 包含了文件重要的元数据，比如说文件的属性和指向数据块的指针，文件系统在这个数据块里存储文件数据和目录元数据。目录项包含文件名和指向 inode 的指针；如果文件系统中的多个目录项指向该文件的 inode，则该文件被称为硬链接，我们不需要这种级别的文件关联，因此我们可以做一些简化：我们的文件系统根本就不使用 inode，而是简单地把所有文件的元数据存到描述这个文件的目录项。

文件和目录在逻辑上包含了一系列的数据块，这些数据块可能会分散在磁盘各处，就好像一个用户环境（进程）的虚拟地址分散映射到各个物理内存页。文件系统环境隐藏了数据块的分布细节，对外展示一个接口去读写文件任意偏移量的字节序列。文件系统环境在内部处理对目录的所有修改，作为执行文件创建和删除等操作的一部分。我们的文件系统确实允许用户环境直接去读取目录的元数据，这意味着用户环境可以自己执行目录扫描操作（实现 ls 程序），而不是依赖于额外的文件系统调用。这种方式的目录扫描的缺点，也是大多数现代 UNIX 所不推荐的，就是如果想更改文件系统的内部布局，应用程序也要跟着修改或者重新编译。

#### 扇区和区块

大多数磁盘无法执行字节粒度的读写，而是直接读写整个扇区。在 JOS 里，每个扇区 512 个字节。文件系统实际上以块为单位分配和使用磁盘存储。请注意这两个术语的区别：扇区大小是磁盘硬件决定的，而区块大小是操作系统决定的。一个文件系统的区块大小必须是该磁盘扇区大小的整数倍。

UNIX xv6 文件系统的一个区块未 512 比特，和扇区大小一样。但是大多数文件系统采用的是更大的区块，因为存储空间很便宜，可以在较大粒度上管理存储。我们的文件系统会使用 4096 比特大小的区块，以便和处理器的页大小匹配。

#### 超级区块（Superblocks）

文件系统把存储整个文件系统元数据的区块放到一个“非常寻找”的磁盘位置（比如说磁盘的起始或末尾），这些元数据存储了区块大小、磁盘大小，以及任何任何寻找根目录所需的元数据，文件系统上次挂载的时间，文件系统上次 check error 的时间等等。这些特别的区块被称为超级区块。

我们的文件系统会有一个超级区块，它永远都是在磁盘的区块 1 上。它的布局被定义在`inc/fs.h`的`Super`结构体里。区块 0 是用来存放 boot loader 和分区表的，所以文件系统一般不会使用磁盘的第一个区块。许多“真实”的文件系统维护了多个超级区块，把这几个超级区块复制到多个宽区域里，这样如果其中一个区域的损坏或磁盘在区域中出现了错误，仍然可以找到其他超级块并使用它们访问文件系统。

#### 文件元数据

我们在`inc/fs.h`里定义的`File`结构体中定义里元数据的布局，这个用来描述一个文件系统的文件。这个元数据包含了文件名、大小、类型（正常文件或目录），以及一个指向存储该文件区块的指针。如上所述，我们不需要 inode，所以这个元数据存储在目录项里。不像大多数“真实“文件系统，为了简化，我们使用一个`File`结构体来代表文件元数据，这个元数据会存储在磁盘，也会出现在内存里。

The f_direct array in struct File contains space to store the block numbers of the first 10 (NDIRECT) blocks of the file, which we call the file's direct blocks. For small files up to 10\*4096 = 40KB in size, this means that the block numbers of all of the file's blocks will fit directly within the File structure itself. For larger files, however, we need a place to hold the rest of the file's block numbers. For any file greater than 40KB in size, therefore, we allocate an additional disk block, called the file's indirect block, to hold up to 4096/4 = 1024 additional block numbers. Our file system therefore allows files to be up to 1034 blocks, or just over four megabytes, in size. To support larger files, "real" file systems typically support double- and triple-indirect blocks as well.

#### 目录 vs 常规文件

我们文件系统的一个文件结构可以代表普通文件或者目录；这两种“文件”通过`type`字段来做区分。文件系统管理普通文件和目录的方式都是一样的，The file system manages regular files and directory-files in exactly the same way, except that it does not interpret the contents of the data blocks associated with regular files at all, whereas the file system interprets the contents of a directory-file as a series of File structures describing the files and subdirectories within the directory.

我们文件系统里的超级区块包含了一个`File`结构体（`Super`结构体里的`root`字段），这里面保存了文件系统根目录的元数据。这个目录里的内容是一串`File`结构体，这些结构体描述了在根目录下的文件和目录。任何根目录下的子目录也会包含`File`结构体来表示孙子目录，以此类推。

## 文件系统

这个实验的目的不是实现整个文件系统，而是实现一些关键的组件。特别的，你要负责把区块读到区块缓存里以及把它们重新刷回磁盘；申请磁盘区块；映射文件偏移到磁盘区块；实现 read、write、open（in the IPC interface）。因为你不用自己实现整个文件系统，但是很重要的是你要熟悉已经提供代码和不同类型的文件系统的接口。

### 访问磁盘

我们操作系统的文件系统需要能访问磁盘，但是我们还没有实现任何磁盘访问功能。我们并不把 IDE 磁盘驱动也放到内核里，而是放到用户级别的文件系统环境里。我们只会轻微地改造内核，以便让文件系统环境有访问磁盘的权限。

It is easy to implement disk access in user space this way as long as we rely on polling, "programmed I/O" (PIO)-based disk access and do not use disk interrupts. It is possible to implement interrupt-driven device drivers in user mode as well (the L3 and L4 kernels do this, for example), but it is more difficult since the kernel must field device interrupts and dispatch them to the correct user-mode environment.

x86 处理器通过 EFLAGS 寄存器里的 IOPL 位来决定是否保护模式代码可以指向 I/O 指令，比如说 IN 和 OUT 指令。因为所有的 IDE 磁盘寄存器都是在 x86 的 I/O 空间，而不是通过存储器映射的方式，给文件系统“I/O 权限”是让它访问这些寄存器的唯一方式。实际上，EFLAGS 寄存器里的 IOPL 位给内核提供了一种“all-or-nothing”的方法来控制用户态代码是否可以访问磁盘。在我们的情况下，我们想要文件系统环境可以访问 I/O 空间，但是我们不想要任何用户环境可以访问 I/O 空间。

练习 1：把`ENV_TYPE_FS`传入用户环境创建函数`env_create`，`i386_init`通过这个参数来区分是不是文件系统环境。修改`env.c`里的的`env_create`，以便给文件系统环境 I/O 权限，但是一定不要给其他环境这个权限。

### 区块缓存

In our file system, we will implement a simple "buffer cache" (really just a block cache) with the help of the processor's virtual memory system. The code for the block cache is in fs/bc.c.

Our file system will be limited to handling disks of size 3GB or less. We reserve a large, fixed 3GB region of the file system environment's address space, from 0x10000000 (DISKMAP) up to 0xD0000000 (DISKMAP+DISKMAX), as a "memory mapped" version of the disk. For example, disk block 0 is mapped at virtual address 0x10000000, disk block 1 is mapped at virtual address 0x10001000, and so on. The diskaddr function in fs/bc.c implements this translation from disk block numbers to virtual addresses (along with some sanity checking).

Since our file system environment has its own virtual address space independent of the virtual address spaces of all other environments in the system, and the only thing the file system environment needs to do is to implement file access, it is reasonable to reserve most of the file system environment's address space in this way. It would be awkward for a real file system implementation on a 32-bit machine to do this since modern disks are larger than 3GB. Such a buffer cache management approach may still be reasonable on a machine with a 64-bit address space.

Of course, it would take a long time to read the entire disk into memory, so instead we'll implement a form of demand paging, wherein we only allocate pages in the disk map region and read the corresponding block from the disk in response to a page fault in this region. This way, we can pretend that the entire disk is in memory.

```
练习2：Implement the bc_pgfault and flush_block functions in fs/bc.c. bc_pgfault is a page fault handler, just like the one your wrote in the previous lab for copy-on-write fork, except that its job is to load pages in from the disk in response to a page fault. When writing this, keep in mind that (1) addr may not be aligned to a block boundary and (2) ide_read operates in sectors, not blocks.

The flush_block function should write a block out to disk if necessary. flush_block shouldn't do anything if the block isn't even in the block cache (that is, the page isn't mapped) or if it's not dirty. We will use the VM hardware to keep track of whether a disk block has been modified since it was last read from or written to disk. To see whether a block needs writing, we can just look to see if the PTE_D "dirty" bit is set in the uvpt entry. (The PTE_D bit is set by the processor in response to a write to that page; see 5.2.4.3 in chapter 5 of the 386 reference manual.) After writing the block to disk, flush_block should clear the PTE_D bit using sys_page_map.
```

The fs_init function in fs/fs.c is a prime example of how to use the block cache. After initializing the block cache, it simply stores pointers into the disk map region in the super global variable. After this point, we can simply read from the super structure as if they were in memory and our page fault handler will read them from disk as necessary.

### 区块位图

After fs_init sets the bitmap pointer, we can treat bitmap as a packed array of bits, one for each block on the disk. See, for example, block_is_free, which simply checks whether a given block is marked free in the bitmap.

Exercise 3. Use free_block as a model to implement alloc_block in fs/fs.c, which should find a free disk block in the bitmap, mark it used, and return the number of that block. When you allocate a block, you should immediately flush the changed bitmap block to disk with flush_block, to help file system consistency.

### 文件操作

We have provided a variety of functions in fs/fs.c to implement the basic facilities you will need to interpret and manage File structures, scan and manage the entries of directory-files, and walk the file system from the root to resolve an absolute pathname. Read through all of the code in fs/fs.c and make sure you understand what each function does before proceeding.

Exercise 4. Implement file_block_walk and file_get_block. file_block_walk maps from a block offset within a file to the pointer for that block in the struct File or the indirect block, very much like what pgdir_walk did for page tables. file_get_block goes one step further and maps to the actual disk block, allocating a new one if necessary.

file_block_walk and file_get_block are the workhorses of the file system. For example, file_read and file_write are little more than the bookkeeping atop file_get_block necessary to copy bytes between scattered blocks and a sequential buffer.

### 文件系统接口

## Spawning Processes

### Sharing library state across fork and spawn

## The keyboard interface

## The Shell
