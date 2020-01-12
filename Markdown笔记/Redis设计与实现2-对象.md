---
title: Redis设计与实现2-对象
category:
  - 数据库
tags:
- 计算机网络
- 数据库
- Redis
- 读书笔记
mathjax: true
date: 2020-01-03 11:01:39

---

前一章介绍了Redis的主要数据结构，但Redis并没有直接使用这些数据结构来实现键值对数据库， 而是基于这些数据结构创建了一个对象系统<!--more--> ，**这个系统包含字符串对象、列表对象、哈希对象、集合对象和有序集合对象**这五种类型的对象。

使用对象有两个好处：

- 执行命令前，根据对象类型来**判断是否可以执行此命令**。
- 针对不同使用场景，为对象**设置多种不同的数据结构实现**，达到优化的目的。

此外，对象系统还引入了**引用计数实现内存回收机制**，以及**对象共享**。

# 1. 对象的类型与编码

Redis 中的**每个键值对的键和值都是一个对象，每个对象都由一个 `redisObject` 结构表示**， 该结构中和保存数据有关的三个属性分别是 `type` 属性、 `encoding` 属性和 `ptr` 属性：

```C
typedef struct redisObject {
    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 指向底层实现数据结构的指针
    void *ptr;
    ...

} robj;
```

> 结构体的冒号表示位域，表示该变量占用的二进制位数

---

对象的 `type` 属性记录了**对象的类型**，属性的值如下表所示：

|    类型常量    |  对象的名称  |
| :------------: | :----------: |
| `REDIS_STRING` |  字符串对象  |
|  `REDIS_LIST`  |   列表对象   |
|  `REDIS_HASH`  |   哈希对象   |
|  `REDIS_SET`   |   集合对象   |
|  `REDIS_ZSET`  | 有序集合对象 |

对于 Redis 数据库保存的键值对来说， **键总是一个字符串对象**， 而值则可以是上表中的其中一个。

所以，当我们称呼一个数据库键为“字符串键”时， 我们指的是“这个数据库键所对应的**值**为字符串对象”；同理，当我们称呼一个键为“列表键”时， 我们指的是“这个数据库键所对应的**值**为列表对象”。

---

对象的 `ptr` 指针指向对象的底层实现数据结构， 而这些数据结构由对象的 `encoding` 属性决定。`encoding` 属性如下表所示：

|          编码常量           |   编码所对应的底层数据结构    |
| :-------------------------: | :---------------------------: |
|    `REDIS_ENCODING_INT`     |       `long` 类型的整数       |
|   `REDIS_ENCODING_EMBSTR`   | `embstr` 编码的简单动态字符串 |
|    `REDIS_ENCODING_RAW`     |        简单动态字符串         |
|     `REDIS_ENCODING_HT`     |             字典              |
| `REDIS_ENCODING_LINKEDLIST` |           双端链表            |
|  `REDIS_ENCODING_ZIPLIST`   |           压缩列表            |
|   `REDIS_ENCODING_INTSET`   |           整数集合            |
|  `REDIS_ENCODING_SKIPLIST`  |         跳跃表和字典          |

# 2. 字符串对象

## 2.1 编码方式

字符串对象的编码可以是 `int` 、 `raw` 或者 `embstr` 。

（1）如果一个字符串对象保存的是**整数值**， 并且这个整数值可以用 `long` 类型来表示， 那么字符串对象会**将整数值保存在字符串对象结构的 `ptr`属性里面**（将 `void*` 转换成 `long` ）， 并将字符串对象的编码设置为 `int` 。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200103112739.png" style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

（2）如果字符串对象保存的是一个字符串值， 并且这个字符串值的长度**大于** `39` 字节， 那么字符串对象将**使用一个简单动态字符串（SDS）来保存这个字符串值**， 并将对象的编码设置为 `raw` 。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200103112837.png" style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

