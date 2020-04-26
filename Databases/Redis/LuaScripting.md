# Redis Lua脚本
Redis支持在服务端使用Lua解释器执行Lua脚本。Redis本身已经提供了非常多的命令，而Lua脚本可以帮助我们使用Lua提供的语言特性来组织这些命令，以达到更好的效果。总的来说，使用Lua脚本带来的好处有：
* **降低网络开销**。客户端每次只能操作一个键，客户端每发出一个命令，都会引发一个完整请求-响应过程，当需要发送大量指令时，这会引起巨大的网络开销。因为Lua脚本是直接在在Redis服务端执行的，将多个命令放在同一个Lua脚本中执行或在同一个Lua脚本中操作多个键可以减少网络传输次数，进而降低网络开销
* **原子操作**。Redis会将整个Lua脚本作为一个整体以原子的方式执行，就像事务一样
* **可复用性**。Lua脚本就像是数据库中的存储程序一样，支持传入参数，反复使用。Redis还会缓存已执行的脚本，这意味着其它的客户端可以直接使用已缓存的脚本

## 执行脚本
执行Lua脚本的命令为 **`EVAL script numkeys key [key ...] arg [arg ...]`**和 **`EVALSHA sha1 numkeys key [key ...] arg [arg ...]`**。

**`EVAL`**具有很多参数：`script`代表待执行的Lua脚本，`numkeys`表示键的数量.，接下来是键名(可以通过全局变量`KEYS`访问)和脚本可能需要用到的参数(可以通过全局变量`ARGV`访问)。
```Redis
> hset hash-key f1 hello f2 Redis
(integer) 2
> eval "return redis.call('HGET', KEYS[1], ARGV[1])" 1 hash-key f2
"Redis"
> eval "return redis.call('HGET', 'hash-key', 'f2')" 0
"Redis"
```
在上面这段脚本里，创建了两个预定义的数组：`KEYS`存放传入所有键名，而`ARGV`存放传入的所有参数。最后一条命令硬编码参数，虽然执行结果和倒数第二条一致，但灵活性非常低。需要注意的是：<u>Lua中的数组下标是基于`1`的</u>。

## Lua和Redis之间的数据类型转换
Redis to Lua：

| Redis Value                              | Lua Value                               |
| ---------------------------------------- | --------------------------------------- |
| Redis Integer                            | Number                                  |
| Redis bulk reply                         | String                                  |
| Redis multi bulk reply                   | Table (with other types nested)         |
| Redis status reply                       | Table with "ok" field containing status |
| Redis error                              | Table with "err" field containing error |
| Redis nil bulk reply and nil multi reply | False (boolean type)                    |

Lua to Redis：

| Lua Value              | Redis Value                        |
| ---------------------- | ---------------------------------- |
| Number                 | Redis Integer                      |
| String                 | Redis bulk reply                   |
| Table (array)          | Redis multi bulk reply             |
| Table with "ok" field  | Redis status reply                 |
| Table with "err" field | Redis error reply                  |
| False (boolean type)   | Redis nil bulk reply               |
| True (boolean type)    | Redis integer reply with value of 1|

还有有三条重要的规则需要注意：
* Lua中没有`integer`和`float`之分，取而代之的是`number`类型。Redis会将Lua中的`number`转为`integer`，对于浮点数来说，其小数部分就会丢失。如果需要保留浮点数的小数部分，我们应该采用`string`来进行存储和查询操作
* Lua数组以`nil`作为结束标志，有效数组元素仅位于第一个`nil`出现之前。所以Redis在转换Lua数组时，遇到`nil`就会停止
* 当Lua数组包含键(和它们的值)时，转换到Redis的数据类型时不会被包含进去

## 管理脚本

### 缓存脚本
对于一个Lua脚本，客户端首先通过网络将它发送给Redis，Redis在执行脚本前需要解析(`parse`)该脚本，脚本执行完成后，执行结果会被传回给客户端。脚本的解析是需要一定开销的，所以Redis会维护一个已编译脚本的缓存。当我们需要反复执行某个脚本时，我们可以使用 **`SCRIPT LOAD script`**命令，它会解析脚本，将其载入缓存并返回脚本的`sha1`摘要值，随后可以使用 **`EVALSHA`**命令通过脚本的`sha1`摘要调用这个脚本。
```Redis
> set s1 "hello"
OK
> script load "local val=redis.call('GET', KEYS[1]) return val"
"dd47ad79bb7b6d7d2b8e0607c344d134412e84e0"
> evalsha dd47ad79bb7b6d7d2b8e0607c344d134412e84e0 1 s1
"hello"
```
只要我们没有手动执行 **`SCRIPT FLUSH`**(清空缓存中的所有脚本)或重启Redis，脚本就会一直位于缓存当中。**`SCRIPT EXISTS sha1 [sha1 ...]`**用于检测脚本是否已在缓存当中。
```Redis
> script exists dd47ad79bb7b6d7d2b8e0607c344d134412e84e0
1) (integer) 1  // 缓存中存在该脚本
```

### 其它命令
有时候，我们的脚本可能有bug或者由于某种原因导致其不能在预期的时间内执行完，我们可以使用 **`SCRIPT KILL`**来终止当前正在执行的脚本。 **`SCRIPT DEBUG YES|SYNC|NO`**可以唤起LDB(Lua Debuger)，进一步帮助我们进行脚本调试。

### 当脚本执行时间过长
当一个脚本开始执行，在一段时间内(默认为**5秒**)，Redis是不会接收其它命令的。
当执行时间超过这个阈值，Redis不会主动终止这个脚本，因为<u>Redis保证以原子的方式执行Lua脚本</u>，主动终止脚本的执行可能会导致数据出现问题。Redis会采取以下方式应对这种情况：
* Redis会在日志里面记录脚本执行时间过长
* Redis会开始接收其它命令。对于脚本执行完之前接收到的这些命令，Redis并不会执行它们，而是直接用一个`busy error`进行响应
* 如果脚本是**只读**的，也就是说脚本没有进行任何写操作，可以直接用 **`SCRIPT KILL`**安全的终止这个脚本
* 如果脚本已经进行过写入操作，那就<u>只能</u>通过 **`SHUTDOWN NOSAVE`**命令来停止服务器，并放弃所有的修改。这些修改包括：当前执行的Lua脚本造成的修改和最近一次刷新数据到磁盘之后的其它修改。

为了防止脚本的执行时间过长，我们需要尽可能地考虑脚本地执行时间，争取使用最短的时间完成相同的工作。其次，脚本其实<u>定义了事务边界</u>，知道这一点，我们可以将一个大脚本拆分为多个可以被组织在一起的原子单元。


## 参考文献
1. [Redis Lua scripting](https://redis.io/commands/eval).