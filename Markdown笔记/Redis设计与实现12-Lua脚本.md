---
title: Redis设计与实现12-Lua脚本
category:
  - 数据库
tags:
- 计算机网络
- 数据库
- Redis
- 读书笔记
mathjax: true
date: 2020-01-08 11:27:51
---

Redis提供了非常丰富的指令集，官网上提供了200多个命令。但是某些特定领域，需要扩充若干指令原子性执行时，仅使用原生命令便无法完成，所以需要Lua脚本进行补充。<!--more-->Redis客户端可以使用Lua脚本，**直接在服务器端原子地执行多个Redis命令。**

比如，使用EVAL命令可以对脚本进行求值

```C
redis> EVAL "return 1+1" 0
(integer) 2
```

使用Lua的原因主要有：

- **拓展原生指令集功能**。
- **减少网络开销**。多个指令集同时发出，作为整体执行。
- **原子操作**。对事务功能的一个替代，避免竞争。
- **复用**。发送的脚本会以函数形式保存在Redis中，其他客户端也能使用。

# 1. Lua环境

Redis在服务器内嵌了一个Lua环境（environ-ment），并**对这个Lua环境进行了一系列修改**，从而确保这个Lua环境可以满足Redis服务器的需要。

步骤如下：

1. 创建一个基础的Lua环境
2. 载入多个函数库到Lua环境里面，让Lua脚本可以使用这些函数库来进行数据操作
3. 创建**全局表格**redis，这个表格包含了对Redis进行操作的函数。
4. 使用Redis自制的随机函数来替换Lua原有的带有副作用的随机函数，从而**避免在脚本中引入副作用。**
5. 创建排序辅助函数，Lua环境使用这个辅佐函数来对一部分Redis命令的结果进行排序，从而**消除这些命令的不确定性**。
6. 创建redis.pcall函数的错误报告辅助函数，这个函数可以提供更详细的出错信息。
7. **对Lua环境中的全局环境进行保护**，防止用户在执行Lua脚本的过程中，将额外的全局变量添加到Lua环境中。
8. 将完成修改的Lua环境保存到服务器状态的lua属性中，等待执行服务器传来的Lua脚本。

**（1）创建Lua环境**

在最开始的这一步，服务器首先调用Lua的C API函数`lua_open`，创建一个新的Lua环境。因为`lua_open`函数创建的**只是一个基本的Lua环境**，为了让这个Lua环境可以满足Redis的操作要求，接下来服务器将对这个Lua环境进行一系列修改。

**（2）载入函数库**

Redis修改Lua环境的第一步，就是将以下函数库载入到Lua环境里面： 

- 基础库：包含了Lua核心函数
- 表格库：用于处理表格的通用函数
- 字符串库
- 数学库：标准C语言数学库的接口
- 调试库：钩子函数和取得钩子函数，还包括元数据相关函数
- Lua CJSON库：用于处理UTF-8编码的JSON格式
- Lua cmsgpack库：用于处理MessagePack格式的数据

**（3）创建redis表格**

服务器将在Lua环境中创建一个redis表格（table），**并将它设为全局变量**。这个redis表格包含以下函数： 

- 用于执行Redis命令的`redis.call`和`redis.pcall`函数。 
- 用于记录Redis日志（log）的`redis.log`函数
- 用于计算SHA1校验和的`redis.sha1hex`函数。 
- 用于返回错误信息的`redis.error_reply`函数和`redis.status_reply`函数。

**（4）自制随机函数替代Lua原有的随机函数**

为了保证相同的脚本可以在不同的机器上产生相同的结果，**Redis要求所有传入服务器的Lua脚本，以及Lua环境中的所有函数，都必须是无副作用（side effect）的纯函数（pure func-tion）。**

> 副作用是指：函数使用时除了返回值以外还破坏了系统环境，比如全局变量。

Redis使用自制的函数替换了math库中原有的`math.random`函数和`math.randomseed`函数，替换之后的两个函数有以下特征：

- 对于相同的seed来说，`math.random`总产生相同的随机数序列
- 除非在脚本中使用`math.randomseed`显式地修改seed，否则每次运行脚本时，Lua环境都使用固定的`math.random-seed（0）`语句来初始化seed。

**（5）创建排序辅助函数**

