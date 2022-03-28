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

The transmit and receive functions of the E1000 are basically independent of each other, so we can work on one at a time. We'll attack transmitting packets first simply because we can't test receive without transmitting an "I'm here!" packet first.

First, you'll have to initialize the card to transmit, following the steps described in section 14.5 (you don't have to worry about the subsections). The first step of transmit initialization is setting up the transmit queue. The precise structure of the queue is described in section 3.4 and the structure of the descriptors is described in section 3.3.3. We won't be using the TCP offload features of the E1000, so you can focus on the "legacy transmit descriptor format." You should read those sections now and familiarize yourself with these structures.

#### C Structures

You'll find it convenient to use C structs to describe the E1000's structures. As you've seen with structures like the struct Trapframe, C structs let you precisely layout data in memory. C can insert padding between fields, but the E1000's structures are laid out such that this shouldn't be a problem. If you do encounter field alignment problems, look into GCC's "packed" attribute.

As an example, consider the legacy transmit descriptor given in table 3-8 of the manual and reproduced here:

```
  63            48 47   40 39   32 31   24 23   16 15             0
  +---------------------------------------------------------------+
  |                         Buffer address                        |
  +---------------+-------+-------+-------+-------+---------------+
  |    Special    |  CSS  | Status|  Cmd  |  CSO  |    Length     |
  +---------------+-------+-------+-------+-------+---------------+
```

The first byte of the structure starts at the top right, so to convert this into a C struct, read from right to left, top to bottom. If you squint at it right, you'll see that all of the fields even fit nicely into a standard-size types:

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

Your driver will have to reserve memory for the transmit descriptor array and the packet buffers pointed to by the transmit descriptors. There are several ways to do this, ranging from dynamically allocating pages to simply declaring them in global variables. Whatever you choose, keep in mind that the E1000 accesses physical memory directly, which means any buffer it accesses must be contiguous in physical memory.

There are also multiple ways to handle the packet buffers. The simplest, which we recommend starting with, is to reserve space for a packet buffer for each descriptor during driver initialization and simply copy packet data into and out of these pre-allocated buffers. The maximum size of an Ethernet packet is 1518 bytes, which bounds how big these buffers need to be. More sophisticated drivers could dynamically allocate packet buffers (e.g., to reduce memory overhead when network usage is low) or even pass buffers directly provided by user space (a technique known as "zero copy"), but it's good to start simple.

Exercise 5. Perform the initialization steps described in section 14.5 (but not its subsections). Use section 13 as a reference for the registers the initialization process refers to and sections 3.3.3 and 3.4 for reference to the transmit descriptors and transmit descriptor array.

Be mindful of the alignment requirements on the transmit descriptor array and the restrictions on length of this array. Since TDLEN must be 128-byte aligned and each transmit descriptor is 16 bytes, your transmit descriptor array will need some multiple of 8 transmit descriptors. However, don't use more than 64 descriptors or our tests won't be able to test transmit ring overflow.

