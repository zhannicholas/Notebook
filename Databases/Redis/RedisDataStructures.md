# Redis数据结构

## Redis键

Redis的键是一个字符串。在Redis中，字符串二进制安全的，也就是说：Redis中的字符串可以是任何二进制序列，即可以是任何类型的数据(比如：一张图片、一个序列化后的Java对象……)。像`2`、`2.3`、`0xff`、空字符串(`""`)等任何二进制序列都可以作为Redis中的键。由于键是一个二进制序列，所以键是区分大小写的(`a`和`A`是两个不同的键)。当前版本的Redis的键最大支持到`512MB`(未来这个值可能还会更大)。过大的键会消耗更多的内存，因此选择合适的键很重要。

通常情况下，我们会使用结构良好且有意义的键名，使用冒号(`:`)作为分隔符。例如：`users:1000:friends`。

### 逻辑数据库
Redis中也有数据库(`database`)的概念，只是是以命名空间(`namespacing`)的形式体现的。在每一个逻辑数据库(`logical database`)内，都存在一个键空间(`key space`)。逻辑数据库通过下标(从`0`开始)进行区分。一个逻辑数据库内的键名都是唯一的，但同一个键名可以出现在多个不同的逻辑数据库中，逻辑数据库的一个作用就是对键名进行隔离。

逻辑数据库的数量也是有限制的，Redis集群只支持`database 0`。

### 获取所有键名
有两个命令可以获取Redis数据库内所有的键名，**`KEYS`**和 **`SCAN`**。我们可以使用这两个命令来迭代数据库内所有的键，也可以只获取满足给定模式的键。

**`KEYS`**命令会一次性迭代完所有的键，在迭代完所有的键之前，会**阻塞**所有其它的操作。若数据库内键的数量非常大，执行这个命令则会很**耗时**，因此在生产环境中使用这个命令时要谨慎，由于 **`KYES`**的使用比 **`SCAN`**要简单，因此它在调试时还是很有用的。

**`SCAN`**命令也会**阻塞**，但它采取的是**增量迭代**的方式，一次只迭代少量的键，阻塞时间不会过长，因此在生产环境中使用它是安全的。

### 删除键
有多种方式可以删除键，每种方式都能保证删除成功，但不同方式的性能不同。

**`DEL key [key ...]`**命令会删除给定的键并回收与键相关联的内存，这个删除操作是以**阻塞**方式进行的。**`UNLINK key [key ...]`**命令和 **`DEL`**命令类似，但它会使用单独的线程来回收内存，因此是**非阻塞**的。

### 检查键的存在性
当我们执行 **`SET`**命令时，若键不存在，将会创建键并设置值。有时候，我们希望只有当某一个键存在时，才给它设置值。**`EXISTS key [key ...]`**命令可以检测给定的键是否存在，若键存在，则返回`1`，否则返回`0`。

### 键过期

我们可以为某个key设置过期时间(`expires`)。当达到设置的过期时间时，对应的key就会被自动删除，就像我们显式的执行 **`DEL`** 命令一样。过期时间的单位可以是秒，也可以是毫秒，还可以是UNIX时间戳。关于`expires`的信息都存放在磁盘上，且有备份。这意味着，即使Redis服务器停止运行，过期时间仍然会有效。实际上，Redis保存的是key过期的确切时间，而不是剩余生存时间，虽然参数可以是秒或毫秒。

有三种类型的命令可以管理键的过期时间：
1. 设置过期时间：**`EXPIRE`**(单位为秒)、**`PEXPIRE`**(单位为毫秒)、**`EXPIREAT`**(单位为秒)、**`PEXPIREAT`**(单位为毫秒)
2. 查看剩余过期时长：**`TTL`**(单位为秒)、**`PTTL`**(单位为毫秒)
3. 移除过期时间：**`PERSIST`**

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

## 数据类型
Redis支持多种数据结构(类型)。我们可以使用 **`TYPE key`**命令来查看`key`对应的数据类型，例如：
```Redis
>> set key1 1
OK
>> rpush key2 a
1
>> type key1
string
>> type key2
list
```
**`OBJECT subcommand [arguments [arguments ...]]`**允许我们查看某个键对应的Redis对象的内部信息。支持的`subcommand`有：
* **`OBJECT REFCOUNT <key>`**：返回`key`对应的值的引用数
* **`OBJECT ENCODING <key>`**：返回`key`对应的值在Redis内部存储的存储形式
* **`OBJECT IDLETIME <key>`**：返回`key`对应的值有多久没有被读/写了，单位为秒
* **`OBJECT FREQ <key>`**：返回`key`对应的值被访问频率的对数值
* **`OBJECT HELP`**：返回简短的帮助信息

