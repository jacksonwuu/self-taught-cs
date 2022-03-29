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

写一个驱动程序需要对硬件有很深的认识，以及用硬件在软件的表现。这个实验的说明会提供关于如何和 E1000 接口协作的高度概览，但是当你写驱动程序的时候，你也要看额外再看一下 Intel 手册。

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

软件通过 MMIO 和 E1000 进行通信。你在这之前已经在 JOS 里看到两次了：CGA console 和 LAPIC 都是那种你可以控制并通过读写“内存”来的设备。但是这些读写不会进入 DRAM，而是直接进入这些设备。

`pci_func_enable`会与 E1000 协调一个 MMIO 区域，并存储它的基址和大小到 BAR0 里（也就是 reg_base[0]和 reg_size[0]）。这是一个分配给设备的物理内存地址范围，这意味着你必须通过虚拟地址来访问它。MMIO 区域被分配了非常高的物理地址(通常超过 3GB)，因为 JOS 的 256MB 限制，你不能使用 KADDR 访问它。因此，您必须创建一个新的内存映射。我们将使用 MMIOBASE 上面的区域(来自 lab 4 的 mmio_map_region 将确保我们不会覆盖 LAPIC 使用的映射)。因为 PCI 设备初始化发生在 JOS 创建用户环境之前，所以可以在 kern_pgdir 中创建映射，这样映射就在之后一直是可用的。

练习 4：在你的附着函数里，调用 mmio_map_region（这是你在 lab 4 里写的）来为 E1000 的 BAR0 创建一个虚拟映射。

您将希望在一个变量中记录这个映射的位置，以便以后可以访问刚刚映射的寄存器。查看 kern/lapic.c 中的 lapic 变量，作为一种实现方法的示例。如果你确实使用了指向设备寄存器映射的指针，一定要声明它为 volatile;否则，编译器被允许缓存值并重新排序访问该内存。

为了测试您的映射，尝试打印出设备状态寄存器(第 13.4.2 节)。这是一个 4 字节的寄存器，从寄存器空间的第 8 字节开始。您应该得到 0x80080783，它表示一个全双工链路以 1000 MB/s 的速度运行。

提示:您需要很多常量，比如寄存器的位置和位掩码的值。试图从开发人员手册中复制这些内容很容易出错，错误可能会导致痛苦的调试。我们建议使用 QEMU 的 e1000_wh .h 头文件作为指导方针。我们不建议逐字复制它，因为它定义的内容远比您实际需要的多，而且可能不会以您需要的方式定义内容，但它是一个很好的起点。

#### DMA

你可以想象通过从 E1000 的寄存器中读写来发送和接收数据包，但是这将会很慢，并且需要 E1000 在内部缓冲数据包数据。相反，E1000 使用直接内存访问(Direct Memory Access)或 DMA 直接从内存读取和写入数据包数据，而不需要 CPU 的参与。驱动程序负责为发送和接收队列分配内存，设置 DMA 描述符，并使用这些队列的位置配置 E1000，但之后的一切都是异步的。为了传输一个包，驱动程序将它复制到传输队列中的下一个 DMA 描述符中，并通知 E1000 有一个包可用;当有时间发送数据包时，E1000 将从描述符中复制数据。同样地，当 E1000 接收到一个包时，它将它复制到接收队列中的下一个 DMA 描述符中，驱动程序可以在下一次读取这个描述符。

接收和发送队列在高度相似。两者都由一系列描述符组成。虽然这些描述符的确切结构各不相同，但每个描述符都包含一些标志和包含包数据的缓冲区的物理地址(可以是卡要发送的包数据，也可以是操作系统分配给卡用来写入接收到的包的缓冲区)。

