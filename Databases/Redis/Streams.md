# Redis Streams
作为Redis 5.0中推出的全新数据结构，`stream`的行为就像`append-only log`一样，但它由基数树(`radix tree`)实现。`stream`由`entry`构成。它具有很多特性：`stream`的`entry`保存了一组`field-value`对，和`hash`十分类似。除了`field-value`对，每个`entry`都具有唯一`ID`，默认情况下，这个`ID`的形式为：`<millisecond-timestamp>-<sequence number>`。`stream`支持基于`ID`的范围查询，若将时间戳作为`ID`的前缀，便可以实现基于时间的范围查询。`entry`一旦被创建，其存储的内容就不能被修改，但我们可以从`stream`中删除某个`entry`。若要向`stream`中添加`entry`，<u>只能</u>通过在`stream`的<u>末尾</u>追加这个`entry`的方式进行。最后，`stream`同时支持阻塞和非阻塞的消费模式，可以被多个不同的消费组消费或处理，这些消费组相当于`stream`的订阅者。`stream`和`Pub/Sub`最大的区别在于：<u>`stream`会为了后续客户端的消费而在内存中保存数据，`Pub/Sub`不保存任何消息</u>。

## Stream生产者
通常情况下，`producer`从一个或多个源读取数据，然后将它们写入到`stream`。

### Redis Stream的生产者API
`Redis Stream Producer API`允许`producer`将任何消息(`entry`)追加到`stream`的末尾，整个API只包含一个命令——**`XADD key ID field value [field value ...]`**。 **`XADD`**命令用于将一个`entry`追加到`stream`，当`key`不存在时，还会新建一个`stream`：第一个参数`key`代表一个特定的`stream`，`ID`表示消息的`ID`，通常情况下我们会使用`*`来告诉Redis我们希望由Redis来生成这个`ID`，接下来是`entry`的内容。**`XADD`**会将新加入`stream`的消息的`ID`作为返回值返回。
```Redis
> xadd numbers * n 0
"1587959118318-0"
> xadd numbers * n 1
"1587959128680-0"
> type numbers
stream
```

#### 消息ID
`ID`是唯一的，并且每个消息有且只有一个`ID`，它代表着消息在`stream`中的位置。也就是说：越接近于`stream`开头的消息的`ID`越小，越接近于`stream`末尾的消息的`ID`越大。消息的`ID`是不可变的，一旦被创建，就不能再被修改。

我们可以在一个命令的前面加上一个数字，然后Redis会多次执行这个命令。这能帮助我们更好的观察Redis自动生成消息`ID`的特点：
```Redis
> 7 xadd letter_number * a 1
"1587960623524-0"
"1587960623525-0"
"1587960623525-1"
"1587960623526-0"
"1587960623526-1"
"1587960623527-0"
"1587960623527-1"
```
这里重复执行了`7`次`xadd letter_number * a 1`。Redis自动生成的消息`ID`的格式为：`<millisecond-timestamp>-<sequence number>`。可以发现：当`timestamp`相同时，`sequence number`会不断自增(多个消息可能会在同一个时间出现，用自增保证`ID`的唯一性)。每到一个新的时刻，`sequence number`会重置为`0`。

实际上，`ID`包括两个数字(64位无符号整数)，数字之间用`-`分隔。`ID`是总是自增的，这体现在组成ID的两个数字的自增上。插入一个`ID`比当前`stream`中最大`entry`的`ID`还小的`entry`是不被接受的。
```Redis
> xadd letter_number 1-1 a 1
(error) ERR The ID specified in XADD is equal or smaller than the target stream top item
```

有时候，我们可能希望使用自己的`ID`，这个时候我们需要保证自己的`ID`是自增的，一个有效的`ID`必须大于`0-0`。
```Redis
> xadd mystream 0-0 max 100
(error) ERR The ID specified in XADD must be greater than 0-0
> xadd mystream 0-1 max 100
"0-1"
> xadd mystream 1-1 max 10
"1-1"
```
插入消息时，我们也可以将`ID`指定为一个数字，称为`partial ID`，这个时候Redis会自动帮我们添加`sequence number`。
```Redis
> xadd mystream 2 max 111
"2-0"   // Redis自动添加sequence number
```

#### 消息内容
**`XADD`**命令不接收空消息，一个消息至少包含一个`field-value`对。每一个`field`上的`value`都是一个普通的Redis字符串。消息的内容对Redis来说时不透明的，Redis本身也不处理消息的内容。

