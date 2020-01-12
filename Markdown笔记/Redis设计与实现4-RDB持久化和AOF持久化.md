---
title: Redis设计与实现4-RDB和AOF持久化
category:
  - 数据库
tags:
- 计算机网络
- 数据库
- Redis
- 读书笔记
mathjax: true
date: 2020-01-04 14:10:19
---

持久化的意思是将数据永久保存在磁盘中。Redis采用RDB和AOF两种策略。<!--more-->

# 1. RDB持久化

将服务器中的非空数据库以及它们的键值对统称为**数据库状态**。下图三个非空数据库，以及其中的键值对就是该服务器的数据库状态。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200104142148.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

在Redis中，只有将数据保存在内存磁盘里才会永久保存，**如果服务器进程退出，服务器中的数据库状态就会消失。**为了解决这个问题，Redis提供了RDB持久化功能，这个功能可以将Redis在内存中的数据库状态保存到磁盘里面。

RDB持久化产生的RDB文件(Redis Database)是一个**经过压缩的二进制文件，该文件可以被还原为数据库状态**，所以即使服务器停机，服务器的数据还是被安全保存在硬盘中。

## 1.1 RDB文件的创建与载入

有两个Redis命令可以用于生成RDB文件，一个是**SAVE**，另一个是**BGSAVE **(BackGround SAVE)。SAVE命令会**阻塞Redis服务器进程**，直到RDB文件创建完毕为止，在服务器进程阻塞期间，服务器不能处理任何命令请求：

```C
redis> SAVE //等待直到RDB文件创建完毕
OK
```

而BGSAVE命令会**增加一个子进程**，负责创建RDB文件。

```C
redis> BGSAVE //派生子进程，并由子进程创建RDB文件
Background saving started
```

BGSAVE执行时，会阻止SAVE、其他BGSAVE和BGREWRITEAOF这三个命令执行，防止竞争。

创建RDB文件的实际工作由`rdb.c/rdbSave`函数完成，SAVE命令和BGSAVE命令会以不同的方式调用这个函数。

---

Redis并没有载入RDB文件的命令，只要服务器启动时**检测到RDB文件存在，他就会自动载入。**比如下面日志的第二条：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200104154834.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

## 1.2 自动间隔性保存

由于BGSAVE可以不阻塞服务器执行，所以我们可以**设置条件**，让服务器每隔一段时间自动保存。举个例子：

```C
save 900 1
save 300 10
save 60 10000
```

这些条件的意思是：900秒内对数据库至少进行了1次修改，300秒内对数据库进行了10次修改....

服务器程序会根据save选项所设置的保存条件，设置服务器状态`redisServer`结构的`saveparams`属性：

```C
struct redisServer 
{ 
    // ... 
    // 记录了保存条件的数组 
    struct saveparam *saveparams; 
    // ...
};
```

`saveparams`属性是一个数组，数组中的每个元素都是一个`saveparam`结构，每个`saveparam`结构都保存了一个save选项设置的保存条件：

```c
struct saveparam 
{ 
    // 秒数 
    time_t seconds; 
    // 修改数 
    int changes;
};
```

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200104161007.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

---

除了设置保存条件的saveparams数组外，服务器状态还维持着一个**dirty计数器**，以及一个**lastsave属性**：

- dirty计数器记录自上一次SAVE和BGSAVE以后，服务器对数据库状态进行了多少次修改（包括增删改）
- lastsave则是一个时间戳，记录了上一次执行保存的时间。

当服务器执行修改命令一次以后，dirty计数器就加一。如果是一次性修改多个元素，计数器此时加N

```C
redis->SADD database0 apple orange watermelon
```

---

Redis的服务器**周期性操作函数serverCron默认每隔100毫秒就会执行一次**，该函数用于对正在运行的服务器进行维护，它的其中一项工作就是检查save选项所设置的保存条件是否已经满足，如果满足的话，就执行BGSAVE命令。

执行完以后，dirty清0，lastsave更新。

## 1.3 RDB文件结构

完整的RDB文件如下，

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200104162540.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

RDB是一个二进制文件而不是文本文件。

> 广义来说，所有文件都是二进制文件。狭义来说，文本文件是基于字符编码的文件，常见的编码有**ASCII编码，UNICODE编码**等等；二进制文件是基于值编码的文件，也可以理解为**自定义编码。**

