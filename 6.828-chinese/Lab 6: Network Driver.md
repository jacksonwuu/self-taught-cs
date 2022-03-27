# Lab 6: Network Driver

这个实验还是非常有难度的，我会通过翻译的形式，来促进我理解任务书里的每一句话。

## Introduction

虽然我们要实现一个网卡驱动，但是这个驱动还不足以让你的操作系统连上互联网。我们在本实验中已经给你提供了一个网络栈和网络服务器。

为了可以写好这个驱动，你要创建一个系统调用接口，以便可以访问网卡驱动。你要实现缺失的代码来在网络栈和驱动之间传递数据包。你也需要组合各种功能来实现一个 web 服务器。有了这个 web 服务器，你就可以利用文件系统来提供文件服务。

大多数内核设备驱动代码你都要从零开始写。这个实验提供的指导比上一个实验少：没有骨架文件，没有写好的系统调用，很多设计由你决定。出于这个原因，我们推荐你把整个指南读完再开始练习。许多学生发现这个实验比上一个实验要难，所以请相应地安排你的时间。

### QEMU 的虚拟网络

我们会使用 QEMU 的用户态网络栈，因为它不需要管理权限就能运行。QEMU 有一些关于 user-net 的[文档](http://wiki.qemu.org/download/qemu-doc.html#Using-the-user-mode-network-stack)。我们会更新 makefile 来开启 QEMU 用户态网络栈和虚拟 E1000 网卡。

虽然 QEMU 虚拟网络允许 JOS 随机创建一个连接到互联网，JOS 的 10.0.2.15 地址对 QEMU 里运行的虚拟网络来说没有任何意义（QEMU 只是作为一个 NAT），所以我们不能直接连接到 JOS 里的服务器，哪怕是来自主机的连接。为了解决这个问题，我们配置了 QEMU 在一些端口上运行一个服务器，简单地连接 JOS 的一些端口，在你的真实主机和虚拟网络之间传输数据。

你会运行 JOS 服务器在端口 7（echo）和 80（http）。

#### Packet Inspection

makefile 也配置了 QEMU 网络栈来记录所有进出的数据包，在实验文件夹的 qemu.pcap 文件里。

可以这样来生成这些数据包的 hex/ASCII 图：

`tcpdump -XXnr qemu.pcap`

或者你也可以使用 Wireshark 来图形化这些 pcap 文件。

#### Debugging the E1000

我们很幸运可以使用模拟硬件。E1000 是允许的软件，它可以给我们提供一个可读形式的报告，来告诉我们它的内部状态和它遇到的任何问题。一般来说，真正在硬件上做开发的开发者享受不到这样的好处。

E1000 可以产生很多 debug 信息，所以你要开启特定的 logging channel，这有一些有用的 channel：

```
Flag	    Meaning
tx          Log packet transmit operations
txerr	    Log transmit ring errors
rx          Log changes to RCTL
rxfilter    Log filtering of incoming packets
rxerr	    Log receive ring errors
unknown	    Log reads and writes of unknown registers
eeprom	    Log reads from the EEPROM
interrupt   Log interrupts and changes to interrupt registers.
```

为了开启“tx”和“txerr”日志，你可以这样：`make E1000_DEBUG=tx,txerr ....`

在模拟硬件上，你还可以进一步 debug。如果你卡住了，不理解为什么 E1000 的响应和你预期的不一样，你可以看一下 QEMU E1000 的实现，在`hw/net/e1000.c`里。

### The Network Server

从零开始写一个网络栈是个艰难的工作。反而，我们会用 lwIP，一个轻量级的 TCP/IP 协议簇。你可以在[这里](https://savannah.nongnu.org/projects/lwip/)找到更多关于 lwIP 的信息。在本任务书里，就我们注意到的，lwIP 是一个黑盒，它实现了 BSD socket interface 以及数据包输入输出端口。

网络服务器由四个环境（进程）组成：

-   核心网络服务环境（包括 socket call dispatcher 和 lwIP）
-   输入环境
-   输出环境
-   定时器环境

下面的图展示了不同环境和它们之间的关系。这个图展示了整个系统，包括设备驱动，我们之后会解释。在这个实验里，你要实现绿色的部分。

![](https://pdos.csail.mit.edu/6.828/2018/labs/lab6/ns.png)

#### The Core Network Server Environment

核心网络服务器环境由两部分组成：socket call dispatcher 和 lwIP 本身。socket call dispatcher 的作用就像文件服务器。用户环境使用`stubs`（`lib/nsipc.c`）来发送 IPC 消息给核心网络环境。如果你查看`lib/nsipc.c`，你会看到我们查找核心网络服务器的方法和查找文件服务器一样：`i386_init`创建`NS`环境，`NS_TYPE_NS`类型，所以我们可以扫描`envs`，来查找这个特别的环境类型。对于每一个用户环境 IPC，网络服务器里的 dispatcher 会代表用户调用合适 BSD socket interface 函数（lwIP 提供的）。

普通用户环境不会直接使用`nsipc_*`，而是使用`lib/sockets.c`里的函数，它提供了一个机遇文件描述符的套接字 API。因此，用户环境通过文件描述符来联系套接字，就像它们联系磁盘文件一样。有一些套接字独有的操作，connect、accept 等，但是 read、write 和 close 操作都要经过`lib/fd.c`中普通的文件描述符设备调度代码。就像文件服务器为打开文件维护内部特有 ID 一样，lwIP 也会为打开的套接字生成特有 ID。在文件服务器和网络服务器里，我们使用存储在 Fd 结构体里的信息来映射每个用户环境的文件描述符到特有 ID 空间。

尽管文件服务器和网络服务器的 IPC 调度程序看起来是一样的，但有一个关键的区别。像 accept 和 recv 这样的 BSD 套接字调用可以无限期地阻塞。如果调度程序要让 lwIP 执行这些阻塞调用中的一个，那么调度程序也会阻塞，并且对于整个系统来说，一次只能有一个未完成的网络调用。由于这是不可接受的，网络服务器使用用户级线程来避免阻塞整个服务器环境。对于每个传入的 IPC 消息，调度程序创建一个线程，并在新创建的线程中处理请求。如果线程阻塞，则只有该线程处于睡眠状态，而其他线程继续运行。

Even though it may seem that the IPC dispatchers of the file server and network server act the same, there is a key difference. BSD socket calls like accept and recv can block indefinitely. If the dispatcher were to let lwIP execute one of these blocking calls, the dispatcher would also block and there could only be one outstanding network call at a time for the whole system. Since this is unacceptable, the network server uses user-level threading to avoid blocking the entire server environment. For every incoming IPC message, the dispatcher creates a thread and processes the request in the newly created thread. If the thread blocks, then only that thread is put to sleep while other threads continue to run.

在核心网络环境之外，还有三个辅助环境。在接收用户应用的消息之外，核心网络环境分发器也会接收来自 input 和 timer 环境的消息。

#### The Output Environment

当为用户环境提供 socket 调用服务时，lwIP 会为网卡生成一个包来传输数据。lwIP 会发送每一个包到 output 环境，通过把包附在 IPC 消息的 page 参数里。output 环境负责接收这些消息以及转发这些包到设备驱动，通过系统调用接口（你马上就会创建的）。

#### The Input Environment

网卡接收到的包需要被注入 lwIP。对于每一个设备驱动接收到的包，input 环境把包从内核空间里拉取出来（使用我们实现的内核系统调用）并通过`NSREQ_INPUT`IPC 消息发送包到内核服务器环境。

包输入功能独立于核心网络环境，因为 JOS 很难去同步 IPC 消息接收和拉取来自于设备驱动的包。JOS 里没有 select 系统调用来让环境监控多个输入源来找出哪一个输入已经准备好要被处理了。

如果你看一线`net/input.c`和`net/output.c`你会发现它们都需要被实现。这主要是因为它们的实现都依赖于你的系统调用接口。在你实现了驱动和系统调用接口后，你也要写这两个辅助环境的代码。

#### The Timer Environment

定时器环境定时发送`NSREQ_TIMER`类型的消息来通知核心网络服务器一个定时器过期了。来自这个线程的定时器消息会被 lwIP 用来实现不同的网络超时。

## Part A: Initialization and transmitting packets

你的内核没有时间概念，所以我们需要添加它。目前有一个由硬件生成的时钟中断，每 10ms 发送一次。在每个时间中断里，我们可以增加一个变量来表示时间已经提前了 10ms。这个在`kern/time.c`实现的，但是还没有完全集成到我们的内核里。

练习 1：在`kern/trap.c`里为每一次时钟中断调用`time_tick`。实现`sys_time_msec`，然后把它添加到`kern/syscall.c`的系统调用里，乍样用户空间就可以访问时间了。

用`make INIT_CFLAGS=-DTEST_NO_NS run-testtime`来测试你的时间代码。你会看到环境从 5 开始数每秒减 1。"-DTEST_NO_NS"是用来取消开启网络服务环境的，因为它会此时会 panic。

### The Network Interface Card

Writing a driver requires knowing in depth the hardware and the interface presented to the software. The lab text will provide a high-level overview of how to interface with the E1000, but you'll need to make extensive use of Intel's manual while writing your driver.

练习 2：熟悉 E1000，浏览 Intel 的[软件开发者手册](https://pdos.csail.mit.edu/6.828/2018/readings/hardware/8254x_GBe_SDM.pdf)。这个手册覆盖了一些非常相关的以太网控制器。QEMU 模拟了 82540EM。

你应该大致浏览第 2 章，找到对该设备的感觉。为了写你的驱动，你需要熟悉第 3、14 章，以及 4.1。你也需要参考第 13 章。其他章节也覆盖了一些你的驱动不会互动的 E1000 的组件。不要太在意细节，现在只要知道文档的结构，等需要的时候知道在哪儿找，就行了。

当你读这个手册等时候，记住 E1000 是非一个非常复杂的设备，它有很多高级功能。一个能工作的 E1000 驱动只需要 NIC 提供的功能和接口的一小部分。仔细考虑最简单的办法来使用这个网卡。我们强烈推荐你先弄出一个能基本工作，再去弄那些高级功能。

#### PCI Interface

E1000 是一个 PCI 设备，意味着它可以插入主板的 PCI 总线上。PCI 总线有数据线、地址线和中断线，它允许 CPU 和 PCI 设备通信以及允许 PCI 设备去读写内存。一个 PCI 在使用之前需要先被发现以及初始化。发现过程是遍历 PCI 总线来查看附属的设备。初始化过程是申请 I/O 和内存空间以及协调设备使用的 IRQ 线。

我们已经在`kern/pci.c`里给你提供了 PCI 代码。为了在 boot 时进行 PCI 初始化，PCI 代码遍历 PCI 总线来发现设备。当它发现了一个设备，它读取设备的厂商 ID 和设备 ID，并用这两个值作为 key 来查找`pci_attach_vendor`数组。这个数组由`struct pci_driver`项组成：

```c
struct pci_driver {
    uint32_t key1, key2;
    int (*attachfn) (struct pci_func *pcif);
};
```

如果被发现的设备厂商 ID 和设备 ID 匹配了数组里的一个项，PCI 代码会调用该项的`attachfn`来进行设备初始化。（设备也可以用 class 来确认，这就是`kern/pci.c`里其他驱动表的作用。）

那个`attachfn`传递一个 PCI 函数去初始化。一个 PCI 卡可以暴露多个函数，即使 E1000 只暴露了一个。这就是我们如何在 JOS 里表示 PCI 函数的：

```c
struct pci_func {
    struct pci_bus *bus;

    uint32_t dev;
    uint32_t func;

    uint32_t dev_id;
    uint32_t dev_class;

    uint32_t reg_base[6];
    uint32_t reg_size[6];
    uint8_t irq_line;
};
```

以上的结构体反映了一些在开发者手册里 4-1 表格里的项。struct pci_func 的最后三项我们特别感兴趣，因为它们记录了协调后的内存，I/O，设备的中断资源。reg_base 和 reg_size 数组包含了多达六个 Base Address Registers or BARs 的信息。reg_base 存储了 memory-mapped I/O 区域的内存基址，reg_size 包含了对应的 reg_baseI/O 端口基础值（reg_base）的字节数（或数字），irq_line 包含了给设备中断分配的 IRQ 线。E1000 BARs 的特定含义在表 4-2 的后半部分给出。

当这个设备的`attachfn`被调用，设备就被发现了，但是还没有被启用。这就意味着 PCI 代码还没有决定要分配哪些资源给设备，比如说地址空间、IRQ 线，也就是 struct pci_func 的最后三个元素还没有被填充。`attachfn`应该调用`pci_func_enable`，它会启用该设备，协调这些资源，并且填充`struct pci_func`。

练习 3：实现一个初始化 E1000 的`attachfn`。添加一个项到`kern/pci.c`的`pci_attach_vendor`数组里，用它来触发你的函数（一旦匹配中的 PCI 设备被发现了），确保在标记表末的{0,0,0}项之前把它添加进去。你可以在小结 5.2 里找到 QEMU 模拟的 82540EM 的厂商 ID 和设备 ID。你也应该在 JOS 扫描 PCI 总线时查看这个清单。

到现在，通过 pci_func_enable 来开启 E1000 设备。本实验的后面，我们会添加更多初始化功能进去。

我们已经提供了 kern/e1000.c 和 kern/e1000.h 给你，所以你不需要弄乱构建系统。它们现在是空白的；你在这个练习里需要去填充它们。你可能也需要在内核的其他地方 include e1000.h。

当你 boot 你的内核时，你应该看到它打印了，E1000 网卡的 PCI 函数被开启了。

#### Memory-mapped I/O

#### DMA

### Transmitting Packets

#### C Structures

### Transmitting Packets: Network Server

## Part B: Receiving packets and the web server

### Receiving Packets

### Receiving Packets: Network Server

### The Web Server

```

```
