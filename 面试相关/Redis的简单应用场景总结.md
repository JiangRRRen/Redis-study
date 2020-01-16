---
title: Redis的简单应用场景总结
category:
  - 数据库
tags:
- 计算机网络
- 数据库
- Redis
- 读书笔记
mathjax: true
date: 2020-01-16 16:15:22
---

# 1. 缓存

比如我要从数据库查看最新的5000条评论：

```sql
SELECT comments FROM user 
ORDER BY time DESC LIMIT 5000
```

这样的操作随着数据的增加会变得越来越慢，因为要进行排序操作。而且这种排序本身不应该发生：因为我们存的时候是按时间存进去的。

我们可以使用redis的列表对象来实现，此时列表由Linkedlist数据结构实现。

每次新评论发表时，我们会将它的ID添加到一个Redis列表`LPUSH latest.comments`，然后将列表裁剪到5000，`LTRIM latest.comments 0 5000`

每次我们需要获取最新评论的项目范围时，我们调用一个函数来完成(使用伪代码)：

```pseudocode
FUNCTION get_latest_comment(num_items):
	id_list=redis.lrange("latest.comments",0,num_items-1)
	IF id_list<num_items
		id_list = MySQL("SELECT ... ORDER BY time LIMIT ...")
	END
	RETURN id_list
END
```

只有超过了5000这个限制时，才会去访问数据库。

# 2. 排行榜

选手报名参加活动，观众可以对选手进行投票，每个观众对同一名选手只能投一票，活动期间最多投四票。后台需要提供以下接口：

- 接口1：返回TOP 10的选手信息及投票数
- 接口2：返回活动总参与选手数及总投票数
- 接口3：对于每个选手，返回自己的投票数，排名，距离上一名差的票数

---

如果是Mysql方案，需要建立一个表来记录投票信息。这个表在入表时首先就需要判断是否重复刷票，有两种方法

1. 在查询时需要组合查询：将投票表和选手信息表组合，统计投票信息，然后排序输出。**耗时较长**。
2. 新建一个排行榜表，每隔一段时间做组合查询，维护这个表。**缺乏实时性**。

---

redis方案可以采用有序集合对象，我们创建一个有序集合`vote_activity`，然后使用`ZINCRBY key increment memeber`命令给指定成员的分数加上增量increment，

```
redis> ZINCRBY vote_activity 1 Bob
"1"

redis> ZINCRBY vote_activity 1 Tim
"1"

redis> ZINCRBY vote_activity 1 Bob
"2"
```

有序列表入队时，按分值排好序了，我们可以方便的用`ZSCORE key member`查询分数。

```
redis> zscore vote_activity Bob
"2"
```

以及获取某人的排名，获取前10名，获取前10名分数等等，

```
#获取Alice排名(从高到低，zero-based)
redis> zrevrank vote_activity Alice
(integer) 0

#获取前10名(从高到低)
redis> zrevrange vote_activity 0 9
1) "Alice"
2) "Bob"

#获取前10名及对应的分数(从高到低)
redis> zrevrange vote_activity 0 9 withscores
1) "Alice"
2) "2"
3) "Bob"
4) "1"
```

# 3. 消息队列

一般来说，消息队列有两种场景：一种是**发布者订阅者模式**；一种是生**产者消费者模式**。利用redis这两种场景的消息队列都能够实现。定义：

- 生产者消费者模式：生产者生产消息放到队列里，多个消费者同时监听队列，谁先抢到消息谁就会从队列中取走消息；即对于**每个消息只能被最多一个消费者拥有**。（常用于处理高并发写操作）
- 发布者订阅者模式：发布者生产消息放到队列里，多个监听队列的消费者都会收到同一份消息；即正常情况下**每个消费者收到的消息应该都是一样的**。（常用来作为日志收集中一份原始数据对多个应用场景）

---

发布者订阅者模式可以直接使用pub/sub指令实现。

---

生产者消费者模式分两种：

- 普通的
- 带有优先级的

普通模式下使用`brpop`指令，可以以阻塞的形式返回数据列表中新添加的参数：

```C++
while(true)
{
    List<string> msgs = redis.brpop(BLOCK_TIMEOUT,listKey);
    Handle(msgs);
}
```

如果是优先级模式，当优先级不是很多是，可以分为两组：

```C
while(true)
{
    List<string> msgs = redis.brpop(['high_task_queue', 'low_task_queue'],0);
    Handle(msgs);
}
```

`brpop`命令可以输入多个键，如果同时都有元素可读，读先输入的那个键。

如果优先级划分很多，就需要再用列表排序的办法了（有序集合不好，因为没有阻塞模式）。假如有1000个优先级，我们可以先分组，分为10组，每组按优先级顺序排列，查找时二分查找。

# 4. 时间轴

所谓时间轴系统就是典型的微博模式：用户在自己的主页可以看到其关注的博主发表的信息列表（按时间排序）；而其它用户可以一个用户的个人主页看到这个人发布的信息列表（按时间排序）。

解决方案主要有两种：

- 推模式：某人发布内容之后推送给所有粉丝，空间换时间，瓶颈在写入；
- 拉模式：粉丝从自己的关注列表中读取内容，时间换空间，瓶颈在读取；

以推模式为例：

**（1）博主发布博文**

我们创建一个哈希对象post，键为博文ID，值为博文内容字符串。存储博文。

```
redis> HSET post 4396 "hahahahah"
```

再使用一个列表，按先后顺序存储该博主的博文：

```
redis>LPUSH Dasima 4396
```

然后使用一个集合，存储该博主的所有粉丝，利用`SMEMBERS`获取这些粉丝的名单。

```
redis> SMEMBERS Dasima
```

每一个粉丝拥有一个timeline列表，存取所有推送博文的ID。之后对所有粉丝的推送列表进行写入。

**（2）用户读取博文推送**

利用`LRANGE`从推送中拉取一定数量的博文，根据拉到的博文ID，读取哈希表的内容。

```
redis>LRANGE timeline 0 30

redis>HGETALL(4396)
```

# 5. 实现分布式锁

**（1）什么是分布式锁？**

对于单进程的程序，采用普通锁即可防止竞争，而对于多进程分布式系统来说需要采用分布式锁来保证一致性。

**（2）redis如何实现分布式锁**

在 Redis 2.6.12 版本开始，`set`命令增加了三个参数，替换以前的`setnx`命令：

- `EX`：设置键的过期时间（单位为秒）
- `PX`：设置键的过期时间（单位为毫秒）
- `NX` | `XX`：当设置为`NX`时，仅当 key 存在时才进行操作，设置为`XX`时，仅当 key 不存在才会进行操作

我们可以以此实现简单的分布式锁：

```
set key "lock" EX 1 XX
```

如果这个操作返回`false`，说明 key 的添加不成功，也就是当前有人在占用这把锁。而如果返回`true`，则说明得了锁，便可以继续进行操作，并且在操作后通过`del`命令释放掉锁。并且即使程序因为某些原因并没有释放锁，**由于设置了过期时间，该锁也会在 1 秒后自动释放**，不会影响到其他程序的运行。

```
del "lock"
```

