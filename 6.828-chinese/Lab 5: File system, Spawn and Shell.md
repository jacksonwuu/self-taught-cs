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

我们在`inc/fs.h`里定义的`File`结构体中定义里元数据的布局，这个用来描述一个文件系统的文件。这个元数据包含了文件名、大小、类型（正常文件或目录），以及一个指向存储该文件区块的指针。如上所述，我们不需要inode，所以这个元数据存储在目录项里。不像大多数“真实“文件系统，为了简化，我们使用一个`File`结构体来代表文件元数据，这个元数据会存储在磁盘，也会出现在内存里。

The f_direct array in struct File contains space to store the block numbers of the first 10 (NDIRECT) blocks of the file, which we call the file's direct blocks. For small files up to 10\*4096 = 40KB in size, this means that the block numbers of all of the file's blocks will fit directly within the File structure itself. For larger files, however, we need a place to hold the rest of the file's block numbers. For any file greater than 40KB in size, therefore, we allocate an additional disk block, called the file's indirect block, to hold up to 4096/4 = 1024 additional block numbers. Our file system therefore allows files to be up to 1034 blocks, or just over four megabytes, in size. To support larger files, "real" file systems typically support double- and triple-indirect blocks as well.

#### 目录 vs 常规文件

A File structure in our file system can represent either a regular file or a directory; these two types of "files" are distinguished by the type field in the File structure. The file system manages regular files and directory-files in exactly the same way, except that it does not interpret the contents of the data blocks associated with regular files at all, whereas the file system interprets the contents of a directory-file as a series of File structures describing the files and subdirectories within the directory.
The superblock in our file system contains a File structure (the root field in struct Super) that holds the meta-data for the file system's root directory. The contents of this directory-file is a sequence of File structures describing the files and directories located within the root directory of the file system. Any subdirectories in the root directory may in turn contain more File structures representing sub-subdirectories, and so on.

## 文件系统



### 进入磁盘

### 区块缓存

### 区块位图

### 文件操作

### 文件系统接口

## Spawning Processes

### Sharing library state across fork and spawn

## The keyboard interface

## The Shell
