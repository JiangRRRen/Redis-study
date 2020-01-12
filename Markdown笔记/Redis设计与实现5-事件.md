---
title: Redis设计与实现5-事件
category:
  - 数据库
tags:
- 计算机网络
- 数据库
- Redis
- 读书笔记
mathjax: true
date: 2020-01-05 13:17:25
---

Redis是一个**事件驱动程序**，前面提到，服务器需要处理文件事件和时间事件。<!--more-->

- **文件事件**：Redis服务器通过套接字与客户端（或者其他Redis服务器）进行连接，而**文件事件就是服务器对套接字操作的抽象**。
- **时间事件**：些操作会在给定的时间点进行，对这类**定时操作的抽象就是时间事件。**

# 1. 文件事件

Redis基于Reactor模式开发了自己的网络事件处理器：这个处理器被称为**文件事件处理器（file event handler）**

> Reactor模式用于高并发，依靠事件驱动。传统的线程连接中，IO连接后需要等待客户的请求。而事件驱动中，IO可以干别的事情，等客户发来请求后再处理。

在Redis中，

- 文件事件处理器使用I/O多路复用（multiplexing）程序来同时监听多个套接字，并根据套接字目前执行的任务来为套接字关联不同的事件处理器。
- 当被监听的套接字准备好执行连接应答（accept）、读取（read）、写入（write）、关闭（close）等操作时，与操作相对应的文件事件就会产生，这时文件事件处理器就会调用套接字之前关联好的事件处理器来处理这些事件。

虽然文件事件处理器**以单线程方式运**行，但通过使用**I/O多路复用**程序来监听多个套接字，文件事件处理器既实现了高性能的网络通信模型，又可以很好地与Redis服务器中其他**同样以单线程方式运行的模块进行对接**，这保持了**Redis内部单线程设计的简单性。**

## 1.1 构成

文件事件处理器的四个组成部分，它们分别是**套接字**、**I/O多路复用程序**、**文件事件分派器（dispatcher）**，以及**事件处理器**。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105132923.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

前面提到，文件事件是对套接字操作的抽象，**当一个套接字准备好后，就会产生一个文件事件。**

尽管多个文件事件可能会并发地出现，但I/O多路复用程序总是会**将所有产生事件的套接字都放到一个队列里面**，然后通过这个队列，以**有序（sequentially）**、**同步（synchronously）**、**每次一个套接字**的方式向文件事件分派器传送套接字。**只有当上一个套接字处理完毕后，复用程序才会向分派器传送下一个套接字。**

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105133514.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

## 1.2 IO多路复用程序的实现

Redis的I/O多路复用程序的所有功能都是**通过包装常见的select、epoll、evport和kqueue这些I/O多路复用函数库来实现的**，每个I/O多路复用函数库在Redis源码中都对应一个单独的文件，比如ae_select.c、ae_epoll.c、ae_kqueue.c。

> ae表示A simple event-driven programming library，一个简单的事件驱动程序库

由于IO复用程序提供了统一的接口，所以**底层实现方法可以互换。**

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105133854.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

## 1.3 事件类型

I/O多路复用程序可以**同时**监听多个套接字的`ae.h/AE_READABLE`和`ae.h/AE_WRITABLE`这两种事件，这两类事件和套接字操作之间的对应关系如下：

- 客户端对套接字执行write操作，客户端对服务器的监听套接字执行connect操作。此时套接字对服务器**变为可读状态**，就会产生`AE_READABLE`事件。
- 客户端对套接字执行read操作。此时套接字对服务器变为**可写状态**，就会产生`AR_WRITABLE`事件。

虽然是可以同时处理这两种事件，但**优先处理可写事件。**

## 1.4 事件处理器

事件处理器有很多，最常用的是**通信的连接应答处理器**、**命令请求处理器**和**命令回复处理器**。

**（1）连接应答处理器**

`networking.c/acceptTcpHandler`函数是Redis的连接应答处理器，具体实现为`sys/socket.h/accept`函数的包装。

当Redis服务器进行**初始化**的时候，程序会将**连接应答处理器**和**服务器监听套接字的`AE_READABLE`事件**关联起来，当有客户端用`sys/socket.h/connec`t函数连接服务器监听套接字的时候，**套接字就会产生`AE_READABLE`事件，引发连接应答处理器执行**。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105143306.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

