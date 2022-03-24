# Lab 6: Network Driver

## Introduction

虽然我们要实现一个网卡驱动，但是这个驱动还不足以让你的操作系统连上互联网。我们在本实验中已经给你提供了一个网络栈和网络服务器。

为了可以写好这个驱动，你要创建一个系统调用接口，以便可以访问网卡驱动。You will implement missing network server code to transfer packets between the network stack and your driver. You will also tie everything together by finishing a web server. With the new web server you will be able to serve files from your file system.

Much of the kernel device driver code you will have to write yourself from scratch. This lab provides much less guidance than previous labs: there are no skeleton files, no system call interfaces written in stone, and many design decisions are left up to you. For this reason, we recommend that you read the entire assignment write up before starting any individual exercises. Many students find this lab more difficult than previous labs, so please plan your time accordingly.

### QEMU's virtual network

We will be using QEMU's user mode network stack since it requires no administrative privileges to run. QEMU's documentation has more about user-net [here](http://wiki.qemu.org/download/qemu-doc.html#Using-the-user-mode-network-stack). We've updated the makefile to enable QEMU's user-mode network stack and the virtual E1000 network card.

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
