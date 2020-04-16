> [Redis](https://redis.io)是一个位于内存中的数据结构存储系统，可用作数据库、缓存和消息中间件。它支持的数据结构有：string、hash、list、set、sorted set with range quries、bitmap、hyperloglogs、geospatial indexes with radius queries以及stream。Redis支持复制、Lua脚本、基于LRU的键驱逐、事务和不同级别的磁盘持久化，并通过哨兵和Redis集群的自动分区机制来保证高可用。

> Redis的存储是基于key-value的，但这里的key-value不是简单的key-value，因为value的类型可以有很多种。在我们传统的key-value存储中，key和value都是字符串，而在Redis中，value不仅可以是字符串，还有可能是像列表、集合这样的复杂的数据结构。

本文主要内容包括：Redis的安装、键和几种常见的数据类型的使用，以及和数据生存期相关的一些操作。

# 安装Redis

参见[Redis Quick Start](https://redis.io/topics/quickstart)。


# Redis键

Redis的键是一个字符串。在Redis中，字符串二进制安全的，也就是说：Redis中的字符串可以包含任何类型的数据(比如：一张图片、一个序列化后的Java对象……)，这些也可以成为Redis中的键。

注意：空字符串(**`""`**)也是一个有效的键。


# 数据类型

## Strings

字符串(`string`)是Redis最基本的数据类型，最大不能超过512MB。

Redis提供了20多个操作`string`类型的命令。下面是一些例子：
```
>> set s1 "Hello, Redis"    // 设置s1的值为"Hello, Redis"
OK                          // 成功返回OK
>> get s1                   // 获取值
"Hello, Redis"
>> setnx s1 "Redis"         // 尝试为s1设置新值
0                           // s1已有值，不进行操作，返回0
>> get s1
"Hello, Redis"
>> set s1 2                 // 设置值
OK
>> get s1                   
"2"
>> incr s1                  // 加1
3
>> incrby s1 10             // 加10
13
>> decr s1                  // 减1
12
>> decrby s1 5              // 减5
7
>> incrbyfloat s1 1.1       // 加1.1
8.1
>> incrbyfloat s1 -3.1      // 减3.1
5
```
**`SET`** 和 **`GET`** 分别用来设置和检索key对应的值(这个值是一个**字符串**)。需要注意的是，如果某个key已经有对应的值，**`SET`** 命令会覆盖掉已有的值(不管之前的值是何种类型都会被覆盖)，和这个key关联的`TTL`也会被丢弃。如果不希望已有key的值被覆盖，可以使用 **`SETNX`** 命令，当key存在时，**`SETNX`** 不会进行任何操作。

虽然值是`string`，但如果这个值是一个数字的字符串形式的话，可以对其进行加减操作。上面的例子中，`s1`的值被重新修改为了`"2"`，随后执行了 **`INCR`** 和 **`INCRBY`** 两个加法命令，值变成了`13`，之后的减法命令 **`DECR`** 和 **`DECRBY`** 将值减到了`7`。更加有趣的是，Redis还提供了 **`INCRBYFLOAT`** 以支持浮点加减运算。

```
>> mset s1 1 s2 2 s3 a
OK
>> mget s1 s2 s3
[
  "1",
  "2",
  "a"
]
```
有时候，一次性设置或获取多个值是很有意义的，这可以通过 **`MSET`** 和 **`MGET`** 命令实现。类似的还有 **`MSETNX`** 命令。

## Lists

在Redis中，列表(`list`)表示由一些元素组成的有序序列。通常情况下，List的实现有数组和链表两种方式，两者各有优劣。数组实现的List可以通过索引快速访问元素，但在插入或删除元素的时候较慢；链表实现的List在插入或删除元素时非常迅速，但访问元素时较慢。Redis中的`list`使用链表实现，因为对于一个数据库系统来说，快速向一个非常长的列表中添加元素至关重要。基于链表的实现还带来了一个重大优势，那就是可以快速的选取列表中的某一部分。

Redis提供了10多个操作`list`的命令，下面时一些例子：
```
>> lpush list1 a 1 b    // 向列表list1的头部依次插入a、1、b这三个元素
3                       // 成功插入，当前列表list1含有3个元素
>> lrange list1 0 -1    // 查看list1
1) "b"
2) "1"
3) "a"
>> rpush list1 c d      // 向列表list1尾部依次插入c、d这两个元素
5                       // 插入成功，当前列表list1含有5个元素
>> lrange list1 0 -1
1) "b"
2) "1"
3) "a"
4) "c"
5) "d"
>> lpop list1           // 移除并返回列表list1中的第一个元素
b
>> rpop list1           // 移除并返回列表list2中的最后一个元素
d
>> lrange list2 0 -1    // 列表list2是空的
>> lpop list2           // 从一个空的列表里面移除元素会返回NULL
(nil)
>> rpop list2
(nil)
>> lrange list1 0 -1
1) "1"
2) "a"
3) "c"
>> lindex list1 0      // 查看列表list1中处于位置0处的元素
1
>> lindex list1 -1     // 查看列表list1中最后一个元素
c
```
命令 **`LPUSH`** 和 **`RPUSH`** 分别用来向列表头部(左侧)和尾部(右侧)添加元素，**`LPOP`** 和 **`RPOP`** 分别用来从列表头部(左侧)和尾部(右侧)移除元素。在一个空的列表上执行 **`LPOP`** 或 **`RPOP`** 将返回 **NULL** 。有时候，我们希望只有当列表中有元素时，才执行POP操作，这时候可以使用 **`BLPOP`** 和 **`BRPOP`** 。两个命令的都是从列表中移除元素，只是方向不一样。以 **`BLPOP`** 为例，**`BLPOP`** 的完整命令如下：
```
BLPOP key [key ...] timeout
```
**`BLPOP`** 接收一个或多个key以及一个超时时间`timeout`(可阻塞时间，单位为秒，0代表一直阻塞)。在这些key中，如果有key对应的列表不为空，将会返回第一个非空列表的key及POP出的值。举个例子：
```
>> lrange list1 0 -1  // 列表list1包含两个元素
1) "a"
2) "c"
>> lrange list2 0 -1  // 列表list2为空
>> blpop list2 list1 0  // list1中的第一个元素被移除
1) "list1"
2) "a"
```
**`BLPOP`** 依次检查list2和list1，然后移除并返回list1的头部元素(list2是一个空列表)。

如果 **`BLPOP`** 命令后面给出的的key对应的都是空列表，**`BLPOP`** 就会阻塞当前连接。一旦另外一个客户端向某一个key对应的列表插入元素，**`BLPOP`** 就会解除阻塞并返回。若阻塞的时间超过了`timeout`，将返回`NULL`。

若要查看列表中的元素，可以使用 **`LRANGE`** 命令，**`LRANGE`** 命令需要两个索引作为参数。这两个索引形成一个区间，分别指向待返回的第一个和最后一个元素在列表中的位置。索引可以是负数，`0`、`-1`、`-2`分别代表链表中的第一个、最后一个和倒数第二个元素。除了区间查看外，还可以使用 **`LINDEX`** 命令查看指定位置的元素。

```
>> llen list1          // 查看列表list1的长度
3
>> ltrim list1 1 2     // 修剪列表list1，只保留位于区间[1,2]中的元素
OK
>> lrange list1 0 -1   // 查看list1中所有元素
1) "a"
2) "c"
```

**`LLEN`** 用于查看`list`的长度。很多时候，我们只需要保留列表中的某一部分，这个时候 **`LTRIM`** 就可以排上用场了，它和 **`LRANGE`** 类似，也接收两个索引作为参数，但它将列表的内容设置为区间内的元素并去除区间外的所有元素。上面的例子中，**`LTRIM`** 告诉Redis只保留列表list1处于区间[1,2]中的元素，然后丢弃其它的元素。

## Hashes

Redis中的哈希表(`hash`)是一个字符串类型的field-value映射表，非常适合用来表示对象。

下面是一些例子：
```
>> hset user:1 name Tome  // 将哈希表user:1的name字段的值设置为Tome
1
>> hget user:1 name       // 获取哈希表user:1中name字段对应的值
Tome
>> hmset user:2 name Bob age 20 education barchelor // 一次向哈希表user:2中插入多个filed-value
OK
>> hmget user:2 name age  // 获取哈希表user:2中name和age字段对应的值
[
  "Bob",
  "20"
]
>> hgetall user:2         // 获取哈希表user:2中所有字段和值
{
  "name": "Bob",
  "age": "20",
  "education": "barchelor"
}
>> hget user:1 age        // 企图获取一个不存在的字段对应的值，会返回NULL
(nil)
>> hexists user:1 age     // 检查哈希表user:1中是否存在字段age
0
>> hvals user:2           // 返回哈希表user:2中所有的值
[
  "Bob",
  "20",
  "barchelor"
]
>> hincrby user:2 age 2  // 将哈希表user:2的age字段对应的值加2
22
>> hget user:2 age
22
```

## Sets

Redis中的集合(`set`)是一个由字符串构成的无序集合。除了基本的插入、删除、存在性检测等操作，Redis还支持集合的交、并、差计算。

下面是基本操作的一些例子：
```
>> sadd set1 red blue white black // 向集合set1中插入4个元素
4
>> smembers set1                  // 查看集合set1内容
[
  "black",
  "white",
  "blue",
  "red"
]
>> sismember set1 yellow          // 检查yellow是否在set1中
0                                 // 集合set1不包含yellow
>> scard set1                     // 查看集合set1的基数(元素个数)
4
>> spop set1                      // 随机从集合set1中移除一个元素
white
>> srem set1 white                // 从集合set1中删除一个不存在的元素
0
>> srem set1 black                // 从集合set1中删除一个存在的元素
1
>> smembers set1
[
  "blue",
  "red"
]
```

下面是集合运算：
```
>> sadd set2 a b c red blue gold
6
>> smembers set2
[
  "red",
  "c",
  "b",
  "a",
  "blue",
  "gold"
]
>> sunion set1 set2 // set1 + set2
[
  "a",
  "blue",
  "gold",
  "red",
  "c",
  "b"
]
>> sinter set1 set2 // set1 x set2
[
  "blue",
  "red"
]
>> sdiff set2 set1  // set2 - set1
[
  "a",
  "gold",
  "c",
  "b"
]
>> sadd set3 f
1
>> sdiff set2 set1 set3 // set2 - set1 - set3
[
  "a",
  "gold",
  "c",
  "b"
]
```

## Sorted sets

Redis中的有序集合(`sorted set`)和集合(`set`)类似，存放不重复的字符串。不过，有序集合中的每个元素都有一个对应的浮点分数(`score`)，这个分数用来维持集合的有序性。由于是有序的，有序集合又具有哈希表的快速访问的优势。

考虑有序集合中的两个元素A和B，有序集合的有序性基于以下两点：

  * 如果A.score > B.score，那么：A > B。
  * 如果A.socre = B.score，A > B的前提是A的字典序大于B。因为集合的元素都是唯一的，A和B的内容不能相同。

下面是一些例子：
```
>> zadd z1 2 a -1 b               // 向有序集合z1中添加a(score=2)和b(score=-1)， ZADD还可以用来更新元素对应的分数
2
>> zrange z1 0 -1                 // 查看z1内容(正序)，因为b.score < a.score，所有b在前面
[
  "b",
  "a"
]
>> zrange z1 0 -1 withscores      // 查看z1内容及对应分数
[
  "b",
  "-1",
  "a",
  "2"
]
>> zrevrange z1 0 -1              // 查看z1内容(逆序)
[
  "a",
  "b"
]
>> zrangebyscore z1 0 inf         // 查看z1中分数在[0, inf)内的内容
[
  "a"
]
>> zrank z1 b                     // 查看z1中内容b的排名
0
>> zremrangebyscore z1 -inf -1    // 移除z1中分数位于(-inf,1]的所有元素
1
>> zrange z1 0 -1
[
  "a"
]

```

有序集合还支持很多的命令，比如：**`ZPOPMAX`**、**`ZRANGEBYLEX`**、**`ZUNIONSTORE`**等等。

## Bitmaps

严格地说，`Bitmap`并不是一种新的数据类型，而是基于`string`的一种数据类型，它提供了一些基于比特位的操作。

相关的命令分为两种：操作单个比特位的和操作一组比特位的。

比特位的设置和检索使用的是 **`BITSET`** 和 **`BITGET`** 命令:
```
>> setbit bits 2 1
0
>> getbit bits 2
1
>> getbit bits 1
0
```

**`BITSET`**给指定比特位设置值，然后返回该位置上**原来的值**，企图使用 **`BITSET`** 设置`0`和`1`以外的值会导致错误。 **`BITGET`** 获取指定位置的值， 如果给定的位置超出了存储用的字符串的长度，将会返回`0`。


操作一组比特位的命令分为三种：

  * **`BITOP`** 进行字符串之间的按位与、按位或、按位异或以及按位取反操作。
  * **`BITCOUNT`** 进行计数操作，返回设置为`1`的比特位的个数。
  * **`BITPOS`** 寻找给定的`0`或`1`出现的第一个位置。

## HyperLogLogs

Redis使用`HyperLogLog`进行计数，这个算法是基于统计的。Redis的实现中，标准误差只有1%，并且在最坏的情况下只需要消耗 **12KB** 的内存。`HyperLogLog`在技术上是一种不同的数据结构，但也是基于`string`实现的。

我们使用 **`SADD`** 向集合中添加元素，类似的，我们也可以使用 **`PFADD`** 向`HyperLogLog`中添加元素。实际上，`HyperLogLog`并不存储我们添加的元素，只是更新内部状态。

```
>> pfadd hll a b c d    // 向hll中加入四个元素
1
>> type hll             // 查看hll类型
string                  // HyperLogLog实际为string
>> pfcount hll          // 对hll进行计数
4
>> pfadd hll1 a b c d
1
>> pfadd hll2 c d e f
1
>> pfmerge hll3 hll1 hll2 // 合并hll1和hll2到hll3
OK
>> pfcount hll3
6
```

## Geospatial indexes

Redis在3.2.0版本中加入了地理空间(`geospatial`)这一数据类型，并支持索引半径查询功能。一个具体的位置信息由三元组(longtitude, latitude, name)确定，当向某一个key添加数据时，数据会被存储为有序集合，这么做为半径查询 **`GEORADIU`** 提供了支持。

下面是一些例子：
```
>> geoadd municipalities 116.4551113869 39.6733986505 beijing 121.6406041505 30.8267595167 shanghai 106.6992091675 29.3055601981 chongqing  // 添加3个地理空间数据
3
>> geodist municipalities beijing chongqing             // 查看beijing和chongqing直接的距离(单位为米)
1457336.8906
>> georadius municipalities 116 40 1000 km              // 查看以经度116、纬度40为中心，1000km为半径内的所有位置
[
  "beijing"
]
>> geohash municipalities beijing shanghai              // 查看beijing和chongqing的Geohash表示
[
  "wx4cdn242c0",
  "wtqrrgzfzs0"
]
>> geopos municipalities chongqing                      // 查看chongqing的地理空间数据
[
  [
    "106.69921070337295532",
    "29.30556015923176716"
  ]
]
>> georadiusbymember municipalities chongqing 1500 km   // 查看以chongqing为中心、1500km为半径内的所有位置
[
  "chongqing",
  "shanghai",
  "beijing"
]

```

## Streams

`Stream`是Redis 5.0中新增加的数据类型，它以更加抽象的方式模拟了日志结构，通常实现为一个仅以追加模式打开的文件。`stream`涉及到的内容比较多，后面会详细了解。这里先跳过。

# 数据过期

我们可以为某个key设置过期时间(`expires`)。当`expires`达到时，对应的key就会被自动删除，就像我们显式的执行 **`DEL`** 命令一样。过期时间的单位可以是秒，也可以是毫秒。关于`expires`的信息都存放在磁盘上，且有备份。这意味着，即使Redis服务器停止运行，过期时间仍然会有效。实际上，Redis保存的是key过期的确切时间，而不是剩余生存时间，虽然参数是秒或毫秒。

下面是一些例子：
```
>> set s1 v1 
OK
>> expire s1 10      // 设置s1于10秒后过期
1
>> ttl s1           // 查看s1的剩余生存时间(单位为秒)
7
>> pttl s1          // 查看s1的剩余生存时间(单位为毫秒)
3042
>> get s1           // 查看s1的值，未经过10秒
"v1"
>> get s1           // 查看s1的值，超过10秒，s1已被删除
(nil)
>> set s2 v2 ex 10  // 在设置值的同时指定过期时间为10秒
OK
>> set s3 v3 px 10000 // 在设置值的同时指定过期时间为10000毫秒
OK
```