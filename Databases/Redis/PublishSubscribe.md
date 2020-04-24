# Publish/Subscribe
Redis通过 **`PUBLISH`**、**`SUBSCRIBE`**等命令实现了发布-订阅模式，这个功能提供了两种信息机制：`simple syndication`和`pattern syndication`。值得注意的是：<u>发布-订阅不保证完成消息的交付。当消息发布时到某个`channel`上时，只有那些订阅了该`channel`并且连接上了的客户端才会收到消息。消息一旦发布，就会被丢弃，只有那些当前订阅了的客户端才会收到该消息。也就是说，客户端只有在消息发布前订阅，才能收到该消息。</u>。

## Simple syndication
在`simple syndication`下，Redis使用 **`PUBLISH channel message`**发布`message`到指定`channel`，然后返回收到该消息的客户端数量。

客户端使用 **`SUBSCRIBE channel [channel ...]`**来监听指定的`channel`，此后客户端不能再发出除 **`SUBSCRIBE`**、**`PSUBSCRIBE`**、**`UNSUBSCRIBE`**、**`PUNSUBSCRIBE`**、**`PING`**和 **`QUIT`**以外的命令。对于`redis-cli`来说，一旦处于订阅模式，就不再接收任何命令并且只能通过`Ctrl-C`来退出。

若客户端不想再监听某个`channel`上的消息，可以执行 **`UNSUBSCRIBE [channel [channel ...]]`**来停止对指定`channel`上消息的订阅。若没有给出`channel`参数，客户端将停止对之前订阅的所有`channel`的监听。

下面是一个例子：
首先`client-2`订阅了`ch1`上的消息：
```Redis
client-2:6379> subscribe ch1
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "ch1"
3) (integer) 1
```
然后`client-1`往`ch1`上发布消息：
```Redis
client-1:6379> publish ch1 "hello"
(integer) 1             // 有一个订阅者收到了消息
```
收到消息的正是`client-2`，`client-2`上应该可以看到这样的信息：
```Redis
client-2:6379> subscribe ch1
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "ch1"
3) (integer) 1
1) "message"
2) "ch1"
3) "hello"
```
### 消息的格式
Redis中，`channel`的订阅者接收到的消息是一个由三个元素组成的数组。其中第一个元素是消息的类型，后两个元素的内容随消息的类型而变化。有三种类型的消息：
* `subscribe`：成功订阅某个`channel`，第二个元素为被订阅`channel`的名字，第三个元素为当前订阅的`channel`数量
* `unsubscribe`：成功取消对某个`channel`的订阅，第二个元素为取消订阅的`channel`的名字，第三个元素为当前订阅的`channel`数量(为`0`表示不再订阅任何`channel`)
* `message`：收到了发布的消息，第二个元素为这个消息的来源(`channel`)，第三个元素为消息的内容

## Pattern syndication
有时候我们想接收所有的消息，这个时候把所有的`channel`都订阅就行了。但有时候我们可能希望对消息的来源进行过滤，只接收满足过滤条件的`channel`上的信息。这个时候就可以使用 **`PSUBSCRIBE pattern [pattern ...]`**来订阅满足给定模式的`channel`了。Redis支持`glob-style`的`pattern`：
* `h?llo`表示订阅`hello`、`hillo`、`hallo`等
* `h*llo`表示订阅`hllo`、`hello`、`haello`等
* `h[ae]llo`表示订阅`hallo`和`hello`，而不是`hxllo`或`hbllo`等

**`PUNSUBSCRIBE [pattern [pattern]]`**表示不再订阅满足给定模式的`channel`。若没有给出`pattern`参数，客户端将停止对之前订阅的所有`channel`的监听。

接着上一部分举例，新开一个`client-3`订阅`ch?`模式的`channel`：
```Redis
client-3:6379> psubscribe ch?
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"         // 订阅pattern成功
2) "ch?"                // 订阅的pattern
3) (integer) 1          // 订阅该pattern的订阅者数量
```
然后`client-1`往`ch3`上发布消息：
```Redis
client-1:6379> publish ch3 "hello, ch3"
(integer) 1             // 有一个客户端收到了消息
```
收到消息的正是`client-3`，由于`client-2`只订阅了`ch1`，因此它不会收到消息。`client-3`上应该可以看到这样的信息：
```Redis
client-3:6379> psubscribe ch?
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "ch?"
3) (integer) 1
1) "pmessage"   // 收到消息
2) "ch?"        // pattern
3) "ch3"        // 消息对应的channel
4) "hello, ch3" // 消息内容
```

### 消息的格式
使用 **`PSUBSCRIBE`**之后，订阅者接收到的消息和 **`SUBSCRIBE`**大体类似，但消息内容的含义稍有变化，具体的见上面的注释。

## 查看Pub/Sub系统状态
**`PUBSUB subcommand [argument [argumeng ...]]`命令允许我们查看发布-订阅系统的内部状态，共支持3个子命令：
* **`PUBSUB CHANNELS [pattern]`**：查看当前处于活动状态的`channel`(有至少一个订阅者的`channel`)。若未提供`pattern`参数，将返回所有的处于活动状态的`channel`，否则只返回满足`pattern`并处于活动状态的`channel`
* **`PUBSUB NUMSUB [channel-1 ... channel-N]`**：查看给定`channel`的订阅者数量(不包括订阅`pattern`的订阅者)，返回结果的格式为`channel,count,channel,count,...,`。若未提供`channel`参数，将返回一个空列表
* **`PUBSUB NUMPAT`**：返回被订阅`pattern`的数量

接着上面的例子，在`client-1`上执行 **`PUBSUB`**，查看系统状态：
```Redis
client-1:6379> pubsub channels       // 不带pattern参数
1) "ch1"
client-1:6379> pubsub channels ch?
1) "ch1"
client-1:6379> pubsub numsub ch1    // 查看ch1上订阅者的数量
1) "ch1"
2) (integer) 1
client-1:6379> pubsub numsub        // 不带参数
(empty list or set)
client-1:6379> pubsub numpat
(integer) 1                         // 当前被订阅的pattern数量为1
```

## 参考文献
1. [Pub/Sub](https://redis.io/topics/pubsub).
2. [Publish–subscribe pattern](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern).