---
title: Redis设计与实现6-客户端与服务器
category:
  - 数据库
tags:
- 计算机网络
- 数据库
- Redis
- 读书笔记
mathjax: true
date: 2020-01-06 10:09:16
---

介绍Redis服务器与客户端。<!--more-->

# 1. 客户端

Redis服务器是典型的一对多服务器程序：一个服务器可以与多个客户端建立网络连接，处理他们的请求。

通过使用**由I/O多路复用技术实现的文件事件处理器**，Redis服务器使用**单线程单进程的方式**来处理命令请求，并与多个客户端进行网络通信。

对于每个与服务器进行连接的客户端，服务器都为这些客户端建立了相应的`redis.h/redisClient`结构（**客户端状态**），这个结构保存了客户端**当前的状态信息**。

在服务器中，用一个**链表**保存客户端的所有状态

```C
struct redisServer 
{ 
    // ... 
    // 一个链表，保存了所有客户端状态 
    list *clients; 
    // ...
};
```

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200106102025.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

## 1.1 客户端属性

**（1）套接字描述符**

客户端状态的fd属性记录了客户端**正在使用的套接字描述符**。根据客户端类型不同，fd的值可以是**-1或大于-1的整数**：

- 伪客户端为-1：伪客户端用于处理的AOF文件或Lua脚本，**而不是网络**。
- 普通客户端为大于-1的整数。

执行CLIENT list命令会列出所有连接到服务器的普通客户端。

```c
redis> CLIENT list
addr=127.0.0.1:53428 fd=6 name= age=1242 idle=0 ...
addr=127.0.0.1:53469 fd=7 name= age=4 idle=4 ...
```

**（2）名字**

默认情况下客户端是没有名字的，比如上面的例子中name处就是空白。使用**CLIENT setname**命令可以为客户端设置一个名字，让客户端的身份变得更清晰。

```C
typedef struct redisClient
{
    //...
    robj *name;
    //...
}redisClient;
```

如果客户端没有名字，那么相应客户端状态的name属性指向NULL指针；相反，如果有名字，那么name属性将指向一个字符串对象。

**（3）标志**

客户端的标志属性flags记录了客户端的角色（role），以及客户端目前所处的状态：

```C
typedef struct redisClient 
{ 
    // ... 
    int flags; 
    // ...
} redisClient;
```

比如`REDIS_BLOCKED`标志表示客户端正在被BRPOP、BLPOP等命令阻塞。flag可以是单个标志，也可以是多个标志的组合。比如：

```C
flags=REDIS_SLAVE | REDIS_PRE_PSYNC;
```

**（4）输入缓冲区**

客户端状态的输入缓冲区用于**保存客户端发送的命令请求**：

```C
typedef struct redisClient 
{ 
    // ... 
    sds querybuf; 
    // ...
} redisClient;
```

保存方式和AOF类似，比如`SET key value`被转化为如下的SDS值：

```
*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n
```

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200106105831.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

**（5）命令与命令参数**

在服务器将客户端发送的命令请求保存到客户端状态的querybuf属性之后，服务器将对命令请求的内容进行**分析**，并将得出的**命令参数**以及**命令参数的个数**分别保存到客户端状态的**argv属性**和**argc属性**：

```C
typedef struct redisClient 
{ 
    // ... 
    robj **argv;
    int argc;
    // ...
} redisClient;
```

argv属性是一个数组，**数组中的每个项都是一个字符串对象**，其中`argv[0]`是**要执行的命令**，而之后的其他项则是**传给命令的参数**。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200106110223.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

**（6）命令的实现函数**

当服务器从协议内容中分析并得出argv属性和argc属性的值之后，服务器**将根据项argv[0]的值，在命令表中查找命令所对应的命令实现函数。**

命令表是一个字典结构，键是SDS结构，保存了命令的名字，**值是redisCommand结构**，保存了：

- 实现函数
- 命令标志
- 命令的参数个数
- 命令的总执行次数
- 总消耗时长

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200106110844.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

当程序在命令表中成功找到argv[0]所对应的redisCommand结构时，**客户端状态的cmd指针指向这个结构**：

```C
typedef struct redisClient 
{ 
    // ... 
    struct redisCommand *cmd;
    // ...
} redisClient;
```

服务器就可以使用cmd属性所指向的redisCommand结构，以及argv、argc属性中保存的命令参数信息，调用命令实现函数，执行客户端指定的命令。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200106114222.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

**（7）输出缓冲区**