队列是用环形数组实现的，这意味着当卡或驱动程序到达数组的末尾时，它会绕回到数组的开头。它们都有一个头指针和一个尾指针，队列的内容是这两个指针之间的描述符。硬件总是从头部使用描述符并移动头部指针，而驱动总是将描述符添加到尾部并移动尾部指针。发送队列中的描述符表示等待发送的包（因此，在稳定状态下，发送队列为空)。对于接收队列，网卡可以把接收到的数据包放进队列中的空闲描述符(因此，在稳定状态下，接收队列由所有可用的接收描述符组成）。正确地更新尾部寄存器而不混淆 E1000 是棘手的；小心!

指向这些数组的指针以及描述符中包缓冲区的地址必须都是物理地址，因为硬件直接在物理 RAM 和物理 RAM 之间执行 DMA，而不需要通过 MMU。

### Transmitting Packets

E1000 的发送和接收功能基本上是独立的，所以我们可以一次做一个。我们首先做发送数据包，因为如果不先发送一个“我在这里”的数据包，那我们就没法测试接收。

首先，你必须按照 14.5 节中描述的步骤来初始化网卡发送功能。发送初始化的第一步是设置发送队列。队列的精确结构在 3.4 节中有描述，描述符的结构在 3.3.3 节中有描述。我们不会使用 E1000 的 TCP offload 特性，所以你可以专注于把重点放在“传统的传输描述符格式（legacy transmit descriptor format）”上。现在你应该阅读这些部分，熟悉这些结构。

#### C Structures

你会发现使用 C 结构体描述 E1000 的结构非常方便。正如你已经看到的结构，如结构 Trapframe，C 结构让你精确地在内存中布局数据。C 可以在字段之间插入填充来对齐，但是 E1000 的结构是这样布置的，内存对齐这个应该不是一个问题。如果您确实遇到了字段对齐问题，请查看 GCC 的“packed”属性。

作为一个例子，看看手册表里 3-8 中给出的旧的传输描述符:

```
  63            48 47   40 39   32 31   24 23   16 15             0
  +---------------------------------------------------------------+
  |                         Buffer address                        |
  +---------------+-------+-------+-------+-------+---------------+
  |    Special    |  CSS  | Status|  Cmd  |  CSO  |    Length     |
  +---------------+-------+-------+-------+-------+---------------+
```

结构体的第一个字节从左上角开始，所以先把这个转化为 C 结构体，从右边读取到左边，从顶部到底部。如果你仔细看，你会发现，所有的字段都很适合标准大小的类型。

```c
struct tx_desc
{
    uint64_t addr;
    uint16_t length;
    uint8_t cso;
    uint8_t cmd;
    uint8_t status;
    uint8_t css;
    uint16_t special;
};
```

你的驱动程序必须为发送描述符数组和指向发送描述符的包缓冲区预留内存。有几种方法你可以做到这一点，从动态分配页面到简单地在全局变量中声明它们。不管你选择的是哪一种，请记住，E1000 直接访问物理内存，这意味着它访问的任何缓冲区都必须是连续的物理内存。

还有多种方法可以处理数据包缓冲。我们建议从最简单的开始，即在驱动程序初始化期间为每个描述符预留一个包缓冲区的空间，并将包数据复制到这些预分配的缓冲区中或从这些缓冲区中取出。以太网数据包的最大大小是 1518 字节，这可以拿来限定这些缓冲区需要的大小。更复杂的驱动程序可以动态分配包缓冲区（比如说在网络使用率低的时候减少内存开销），甚至传递由用户空间直接提供的缓冲区（一种称为“零拷贝”的技术），但是我们最好先从简单的开始。

练习 5：执行第 14.5 节(但不是它的子节)中描述的初始化步骤。使用第 13 节作为寄存器的参考，初始化过程引用寄存器，第 3.3.3 和 3.4 节引用发送描述符和发送描述符数组。

请注意发送描述符数组的对齐要求和该数组的长度限制。由于 TDLEN 必须是 128 字节对齐的，并且每个发送描述符都是 16 字节，所以您的发送描述符数组将需要 8 倍的发送描述符空间。但是，不要使用超过 64 个描述符，否则我们的测试将无法测试发送环溢出。

对于 TCTL.COLD，你可以假设全双工操作。对于 TIPG，请参考 IEEE 802.3 标准 IPG 中的 13.4.34 节，表 13-77 中描述的默认值（不要使用 14.5 节表中的值）。

尝试运行`make E1000_DEBUG=TXERR,TX qemu`。If you are using the course qemu, you should see an "e1000: tx disabled" message when you set the TDT register (since this happens before you set TCTL.EN) and no further "e1000" messages.

现在，发送功能已经初始化了，你必须编写代码来发送一个包，并通过系统调用让用户空间可以访问它。要发送一个包，你需要把它添加到发送队列的末尾，这意味着将数据包数据复制到下一个数据包缓冲区，然后更新 TDT(发送描述符尾部)寄存器，以通知网卡在发送队列中有另一个数据包。(注意 TDT 是发送描述符数组的索引，而不是字节偏移量；文档对此并不是写地很清楚。)

然而，发送队列只有这么大。如果网卡落后于传输包，且传输队列已满，会发生什么？为了检测这个条件，您需要 E1000 提供一些反馈。不幸的是，你不能只使用 TDH(发送描述符头)寄存器；文档明确指出，从软件读取这个寄存器是不可靠的。然而，如果您在传输描述符的命令字段中设置 RS 位，那么，当网卡在该描述符中传输了数据包时，网卡将在描述符的状态字段中设置 DD 位。如果设置了描述符的 DD 位，那么就可以安全地回收该描述符并使用它来传输另一个包。

如果用户调用了您的传输系统调用，但是下一个描述符的 DD 位没有设置，表明传输队列已满，该怎么办?在这种情况下，你必须决定怎么做。你可以直接扔掉这个包裹。网络协议对此很有弹性，但如果您丢弃了大量的数据包，协议可能无法恢复。相反，您可以告诉用户环境必须重试，就像您对 sys_ipc_try_send 所做的那样。这样做的好处是可以推迟生成数据的环境（进程）。

练习 6：编写一个函数，通过检查下一个描述符是否空闲，将包数据复制到下一个描述符，并更新 TDT 来传输包。确保传输队列已满。

现在是测试数据包传输代码的好时机。通过直接从内核调用传输函数，尝试只传输几个包。您不必为了测试这一点而创建符合任何特定网络协议的数据包。运行`make E1000_DEBUG=TXERR,TX qemu`运行测试。你应该看到

```
e1000: index 0: 0x271f00 : 9000002a 0
...
```

当你传输数据包时。每一行都给出了发送数组中的索引、该发送描述符的缓存地址、cmd/CSO/length 字段，special/CSS/status 字段。如果 QEMU 没有打印您希望从传输描述符中得到的值，请检查您是否填写了正确的描述符，以及是否正确配置了 TDBAL 和 TDBAH。如果你得到了“e1000: TDH wraparound @0, TDT x, TDLEN y”这样的消息，那就意味着 E1000 一直运行在发送队列中而没有停止（如果 QEMU 没有检查这个，它将进入一个无限循环），这可能意味着您没有正确地操作 TDT。如果你得到大量的“e1000: tx disabled”消息，那么你没有设置对发送控制寄存器。

一旦 QEMU 运行，您就可以运行`tcpdump -XXnr QEMU`。Pcap 查看你传输的数据包数据。如果您看到预期的“e1000: index”消息来自 QEMU，但是您的包捕获是空的，请仔细检查您是否在发送描述符中填写了所有必要的字段和位(e1000 可能检查了您的发送描述符，但不认为它必须发送任何东西)。

练习 7：添加一个允许您从用户空间传输数据包的系统调用。具体的接口由您决定。不要忘记检查从用户空间传递到内核的指针。

### Transmitting Packets: Network Server

现在已经有了设备驱动程序的传输侧的系统调用接口，是时候该发送包了。输出助手环境的目标是循环执行以下操作：接受来自核心网络服务器的 NSREQ_OUTPUT IPC 消息，并使用上面添加的系统调用，并将包伴随这这些 IPC 消息发送到网络设备驱动程序。NSREQ_OUTPUT IPC 是由 net/lwip/jos/jif/jif.c 中的 low_level_output 函数发送的，它将 lwIP 堆栈粘到 JOS 的网络系统上。每个 IPC 会包括进一个页，这个页由一个 Nsipc 的 union 和它的结构 jif_pkt pkt 字段中的包组成(参见 inc/ns.h)。结构 jif_pkt 看起来是这样的：

```c
struct jif_pkt {
    int jp_len;
    char jp_data[0];
};
```

jp_len 表示数据包的长度。IPC 页上的所有后续字节都专用于包的内容。在结构的末尾使用 jp_data 这样的零长度数组是一个常见的 C 技巧(有些人会说讨厌)，用于表示没有预先确定长度的缓冲区。由于 C 语言不做数组边界检查，只要确保 struct 后面有足够的未使用内存，就可以像使用任意大小的数组一样使用 jp_data。

当设备驱动程序的传输队列中没有更多空间时，要注意设备驱动程序、输出环境和核心网络服务器之间的交互。核心网络服务器通过 IPC 向输出环境发送数据包。如果输出环境由于发送包系统调用而挂起，因为驱动程序没有更多的缓冲空间用于新的包，核心网络服务器将阻塞，等待输出服务器接受 IPC 调用。

练习 8：实现 net/output.c.

你可以使用 net/testoutput.c 来测试你的输出代码，不需要整个网络服务器参与进来。尝试运行 `make E1000_DEBUG=TXERR,TX run-net_testoutput`。你应该看到如下：

```
Transmitting packet 0
e1000: index 0: 0x271f00 : 9000009 0
Transmitting packet 1
e1000: index 1: 0x2724ee : 9000009 0
...
```

以及 `tcpdump -XXnr qemu.pcap` 会输出：

```
reading from file qemu.pcap, link-type EN10MB (Ethernet)
-5:00:00.600186 [|ether]
        0x0000: 5061 636b 6574 2030 30 Packet.00
-5:00:00.610080 [|ether]
        0x0000: 5061 636b 6574 2030 31 Packet.01
...
```

为了测试更大的数据包量，尝试`make E1000_DEBUG=TXERR,TX NET_CFLAGS=-DTESTOUTPUT_COUNT=100 run-net_testoutput`。如果这个让你的发送环移除了，那就多检查一下你正确处理了 DD 状态位，以及你是否正确地让硬件也设置了 DD 状态位（用 RS 命令位）。

你的代码应该通过`make grade`测试输出。

问题 1：你如何组织你的发送实现的？详细来说，如果发送环满了你会怎么做？

## Part B: Receiving packets and the web server

### Receiving Packets

Just like you did for transmitting packets, you'll have to configure the E1000 to receive packets and provide a receive descriptor queue and receive descriptors. Section 3.2 describes how packet reception works, including the receive queue structure and receive descriptors, and the initialization process is detailed in section 14.4.

Exercise 9. Read section 3.2. You can ignore anything about interrupts and checksum offloading (you can return to these sections if you decide to use these features later), and you don't have to be concerned with the details of thresholds and how the card's internal caches work.

The receive queue is very similar to the transmit queue, except that it consists of empty packet buffers waiting to be filled with incoming packets. Hence, when the network is idle, the transmit queue is empty (because all packets have been sent), but the receive queue is full (of empty packet buffers).

When the E1000 receives a packet, it first checks if it matches the card's configured filters (for example, to see if the packet is addressed to this E1000's MAC address) and ignores the packet if it doesn't match any filters. Otherwise, the E1000 tries to retrieve the next receive descriptor from the head of the receive queue. If the head (RDH) has caught up with the tail (RDT), then the receive queue is out of free descriptors, so the card drops the packet. If there is a free receive descriptor, it copies the packet data into the buffer pointed to by the descriptor, sets the descriptor's DD (Descriptor Done) and EOP (End of Packet) status bits, and increments the RDH.