另一个可能产生不一致数据的地方是**那些带有不确定性质的命令**。比如对于一个集合键来说，因为集合元素的排列是无序的，所以即使两个集合的元素完全相同，它们的输出结果也可能并不相同。

为了消除这些命令带来的不确定性，服务器会为Lua环境创建一个排序辅助函数`__redis__compare_helper`，当Lua脚本执行完一个带有不确定性的命令之后，程序会使用`__redis__compare_helper`**作为对比函数自动调用`table.sort`函数对命令的返回值做一次排序，以此来保证相同的数据集总是产生相同的输出。**

如果我们在Lua脚本中对fruit集合和anotherfruit集合执行`SMEMBERS`命令，那么两个脚本将得出相同的结果：

```C
redis> EVAL "return redis.call('SMEMBERS', KEYS[1])" 1 anotherfruit
1) "apple"
2) "banana"
3) "cherry"

redis> EVAL "return redis.call('SMEMBERS', KEYS[1])" 1 fruit
1) "apple"
2) "banana"
3) "cherry"
```

**（6）创建redis.pcall函数的错误报告辅助函数**

服务器将为Lua环境创建一个名为`__redis__err__handler`的错误处理函数，当脚本调用`redis.pcall`函数执行Redis命令，并且被执行的命令出现错误时，`__re-dis__err__handler`就会打印出错代码的来源和发生错误的行数，为程序的调试提供方便。

**（7）保护Lua的全局环境**

**确保传入服务器的脚本不会因为忘记使用local关键字而将额外的全局变量添加到Lua环境里面**。

如果误操作，程序会报错：

```
redis> EVAL "x = 10" 0
(error) ERR Error running script
(call to f_df1ad3745c2d2f078f0f41377a92bb6f8ac79af0):
@enable_strict_lua:7: user_script:1:
Script attempted to create global variable 'x'
```

试图获取一个不存在的全局变量也会引发一个错误

不过**Redis并未禁止用户修改已存在的全局变量**，所以在执行Lua脚本的时候，必须非常小心，以免错误地修改了已存在的全局变量。

```C
redis> EVAL "redis = 10086; return redis" 0
(integer) 10086
```

**（8）将Lua环境保存到服务器状态的lua属性里面**

最后的这一步，服务器会将Lua环境和服务器状态的lua属性关联起来

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200108150354.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

# 2. Lua环境协作组件

## 2.1 伪客户端

伪客户端负责处理Lua脚本中包含的所有Redis命令。Lua脚本使用`redis.call`函数或者`redis.pcall`函数执行一个Redis命令，需要完成以下步骤：

1. `redis.call`函数或者`redis.pcall`函数想要执行的命令传给伪客户端。
2. 伪客户端将命令传递给命令执行器
3. 命令执行器执行，将结果返回给伪客户端
4. 伪客户端接受结果，返回给Lua环境
5. Lua环境在接收到命令结果之后，将该结果返回给`redis.call`函数或者`redis.pcall`函数。
6. 接收到结果的`redis.call`函数或者`redis.pcall`函数会将命令结果作为函数返回值返回给脚本中的调用者。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200108150944.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

## 2.2 lua_script字典

这个字典的**键**为某个Lua脚本的SHA1校验和（checksum），而字典的**值**则是SHA1校验和对应的Lua脚本：

```C
struct redisServer 
{ 
// ... 
dict *lua_scripts; 
// ...
};
```

Redis服务器会**将所有被EVAL命令执行过的Lua脚本，以及所有被`SCRIPT LOAD`命令载入过的Lua脚本都保存到`lua_scripts`字典里面。**

如果客户端给服务器发送以下命令：

```C
redis> SCRIPT LOAD "return 'hi'"
"2f31ba2bb6d6a0f42cc159d2e2dad55440778de3"

redis> SCRIPT LOAD "return 1+1"
"a27e7e8a43702b7046d4f6a7ccf5b60cef6b9bd9"

redis> SCRIPT LOAD "return 2*2"
"4475bfb5919b5ad16424cb50f74d4724ae833e72"
```

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200108151150.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

# 3. Lua相关命令的实现

## 3.1 EVAL命令

EVAL命令有三个参数：

- Lua脚本
- 脚本中使用键的个数
- 键参数和脚本参数

比如，下面的例子中，2表示有两个键，名字为Key1和key2，first和second为脚本参数。

