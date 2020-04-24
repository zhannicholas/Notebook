# Redis中的事务
Redis中提供了5个和事务相关的命令：**`MULTI`**、**`EXEC`**、**`DISCARD`**、**`WATCH key [key ...]`**和 **`UNWATCH`**。Redis事务一次执行一组命令，并保证：
* 一个事务中的所有命令按顺序串行执行。若某一个连接已开启事务，其它连接提交的命令并不会插入到这个事务中，而是会单独执行，即**隔离性**——每个会话之间是相互隔离的
* 要么所有的命令都执行，要么一条命令都不执行，即**原子性**。Redis保证当某个连接断开后，之前在该连接上所有`QUEUED`的命令一个也不会执行

## 使用事务
**`MULTI`**标志着一个事务块的开始，随后的命令不会立即执行，而是会被放入一个队列。当遇到 **`EXEC`**时，队列中的命令会以原子的方式按入队顺序执行，队列中的所有命令构成了一个单一的原子操作。而当遇到 **`DISCARD`**命令时，事务队列会被清空，队列中的所有命令被丢弃，事务结束，连接回到正常状态，如果某些键处于`watched`状态，它们将不再被`watch`。

下面是一个例子，从`acount:B`划`20`到`acount:A`的这个操作是原子的：
```Redis
> mset account:A 100 account:B 200
OK
> multi
OK
> decrby account:B 20
QUEUED
> incrby account:A 20
QUEUED
> exec
1) (integer) 180
2) (integer) 120
> mget account:A account:B
1) "120"
2) "180"
```
从上面这段代码可以发现，**`MULTI`**后的命令都被`QUEUED`，执行 **`EXEC`**会返回一个数组，数组中的元素就是之前入队的命令的执行结果，它们是顺序对应的。

## 当事务中发生错误
在一个事务内，可能发生以下两类错误：
* 调用 **`EXEC`**之前有命令入队失败。多数情况下是由于编程错误(例如语法错误)，也可能是系统错误(例如内存用尽)，也可能是被当前线程 **`WATCH`**的键在调用 **`MULTI`**之后被其它线程修改
* 调用 **`EXEC`**之后有命令执行失败。例如在特定类型的数据上执行该类型不支持的操作

对于第一种情况，当前事务会被标记为**无效**状态，错误的命令并不会入队，其后的命令会继续入队，但当调用 **`EXEC`**时，并不会开启事务，而是会异常中止。当`xx`命令由于语法错误而入队失败，事务处于无效状态，调用 **`EXEC`**并不会开启事务，而是直接终止，**`INCR account:B`**也没有执行：
```Redis
> multi
OK
> xx
(error) ERR unknown command `xx`, with args beginning with:
> incr account:B
QUEUED
>exec
(error) EXECABORT Transaction discarded because of previous errors.
```

对于第二种情况，**当某个命令执行失败时，Redis会继续执行后续命令，因为Redis不支持回滚**。继续以`account:A`和`account:B`举例，我们先在`string`上执行 **`LPOP`**命令，命令本身没有语法错误，因此入队成功，开启事务后，`lpop account:A`执行失败，而后续命令正常执行：
```Redis
> multi
OK
> lpop account:A
QUEUED
> incr account:B
QUEUED
> exec
1) (error) WRONGTYPE Operation against a key holding the wrong kind of value
2) (integer) 181
> mget account:A account:B
1) "120"
2) "181"
```

第一种情况和第二种情况最大的区别在于：前者会导致 **`EXEC`**执行失败，而后者不会。

## 乐观并发控制
**`WATCH`**为Redis事务提供了基于`check-and-set(CAS)`的乐观并发控制机制，作用范围为单个连接。**`WATCH`**必须在 **`MULTI`**之前调用，它在调用 **`EXEC`**命令之前持续`watch`一组键，若被`watch`的键中有至少一个被其它客户端修改，整个事务就会中止，**`EXEC`**也会返回`NULL`以表示事务失败(没有任何命令被执行)。**`WATCH`**可以被多次调用，后面的调用不会重置之前被`watch`的键，而是持续增加新的被`watch`的键。

**`UNWATCH`**会使得当前被`watch`的所有键不再被监视。如果已经在 **`WATCH`**之后调用了 **`EXEC`**或 **`DISCARD`**，就不需要再次手动调用 **`UNWATCH`**了，因为它们会自动`unwatch`。

## 参考文献
1. [Transactions](https://redis.io/topics/transactions).