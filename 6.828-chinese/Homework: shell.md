# Homework: shell

This assignment will make you more familiar with the Unix system call interface and the shell by implementing several features in a small shell, which we will refer to as the 6.828 shell.

先阅读 xv6-book 的第 0 章。

The 6.828 shell contains two main parts: parsing shell commands and implementing them.

```shell
ls > y
cat < y | sort | uniq | wc > y1
cat y1
rm y1
ls |  sort | uniq | wc
rm y
```

为了给这个 shell 添加功能，首先理解 shell 做了什么，其实 shell 的工作主要分为两个步骤，第一步骤是解析输入的命令字符串，第二步骤是创建子进程去执行相应的命令。

所以要改变 parser 和 runcmd 这两块的代码。

理解这个 shell 程序的数据数据结构是第一步，也是最重要的。

#### I/O 重定向的实现

在此之前要理解文件描述符 fd 和相关的系统调用：open,close,write,read。

重定向其实就是通过把特殊的文件描述符组合起来实现的。

#### 管道的实现

管道其实就是把前一个管道的标准输出和后一个管道的标准输入连接起来实现的。其实也是 I/O 重定向的一种。

#### 额外挑战

如果要添加一些额外的功能，要考虑改造数据结构。

-   Implement lists of commands, separated by ";"
-   Implement sub shells by implementing "(" and ")"
-   Implement running commands in the background by supporting "&" and "wait"
