>[Redis](https://redis.io)是一个位于内存中的数据结构存储系统，可用作数据库、缓存和消息中间件。它支持的数据结构有：string、hash、list、set、sorted set with range quries、bitmap、hyperloglogs、geospatial indexes with radius queries以及stream。Redis支持复制、Lua脚本、基于LRU的键驱逐、事务和不同级别的磁盘持久化，并通过哨兵和Redis集群的自动分区机制来保证高可用。

Redis的存储是基于key-value的，但这里的key-value不是简单的key-value，因为value的类型可以有很多种。在我们传统的key-value存储中，key和value都是字符串，而在Redis中，value不仅可以是字符串，还有可能是像列表、集合这样的复杂的数据结构。

若要在安装Redis，可以参考[Redis Quick Start](https://redis.io/topics/quickstart)。