**命令回复**会被保存在客户端状态的输出缓冲区里面，每个客户端都有两个输出缓冲区可用，**一个缓冲区的大小是固定的，另一个缓冲区的大小是可变的**：

- 固定大小的缓冲区用于**保存那些长度比较小的回复**，比如OK、简短的字符串值、整数值、错误回复等等。
- 可变大小的缓冲区用于保存那些长度比较大的回复，比如一个非常长的字符串值，一个由很多项组成的列表，一个包含了很多元素的集合等等。

**（8）身份验证**

客户端状态的**authenticated**属性用于记录客户端是否通过了身份验证：

```C
typedef struct redisClient 
{ 
    // ... 
    int authenticated;
    // ...
} redisClient;
```

为0表示没有通过验证，为1表示通过。如果没有通过，**除了AUTH命令之外，客户端发送的所有其他命令都会被服务器拒绝执行**。

```C
redis> SET msg "hello world"
(error) NOAUTH Authentication required.
```

当客户端通过AUTH命令成功进行身份验证之后，客户端状态authenticated属性的值就会从0变为1.

**（9）时间**

客户端还有几个和时间有关的属性：

```C
typedef struct redisClient 
{ // ... 
    time_t ctime; 
    time_t lastinteraction; 
    time_t obuf_soft_limit_reached_time; // ...
} redisClient;
```

- ctime属性记录了创建客户端的时间，这个时间可以用来计算客户端与服务器已经连接了多少秒。
- lastinteraction属性记录了客户端与服务器最后一次进行互动（interaction）的时间。（收或者发命令）
- obuf_soft_limit_reached_time属性记录了输出缓冲区第一次到达软性限制（soft limit）的时间

## 1.2 创建与关闭客户端

**（1）普通客户端**

所谓普通客户端是**指客户端通过网络与服务器连接**，客户端使用connect函数连接到服务器时就会**调用连接事件处理器**。

之后，会将新客户端的状态添加到clients链表的末尾。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200107101305.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

关闭的原因可能有很多种

- 客户端进程退出或被杀死
- 客户端向服务器发送了带有不符合协议格式的命令请求
- 如果客户端成为了CLIENT KILL命令的目标
- 客户端空转超时
- 客户端发送的命令请求的大小超过了输入缓冲区的限制大小（默认为1 GB）
- 如果要发送给客户端的命令回复的大小超过了输出缓冲区的限制大小

除了超过1GB大小的**硬性限制**外，还有**软性限制**，用到了之前提到的`obuf_soft_limit_reached_time`属性。

如果输出缓冲区的大小超过了软性限制所设置的大小，但还没超过硬性限制，那么服务器将使用客户端状态结构的`obuf_soft_limit_reached_time`属性记录下客户端到达软性限制的起始时间。之后服务器会继续监视客户端，**如果输出缓冲区的大小一直超出软性限制，并且持续时间超过服务器设定的时长，那么服务器将关闭客户端**

**（2）Lua脚本的伪客户端**

服务器会在**初始化时**创建负责执行Lua脚本中包含的Redis命令的伪客户端，并将这个伪客户端关联在服务器状态结构的lua_client属性中：

```C
struct redisServer 
{ 
    // ... 
    redisClient *lua_client; 
    // ...
};
```

Lua脚本会一直存在于服务器生命周期，只有服务器被关闭时他才会停止。

服务器在载入AOF文件时，会创建用于执行AOF文件包含的Redis命令的伪客户端，并在载入完成之后，关闭这个伪客户端。

# 2. 服务器

## 2.1 命令请求的执行过程

在处理`SET KEY VALUE`的过程中，客户端和服务器共需要执行以下操作：

1. 客户端向服务器发送命令请求`SET KEY VALUE`
2. 服务器接收并处理，产生回复命令OK
3. 服务器发送OK给客户端
4. 客户端接收到命令，并打印给用户

**（1）发送命令请求**

用户在客户端键入一个请求时，客户端会**将命令请求转换为协议格式**，然后通过套接字发送给服务器。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200107103344.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

**（2）读取命令请求**

当客户端与服务器之间的连接套接字因为客户端的写入而**变得可读**时，服务器将调用命令请求处理器来执行以下操作：

1. 读取套接字中的协议请求，并保存在客户端状态的输入缓冲区内
2. 分析命令，提取命令参数和个数，存在argv和argc属性中
3. 调用命令执行器

**（3）命令执行器**

前面提到过，命令执行器会在命令表中查找命令，并将找到的结果保存在客户端状态cmd中。

字典的键是一个命令的字符串格式，值则是一个redisCommand结构：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200107103728.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

