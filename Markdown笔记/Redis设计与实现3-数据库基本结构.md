---
Rtitle: Redis设计与实现3-数据库基本结构
category:
  - 数据库
tags:
- 计算机网络
- 数据库
- Redis
- 读书笔记
mathjax: true
date: 2020-01-04 09:59:28
---

本章将对Redis服务器的数据库实现进行介绍，介绍键空间、过期键，数据库通知的实现方法。<!--more-->

# 1. 数据库的切换

Redis服务器将所有数据库都保存在**服务器状态**`redis.h/redisServer`结构的db数组中，db数组的每个项都是一个`redis.h/redisDb`结构，**每个redisDb结构代表一个数据库**：

```C
struct redisServer 
{ 
    // ... 
    redisDb *db; 
    int dbnum;
    // ...
};
```

初始化时，程序会根据当前服务器的`dbnum`属性来**决定建立数据库的个数**，默认创建16个。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200104100601.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

---

每个Redis客户端都有自己的目标数据库，当客户端执行读写命令时，就需要**切换数据库**。

默认情况下，Redis客户端的目标数据库为0号数据库，但客户端可以通过执行**SELECT命令**来切换目标数据库。

```c
redis> SET msg "hello world"
OK

redis> GET msg
"hello world"

redis> SELECT 2
OK

redis[2]> GET msg
(nil)
```

在服务器内部，**客户端状态**redisClient结构的db属性记录了**客户端当前的目标数据库**，这个属性是一个指向redisDb结构的指针：

```C
typedef struct redisClient 
{
    // ...
    //记录客户端当前正在使用的数据库
    redisDb *db;
    // ...
} redisClient;
```

如果某个**客户端的目标数据库**为1号数据库，那么这个客户端所对应的客户端状态和**服务器状态**之间的关系如图：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200104103307.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

通过修改指针，使他指向服务器中不同的数据库，从而达到切换的目的。

# 2. 数据库键空间

## 2.1 键空间结构

Redis是一个**键值对数据库服务器**，每个数据库都是一个redis.h/redisDb结构。**其中`dict`字典保存了数据库中所有的键值对**，我们将这个字典称为**键空间(key space)**。

```C
typedef struct redisDb 
{ // ... 
    // 数据库键空间，保存着数据库中的所有键值对 
    dict *dict; 
    // ...
} redisDb;
```

键空间的键就是数据库的键，每个键是一个**字符串对象**。键空间的值就是数据库的值，可以是字符串对象、列表对象、哈希表对象、集合对象和有序集合对象中的一种。

当我们输入以下命令时：

```C
redis> SET message "hello world"
OK

redis> RPUSH alphabet "a" "b" "c"
(integer)3

redis> HSET book name "Redis in Action"
(integer) 1

redis> HSET book author "Josiah L. Carlson"
(integer) 1

redis> HSET book publisher "Manning"
(integer) 1
```

数据库的键空间结构如下：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200104105103.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

## 2.2 键空间的增删改查

**（1）添加和修改键**

添加新键值对和修改键值的操作是一样的，**区别在于键是新的是旧的**。

|    对象    | 命令                                                         |
| :--------: | :----------------------------------------------------------- |
| 字符串对象 | **SET**  date  2020/1/1<br />**MSET** date1 19 date2 20      |
|  哈希对象  | **HSET**  book  name  C++primer <br />**HMSET**  fruit  name apple size  large |
|  列表对象  | **LSET**  cloth   0 shirt<br />**LPUSH**  food  potato <br />**RPUSH**  brand  apple<br />**LRANGE**  level  0 5 |
|  集合对象  | **SADD**  occupation  firefighter                            |
|  有序集合  | **ZADD**  grade  87 Tom  65 Terry<br />                      |

**（2）删除键**