If the E1000 receives a packet that is larger than the packet buffer in one receive descriptor, it will retrieve as many descriptors as necessary from the receive queue to store the entire contents of the packet. To indicate that this has happened, it will set the DD status bit on all of these descriptors, but only set the EOP status bit on the last of these descriptors. You can either deal with this possibility in your driver, or simply configure the card to not accept "long packets" (also known as jumbo frames) and make sure your receive buffers are large enough to store the largest possible standard Ethernet packet (1518 bytes).

Exercise 10. Set up the receive queue and configure the E1000 by following the process in section 14.4. You don't have to support "long packets" or multicast. For now, don't configure the card to use interrupts; you can change that later if you decide to use receive interrupts. Also, configure the E1000 to strip the Ethernet CRC, since the grade script expects it to be stripped.

By default, the card will filter out all packets. You have to configure the Receive Address Registers (RAL and RAH) with the card's own MAC address in order to accept packets addressed to that card. You can simply hard-code QEMU's default MAC address of 52:54:00:12:34:56 (we already hard-code this in lwIP, so doing it here too doesn't make things any worse). Be very careful with the byte order; MAC addresses are written from lowest-order byte to highest-order byte, so 52:54:00:12 are the low-order 32 bits of the MAC address and 34:56 are the high-order 16 bits.

The E1000 only supports a specific set of receive buffer sizes (given in the description of RCTL.BSIZE in 13.4.22). If you make your receive packet buffers large enough and disable long packets, you won't have to worry about packets spanning multiple receive buffers. Also, remember that, just like for transmit, the receive queue and the packet buffers must be contiguous in physical memory.

You should use at least 128 receive descriptors

You can do a basic test of receive functionality now, even without writing the code to receive packets. Run make E1000_DEBUG=TX,TXERR,RX,RXERR,RXFILTER run-net_testinput. testinput will transmit an ARP (Address Resolution Protocol) announcement packet (using your packet transmitting system call), which QEMU will automatically reply to. Even though your driver can't receive this reply yet, you should see a "e1000: unicast match[0]: 52:54:00:12:34:56" message, indicating that a packet was received by the E1000 and matched the configured receive filter. If you see a "e1000: unicast mismatch: 52:54:00:12:34:56" message instead, the E1000 filtered out the packet, which means you probably didn't configure RAL and RAH correctly. Make sure you got the byte ordering right and didn't forget to set the "Address Valid" bit in RAH. If you don't get any "e1000" messages, you probably didn't enable receive correctly.

Now you're ready to implement receiving packets. To receive a packet, your driver will have to keep track of which descriptor it expects to hold the next received packet (hint: depending on your design, there's probably already a register in the E1000 keeping track of this). Similar to transmit, the documentation states that the RDH register cannot be reliably read from software, so in order to determine if a packet has been delivered to this descriptor's packet buffer, you'll have to read the DD status bit in the descriptor. If the DD bit is set, you can copy the packet data out of that descriptor's packet buffer and then tell the card that the descriptor is free by updating the queue's tail index, RDT.