（3）如果字符串对象保存的是一个字符串值， 并且这个字符串值的长度**小于等于** `39` 字节， 那么字符串对象将使用 `embstr` 编码的方式来保存这个字符串值。

`embstr` 编码是**专门用于保存短字符串**的一种优化编码方式。这种编码和 `raw` 编码一样， 都使用 `redisObject` 结构和 `sdshdr` 结构来表示字符串对象，区别在于：

- `raw` 编码会**调用两次内存分配**函数来**分别**创建 `redisObject` 结构和 `sdshdr` 结构。
- `embstr` 编码则通过**调用一次**内存分配函数来分配一块**连续的空间**。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200103113250.png" style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

由于减少了内存分配的次数，以及将零散的内存整合到一起，这种编码的字符串对象比起 `raw` 编码能够**更好地利用缓存带来的优势**。

如果保存浮点数，则会先转化为字符串类型保存。比如保存3.14就会先转化为`"3.14"`。下表是值和对应的编码类型

|                         值                         |        编码         |
| :------------------------------------------------: | :-----------------: |
|           可以用 `long` 类型保存的整数。           |        `int`        |
|      可以用 `long double` 类型保存的浮点数。       | `embstr` 或者 `raw` |
| 字符串值，长度太大没办法用 `long` 类型表示的整数。 | `embstr` 或者 `raw` |

## 2.2 编码转换

编码之间也会有**互相转换**的情况。对于 `int` 编码的字符串对象来说，如果因为命令导致这个对象保存的不再是整数值， 而是一个字符串值， 那么字符串对象的编码将从 `int` 变为 `raw` 。

```C
redis> SET number 10086
OK
    
redis> OBJECT ENCODING number
"int"
    
redis> APPEND number " is a good number!"
(integer) 23
    
redis> GET number
"10086 is a good number!"
    
redis> OBJECT ENCODING number
"raw"
```

 因为 Redis 没有为 `embstr` 编码的字符串对象编写任何相应的修改程序 ， 所以 `embstr` 编码的字符串对象**实际上是只读的**。当修改`embstr` 编码的字符串对象， 程序会**先将对象的编码从 `embstr` 转换成 `raw` ， 然后再执行修改命令**。

```C
redis> SET msg "hello world"
OK

redis> OBJECT ENCODING msg
"embstr"

redis> APPEND msg " again!"
(integer) 18

redis> OBJECT ENCODING msg
"raw"
```

# 3. 列表对象

## 3.1 编码方式

列表对象的编码可以是 `ziplist` 或者 `linkedlist`。

如同前面提到的，**压缩列表每个节点(entry)只保存一个列表元素**。下面例子中，我们输入数字1，字符"three"和数字5，

```C
redis> RPUSH numbers 1 "three" 5
(integer) 3
```

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200103132547.png" style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

---

如果使用的不是 `ziplist` 编码， 而是 `linkedlist`双端链表 编码， 那么 

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200103132714.png" style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

这其实是一个**嵌套编码**，Redis使用了一个带有 `StringObject` 来表示一个字符串对象，编码方式如同上面提到的那三种。**如果编码对象时字符串值**，展开后就是：

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200103132918.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

## 3.2 编码转换

当列表对象可以同时满足以下两个条件时， 列表对象使用 `ziplist` 编码：

1. 列表对象保存的**所有**字符串元素的长度都小于 `64` 字节；
2. 列表对象保存的元素数量小于 `512` 个；

当上述条件任意一个不满足时，就会执行**转换操作**： 原本保存在压缩列表里的所有列表元素都会被转移并保存到双端链表里面， 对象的编码也会从 `ziplist` 变为 `linkedlist` 。

# 4. 哈希对象

## 4.1 编码方式

哈希对象的编码可以是 `ziplist` 或者 `hashtable` 。