Redis中的对象在Redis内部可以被存储为多种形式：
* `String`可以被编码成`raw`(普通字符串)和`int`(64位有符号整数会被这么存储以节省空间)
* `List`可以被编码成`ziplist`(用于小型`list`，可以节省空间)和`linkedlist`
* `Set`可以被编码成`intset`(用于只由整数组成的小型集合)和`hashtable`
* `Hash`可以被编码成`ziplist`(用于小型`hash`)和`hashtable`
* `Sorted Set`可以被编码成`ziplist`(用于小型`sorted set`)和`skiplist`

当Redis无法继续维持为节省空间而是用的编码格式时，都会将编码格式转为各种类型对应的通用格式。

下面的例子展示了如何查看数据的编码格式：
```Redis
>> set foo 1
OK
>> object encoding foo
"int"
>> append foo a
(integer) 2
>> object encoding foo
"raw" 
```

### Strings
字符串(`string`)是Redis中最基本的数据类型，它不仅可以存储文本数据，还可以存储整数、浮点数和二进制数据。`string`是二进制安全的，最大不能超过512MB。

Redis提供了20多个操作`string`类型的命令。下面是一些例子：
```Redis
>> set s1 "Hello, Redis"    // 设置s1的值为"Hello, Redis"
OK                          // 成功返回OK
>> get s1                   // 获取值
"Hello, Redis"
>> setnx s1 "Redis"         // 尝试为s1设置新值
(integer) 0                 // s1已有值，不进行操作，返回0
>> set s1 2                 // 设置值
OK
>> get s1                   
"2"
>> incr s1                  // 加1
(integer) 3
>> incrby s1 10             // 加10
(integer) 13
>> decr s1                  // 减1
(integer) 12
>> decrby s1 5              // 减5
(integer) 7
>> incrbyfloat s1 1.1       // 加1.1
"8.09999999999999964"
>> incrbyfloat s1 -3.1      // 减3.1
"5"
```
**`SET`** 和 **`GET`** 分别用来设置和检索key对应的值(这个值是一个**字符串**)。需要注意的是，如果某个key已经有对应的值，**`SET`** 命令会覆盖掉已有的值(不管之前的值是何种类型都会被覆盖)，和这个key关联的`TTL`也会被丢弃。如果不希望已有key的值被覆盖，可以使用 **`SETNX`** 命令，当key存在时，**`SETNX`** 不会进行任何操作。

虽然值是`string`，但如果这个值是一个数字的字符串形式的话，可以对其进行加减操作。上面的例子中，`s1`的值被重新修改为了`"2"`，随后执行了 **`INCR`** 和 **`INCRBY`** 两个加法命令，值变成了`13`，之后的减法命令 **`DECR`** 和 **`DECRBY`** 将值减到了`7`。更加有趣的是，Redis还提供了 **`INCRBYFLOAT`** 以支持浮点加减运算。

有时候，一次性设置或获取多个值是很有意义的，这可以通过 **`MSET key value [key value ...]`** 和 **`MGET key [key ...]`** 命令实现。类似的还有 **`MSETNX`** 命令。下面是一个例子：
```Redis
>> mset s1 1 s2 2 s3 a
OK
>> mget s1 s2 s3
1) "1"
2) "2"
3) "a"
```

### Lists
在Redis中，列表(`list`)表示由一系列元素组成的有序序列。通常情况下，List的实现有数组和链表两种方式，两者各有优劣。数组实现的List可以通过索引快速访问元素，但在插入或删除元素的时候较慢；链表实现的List在插入或删除元素时非常迅速，但访问元素时较慢。Redis中的`list`使用双向链表实现，支持双向操作。基于链表的实现还带来了一个重大优势，那就是可以快速的截取列表中的某一部分。

