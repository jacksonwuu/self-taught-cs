# Lab 6: Network Driver

## Introduction

虽然我们要实现一个网卡驱动，但是这个驱动还不足以让你的操作系统连上互联网。我们在本实验中已经给你提供了一个网络栈和网络服务器。

为了可以写好这个驱动，你要创建一个系统调用接口，以便可以访问网卡驱动。你要实现缺失的代码来在网络栈和驱动之间传递数据包。你也需要组合各种功能来实现一个 web 服务器。有了这个 web 服务器，你就可以利用文件系统来提供文件服务。

大多数内核设备驱动代码你都要从零开始写。这个实验提供的指导比上一个实验少：没有骨架文件，没有写好的系统调用，很多设计由你决定。出于这个原因，我们推荐你把整个指南读完再开始练习。许多学生发现这个实验比上一个实验要难，所以请相应地安排你的时间。

### QEMU 的虚拟网络

我们会使用 QEMU 的用户态网络栈，因为它不需要管理权限就能运行。QEMU 有一些关于 user-net 的[文档](http://wiki.qemu.org/download/qemu-doc.html#Using-the-user-mode-network-stack)。我们会更新 makefile 来开启 QEMU 用户态网络栈和虚拟 E1000 网卡。

While QEMU's virtual network allows JOS to make arbitrary connections out to the Internet, JOS's 10.0.2.15 address has no meaning outside the virtual network running inside QEMU (that is, QEMU acts as a NAT), so we can't connect directly to servers running inside JOS, even from the host running QEMU. To address this, we configure QEMU to run a server on some port on the host machine that simply connects through to some port in JOS and shuttles data back and forth between your real host and the virtual network.

You will run JOS servers on ports 7 (echo) and 80 (http). To avoid collisions on shared Athena machines, the makefile generates forwarding ports for these based on your user ID. To find out what ports QEMU is forwarding to on your development host, run make which-ports. For convenience, the makefile also provides make nc-7 and make nc-80, which allow you to interact directly with servers running on these ports in your terminal. (These targets only connect to a running QEMU instance; you must start QEMU itself separately.)

#### Packet Inspection

#### Debugging the E1000

### The Network Server

Writing a network stack from scratch is hard work.Instead, we will be using lwIP, an open source lightweight TCP/IP protocol suite that among many things includes a network stack. You can find more information on lwIP [here](https://savannah.nongnu.org/projects/lwip/). In this assignment, as far as we are concerned, lwIP is a black box that implements a BSD socket interface and has a packet input port and packet output port.

The network server is actually a combination of four environments:

-   core network server environment (includes socket call dispatcher and lwIP)
-   input environment
-   output environment
-   timer environment

The following diagram shows the different environments and their relationships. The diagram shows the entire system including the device driver, which will be covered later. In this lab, you will implement the parts highlighted in green.

![](https://pdos.csail.mit.edu/6.828/2018/labs/lab6/ns.png)

#### The Core Network Server Environment

#### The Output Environment

#### The Input Environment

#### The Timer Environment

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
