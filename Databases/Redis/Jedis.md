# Jedis
[Jedis](https://github.com/xetorthio/jedis)是一个Java编写的Redis客户端，提供了完整的Redis API。Redis客户端通常需要具备三种能力：管理Redis连接、实现Redis序列化协议[RESP](https://redis.io/topics/protocol)(REdis Serialization Protocol)和实现可供编程人员调用的Redis API(GET、SET等)。

除了Jedis，还有两个Java编写的Redis客户端非常流行，它们是[Lettuce](https://github.com/mp911de/lettuce)和[Redisson](https://github.com/mrniko/redisson)。

## 连接到Redis服务
```Java
Jedis jedis = new Jedis("127.0.0.1", 6379);
jedis.set("hello", "Redis");
assertThat(result, is("OK"));
String value = jedis.get("hello");
assertThat(value, is("Redis"));
jedis.close();  // 关闭TCP连接，防止发生连接泄漏
```
Jedis API的使用和redis-cli非常类似，例如redis-cli中的 **`SET`** 和 **`GET`**在Jedis中就是`jedis.set(...)`和`jedis.get(...)`。

### 在多线程环境下使用Jedis
`Jedis`实例会维护一个到Redis的TCP连接，它<u>不是线程安全的</u>。因此，为了避免遇到未知问题，我们不应该让同一个`Jedis`实例被多个线程共享。为每一个线程都创建一个`Jedis`实例也不是一个好主意，因为这会创建大量的Socket连接，也可能导致未知问题。为了避免这些问题，我们可以在多线程环境下使用`JedisPool`，它是一个线程安全的网络连接池。使用`JedisPool`既可以帮助我们克服非线程安全引发的未知问题，同时，由于复用了连接，应用的性能也会有所提高。为了使用`JedisPool`，我们需要先初始化它：
```Java
JedisPool jedisPool = new JedisPool(new JedisPoolConfig(), "127.0.0.1", 6379);
```
`JedisPool`基于[Apache Commons Pool 2](http://commons.apache.org/proper/commons-pool/index.html)实现。`JedisPoolConfig`也继承于[GenericObjectPoolConfig](http://commons.apache.org/proper/commons-pool/apidocs/org/apache/commons/pool2/impl/GenericObjectPoolConfig.html)，它包含了一些非常有用的连接池默认值。我们可以这么使用`JedisPool`:
```Java
try (Jedis jedis = jedisPool.getResource()) {
    String result = jedis.set("hello", "Redis");
    assertThat(result, is("OK"));
    String value = jedis.get("hello");
    assertThat(value, is("Redis"));
}
jedisPool.close();
```

### 选择正确的Java类去连接Redis

| Thread-safe | Deployment Type  | Connection        |
| ----------- | ---------------- | ----------------- |
| No          | Single Redis     | Jedis             |
| No          | Redis Enterprise | Jedis             |
| Yes         | Single Redis     | JedisPool         |
| Yes         | Redis Enterprise | JedisPool         |
| Yes         | Redis Sentinel   | JedisSentinelPool |
| Yes         | RedisCluster     | JedisCluster      |

## Java-Redis类型映射

| Redis Type | Java Type           |
| ---------- | ------------------- |
| string     | String              |
| list       | List<String>        |
| set        | Set<String>         |
| hash       | Map<String, String> |
| float      | Double              |
| integer    | Long                |

## Pipelining
pipeline内的所有命令都是独立的，它们不是一个原子的整体。
```Java
Pipeline p = jedis.pipelined();
Response<Long> hsetResponse = p.hset(statusKey, "available", "true");
Response<Long> expireResponse = p.expire(statusKey, 1000);
Response<Long> saddResponse = p.sadd(availableKey, "1");
p.sync();   // 通过pipeline发送所有的命令并获取返回值
assertThat(hsetResponse.get(), is(1L));
assertThat(expireResponse.get(), is(1L));
assertThat(saddResponse.get(), is(1L));
```

## Transactions
很多客户端用pipeline实现事务，这么做既高效，又能保证原子性。
```Java
Transaction t = jedis.multi();
Response<Long> hsetResponse = t.hset(statusKey, "available", "true");
Response<Long> expireResponse = t.expire(statusKey, 1000);
Response<Long> saddResponse = t.sadd(availableKey, "1");
t.exec();
assertThat(hsetResponse.get(), is(1L));
assertThat(expireResponse.get(), is(1L));
assertThat(saddResponse.get(), is(1L));
```

## 参考文献
1. [Jedis wiki](https://github.com/xetorthio/jedis/wiki).