# Homework: xv6 lazy page allocation

Lazy allocation simply means not allocating a resource until it is actually needed. This is common with singleton objects, but strictly speaking, any time a resource is allocated as late as possible, you have an example of lazy allocation.


懒加载是操作系统里普遍的设计理念，只有当一个资源真正需要的时候，才分配给申请者。

操作系统里有一个概念叫做“写时复制”，就是新进程需要对内存进行写操作的时候才分配新内存给它。

