# 简述 Redis 的过期机制和内存淘汰策略

内存淘汰策略用于处理内存不足时的需要申请额外空间的数据，内存淘汰策略的选取并不会影响过期的 key 的处理。

过期删除策略用于处理过期的缓存数据。



## 过期删除

过期删除策略通常有三种：

1.定时过期(Redis未使用)

每个设置过期时间的 key 都需要创建一个定时器，到过期时间就会立即清除。该策略可以立即清除过期的数据，对内存很友好；但是会占用大量的 CPU 资源去处理过期的数据，从而影响缓存的响应时间和吞吐量。

2.惰性过期

只有当访问一个 key 时，才会判断该 key 是否已过期，过期则清除。该策略可以最大化地节省 CPU 资源，却对内存非常不友好。极端情况可能出现大量的过期 key 没有再次被访问，从而不会被清除，占用大量内存。

惰性过期在4.0之前是主线程删除，4.0之后是如果删除的数据比较大，异步后台线程进行处理。

BIO目前有三种，关闭文件线程、AOF刷盘线程、异步删除线程

3.定期过期

每隔一定的时间，会扫描一定数量的数据库的 expires 字典中一定数量的 key，并清除其中已过期的 key。该策略是前两者的一个折中方案。通过调整定时扫描的时间间隔和每次扫描的限定耗时，可以在不同情况下使得 CPU 和内存资源达到最优的平衡效果。

- Redis配置项hz定义了serverCron任务的执行周期，默认为10，即CPU空闲时每秒执行10次;
- 每次过期key清理的时间不超过CPU时间的25%，即若hz=1，则一次清理时间最大为250ms，若hz=10，则一次清理时间最大为25ms;
- 清理时依次遍历所有的db;
- 从db中随机取20个key，判断是否过期，若过期，则逐出;
- 若有5个以上key过期，则重复步骤4，否则遍历下一个db;
- 在清理过程中，若达到了25%CPU时间，退出清理过程;

**Redis中同时使用了惰性过期和定期过期两种过期策略。**



## 内存淘汰

**noeviction**

我不会继续服务写请求 (DEL 请求可以继续服务)，读请求可以继续进行。这样可以保证不会丢失数据，但是会让线上的业务不能持续进行。这是默认的淘汰策略。

**volatile-lru**

尝试淘汰设置了过期时间的 key，最少使用的 key 优先被淘汰。没有设置过期时间的 key 不会被淘汰，这样可以保证需要持久化的数据不会突然丢失。

**volatile-lfu**

与上面LRU类似，不过用的是LFU。

**volatile-ttl**

跟上面一样，除了淘汰的策略不是 LRU，而是 key 的剩余寿命 ttl 的值，ttl 越小越优先被淘汰。

**volatile-random**

跟上面一样，不过淘汰的 key 是过期 key 集合中随机的 key。

**allkeys-lru**

区别于 volatile-lru，这个策略要淘汰的 key 对象是全体的 key 集合，而不只是过期的 key 集合。这意味着没有设置过期时间的 key 也会被淘汰。

**allkeys-lfu**

遇上面LRU类似，不过用的是LFU。	

**allkeys-random**

跟上面一样，不过淘汰的策略是随机的 key。

volatile-xxx 策略只会针对带过期时间的 key 进行淘汰，allkeys-xxx 策略会对所有的 key 进行淘汰。如果你只是拿 Redis 做缓存，那应该使用 allkeys-xxx，客户端写缓存时不必携带过期时间。如果你还想同时使用 Redis 的持久化功能，那就使用 volatile-xxx 策略，这样可以保留没有设置过期时间的 key，它们是永久的 key 不会被 LRU 算法淘汰。