If the DD bit isn't set, then no packet has been received. This is the receive-side equivalent of when the transmit queue was full, and there are several things you can do in this situation. You can simply return a "try again" error and require the caller to retry. While this approach works well for full transmit queues because that's a transient condition, it is less justifiable for empty receive queues because the receive queue may remain empty for long stretches of time. A second approach is to suspend the calling environment until there are packets in the receive queue to process. This tactic is very similar to sys_ipc_recv. Just like in the IPC case, since we have only one kernel stack per CPU, as soon as we leave the kernel the state on the stack will be lost. We need to set a flag indicating that an environment has been suspended by receive queue underflow and record the system call arguments. The drawback of this approach is complexity: the E1000 must be instructed to generate receive interrupts and the driver must handle them in order to resume the environment blocked waiting for a packet.

Exercise 11. Write a function to receive a packet from the E1000 and expose it to user space by adding a system call. Make sure you handle the receive queue being empty.

### Receiving Packets: Network Server

In the network server input environment, you will need to use your new receive system call to receive packets and pass them to the core network server environment using the NSREQ_INPUT IPC message. These IPC input message should have a page attached with a union Nsipc with its struct jif_pkt pkt field filled in with the packet received from the network.

Exercise 12. Implement net/input.c.

Run testinput again with make E1000_DEBUG=TX,TXERR,RX,RXERR,RXFILTER run-net_testinput. You should see