`ziplist` 编码时，每当有新的键值对要加入到哈希对象时， 程序会**先将保存了键**的压缩列表节点推入到压缩列表表尾， 然后**再将保存了值**的压缩列表节点推入到压缩列表表尾， 因此：

- 保存了同一键值对的两个节点总是**紧挨在一起， 键前值后。**
- **先添加**到哈希对象中的键值对会被放在压缩列表的**表头方向**， 而**后来添加**到哈希对象中的键值对会被放在压缩列表的**表尾方向**。

比如：

```C
redis> HSET profile name "Tom"
(integer) 1

redis> HSET profile age 25
(integer) 1

redis> HSET profile career "Programmer"
(integer) 1
```

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200103133909.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

---

`hashtable` 编码时， 哈希对象中的每个键值对都使用一个字典键值对来保存：

- 字典的每个键都是一个**字符串对象**， 对象中保存了键值对的键；
- 字典的每个值都是一个**字符串对象**， 对象中保存了键值对的值。

比如，上面的例子改为`hashtable` 编码

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200103134109.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

## 4.2 编码转换

当哈希对象可以同时满足以下两个条件时， 哈希对象使用 `ziplist` 编码：

1. 哈希对象保存的所有键值对的键和值的字符串长度都小于 `64` 字节；
2. 哈希对象保存的键值对数量小于 `512` 个；

和列表对象一样，不满足条件时原本保存在压缩列表里的所有键值对都会被转移并保存到字典里面， 对象的编码也会从 `ziplist` 变为 `hashtable` 。

# 5. 集合对象

Redis中集合和列表结构相似，但**集合具有唯一性，列表不具有**。

## 5.1 编码方式

集合对象的编码可以是 `intset` 或者 `hashtable` 。

`intset` 编码时，元素将被密集得堆叠在位上，比如

```C
redis> SADD numbers 1 3 5
(integer) 3
```

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200103134827.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

---

另一方面， `hashtable` 编码的集合对象使用字典作为底层实现， 字典的**每个键都是一个字符串对象**， 每个字符串对象包含了一个集合元素， 而**字典的值则全部被设置为 `NULL` 。**

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200103134945.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

## 5.2 编码的转换

当集合对象可以同时满足以下两个条件时， 对象使用 `intset` 编码：

1. 集合对象保存的所有元素**都是整数值**；
2. 集合对象保存的元素数量不超过 `512` 个；

当使用 `intset` 编码所需的两个条件的任意一个不能被满足时， 对象的编码转换操作就会被执行： 原本保存在整数集合中的所有元素都会被转移并保存到字典里面， 并且对象的编码也会从 `intset` 变为 `hashtable` 。

# 6. 有序集合对象

## 6.1 编码方式

有序集合的编码可以是 `ziplist` 或者 `skiplist` 。

`ziplist` 编码的有序集合对象使用压缩列表作为底层实现， 每个集合元素使用两个紧挨在一起的压缩列表节点来保存， 第一个节点保存元素的成员（member）， 而第二个元素则保存元素的分值（score）。

```C
redis> ZADD price 8.5 apple 5.0 banana 6.0 cherry
(integer) 3
```

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200103135248.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

---

`skiplist` 编码的有序集合对象使用 `zset` 结构作为底层实现， 一个 `zset` 结构同时**包含一个字典和一个跳跃表**：

```C
typedef struct zset {
    zskiplist *zsl;
    dict *dict;
} zset;
```

起作用主要是跳跃表，字典是辅助加速用。**字典的键记录了元素的成员，而值则保存了元素的分值**。通过字典，能实现$O(1)$复杂度的查找给定成员分值。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200103140005.png"  style="zoom:80%;display: block; margin: 0px auto; vertical-align: middle;">

---

有序集合**每个元素的成员都是一个字符串对象**， 而每个元素的**分值都是一个 `double` 类型的浮点数**。