### 管理Stream的长度
**`XLEN key`**可以用来查看`stream`中消息(`entry`)的数量，若`key`不存在，将返回0(就像`stream`是空的一样)。
```Redis
> xlen ms-1
(integer) 0
> xadd ms-1 0-1 f1 v1 f2 v2
"0-1"
> xadd ms-1 1-1 f1 v1 f2 v2
"1-1"
> xlen ms-1
(integer) 2
```

**`XDEL key ID [ID ...]`**用来从`stream`中删除消息，返回实际被删除消息的数量。虽然`stream`是一个`append-only`的数据结构，但它是保存在内存中的，所以我们可以执行删除操作。当删除某个消息时，并不是真正删除，而是[标记删除](https://redis.io/commands/xdel#understanding-the-low-level-details-of-entries-deletion)，当满足一定条件时，才会进行真正的删除操作。即使`stream`中的所有消息都被删除，`stream`为空时，`stream`本身也不会被删除，要删除它，可以使用 **`DEL`**或 **`UNLINK`**。
```Redis
> xdel ms-1 0-1 1-1
(integer) 2
> xlen ms-1
(integer) 0
> exists ms-1
(integer) 1
> del ms-1
(integer) 0
```

使用 **`XDEL`**可以限制`stream`的长度，但我们需要知道消息的`ID`。为了防止`stream`无限扩大，Redis也允许我们对`stream`进行修剪(`trim`)，即限制`stream`长度的上限。 **`XTRIM key MAXLEN [~] count`**将`stream`的长度限制为`count`，当`stream`长度超过上限，**`XTRIM`**会开始删除`stream`中`ID`最小的消息。然而，使用`MAXLEN`对`stream`进行修建的开销非常大，这和`entry`底层的基数树(`radix tree`)中采用的宏结点(`macro node`)有关系。命令提供了一个可选参数`~`，`XTRIM mystream MAXLEN ~ 1000`表示：并不是真的不能超过1000，消息的数量可以操作1000且至少是1000。当Redis移除整个宏结点时才会进行修剪操作。
```Redis
> xadd ms-2 0-1 f1 v1
"0-1"
> xadd ms-2 0-2 f1 v1
"0-2"
> xadd ms-2 0-3 f1 v1
"0-3"
> xtrim ms-2 MAXLEN 2
(integer) 1     // 有一个entry被删除
> xlen ms-2
(integer) 2
```
如果需要一直限制`stream`的长度，除了周期性的手动调用 **`XTRIM`**，还可以利用 **`XADD`**提供的`MAXLEN`来在我们添加`entry`时自动修剪`strean`，例如：`xadd ms-3 maxlen ~ 2 * f1 v1`。

## 范围查询
**`XRANGE key start end [COUNT count]`**查询`ID`在`[start, end]`之间的所有`entry`。
```Redis
> 10 xadd ms-3 * f1 v1  // 生成10条消息
"1587969310313-0"
"1587969310313-1"
"1587969310314-0"
"1587969310314-1"
"1587969310314-2"
"1587969310315-0"
"1587969310315-1"
"1587969310316-0"
"1587969310316-1"
"1587969310316-2"
> xrange ms-3 1587969310313-0 1587969310313-1
1) 1) "1587969310313-0"
   2) 1) "f1"
      2) "v1"
2) 1) "1587969310313-1"
   2) 1) "f1"
      2) "v1"
```
`-`和`+`表示`stream`可能存在的最小`ID`和最大`ID`，`-`等价于`0-0`，`+`等价于`18446744073709551615-18446744073709551615`。
```Redis
> xrange ms-3 - +           // 使用-和+
 1) 1) "1587969310313-0"
    2) 1) "f1"
       2) "v1"
 2) 1) "1587969310313-1"
    2) 1) "f1"
       2) "v1"
 3) 1) "1587969310314-0"
    2) 1) "f1"
       2) "v1"
 4) 1) "1587969310314-1"
    2) 1) "f1"
       2) "v1"
 5) 1) "1587969310314-2"
    2) 1) "f1"
       2) "v1"
 6) 1) "1587969310315-0"
    2) 1) "f1"
       2) "v1"
 7) 1) "1587969310315-1"
    2) 1) "f1"
       2) "v1"
 8) 1) "1587969310316-0"
    2) 1) "f1"
       2) "v1"
 9) 1) "1587969310316-1"
    2) 1) "f1"
       2) "v1"
10) 1) "1587969310316-2"
    2) 1) "f1"
       2) "v1"
```
此外，还可以使用`partial ID`。例如：
```Redis
>  xrange ms-3 1587969310313 1587969310314
1) 1) "1587969310313-0"
   2) 1) "f1"
      2) "v1"
2) 1) "1587969310313-1"
   2) 1) "f1"
      2) "v1"
3) 1) "1587969310314-0"
   2) 1) "f1"
      2) "v1"
4) 1) "1587969310314-1"
   2) 1) "f1"
      2) "v1"
5) 1) "1587969310314-2"
   2) 1) "f1"
      2) "v1"
```
可选参数`count`可以限制返回结果数量的上限：
```Redis
>  xrange ms-3 1587969310313 1587969310314 count 1
1) 1) "1587969310313-0"
   2) 1) "f1"
      2) "v1"
```

### 迭代Stream
结合`ID`自增的特性和 **`XRANGE`**的`count`参数，我们可以完成对整个`stream`的迭代。首先：
```Redis
> xrange ms-3 - + count 2
1) 1) "1587969310313-0"
   2) 1) "f1"
      2) "v1"
2) 1) "1587969310313-1"
   2) 1) "f1"
      2) "v1"
```
然后将当前迭代返回的最后一个`ID`的`sequence number`加一，作为下一次迭代的起始`ID`：
```Redis
> xrange ms-3 1587969310313-1 + count 2
1) 1) "1587969310313-1"
   2) 1) "f1"
      2) "v1"
2) 1) "1587969310314-0"
   2) 1) "f1"
      2) "v1"
```
重复这个过程即可。以上演示的是`ID`从小到大的正向迭代过程，若要从大到小进行反向迭代，可以使用 **`XREVRANGE key end start [COUNT count]`**。

### 查询单条消息
如果只需要查询单条消息，可以将`count`参数设为`1`。
```Redis
> xrange ms-3 1587969310313-0 + count 1
1) 1) "1587969310313-0"
   2) 1) "f1"
      2) "v1"
```
因为 **`XRANGE`**和 **`XREVRANGE`**采用的是闭区间，所以当`start`和`end`相同时，查询的结果就是单条消息。
```Redis
> xrange ms-3 1587969310313-0 1587969310313-0
1) 1) "1587969310313-0"
   2) 1) "f1"
      2) "v1"
> xrevrange ms-3 1587969310313-0 1587969310313-0
1) 1) "1587969310313-0"
   2) 1) "f1"
      2) "v1"
```

## 消费者
有时候，我们并不希望主动去查询`stream`，而是希望当`stream`中有新的消息时，新的消息可以直接推送给我们，也就是订阅`stream`。**`XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] id [id ...]`**正是用来监听`stream`中新的消息的。例如：
```Redis
> xread count 1 streams ms-3 0
1) 1) "ms-3"
   2) 1) 1) "1587969310313-0"
         2) 1) "f1"
            2) "v1"
```
**`XREAD`**不支持`-`和`+`，`id`表示消费者上一次收到的消息`ID`，所以返回的消息`ID`都会大于`id`。
```Redis
> xread count 1 streams ms-3 1587969310313-0
1) 1) "ms-3"
   2) 1) 1) "1587969310313-1"
         2) 1) "f1"
            2) "v1"
```

### 阻塞型消费者
默认情况下，**`XREAD`**是非阻塞的。通过使用可选的`[BLOCK] milliseconds`参数，我们可以将 **`XREAD`**变成阻塞操作。在消息到来之前，**`XREAD`**会在新消息到来之前阻塞阻塞一段时间，若在超时之前有新的消息到来，则结束阻塞并返回新的消息，否则超时自动返回。
```Redis
> xrevrange ms-3 + - count 2    // 获取最新的两条消息
1) 1) "1587969310316-2"
   2) 1) "f1"
      2) "v1"
2) 1) "1587969310316-1"
   2) 1) "f1"
      2) "v1"
> xread count 1 block 1000 streams ms-3 1587969310316-1 // 因为有消息，直接返回
1) 1) "ms-3"
   2) 1) 1) "1587969310316-2"
         2) 1) "f1"
            2) "v1"
> xread count 1 block 1000 streams ms-3 1587969310316-2 // 没有新的消息，阻塞，超时返回
(nil)
(1.03s)
```
`BLOCK 0`表示新消息到来之前一直阻塞，不会超时返回。在`client-2`上阻塞，`$`表示当前`stream`中最新消息的`ID`(使用`$`之后，每个消息最多收到一次)，所以`client-2`会一直阻塞：
```Redis
client-2:6379> xread count 1 block 0 streams ms-3 $

```
这个时候在`client-1`上向`ms-3`中写入一条消息：
```Redis
client-1:6379> xadd ms-3 * f1 v1
"1587974321051-0"
```
`client-2`收到消息，结束阻塞，返回：
```Redis
client-2:6379> xread count 1 block 0 streams ms-3 $
1) 1) "ms-3"
   2) 1) 1) "1587974321051-0"
         2) 1) "f1"
            2) "v1"
(23.59s)
```

## 消费组
**`XREAD`**可以让多个消费者都获取到`stream`中的所有消息。将所有的消息同时分发给多个消费者有时候并不能提高效率，因为多个消费者可能做着重复的工作，而我们只需要一份结果。一个比较好的处理方式是：<u>将`stream`中的消息划分为多个彼此不重叠的子集，然后将不同的子集分发给不同消费者处理</u>。为了处理一个大的任务，多个物理消费者可以联合起来，每个物理消费者处理不同的部分，外界看起来就像只有一个消费者一样，这样，多个物理消费者构成了一个逻辑消费者，外界看到的正是这个逻辑消费者。在Redis里，这个逻辑消费者就是一个消费组(**Consumer Group**)，物理消费者可以根据实际需要加入或者离开逻辑消费者，逻辑消费者从`stream`中获取数据，数据由多个物理消费者进行处理，并且：
1. 同一个消息不会被分发给多个消费者
2. 消费组中的每一个消费者都通过名字唯一识别，名字是一个大小写敏感的字符串
3. 每个消费组都有*first ID never consumed*的概念，这样一来，当一个消费者请求新的消息时，它可以提供从未被交付过的消息
4. 使用特定的命令作为某个消息已经被正确处理的确认，此后该消息可以被移出消费组了
5. 消费组追踪每个当前被分发给某个消费者但还未收到处理完成的确认的消息，消息的这种状态被称为`pending`。Redis使用`Pending Entries List (PEL)`来追踪这些消息

**`XGROUP [CREATE key groupname id-or-$] [SETID key groupname id-or-$] [DESTROY key groupname] [DELCONSUMER key groupname consumername]`**是一个用来管理和`key`上`stream`相关联的消费组的命令。

### 创建消费组
要创建一个消费组，可以使用 **`XGROUP CREATE key groupname id-or $`**子命令。最后一个参数`id-or-$`是消费组开始消费的消息的`ID`(不包括这个`ID`代表的消息)。如果希望消费组从下一个最新的消息开始消费，可以使用`$`，`$`表示`stream`中的最后一个消息的`ID`。如果希望消费组消费`stream`中的所有消息，可以使用`0`作为传入`ID`。如果消费组已经存在，命令将放回一个`-BUSYGROUP`错误，否则创建成功并返回OK。

如果`stream`不存在，命令会返回一个错误，可以在`ID`后面新增一个可选`MKSTREAM`子命令让Redis自动创建一个`stream`，这个时候创建出来的`stream`的长度为`0`。
```Redis
> xgroup create ms-4 group0 0   // ms-4不存在
(error) ERR The XGROUP subcommand requires the key to exist. Note that for CREATE you may want to use the MKSTREAM option to create an empty stream automatically.
> xgroup create ms-4 group0 0 MKSTREAM  // 创建group0，并让Redis自动创建出stream
OK
> exists ms-4
(integer) 1
> xadd ms-4 0-1 f1 v1   // 添加一条消息到ms-4
"0-1"
> xgroup create ms-4 group1 $   // 创建group1
OK
> xadd ms-4 1-1 f1 v1
"1-1"
> xinfo groups ms-4
1) 1) "name"
   2) "group0"
   3) "consumers"
   4) (integer) 0
   5) "pending"
   6) (integer) 0
   7) "last-delivered-id"
   8) "0-0"
2) 1) "name"
   2) "group1"
   3) "consumers"
   4) (integer) 0
   5) "pending"
   6) (integer) 0
   7) "last-delivered-id"
   8) "0-1"
```
每个消费组都与一个`stream`以及最近交付的消息`ID`(即上面的`last-delivered-id`)相关联。

#### 添加消费者到消费组
**`XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] ID [ID ...]`**是 **`XREAD`**的一个特殊版本，它支持Consumer Group。`STREAMS`选项的`ID`可以是以下两种形式：
* `>`。`>`是一个特殊的`ID`，表示消费者只想接收那些从未被交付给任何其他消费者的消息，即消费者想收到一个从未被交付过的消息
* 任何其它的`ID`，即`0`，其它有效`ID`或`partial ID`。这个时候，我们可以得到消费者的`PEL`(返回的`PEL`中的`ID`比传递的`ID`都要大)，而`BLOCK`和`NOACK`参数会被忽略。

```Redis
> xreadgroup group group0 consumerA count 1 block 1000 streams ms-4 >
1) 1) "ms-4"
   2) 1) 1) "0-1"
         2) 1) "f1"
            2) "v1"
> xreadgroup group group0 consumerA streams ms-4 0  // 查看consumerA的PEL
1) 1) "ms-4"
   2) 1) 1) 0-1"
         2) 1) "f1"
            2) "v1"
```
上面这条命令让消费者`consumerA`加入到了`group0`中并消费了一条消息。如果这个时候我们再次执行`xinfo groups ms-4`，会得到和上一次不同的结果：
```Redis
> xinfo groups ms-4
1) 1) "name"
   2) "group0"
   3) "consumers"
   4) (integer) 1
   5) "pending"
   6) (integer) 1
   7) "last-delivered-id"   // 0-1已被group0中的consumerA消费
   8) "0-1"
2) 1) "name"
   2) "group1"
   3) "consumers"
   4) (integer) 0
   5) "pending"
   6) (integer) 0
   7) "last-delivered-id"
   8) "0-1"
```
若要查看某个消费组中的消费者信息，可以使用 **`XINFO CONSUMERS <key> <group>`**：
```Redis
> xinfo consumers ms-4 group0
1) 1) "name"
   2) "comsumerA"
   3) "pending"
   4) (integer) 1
   5) "idle"
   6) (integer) 905911
```

### 消息处理完成确认
**`XACK key group ID [ID ...]`**从消费组的`PEL`移除一个或多个消息，表示他们已被成功处理。假设`group0`中的consumerA已经将`0-1`这条消息处理完成，它需要告知Redis服务器它已经这条消息处理完毕：
```Redis
> xack ms-4 group0 0-1  // 发送确认
(integer) 1
> xreadgroup group group0 consumerA streams ms-4 0  // 查看consumerA的PEL
1) 1) "ms-4"
   2) (empty list or set)
```

当消息处理失败时，没有 **`XACK`**确认，消费者可以再次进行消费。有时候，当失败在可接受范围内时，我么可以使用 **`XREADGROUP`**中的子命令 **`NOACK`**来告诉Redis不需要进行确认，Redis会认为 **`XREADGROUP`**返回的所有消息都已确认，因此不用再维护对应的`PEL`。

### 管理消费组
先准备一些测试数据，创建3个消费组，它们都从最新的消息开始消费：
```Redis
> xgroup create ms-5 group0 $ MKSTREAM
OK
> xgroup create ms-5 group1 $
OK
> xgroup create ms-5 group3 $
OK
```
向`ms-5`中写入7条消息：
```Redis
> 7 xadd ms-5 * f1 v1
"1587991174542-0"
"1587991174542-1"
"1587991174542-2"
"1587991174543-0"
"1587991174543-1"
"1587991174543-2"
"1587991174544-0"
```

#### 修改消费者的位置
**`XGROUP SETID key groupname id-or-$`**允许我们修改消费组的`last-delivered-id`的值，进而改变接下来要消费的消息(即消费者的位置)。若要重新消费所有消息，可以使用：
```Redis
> xgroup setid ms-5 group1 0
OK
```
若现在指向消费最新的消息，则可以使用：
```Redis
> xgroup setid ms-5 group1 $
OK
```
#### 删除消费组
因为并不会自动删除未使用的消费组，所以我们需要自己来做这件事情。**`XGROUP DESTROY key groupname`**会永久删除某个消费组和相关的消费者。
```Redis
> xgroup destroy ms-5 group3  // 删除group3
(integer) 1
```

#### 从消费组删除消费者
**`XGROUP DELCONSUMER key groupname consumername`**命令用于从消费组里面删除消费者，它还会删除消费者对应的`PEL`，返回值为被删消费的的`PEL`中的消息数量。
```Redis
> xreadgroup group group0 consumerA count 1 block 1000 streams ms-5 >
1) 1) "ms-5"
   2) 1) 1) "1587991174542-0"
         2) 1) "f1"
            2) "v1"
> xreadgroup group group0 consumerB count 1 block 1000 streams ms-5 >
1) 1) "ms-5"
   2) 1) 1) "1587991174542-1"
         2) 1) "f1"
            2) "v1"
> xinfo consumers ms-5 group0       // 查看group0中的消费者
1) 1) "name"
   2) "consumerA"
   3) "pending"
   4) (integer) 1
   5) "idle"
   6) (integer) 88581
2) 1) "name"
   2) "consumerB"
   3) "pending"
   4) (integer) 1
   5) "idle"
   6) (integer) 79710
> xgroup delconsumer ms-5 group0 consumerB   // 删除group0中的consumerB
(integer) 1
> xinfo consumers ms-5 group0
1) 1) "name"
   2) "consumerA"
   3) "pending"
   4) (integer) 1
   5) "idle"
   6) (integer) 1540978
```
在删除消费者的时候一定要消息，因为它可能还有未确认的消息(`PEL`不为空)。

### 消费失败问题
理想情况下，所有交付的消息都会被确认(**`XACK`**)。但是有时候，消费者可能在发出 **`XACK`**之前就已经下线了，这个时候未确认的消息就会一直留在`PEL`中。

#### 查看未确认的消息
有两种方式可以查看未确认的消息：**`XINFO CONSUMERS key group`**和 **`XPENDING key group [start end count] [consumer]`**。

先准备测试数据，创建消费组`group0`并向`ms-6`中写入5条消息，然后添加消费者`consumerA`：
```Redis
> xgroup create ms-6 group0 0 MKSTREAM
OK
> 5 xadd ms-6 * f1 v1
"1587996240331-0"
"1587996240331-1"
"1587996240332-0"
"1587996240332-1"
"1587996240333-0"
> xreadgroup group group0 consumerA count 1 block 1000 streams ms-6 >
1) 1) "ms-6"
   2) 1) 1) "1587996240331-0"
         2) 1) "f1"
            2) "v1"
```
查看消息确认状态：
```Redis
> xinfo consumers ms-6 group0
1) 1) "name"
   2) "consumerA"
   3) "pending"
   4) (integer) 1
   5) "idle"
   6) (integer) 44813
> xpending ms-6 group0
1) (integer) 1
2) "1587996240331-0"
3) "1587996240331-0"
4) 1) 1) "consumerA"
      2) "1"
```
结果显示`consumerA`消费的一条消息还未确认。

#### 更换消费者
假设`consumerA`在发送确认消息之前下线了，为了继续处理`consumerA`在下线前未处理完的消息，我们可以使用 **`XCLAIM key group consumer min-idle-time ID [ID ...] [IDLE ms] [TIME ms-unix-time] [RETRYCOUNT count] [FORCE] [JUSTID]`**命令让另一个消费者获得未处理完的消息并继续处理。

先创建一个消费者`consumerB`：
```Redis
> xreadgroup group group0 consumerB count 1 block 1000 streams ms-6 >
1) 1) "ms-6"
   2) 1) 1) "1587996240331-1"
         2) 1) "f1"
            2) "v1"
> xack ms-6 group0 1587996240331-1
(integer) 1
> xreadgroup group group0 consumerB streams ms-6 0
1) 1) "ms-6"
   2) (empty list or set)     // consumerB的消息都已经处理完并确认
```
将`consumerA`未处理完的消息`1587996240331-0`转交给`consumerB`：
```Redis
> xclaim ms-6 group0 consumerB 1000 1587996240331-0
1) 1) "1587996240331-0"
   2) 1) "f1"
      2) "v1"
```
查看`consumerA`和`consumerB`的`PEL`：
```Redis
> xreadgroup group group0 consumerA streams ms-6 0
1) 1) "ms-6"
   2) (empty list or set)
> xreadgroup group group0 consumerB streams ms-6 0
1) 1) "ms-6"
   2) 1) 1) "1587996240331-0"
         2) 1) "f1"
            2) "v1"
```
如果我们运行 **`XPENDING`**，也可以发现消息`1587996240331-0`现在归`consumerB`所有：
```Redis
> xpending ms-6 group0
1) (integer) 1
2) "1587996240331-0"
3) "1587996240331-0"
4) 1) 1) "consumerB"
      2) "1"
```

## 参考文献
1. [Introduction to Redis Streams](https://redis.io/topics/streams-intro).