```
Sending ARP announcement...
Waiting for packets...
e1000: index 0: 0x26dea0 : 900002a 0
e1000: unicast match[0]: 52:54:00:12:34:56
input: 0000   5254 0012 3456 5255  0a00 0202 0806 0001
input: 0010   0800 0604 0002 5255  0a00 0202 0a00 0202
input: 0020   5254 0012 3456 0a00  020f 0000 0000 0000
input: 0030   0000 0000 0000 0000  0000 0000 0000 0000
```

The lines beginning with "input:" are a hexdump of QEMU's ARP reply.

Your code should pass the testinput tests of make grade. Note that there's no way to test packet receiving without sending at least one ARP packet to inform QEMU of JOS' IP address, so bugs in your transmitting code can cause this test to fail.

To more thoroughly test your networking code, we have provided a daemon called echosrv that sets up an echo server running on port 7 that will echo back anything sent over a TCP connection. Use make E1000_DEBUG=TX,TXERR,RX,RXERR,RXFILTER run-echosrv to start the echo server in one terminal and make nc-7 in another to connect to it. Every line you type should be echoed back by the server. Every time the emulated E1000 receives a packet, QEMU should print something like the following to the console:

```
e1000: unicast match[0]: 52:54:00:12:34:56
e1000: index 2: 0x26ea7c : 9000036 0
e1000: index 3: 0x26f06a : 9000039 0
e1000: unicast match[0]: 52:54:00:12:34:56
```

At this point, you should also be able to pass the echosrv test.

Question 2: How did you structure your receive implementation? In particular, what do you do if the receive queue is empty and a user environment requests the next incoming packet?

### The Web Server

A web server in its simplest form sends the contents of a file to the requesting client. We have provided skeleton code for a very simple web server in user/httpd.c. The skeleton code deals with incoming connections and parses the headers.

Exercise 13. The web server is missing the code that deals with sending the contents of a file back to the client. Finish the web server by implementing send_file and send_data.

Once you've finished the web server, start the webserver (make run-httpd-nox) and point your favorite browser at http://host:port/index.html, where host is the name of the computer running QEMU (If you're running QEMU on athena use hostname.mit.edu (hostname is the output of the hostname command on athena, or localhost if you're running the web browser and QEMU on the same computer) and port is the port number reported for the web server by make which-ports . You should see a web page served by the HTTP server running inside JOS.

At this point, you should score 105/105 on make grade.

Question 3: What does the web page served by JOS's web server say?
Question 4: How long approximately did it take you to do this lab?

This completes the lab. As usual, don't forget to run make grade and to write up your answers and a description of your challenge exercise solution. Before handing in, use git status and git diff to examine your changes and don't forget to git add answers-lab6.txt. When you're ready, commit your changes with git commit -am 'my solutions to lab 6', then make handin and follow the directions.