**（2）命令请求处理器**

`networking.c/readQueryFromClient`函数是Redis的命令请求处理器，这个处理器负责从套接字中读入客户端发送的命令请求内容，具体实现为`unistd.h/read`函数的包装。

和上面一样，当客户端**通过连接应答处理器成功连接到服务器后**，服务器会将**客户端套接字的AE_READABLE事件**和**命令请求处理器**关联起来，当客户端向服务器发送命令请求的时候，**套接字就会产生AE_READABLE事件**，**引发命令请求处理器执行**。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105143323.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

**（3）命令回复处理器**

`networking.c/sendReplyToClient`函数是Redis的命令回复处理器，这个处理器**负责将服务器执行命令后得到的命令通过套接字返回给客户端**，具体实现为`unistd.h/write`函数的包装。

当服务器有命令回复需要传送给客户端的时候，服务器会将**客户端套接字的AE_WRITABLE事件**和**命令回复处理器**关联起来，当客户端准备好接收服务器传回的命令回复时，就会**产生AE_WRITABLE事件，引发命令回复处理器执行**。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105143336.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

# 2. 时间事件

Redis时间事件分为两类：

- **定时事件**：程序在指定时间后执行一次。
- **周期性事件**：每隔一段时间就执行，循环往复。

一个时间事件主要由以下三个属性组成：

- **id**：服务器为时间事件创造全局唯一ID作为识别，新事件比旧事件号码要大。
- **when**：毫秒级UNIX时间戳，记录时间事件到达时间。
- **timeProc**：时间事件处理器，到时间后处理事件。

## 2.1 构成

服务器将所有时间事件都放在一个**无序链表**中，每当时间事件执行器运行时，它就**遍历整个链表**，**查找所有已到达的时间事件**，并调用相应的事件处理器。

因为新的事件总是放在表头，所以三个时间事件分别按逆序ID排列：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105150113.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

注意，我们说保存时间事件的链表为无序链表，指的不是链表不按ID排序，而是说，**该链表不按when属性的大小排序**。

## 2.2 API

`ae.c/aeCreateTimeEvent`函数接受一个毫秒数milliseconds和一个时间事件处理器proc作为参数，**将一个新的时间事件添加到服务器**。

`ae.c/aeDeleteFileEvent`函数接受一个时间事件ID作为参数，然后从服务器中**删除**该ID所对应的时间事件。

`ae.c/aeSearchNearestTimer`函数返回到达时间距离当前时间最接近的那个时间事件。

`ae.c/processTimeEvents`函数是时间事件的执行器，这个函数会**遍历所有已到达的时间事件**，并调**用这些事件的处理器**。已到达指的是，时间事件的when属性记录的UNIX时间戳等于或小于当前时间的UNIX时间戳。

## 2.3 severCron函数

持续运行的Redis服务器需要**定期**对自身的资源和状态进行检查和调整，这些定期操作由`redis.c/serverCron`函数负责执行，它的主要工作包括：

- 更新服务器统计信息，包括事件、内存占用等情况
- 清理过期键值对
- 关闭和清理失效的客户端连接
- AOF和RDB持久化操作
- 如果sever是主服务器，则对从服务器进行定期同步
- 如果是集群模式，对集群进行定期同步和连接测试

> cron在unix中表示计划任务，计时程序

默认频率是100毫秒一次，用户可以在redis.conf中修改hz选项来改变。

## 2.3 事件的调度与执行

因为服务器中同时存在文件事件和时间事件两种事件类型，所以服务器必须对这两种事件进行调度，**决定何时应该处理什么文件，以及花多少时间来处理它们等等。**事件的调度和执行由`ae.c/aeProcessEvents`函数负责。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105152029.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

对事件处理的原则是：

- 如果等待并处理完一次文件事件之后，仍未有任何时间事件到达，那么服务器将**再次等待并处理文件事件。**
- 对两种事件处理都是**同步、有序、原子**地执行的，服务器**不会中途中断事件处理，也不会对事件进行抢占**，因此需要尽可能地减少程序的阻塞时间，并在有需要时主动让出执行权。（比如写入字节太长，命令回复处理器就会break跳出，将余下的数据留到下次）
- 由于不能抢占，时间事件到达后需要等待文件事件处理完成，所以**一般会稍晚于到达时间。**

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105152515.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">