- 开头的REDIS占5个字节，这5个字符**用于检查是不是RDB文件。**
- db_version长度为4字节，值被解析为**RDB版本**，比如"0006"就代表第6版。
- database部分包含着**多个数据库的键值对数据**，根据大小不同，长度有所不同。
- EOF占1个字节，结束位标志。
- check_sum是占8字节，保存**校验和**。服务器在载入时会根据读入的实际数据计算出一个数来和校验值比较，以此来检查是否有损坏。

### 1.3.1 database部分

每个非空数据库在RDB文件中都可以保存为`SELECTDB`、`db_number`、`key_value_pairs`三个部分，如图所示。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105101246.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

- `SELECTDB`，1字节，当读取到此值时，程序知道接下来要读入一个数据库号码。
- `db_number`，1、2、5字节，保存数据库号码。
- `key_value_pairs`，保存键值对，包括过期时间。

### 1.3.2 key_value_pairs部分

不带过期时间的键值对在RDB文件中由TYPE、key、value三部分组成，

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105101548.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

带有过期时间的键值对在RDB中的结构如下

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105101713.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

- EPIRETIME_MS，1字节，告诉程序接下来读取一个以毫秒为单位的过期时间。
- ms，8字节带符号整数，记录一个以毫秒为单位的UNIX时间戳。

### 1.3.3 value部分

**（1）字符串对象**

如果TYPE的值为`REDIS_RDB_TYPE_STRING`，那么value保存的就是一个字符串对象，字符串对象的编码可以是`REDIS_ENCODING_INT`或者`REDIS_ENCODING_RAW`。

如果是INT，则表示对象是一个**长度不超过32位的整数**，保存方式如下：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105103613.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

其中，`ENCODING`的值可以是`REDIS_RDB_ENC_INT8`、`REDIS_RDB_ENC_INT16`或者`REDIS_RDB_ENC_INT32`三个常量的其中一个，它们分别代表RDB文件使用8位、16位或者32位来保存整数值integer。

如果是RAW格式，则说明对象是一个**字符串值**，有压缩和不压缩两种方法来保存。对于没有压缩的字符串，保存格式如下：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105103834.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

压缩后的字符串，保存格式如下：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105103859.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

- REDIS_RDB_ENC_LZF，表明已被LZF算法压缩
- compressed_len，被压缩后的字符串长度
- origin_len，原来的长度
- compressed_string，被压缩后的字符串

**（2）列表对象**

如果TYPE的值为`REDIS_RDB_TYPE_LIST`，那么value保存的就是一个`REDIS_ENCODING_LINKEDLIST`编码的列表对象，RDB文件保存这种对象的结构如图所示。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105104226.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

**每一个列表项都是一个字符串对象**，所以程序会以字符串对象的方式来保存。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105104419.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

结构中，3表示列表长度，5表示第一个列表项长度为5，内容为"hello"。

**（3）集合对象**

如果TYPE的值为`REDIS_RDB_TYPE_SET`，那么value保存的就是一个`REDIS_ENCODING_HT`编码的集合对象，RDB文件保存这种对象的结构如图所示。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105104627.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

图中elem代表集合的元素，**每个集合元素都是一个字符串对象。**

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105104715.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

和列表一样，4代表集合大小，5代表元素长度，值为"apple"。

**（4）哈希表对象**

如果TYPE的值为`REDIS_RDB_TYPE_HASH`，那么value保存的就是一个`REDIS_ENCODING_HT`编码的集合对象，RDB文件保存这种对象的结构如图所示。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105105142.png"  style="zoom:77%;display: block; margin: 0px auto; vertical-align: middle;">

例子如下，

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105105208.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

哈希表长度为2，第一个键值对，键长度为1的字符串"a"，值为5的字符串"apple"。

**（5）有序集合对象**

如果TYPE的值为`REDIS_RDB_TYPE_ZSET`，那么value保存的就是一个`REDIS_ENCODING_SKIPLIST`编码的有序集合对象，RDB文件保存这种对象的结构如图所示。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105105658.png" style="zoom:77%;display: block; margin: 0px auto; vertical-align: middle;">

比如：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105105726.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

大小为2，第一个元素是长度为2的字符串"pi"，分值被转换为长度为4的字符串"3.14"。

**（6）INTSET编码的集合**

如果TYPE的值为`REDIS_RDB_TYPE_SET_INTSET`，那么value保存的就是一个**整数集合对象**，RDB文件保存这种对象的方法是，**先将整数集合转换为字符串对象**，然后将这个字符串对象保存到RDB文件里面。

**（7）ZIPLIST编码的列表、哈希表和有序集合**

如果TYPE的值为`REDIS_RDB_TYPE_LIST_ZIPLIST`、`REDIS_RDB_TYPE_HASH_ZIPLIST`或者`REDIS_RDB_TYPE_ZSET_ZIPLIST`，那么value保存的就是一个**压缩列表对象**，保存策略和上面一一样：先转化为字符串对象。

