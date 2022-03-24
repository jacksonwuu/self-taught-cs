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

The core network server environment is composed of the socket call dispatcher and lwIP itself. The socket call dispatcher works exactly like the file server. User environments use stubs (found in lib/nsipc.c) to send IPC messages to the core network environment. If you look at lib/nsipc.c you will see that we find the core network server the same way we found the file server: i386_init created the NS environment with NS_TYPE_NS, so we scan envs, looking for this special environment type. For each user environment IPC, the dispatcher in the network server calls the appropriate BSD socket interface function provided by lwIP on behalf of the user.

Regular user environments do not use the nsipc\_\* calls directly. Instead, they use the functions in lib/sockets.c, which provides a file descriptor-based sockets API. Thus, user environments refer to sockets via file descriptors, just like how they referred to on-disk files. A number of operations (connect, accept, etc.) are specific to sockets, but read, write, and close go through the normal file descriptor device-dispatch code in lib/fd.c. Much like how the file server maintained internal unique ID's for all open files, lwIP also generates unique ID's for all open sockets. In both the file server and the network server, we use information stored in struct Fd to map per-environment file descriptors to these unique ID spaces.

Even though it may seem that the IPC dispatchers of the file server and network server act the same, there is a key difference. BSD socket calls like accept and recv can block indefinitely. If the dispatcher were to let lwIP execute one of these blocking calls, the dispatcher would also block and there could only be one outstanding network call at a time for the whole system. Since this is unacceptable, the network server uses user-level threading to avoid blocking the entire server environment. For every incoming IPC message, the dispatcher creates a thread and processes the request in the newly created thread. If the thread blocks, then only that thread is put to sleep while other threads continue to run.

In addition to the core network environment there are three helper environments. Besides accepting messages from user applications, the core network environment's dispatcher also accepts messages from the input and timer environments.

#### The Output Environment

When servicing user environment socket calls, lwIP will generate packets for the network card to transmit. LwIP will send each packet to be transmitted to the output helper environment using the NSREQ_OUTPUT IPC message with the packet attached in the page argument of the IPC message. The output environment is responsible for accepting these messages and forwarding the packet on to the device driver via the system call interface that you will soon create.

#### The Input Environment

Packets received by the network card need to be injected into lwIP. For every packet received by the device driver, the input environment pulls the packet out of kernel space (using kernel system calls that you will implement) and sends the packet to the core server environment using the NSREQ_INPUT IPC message.

The packet input functionality is separated from the core network environment because JOS makes it hard to simultaneously accept IPC messages and poll or wait for a packet from the device driver. We do not have a select system call in JOS that would allow environments to monitor multiple input sources to identify which input is ready to be processed.

If you take a look at net/input.c and net/output.c you will see that both need to be implemented. This is mainly because the implementation depends on your system call interface. You will write the code for the two helper environments after you implement the driver and system call interface.

#### The Timer Environment

The timer environment periodically sends messages of type NSREQ_TIMER to the core network server notifying it that a timer has expired. The timer messages from this thread are used by lwIP to implement various network timeouts.

## Part A: Initialization and transmitting packets

### The Network Interface Card

#### PCI Interface

#### Memory-mapped I/O

#### DMA

### Transmitting Packets

#### C Structures

### Transmitting Packets: Network Server

## Part B: Receiving packets and the web server

### Receiving Packets

### Receiving Packets: Network Server

### The Web Server