```lua
-> EVAL "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
1) "key1"
2) "key2"
3) "first"
4) "second"
```

下面的例子中，比较了两种不同的用法：

```lua
-> EVAL "return redis.call('set','foo','bar')" 0
OK

-> EVAL "return redis.call('set',KEYS[1],'bar')" 1 foo
OK
```

EVAL命令实现的过程分三步：

1. 根据客户端给定的Lua脚本，在Lua环境中定义一个Lua函数。
2. 将客户端给定的脚本保存到`lua_scripts`字典
3. 执行刚刚在Lua环境中定义的函数，以此来执行客户端给定的Lua脚本

**（1）定义脚本函数**

服务器首先要做的就是在Lua环境中，为传入的脚本**定义一个与这个脚本相对应的Lua函数**。Lua函数的名字为`"_f"+SHA1校验和`，而函数的体（body）则是脚本本身。

```lua
function f_5332031c6b470dc5a0dd9b4bf2030dea6d65de91() 
    return 'hello world'
end
```

使用函数来保存客户端传入的脚本可以**让Lua环境保持清洁**：减少了垃圾回收的工作量，并且避免了使用全局变量。 

**（2）脚本保存到lua_scripts字典**

首先服务器向lua_stripts字典中添加一个键值对，键为Lua脚本的SHA1校验和，值为Lua脚本本身（一个字符串）

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200108151935.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

**（3）执行脚本函数**

执行过程如下：

1. 将EVAL命令中传入的**键名（key name）参数和脚本参数分别保存到KEYS数组和ARGV数组**，然后将这两个数组作为全局变量传入到Lua环境里面。
2. 为Lua环境装载**超时处理钩子（hook）**，这个钩子可以在脚本出现超时运行情况时，让客户端通过SCRIPT KILL命令停止脚本，或者通过SHUTDOWN命令直接关闭服务器。
3. 执行脚本函数
4. 卸载钩子
5. 结果保存到客户端状态的输出缓冲区里面，等待服务器将结果返回给客户端。
6. 对Lua环境执行垃圾回收操作。

## 3.2 EVALSHA命令的实现

只要脚本对应的函数曾经在Lua环境里面定义过，那么**即使不知道脚本的内容本身，客户端也可以根据脚本的SHA1校验和来调用脚本对应的函数**，从而达到执行脚本的目的，这就是EVALSHA命令的实现原理。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200108154429.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">

举个例子，当服务器执行完以下EVAL命令之后：

```lua
redis> EVAL "return 'hello world'" 0
"hello world"
```

当客户端执行以下EVALSHA命令时：

```lua
redis> EVALSHA "5332031c6b470dc5a0dd9b4bf2030dea6d65de91" 0
"hello world"
```

## 3.3 脚本管理命令的实现

除了EVAL命令和EVALSHA命令之外，Redis中与Lua脚本有关的命令还有四个，它们分别是**SCRIPT FLUSH命令、SCRIPT EXISTS命令、SCRIPT LOAD命令、以及SCRIPT KILL命令**。

**（1）SCRIPT FLUSH**

释放并重建lua_scripts字典，关闭现有的Lua环境并重新创建一个新的Lua环境。

**（2）SCRIPT EXISTS**

检查校验和对应的脚本是否存在于服务器中

**（3）SCRIPT LOAD**

Load和EVAL比较相似，区别在于Load装载后并不执行。

```lua
redis> SCRIPT LOAD "return 'hi'"
"2f31ba2bb6d6a0f42cc159d2e2dad55440778de3"
```

执行后，服务器创建此函数，于是我们可以：

```lua
redis> EVALSHA "2f31ba2bb6d6a0f42cc159d2e2dad55440778de3" 0
"hi"
```

**（4）SCRIPT KILL**

如果服务器设置了`lua-time-limit`配置选项，那么在每次执行Lua脚本之前，服务器都会在Lua环境里面设置一个超时处理钩子（hook）。

一旦钩子发现脚本的运行时间已经超过了lua-time-limit选项设置的时长，**钩子将定期在脚本运行的间隙中，查看是否有SCRIPT KILL命令或者SHUTDOWN命令到达服务器。**达到类似于**中断**的效果。

<img src="https://bucket-1259555870.cos.ap-chengdu.myqcloud.com/20200108160142.png"  style="zoom:75%;display: block; margin: 0px auto; vertical-align: middle;">