Redis提供了10多个操作`list`的命令，下面是一些例子：
```Redis
>> lpush list1 a 1 b    // 向列表list1的头部依次插入a、1、b这三个元素
(integer) 3             // 成功插入，当前列表list1含有3个元素
>> lrange list1 0 -1    // 查看list1中的所有元素
1) "b"
2) "1"
3) "a"
>> llen list1           // 获取list1的长度
(integer) 3
>> rpush list1 c d      // 向列表list1尾部依次插入c、d这两个元素
(integer) 5             // 插入成功，当前列表list1含有5个元素
>> lrange list1 0 -1
1) "b"
2) "1"
3) "a"
4) "c"
5) "d"
>> lpop list1           // 移除并返回列表list1中的第一个元素
"b"
>> rpop list1           // 移除并返回列表list2中的最后一个元素
"d"
>> lrange list2 0 -1    // 列表list2是空的
(empty list or set)
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
"c"
>> ltrim list1 1 2     // 修剪列表list1，只保留位于区间[1,2]中的元素
OK
>> lrange list1 0 -1   // 查看list1中所有元素
1) "a"
2) "c"
```
命令 **`LPUSH key value [value ...]`** 和 **`RPUSH key value [value ...]`** 分别用来向列表头部(左侧)和尾部(右侧)添加元素，**`LPOP key`** 和 **`RPOP key`** 分别用来从列表头部(左侧)和尾部(右侧)移除元素。在一个空的列表上执行 **`LPOP`** 或 **`RPOP`** 将返回 **NULL** 。

若要查看列表中的元素，可以使用 **`LRANGE key start stop`** 命令，**`LRANGE`** 命令需要两个索引作为参数。这两个索引形成一个区间，分别指向待返回的第一个和最后一个元素在列表中的位置。索引可以是负数，`0`、`-1`、`-2`分别代表链表中的第一个、最后一个和倒数第二个元素。除了区间查看外，还可以使用 **`LINDEX key index`** 命令查看指定位置的元素。

**`LLEN key`** 用于查看`list`的长度。很多时候，我们只需要保留列表中的某一部分，这个时候 **`LTRIM key start end`** 就可以排上用场了，它和 **`LRANGE`** 类似，也接收两个索引作为参数，但它将列表的内容设置为区间内的元素并去除区间外的所有元素。上面的例子中，**`LTRIM`** 告诉Redis只保留列表list1处于区间[1,2]中的元素，然后丢弃其它的元素。

有时候，我们希望只有当列表中有元素时，才执行POP操作，这时候可以使用 **`BLPOP key [key ...] timeout`** 和 **`BRPOP key [key ...] timeout`** 。两个命令的都是从列表中移除元素，只是方向不一样。以 **`BLPOP`** 为例，它接收一个或多个`key`以及一个超时时间`timeout`(可阻塞时间，单位为秒，0代表一直阻塞)。在这些`key`中，如果有key对应的列表不为空，将会返回第一个非空列表的key及POP出的值。举个例子：
```Redis
>> lrange list1 0 -1  // 列表list1包含两个元素
1) "a"
2) "c"
>> lrange list2 0 -1  // 列表list2为空
(empty list or set)
>> blpop list2 list1 0  // list1中的第一个元素被移除
1) "list1"
2) "a"
```
**`BLPOP`** 依次检查list2和list1，然后移除并返回list1的头部元素(list2是一个空列表)。

如果 **`BLPOP`** 命令后面给出的的key对应的都是空列表，**`BLPOP`** 就会阻塞当前连接。一旦另外一个客户端向某一个key对应的列表插入元素，**`BLPOP`** 就会解除阻塞并返回。若阻塞的时间超过了`timeout`，将返回`NULL`。

### Hashes

Redis中的哈希表(`hash`)是一个字符串类型的field-value映射表，不支持多级嵌套，非常适合表示对象。

下面是一些例子：
```Redis
>> hset user:1 name Tome  // 将哈希表user:1的name字段的值设置为Tome
(integer) 1
>> hget user:1 name       // 获取哈希表user:1中name字段对应的值
"Tome"
>> hmset user:2 name Bob age 20 education barchelor // 一次向哈希表user:2中插入多个filed-value
OK
>> hmget user:2 name age  // 获取哈希表user:2中name和age字段对应的值
1) "Bob"
2) "20"
>> hgetall user:2         // 获取哈希表user:2中所有字段和值
1) "name"
2) "Bob"
3) "age"
4) "20"
5) "education"
6) "barchelor"
>> hget user:1 age        // 企图获取一个不存在的字段对应的值，会返回NULL
(nil)
>> hexists user:1 age     // 检查哈希表user:1中是否存在字段age
(integer) 0
>> hvals user:2           // 返回哈希表user:2中所有的值
1) "Bob"
2) "20"
3) "barchelor"
>> hincrby user:2 age 2  // 将哈希表user:2的age字段对应的值加2
(integer) 22
>> hget user:2 age
"22"
```