下表展示了slags属性可以使用的标识和意义：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200107103852.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

比如set执行时，就会：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200107104058.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

---

现在已经成功完成了：连接所需函数，参数，参数个数。但在真正执行之前还需要进行检查：

- 检查客户端状态cmd是否指向NULL
- 根据redisCommand结构的arity属性，检查参数个数是否正确
- 检查身份验证
- 检查内存占用
- 如果当前客户端正在SUBSCRIBE命令订阅频道，则只接受订阅命令，其他命令会被拒绝。
- 服务器因Lua脚本超时并阻塞，服务器只会执行关闭命令，其他会被拒绝。
- 如果客户端正在执行事务，则服务器只会执行客户端发来的EXEC、DISCARD、MULTI、WATCH四个命令，其他命令都会被放进事务队列中。
- 如果打开了监视器功能，服务器会把将要执行的命令发送给监视器。

---

在执行命令时，先找到客户端状态指针client，然后找到命令字典cmd，然后查找命令的函数指针proc

```C
client->cmd->proc(client);
```

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200107105613.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

处理完毕后，产生回复，**保存在输出缓冲区里面**，之后实现函数还会为客户端的套接字**关联命令回复处理器**，这个处理器负责将命令回复返回给客户端。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200107105749.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

## 2.2 severCron函数

Redis服务器中的serverCron函数默认每隔100毫秒执行一次，这个函数负责管理服务器的资源，并保持服务器自身的良好运转。severCorn函数的常见功能如下：

**（1）更新服务器时间缓存**

Redis服务器中有不少功能需要**获取系统的当前时间**，而每次获取系统的当前时间都需要执行一次系统调用，**为了减少系统调用的执行次数**，服务器状态中的unixtime属性和mstime属性被用作当前时间的缓存：

```C
struct redisServer 
{ 
    // ... 
    // 保存了秒级精度的系统当前UNIX时间戳 
    time_t unixtime; 
    // 保存了毫秒级精度的系统当前UNIX时间戳 
    long long mstime;
};
```

因为serverCron函数默认会以每100毫秒一次的频率更新unixtime属性和mstime属性，所以**这两个属性记录的时间的精确度并不高**。

**（2）更新LRU时钟**

LRU是Least Recent Used，原理如下：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200107110155.png"  style="zoom:65%;display: block; margin: 0px auto; vertical-align: middle;">

服务器状态中的lruclock属性保存了服务器的LRU时钟，这个属性和上面介绍的unixtime属性、mstime属性一样，都是**服务器时间缓存的一种**。

```C
struct redisServer 
{ 
    // ... 
    // 默认每10秒更新一次的时钟缓存， 
    // 用于计算键的空转（idle）时长。
    unsigned lruclock:22;
    //...
};
```

每个Redis对象都会有一个lru属性，这个lru属性保存了对象最后一次被命令访问的时间：

```C
typedef struct redisObject 
{ 
    // ... 
    unsigned lru:22; 
    // ...
} robj;
```

当服务器要**计算一个数据库键的空转时间**（也即是数据库键对应的值对象的空转时间），程序会用服务器的lruclock属性记录的时间减去对象的lru属性记录的时间。

由于是10秒更新一次，所以时钟并不是实时的，这个LRU时间只是一个模糊的估算值。

**（3）更新服务器每秒执行命令次数**

`serverCron`函数中的`trackOperationsPerSecond`函数会以每100毫秒一次的频率执行，这个函数的功能是以**抽样**计算的方式，**估算并记录服务器在最近一秒钟处理的命令请求数量**。

`trackOperationsPerSecond`函数每次运行，都会根据`ops_sec_last_sample_time`记录的上一次抽样时间和服务器的当前时间，以及`ops_sec_last_sample_ops`**记录的上一次抽样的已执行命令数量和服务器当前的已执行命令数量**，计算出两次`trackOperationsPerSecond`调用之间，服务器**平均每一毫秒处理了多少个命令请求，然后将这个平均值乘以1000，这就得到了服务器在一秒钟内能处理多少个命令请求的估计值**，这个估计值会被作为一个新的数组项被放进`ops_sec_samples`环形数组里面。

**（4）更新内存峰值记录**

服务器状态中的`stat_peak_memory`属性记录了服务器的内存峰值大小：

```C
struct redisServer 
{ 
    // ... 
    // 已使用内存峰值 
    size_t stat_peak_memory; 
    // ...
};
```