|    对象    | 命令                                                         |
| :--------: | :----------------------------------------------------------- |
| 字符串对象 | **DEL**  date                                                |
|  哈希对象  | **HDEL**  myhash field1  myhash field2                       |
|  列表对象  | **BLPOP** list1 100<br />**BRPOP** list1 150<br />**LPOP** list2<br />**LREM** list3 -2 "hello" |
|  集合对象  | **SPOP** food "rice"<br />**SREM** food "noodle"             |
|  有序集合  | **ZREM** website google.com<br />**ZREMRANGEBYLEX** drink [sprit (coco<br />**ZREMRANGEBYRANK** salary 0 2<br />**ZREMRANGEBYSCORE** salary 1500 3500 |

POP在删除的同时，会返回结果，打印到控制台，而REM则是单纯的删除。BLPOP在移除元素时，如果列表没有元素则会等待至超时或发现元素为止。

有序集合范围删除中，LEX表示键， [ ( 表示区间开闭。而`ZREMRANGEBYRANK salary 0 2`表示删除salary最高的三个。

**（3）查询键**

|    对象    | 命令                                                         |
| :--------: | :----------------------------------------------------------- |
| 字符串对象 | **GET**  time<br />**MGET**  time1  time2                    |
|  哈希对象  | **HGET**  site  baidu<br />**HMGET** site baidu google<br />**HGETALL** site<br />**HKEYS** site |
|  列表对象  | **LINDEX** mylist 2<br />**LRANGE** mylist 0 2               |
|  集合对象  | **SISMEMBER** myset1 "hello"                                 |
|  有序集合  | ...                                                          |

HGET是根据键返回值，HGETALL则返回所有键值对，HKEYS返回所有键。列表对象根据主要根据下标返回结果。

# 3. 过期键

## 3.1 设置过期时间

通过**EXPIRE命令**或者**PEXPIRE命令**，**客户端**可以以**秒**或者**毫秒**精度某个键设置**生存时间（Time To Live，TTL）**，在经过指定的秒数或者毫秒数之后，服务器就会**自动删除生存时间为0的键**：

```C
redis> SET key value
OK

redis> EXPIRE key 5
(integer) 1

redis> GET key // 5秒之内
"value"
redis> GET key // 5秒之后
(nil)
```

与前面相似，客户端可以通过**EXPIREAT命令**或**PEXPIREAT命令**，以秒或者毫秒精度给数据库中的某个键设置**过期时间（expire time）**。过期时间由UNIX时间戳表示。

而**TTL命令**和**PTTL命令**则返回**一个键的剩余生存时间。**

所有的命令在Redis中**最终都会转化为PEXPIREAT**执行。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200104125252.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

---

在RedisDb结构中，**在键空间之外**，有一个expires字典专门保存所有键的过期时间，我们称之为**过期字典**。过期字典保存的值是long long 类型整数，**保存一个毫秒精度的UNIX时间戳**。

```C
typedef struct redisDb 
{ // ... 
    // 过期字典，保存着键的过期时间 
    dict *expires; 
    // ...
} redisDb;
```

虽然键空间和过期时间都有相同的键，但他们以指针形式指向同一个键，不会造成空间浪费。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200104130057.png"  style="zoom:72%;display: block; margin: 0px auto; vertical-align: middle;">

## 3.2 过期键的删除策略

通过过期字典知道了哪些键已经过期，那么**过期的键什么时候会被删除呢？**删除策略有三种：

- 定时删除：在设置键的过期时间的同时，创建一个定时器（timer），定时结束后删除。
- 惰性删除：放着不管，每次从键空间获取时检查是否过期，过期就删除。
- 定期删除：每隔一段时间，程序检查一次数据库，删除过期键。

**（1）定时删除**

定时删除**有利于内存管理**，但**对CPU不友好**。如果过期键太多，删除会占用相当一部分CPU。

所以策略应该是：当有大量命令请求服务器处理时，并且服务器内存充足，就应该优先将CPU资源安排在处理客户端请求上，而不是删除过期键。

创建一个定时器需要用到Redis服务器中的**时间事件**，而当前时间事件的实现方式——无序链表，查找一个事件的时间复杂度为$O(N)$，并**不能高效地处理大量时间事件**。

**（2）惰性删除**

**对CPU最友好**，**但浪费内存**。如果数据库中有很多过期键，而这些过期键永远也不会被访问的话，他们就会永远占据空间，可视为**内存泄漏**。

一些和时间有关的数据，比如日志，在某个时间点后，他们的访问就会很少。如果这类过期数据大量积压，会造成严重的内存浪费。

**（3）定期删除**

定期删除是一种折中，通过选择较为空闲的时间点来处理过期键，减少CPU压力。同时也能及时释放内存，避免内存泄漏。

---

在Redis中，**实际使用的是惰性删除和定期删除这两种**。

**（1）Redis中的惰性删除**

存在于`db.c/expireIfNeeded`函数。**所有读写数据库的Redis命令在执行之前都会调用`expireIfNeeded`函数对输入键进行检查**：

- 过期，函数将输入键删除
- 不过期，函数不动作

**（2）Redis中的定期删除**

过期键的定期删除策略由`redis.c/activeExpireCycle`函数实现，**每当Redis的服务器周期性操作`redis.c/serverCron`函数执行时**，`activeExpireCycle`函数就会被调用，它在规定的时间内，**分多次遍历服务器中的各个数据库**，从数据库的expires字典中**随机检查一部分键的过期时间**，并删除其中的过期键。

全局变量`current_db`会**记录当前`activeExpireCycle`函数检查的进度**，并在**下一次检查时接着上一次的进度进行处理**。比如说，如果当前`activeExpireCycle`函数在遍历10号数据库时返回了，那么下次就会从11号数据库开始工作。

如果所有数据库都被检查了一遍，则`current_db`将会被置0，然后开始新一轮检查。

# 4. 数据库通知

通知是Redis2.8新增的功能，可以**让客户端通过订阅给定的频道或者模式，来获知数据库中键的变化，以及数据库中命令的执行情况。**

## 4.1 订阅通知

订阅有两种模式：

- 订阅某一个键，返回键的所有操作
- 订阅某一个操作，返回执行这个操作的键

情况1，从0号数据库订阅了键message的消息。如果此时有其他客户端操作了message，则会将消息通知到此处。

```C
127.0.0.1:6379> SUBSCRIBE _ _keyspace@0_ _:message
Reading messages... (press Ctrl-C to quit)

1) "subscribe" // 订阅信息
2) "__keyspace@0__:message"
3) (integer) 1

1) "message" //执行SET命令
2) "_ _keyspace@0_ _:message"
3) "set"

1) "message" //执行EXPIRE命令
2) "_ _keyspace@0_ _:message"
3) "expire"
```

情况2，客户端订阅了0号数据库中的DEL命令。

```C
127.0.0.1:6379> SUBSCRIBE _ _keyevent@0_ _:del
Reading messages... (press Ctrl-C to quit)
1) "subscribe" // 订阅信息
2) "_ _keyevent@0_ _:del"
3) (integer) 1

1) "message" //键key执行了DEL命令
2) "_ _keyevent@0_ _:del"
3) "key"

1) "message" //键number执行了DEL命令
2) "_ _keyevent@0_ _:del"
3) "number"
```

## 4.2 发送通知

发送数据库通知的功能是由`notify.c/notifyKeyspaceEvent`函数实现，函数声明如下：

```C
void notifyKeyspaceEvent(int type,char *event,robj *key,int dbid);
```

**type参数是发送的通知的类型，event、keys和dbid分别是事件的名称、产生事件的键，以及产生事件的数据库编号**，函数会根据type参数以及这三个参数来构建事件通知的内容，以及接收通知的频道名。

比如SADD命令的实现函数中，通知的发送方式是

```C
void saddCommand(redisClient* c)
{
    //...
    if(added)
    {
        //...添加成功，发送通知
       	notifyKeyspaceEvent(REDIS_NOTIFY_SET,"add",c->argv[1],c->db->id);
        //...
    }
}
```

当SADD命令成功地向集合添加了一个集合元素之后，命令就会发送通知，**该通知的类型为REDIS_NOTIFY_SET（表示这是一个集合键通知）**，名称为sadd（表示这是执行SADD命令所产生的通知）。

发布时调用的`notifyKeyspaceEvent`函数逻辑是：

1. 检查服务器是否允许发送此类通知，如果不允许就返回
2. 是否允许发送**键空间通知**（4.1提到的情况1），允许就发送
3. 是否允许发送**键事件通知**（4.2提到的情况2），允许就发送