### Sets
Redis中的集合(`set`)是一个由<u>字符串</u>构成的无序集合，集合中不存在重复元素。除了基本的插入、删除、存在性检测等操作，Redis还支持集合的交、并、差计算。

下面是基本操作的一些例子：
```Redis
>> sadd colors red blue white black // 向集合colors中插入4个元素
(integer) 4
>> smembers colors                  // 查看集合colors内容
1) "blue"
2) "white"
3) "red"
4) "black"
>> sismember colors yellow        // 检查yellow是否在colors中
(integer) 0                       // 集合colors不包含yellow
>> scard colors                   // 查看集合colors的基数(元素个数)
(integer) 4
>> spop colors                    // 随机从集合colors中移除一个元素
"black"
>> srem set1 black                // 从集合colors中删除一个不存在的元素
(integer) 0
>> srem colors white              // 从集合colors中删除一个存在的元素
(integer) 1
>> smembers colors
1) "blue"
2) "red"
```

下面是集合运算：
```Redis
>> sadd set2 a b c red blue gold
(integer) 6
>> smembers set2
1) "c"
2) "a"
3) "gold"
4) "b"
5) "red"
6) "blue"
>> sunion colors set2   // colors ∩ set2
1) "gold"
2) "red"
3) "b"
4) "a"
5) "blue"
6) "c"
>> sinter colors set2   // colors ∪ set2
1) "blue"
2) "red"
>> sdiff set2 colors   // set - colors
1) "gold"
2) "b"
3) "c"
4) "a"
>> sadd set3 f
(integer) 1
>> sdiff set2 colors set3   // set2 - colors - set3
1) "gold"
2) "b"
3) "c"
4) "a"
```

### Sorted sets
Redis中的有序集合(`sorted set`)和集合(`set`)类似，存放不重复的字符串。不过，有序集合中的每个元素都有一个对应的浮点类型的分数(`score`)，这个分数用来维持集合的有序性。由于是有序的，有序集合又具有哈希表的快速访问的优势。

考虑有序集合中的两个元素`A`和`B`，有序集合的有序性基于以下两点：
* 如果`A.score > B.score`，那么：`A > B`
* 如果`A.socre = B.score`，`A > B`的前提是`A`的字典序大于`B`。因为集合的元素都是唯一的，所以`A`和`B`的内容不可能相同

下面是一些例子：
```Redis
>> zadd z1 2 a -1 b               // 向有序集合z1中添加a(score=2)和b(score=-1)， ZADD还可以用来更新元素对应的分数
(integer) 2
>> zrange z1 0 -1                 // 查看z1内容(正序)，因为b.score < a.score，所以b在前面
1) "b"
2) "a"
>> zrange z1 0 -1 withscores      // 查看z1内容及对应分数
1) "b"
2) "-1"
3) "a"
4) "2"
>> zrevrange z1 0 -1              // 查看z1内容(逆序)
1) "a"
2) "b"
>> zrangebyscore z1 0 inf         // 查看z1中分数在[0, inf)内的内容
1) "a"
>> zrank z1 b                     // 查看z1中内容b的排名
(integer) 0
>> zremrangebyscore z1 -inf -1    // 移除z1中分数位于(-inf,1]的所有元素
(integer) 1
>> zrange z1 0 -1
1) "a"
```
有序集合还支持很多的命令，比如：**`ZPOPMAX`**、**`ZRANGEBYLEX`**、**`ZUNIONSTORE`**等等。

### Bitmaps
严格地说，`Bitmap`并不是一种新的数据类型，而是基于`string`的一种数据类型，它提供了一些基于比特位的操作。

相关的命令分为两种：操作单个比特位的和操作一组比特位的。

比特位的设置和检索使用的是 **`BITSET`** 和 **`BITGET`** 命令:
```Redis
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

### HyperLogLogs
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

### Geospatial indexes
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

### Streams
`Stream`是Redis 5.0中新增加的数据类型，它以更加抽象的方式模拟了日志结构，通常实现为一个仅以追加模式打开的文件。`stream`涉及到的内容比较多，后面会详细了解。这里先跳过。

## 参考文献
1. [redis.io](https://redis.io).