每次serverCron函数执行时，**程序都会查看服务器当前使用的内存数量，并与stat_peak_memory保存的数值进行比较**，如果当前使用的内存数量比stat_peak_memory属性记录的值要大，那么就替换峰值。

**（5）处理SIGTERM信号**

在启动服务器时，Redis会为服务器进程的SIGTERM信号关联处理器`sigtermHandler`函数，这个信号处理器负责在服务器**接到SIGTERM信号时，打开服务器状态的shutdown_asap标识**。

每次serverCron函数运行时，程序都会对服务器状态的shutdown_asap属性进行检查，并**根据属性的值决定是否关闭服务器**。

**（6）管理客户端资源**

serverCron函数每次执行都会调用clientsCron函数，检查：

- 客户端服务器连接超时（长时间没有互动），程序将释放这个客户端。
- 客户端在上一次执行命令后，输入缓冲区大小超过一定长度，程序会释放客户端当前的输入缓冲区。

**（7）管理数据库资源**

serverCron函数每次执行都会调用databasesCron函数，这个函数会对服务器中的一部分数据库进行检查，删除其中的过期键，并在有需要时，对字典进行收缩操作。参见[Redis中的定期检查]([https://jiangren.work/2020/01/04/Redis%E8%A7%A3%E6%9E%903-%E6%95%B0%E6%8D%AE%E5%BA%93/#2-%E6%95%B0%E6%8D%AE%E5%BA%93%E9%94%AE%E7%A9%BA%E9%97%B4](https://jiangren.work/2020/01/04/Redis解析3-数据库/#2-数据库键空间))

**（8）执行被延迟的BGREWRITEAOF**

在服务器执行BGSAVE命令的期间，如果客户端向服务器发来BGREWRITEAOF命令，那么服务器会将BGREWRITEAOF命令的执行时间延迟到BGSAVE命令执行完毕之后。

每次serverCron函数执行时，函数都会检查BGSAVE命令或者BGREWRITEAOF命令是否正在执行，如果这两个命令都没在执行，且有被延迟的BGREWRITEAOF，则执行。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200107113952.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

**（9）将AOF缓冲区内容写入AOF文件**

**（10）关闭输出缓冲区超限的客户端**

**（11）增加cronloops计数**

服务器状态的cronloops属性记录了serverCron函数执行的次数，每执行一次就增加计数。作用是：在**复制模块**中实现“**每执行serverCron函数N次就执行一次指定代码**”的功能。

## 2.3 服务器初始化

服务器初始化要完成以下几个任务：

- 初始化服务器状态结构
- 载入配置选项
- 初始化服务器数据结构
- 还原数据库状态
- 执行时间循环

---

**（1）初始化服务器状态结构**

创建一个`struct redisServer`类型的实例变量`server`作为服务器的状态，并为结构中的各个属性设置默认值。

初始化server变量的工作由`redis.c/initServerConfig`函数完成，主要工作：

- 设置服务器ID
- 设置服务器运行默认频率
- 设置配置文件路径
- 设置运行架构
- 设置默认端口号
- 设置RDB持久化和AOF持久化条件
- 初始化LRU时钟
- 创建命令表

**（2）载入配置选项**

完成初始化服务器状态结构后**，所有变量会被附上默认的值**，但是实际上用户可能修改了某些参数。此时，载入用户的配置选项，**替换掉那些被修改后的默认值**。

**（3）初始化服务器数据结构**

除了在之前执行`initServerConfig`函数初始化server状态时，程序只创建了命令表一个数据结构，在这个阶段还需要创建其他数据结构：

- `server.client`链表，记录了所有与服务器相连的客户端状态结构。
- `server.db`数组，数组中包含了所有数据库。
- `server.pubsub_channels`字典，保存模式订阅信息的`server.pubsub_patterns`链表。
- `server.lua`，用于执行Lua脚本的Lua环境
- `server.slowlog`，用于保存慢查询日志。

服务器到现在才初始化数据结构的原因在于，服务器**必须先载入用户指定的配置选项，然后才能正确地对数据结构进行初始化**。

**（4）还原数据库状态**

在完成了对服务器状态server变量的初始化之后，服务器需要载入RDB文件或者AOF文件，并根据文件记录的内容来还原服务器的数据库状态。

如果启用了AOF持久化功能，则会使用AOF来还原，否则用RDB文件还原。

**（5）执行事件循环**

在初始化的最后一步，服务器将打印出以下日志：

```
[5244] 21 Nov 22:43:49.084 * The server is now ready to accept connections on port 6379
```

开始执行事件循环，意味着服务器现在开始可以接受客户端的连接请求了。