For the TCTL.COLD, you can assume full-duplex operation. For TIPG, refer to the default values described in table 13-77 of section 13.4.34 for the IEEE 802.3 standard IPG (don't use the values in the table in section 14.5).

Try running make E1000_DEBUG=TXERR,TX qemu. If you are using the course qemu, you should see an "e1000: tx disabled" message when you set the TDT register (since this happens before you set TCTL.EN) and no further "e1000" messages.

Now that transmit is initialized, you'll have to write the code to transmit a packet and make it accessible to user space via a system call. To transmit a packet, you have to add it to the tail of the transmit queue, which means copying the packet data into the next packet buffer and then updating the TDT (transmit descriptor tail) register to inform the card that there's another packet in the transmit queue. (Note that TDT is an index into the transmit descriptor array, not a byte offset; the documentation isn't very clear about this.)

However, the transmit queue is only so big. What happens if the card has fallen behind transmitting packets and the transmit queue is full? In order to detect this condition, you'll need some feedback from the E1000. Unfortunately, you can't just use the TDH (transmit descriptor head) register; the documentation explicitly states that reading this register from software is unreliable. However, if you set the RS bit in the command field of a transmit descriptor, then, when the card has transmitted the packet in that descriptor, the card will set the DD bit in the status field of the descriptor. If a descriptor's DD bit is set, you know it's safe to recycle that descriptor and use it to transmit another packet.

What if the user calls your transmit system call, but the DD bit of the next descriptor isn't set, indicating that the transmit queue is full? You'll have to decide what to do in this situation. You could simply drop the packet. Network protocols are resilient to this, but if you drop a large burst of packets, the protocol may not recover. You could instead tell the user environment that it has to retry, much like you did for sys_ipc_try_send. This has the advantage of pushing back on the environment generating the data.

Exercise 6. Write a function to transmit a packet by checking that the next descriptor is free, copying the packet data into the next descriptor, and updating TDT. Make sure you handle the transmit queue being full.

Now would be a good time to test your packet transmit code. Try transmitting just a few packets by directly calling your transmit function from the kernel. You don't have to create packets that conform to any particular network protocol in order to test this. Run make E1000_DEBUG=TXERR,TX qemu to run your test. You should see something like

e1000: index 0: 0x271f00 : 9000002a 0
...
as you transmit packets. Each line gives the index in the transmit array, the buffer address of that transmit descriptor, the cmd/CSO/length fields, and the special/CSS/status fields. If QEMU doesn't print the values you expected from your transmit descriptor, check that you're filling in the right descriptor and that you configured TDBAL and TDBAH correctly. If you get "e1000: TDH wraparound @0, TDT x, TDLEN y" messages, that means the E1000 ran all the way through the transmit queue without stopping (if QEMU didn't check this, it would enter an infinite loop), which probably means you aren't manipulating TDT correctly. If you get lots of "e1000: tx disabled" messages, then you didn't set the transmit control register right.

Once QEMU runs, you can then run tcpdump -XXnr qemu.pcap to see the packet data that you transmitted. If you saw the expected "e1000: index" messages from QEMU, but your packet capture is empty, double check that you filled in every necessary field and bit in your transmit descriptors (the E1000 probably went through your transmit descriptors, but didn't think it had to send anything).

Exercise 7. Add a system call that lets you transmit packets from user space. The exact interface is up to you. Don't forget to check any pointers passed to the kernel from user space.

### Transmitting Packets: Network Server

Now that you have a system call interface to the transmit side of your device driver, it's time to send packets. The output helper environment's goal is to do the following in a loop: accept NSREQ_OUTPUT IPC messages from the core network server and send the packets accompanying these IPC message to the network device driver using the system call you added above. The NSREQ_OUTPUT IPC's are sent by the low_level_output function in net/lwip/jos/jif/jif.c, which glues the lwIP stack to JOS's network system. Each IPC will include a page consisting of a union Nsipc with the packet in its struct jif_pkt pkt field (see inc/ns.h). struct jif_pkt looks like

```c
struct jif_pkt {
    int jp_len;
    char jp_data[0];
};
```

jp_len represents the length of the packet. All subsequent bytes on the IPC page are dedicated to the packet contents. Using a zero-length array like jp_data at the end of a struct is a common C trick (some would say abomination) for representing buffers without pre-determined lengths. Since C doesn't do array bounds checking, as long as you ensure there's enough unused memory following the struct, you can use jp_data as if it were an array of any size.

Be aware of the interaction between the device driver, the output environment and the core network server when there is no more space in the device driver's transmit queue. The core network server sends packets to the output environment using IPC. If the output environment is suspended due to a send packet system call because the driver has no more buffer space for new packets, the core network server will block waiting for the output server to accept the IPC call.

Exercise 8. Implement net/output.c.

You can use net/testoutput.c to test your output code without involving the whole network server. Try running make E1000_DEBUG=TXERR,TX run-net_testoutput. You should see something like

Transmitting packet 0
e1000: index 0: 0x271f00 : 9000009 0
Transmitting packet 1
e1000: index 1: 0x2724ee : 9000009 0
...
and tcpdump -XXnr qemu.pcap should output

reading from file qemu.pcap, link-type EN10MB (Ethernet)
-5:00:00.600186 [|ether]
0x0000: 5061 636b 6574 2030 30 Packet.00
-5:00:00.610080 [|ether]
0x0000: 5061 636b 6574 2030 31 Packet.01
...
To test with a larger packet count, try make E1000_DEBUG=TXERR,TX NET_CFLAGS=-DTESTOUTPUT_COUNT=100 run-net_testoutput. If this overflows your transmit ring, double check that you're handling the DD status bit correctly and that you've told the hardware to set the DD status bit (using the RS command bit).

Your code should pass the testoutput tests of make grade.

Question1：How did you structure your transmit implementation? In particular, what do you do if the transmit ring is full?

## Part B: Receiving packets and the web server

### Receiving Packets

### Receiving Packets: Network Server

### The Web Server