虽然 `zset` 结构同时使用跳跃表和字典来保存有序集合元素， 但这两种数据结构都会**通过指针来共享相同元素的成员和分值**， 所以同时使用跳跃表和字典来保存集合元素不会产生任何重复成员或者分值， 也不会因此而浪费额外的内存。

## 6.2 编码转换

当有序集合对象可以同时满足以下两个条件时， 对象使用 `ziplist` 编码：

1. 有序集合保存的元素数量小于 `128` 个；
2. 有序集合保存的所有元素成员的长度都小于 `64` 字节；

不能满足以上两个条件的有序集合对象将使用 `skiplist` 编码。

# 7. 内存回收、对象共享和空转时长

对象中包括了一个引用计数器：

```C
typedef struct redisObject {
    // ...
    // 引用计数
    int refcount;
    // ...
} robj;
```

对象的引用计数信息会随着对象的使用状态而不断变化：

- 在创建一个新对象时， 引用计数的值会被初始化为 `1` ；
- 当对象被引用时，计数值+1；
- 当对象不再被引用时，计数值-1；
- 当对象的引用计数值变为 `0` 时， 对象所占用的内存会被释放。

|      函数       |                             作用                             |
| :-------------: | :----------------------------------------------------------: |
| `incrRefCount`  |                   将对象的引用计数值增一。                   |
| `decrRefCount`  | 将对象的引用计数值减一， 当对象的引用计数值等于 `0` 时， 释放对象。 |
| `resetRefCount` | 将对象的引用计数值设置为 `0` ， 但并不释放对象， 这个函数通常在需要重新设置对象的引用计数值时使用。 |

---

通过引用机制，还能实现对象共享。共享**只针对整数值对象，不针对包含字符串的对象。**

 假设键 A 创建了一个包含整数值 `100` 的字符串对象作为值对象，

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200103141637.png"  style="zoom:67%;display: block; margin: 0px auto; vertical-align: middle;">

如果这时键 B 也要创建一个同样保存了整数值 `100` 的字符串对象作为值对象， 那么服务器有以下两种做法：

1. 为键 B 新创建一个包含整数值 `100` 的字符串对象；
2. 让键 A 和键 B 共享同一个字符串对象；

以上两种方法很明显是第二种方法更节约内存。在 Redis 中， 让多个键共享同一个值对象需要执行以下两个步骤：

1. 将数据库键的值指针指向一个现有的值对象；
2. 将被共享的值对象的引用计数增一。

 Redis 会在初始化服务器时， 创建一万个**字符串对象**， 这些对象包含了从 `0` 到 `9999` 的所有整数值， 当服务器需要用到值为 `0`到 `9999` 的字符串对象时， 服务器就会使用这些共享对象， 而不是新创建对象。

为什么 Redis 不共享包含字符串的对象？

判断是否共享时要检验**共享对象和目标对象是否相同**。复杂度如下，

|             共享对象             |  复杂度  |
| :------------------------------: | :------: |
|       保存整数值字符串对象       | $O( 1)$  |
|     保存字符串值的字符串对象     |  $O(N)$  |
| 包含了多个值的对象（列表或哈希） | $O(N^2)$ |

---

`redisObject` 结构包含的最后一个属性为 `lru` 属性， 该属性记录了对象最后一次被命令程序访问的时间：

```C
typedef struct redisObject {
    // ...
    unsigned lru:22;
    // ...

} robj;
```

`OBJECT IDLETIME` 命令可以打印出给定键的空转时长， 这一空转时长就是通过将当前时间减去键的值对象的 `lru` 时间计算得出的：

```C
redis> SET msg "hello world"
OK

# 等待一小段时间
redis> OBJECT IDLETIME msg
(integer) 20
```

注意`OBJECT IDLETIME` 命令的实现是特殊的， 这个命令在访问键的值对象时， **不会修改值对象的 `lru` 属性**。这类似于`std::weak_ptr`的作用。

当内存满时，空转时长较长的键会被优先释放。