# 2. AOF持久化

RDB持久化记录的是数据库本身，而AOF(Append Only File)则**记录Redis服务器所执行的写命令**。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105111552.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

假如使用如下命令:

```C
redis> SET msg "hello"
OK
```

则AOF记录形式如下：

```
*2\r\n$6\r\nSELECT\r\n$1\r\n0\r\n
*3\r\n$3\r\nSET\r\n$3\r\nmsg\r\n$5\r\nhello\r\n
```

## 2.1 AOF实现原理

AOF如其名所示，Append Only File，AOF持久化功能的实现可以分为**命令追加（append）、文件写入与同步（sync）**

**（1）命令追加**

如果AOF被打开，则服务器执行完一个命令后，会以协议格式将命令**追加到服务器状态aof_buf缓冲区的结尾**：

```C
struct redisServer 
{ 
    // ...  
    sds aof_buf;     // AOF缓冲区
    // ...
};
```

比如执行了`SET KEY VALUE`后，会将以下协议内容加载到aof_buf缓冲区：

```
*3\r\n$3\r\nSET\r\n$3\r\nKEY\r\n$5\r\nVALUE\r\n
```

**（2）AOF文件的写入与同步**

Redis的服务器进程就是一个**事件循环（loop）**，这个循环中的**文件事件负责接收客户端的命令请求**，**以及向客户端发送命令回复**，而**时间事件则负责执行像`serverCron`函数这样需要定时运行的函数**。

服务器每次结束一个事件循环之前，它都会调用`flushAppendOnlyFile`函数，**考虑是否需要将`aof_buf`缓冲区中的内容写入和保存到AOF文件里面**。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105120013.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

这个函数的行为有服务器配置的`appendfsync`选项来设置，默认为`everysec`：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200105120224.png"  style="zoom:80%;display: block; margin: 0px auto; vertical-align: middle;">

默认情况下，距离上次同步过了一秒钟，则服务器会将aof_buf内容写入AOF文件中。

## 2.2 AOF文件的载入与数据还原

因为AOF文件里面包含了重建数据库状态所需的所有写命令，所以服务器只要**读入并重新执行一遍AOF文件里面保存的写命令**，就可以还原服务器关闭之前的数据库状态。

AOF还原数据库的步骤如下：

1. 创建一个不带网络连接的**伪客户端（fake client）**：因为Redis的命令只能在客户端上下文中执行。
2. 从AOF中读出一条命令。
3. 使用伪客户端执行被读出的写命令。
4. 重复23步

## 2.3 AOF重写

随着时间的增长，AOF文件的大小将会越来越大。为了解决这个问题，Redis提供了**AOF重写**功能。

重写后，Redis服务器可以创建一个新的AOF文件来替代现有的AOF文件，新旧两个**AOF文件保存的数据库状态完全相同**。

如果要保存一个键值对，我们其实只关心它当前的状态。所以重写策略是：首先**从数据库中读取键现在的值，然后用一条命令去记录键值对**，用到了`aof_rewrite`函数。

比如，对list进行`RPUSH`操作填入"A"、"B"、"C"，然后再`LPOP`一次，我们操作了4次，但其实用`RPUSH list A B`这一条指令就可以代替。

`aof_rewrite`函数包含了大量写入操作，调用时会导致线程被长时间阻塞，所以Redis**将AOF重写放入子进程里**。

----

还有一个问题：子进程AOF重写时，主进程也在写命令，导致两者状态不一致。因此，**Redis服务器设置了一个AOF重写缓冲区**，这个缓冲区在服务器创建子进程之后开始使用，当Redis服务器执行完一个写命令之后，它会**同时**将这个写命令发送给**AOF缓冲区**和**AOF重写缓冲区**。

换句话说，子进程执行AOF期间，服务器进程需要：

- 执行客户端指令
- 将执行后的命令追加到AOF缓冲区
- 将执行后的命令追加到AOF重写缓冲区

---

子进程执行完AOF后，向父进程发送一个信号。父进程接收后：

1. 将AOF重写缓冲区的内容写入AOF文件中，保证一致性。
2. 对新AOF文件改名，原子的(atomic)覆盖现有AOF文件。

在整个AOF后台重写过程中，**只有信号处理函数执行时会对服务器进程（父进程）造成阻塞**，在其他时候，AOF后台重写都不会阻塞父进程，这将AOF重写对服务器性能造成的影响降到了最低。