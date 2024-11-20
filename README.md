
## 前言


Redis在我们的日常开发工作中，使用频率非常高，已经变成了必不可少的技术之一。


Redis的使用场景也很多。


比如：保存用户登录态，做限流，做分布式锁，做缓存提升数据访问速度等等。


那么问题来了，Redis的性能要如何优化？


为了充分发挥Redis的性能，这篇文章跟大家一起聊聊Redis性能优化的18招，希望对你会有所帮助。


## 1\. 选择合适的数据结构


Redis支持多种数据结构，如字符串、哈希、列表、集合和有序集合。根据实际需求选择合适的数据结构可以提高性能。


如果要存储用户信息，考虑使用哈希而不是多个字符串：



```
jedis.hset("user:1001", "name", "Alice");
jedis.hset("user:1001", "age", "30");

```

这样可以高效地存储和访问多个属性。


## 2\. 避免使用过大的key和value


较长的key和value会占用更多内存，还可能影响性能。


保持key简短，并使用简洁的命名约定。


比如：
将“user:1001:profile”简化为“u:1001:p”。


还可以做压缩等其他优化。


如果对大key问题，比较感兴趣可以看看我的另一篇文章《[从2s优化到0\.1s，我用了这5步](https://github.com)》，里面有非常详细的介绍。


## 3\. 使用Redis Pipeline


对多个命令的批量操作，使用Pipeline可以显著降低网络延迟，提升性能。


比如，批量设置key可以这样做：



```
Pipeline p = jedis.pipelined();
for (int i = 0; i < 1000; i++) {
    p.set("key:" + i, "value:" + i);
}
p.sync();

```

这样一次性可以发送多个命令，减少了网络往返时间，能够提升性能。


## 4\. 控制连接数量


过多的连接会造成资源浪费，使用`连接池`可以有效管理连接数量。


比如，使用JedisPool：



```
JedisPool pool = new JedisPool("localhost");
try (Jedis jedis = pool.getResource()) {
    jedis.set("key", "value");
}

```

有了连接池，这样连接就会被复用，而不是每次都创建新连接，使用完之后，又放回连接池。


能有效的节省连接的创建和销毁时间。


## 5\. 合理使用过期策略


设置合理的过期策略，能防止内存被不再使用的数据占满。


例如，缓存热点数据可以设置过期时间。


比如，对会话数据设置过期时间：



```
jedis.setex("session:12345", 3600, "data");

```

Redis内部会定期清理过期的缓存。


## 6\. 使用Redis集群


数据量增大时，使用Redis集群可以将数据分散到多个节点，提升并发性能。


可以将数据哈希分片到多个Redis实例。


这样可以避免单个Redis实例，数据太多，占用内存过多的问题。


## 7\. 充分利用内存优化


选择合适的内存管理策略，Redis支持LRU（Least Recently Used）策略，可以自动删除不常用的数据。


比如，配置Redis的maxmemory：



```
maxmemory 256mb
maxmemory-policy allkeys-lru

```

## 8\. 使用Lua脚本


Lua脚本让多条命令在Redis中原子性执行，减少网络延迟。


比如，使用Lua防止多个命令的网络延迟：



```
EVAL "redis.call('set', KEYS[1], ARGV[1]) return redis.call('get', KEYS[1])" 1 "key" "value"

```

使用Lua脚本，可以保证Redis的多个命令是原子性操作。


## 9\. 监控与调优


使用INFO命令监控Redis性能数据，如命令支持、内存使用等，及时调优。


比如，使用命令获取监控信息：



```
INFO memory
INFO clients

```

## 10\. 避免热点key


热点key会造成单一节点的压力，通过随机化访问来避免。


比如，可以为热点key加随机后缀：



```
String key = "hotkey:" + (System.currentTimeMillis() % 10);
jedis.incr(key);

```

## 11\. 使用压缩


存储大对象时，考虑使用压缩技术来节省内存。


比如，可以使用`GZIP`压缩JSON数据：



```
byte[] compressed = gzipCompress(jsonString);
jedis.set("data", compressed);

```

## 12\. 使用Geo位置功能


Redis支持地理位置存储和查询，使用`GEOADD`可以高效管理地理数据。


比如，存储地点信息：



```
jedis.geoadd("locations", longitude, latitude, "LocationName");

```

## 13\. 控制数据的持久化


合理设置`RDB`和`AOF`的持久化策略，避免频繁写盘造成性能下降。


示例：
设置持久化的时间间隔：



```
save 900 1
appendonly yes

```

## 14\. 尽量减少事务使用


在高并发场景下，避免过度使用MULTI/EXEC，因为事务会锁住key。


可以直接使用单条命令替代事务。


## 15\. 合理配置客户端


调整客户端的连接超时和重连策略，以适应高负载场景，确保连接稳定。


例如：



```
JedisPoolConfig poolConfig = new JedisPoolConfig();
poolConfig.setMaxTotal(128); // 最大连接数
poolConfig.setMaxIdle(64); // 最大空闲连接
poolConfig.setMinIdle(16); // 最小空闲连接
poolConfig.setTestOnBorrow(true);
poolConfig.setTestOnReturn(true);
poolConfig.setTestWhileIdle(true);

JedisPool jedisPool = new JedisPool(poolConfig, "localhost", 6379, 2000); // 连接超时2000ms

```

## 16\. 使用Redis Sentinel


使用`Sentinel`进行监控，实现高可用性，确保系统在故障时能够快速切换。


配置Sentinel进行主从复制。


## 17\. 优化网络配置


保证Redis服务器有良好的网络带宽，避免网络瓶颈。


使用服务器内部专线，减少延迟。


## 18\. 定期清理不必要的数据


生命周期管理很关键，定期删除过期或不必要的数据，保持内存高效利用。


可以设置`Cron`任务定期清理。


虽说Redis内部会清理过期的数据，但有些长期存在的垃圾数据，也建议及时清理。


## 总结


以上就是Redis性能优化的18条军规，灵活应用这些策略能够为你的项目带来显著的性能提升。希望能帮助到你，欢迎分享你的优化经验！


## 最后说一句(求关注，别白嫖我)


如果这篇文章对您有所帮助，或者有所启发的话，帮忙扫描下发二维码关注一下，您的支持是我坚持写作最大的动力。
求一键三连：点赞、转发、在看。
关注公众号：【苏三说技术】，在公众号中回复：进大厂，可以免费获取我最近整理的10万字的面试宝典，好多小伙伴靠这个宝典拿到了多家大厂的offer。


 本博客参考[milou加速器](https://xinminxuehui.org)。转载请注明出处！
