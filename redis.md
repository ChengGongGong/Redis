# redis集群方案

## 1. 主从高可用方案

## 2.客户端分片:典型的jedis分片
### 2.1 维护了两个map：
  
  一个TreeMap，Hash(shardInfo)<->JedisShardInfo,每个JedisShardInfo的hash值及其对应的JedisShardInfo
  
  一个LinkedHashMap，JedisShardInfo <->Jedis，每个JedisShardInfo及其对应的Jedis实例
  
  初始化：JedisShardInfo -> ShardedJedisPool(基于apache的common-pool2的对象池实现，borrorObject()方法，调用ShardedJedisFactory创建对象) -> ShardedJedis
        -> BinaryShardedJedis -> Sharded <Jedis, JedisShardInfo>(维护两个map)；
  
  数据定位：获取的时候调用ShardedJedis的getShard方法获取一个Jedis实例，具体对key做hash，去TreeMap中判断，当前的key落在哪个区间上，根据该区间上的shardInfo去获取对应的Jedis实例；
### 2.2 分片算法：
  
    redis.clients.jedis.util.Sharded#initialize()
    
    1. 根据redis节点集合创建虚拟节点(160*weight),根据每个redis节点的name计算出对应的hash值，如果没有配置节点名称，使用默认的名称；
    
    2. 将hash值和对应的JedisShardInfo放入TreeMap中，将JedisShardInfo和对应的Jedis实例放入LinkedHashMap中去；
    
    redis.clients.jedis.util.Sharded#getShard(java.lang.String)
    
    3.通过key或者keyTag计算出hash值，在TreeMap中找到比这个hash值大的第一个虚拟节点(这个过程是在一致性hash环上顺时针查找的过程)，
    然后获取对应的JedisShardInfo,在LinkedHashMap中获取对应的Jedis实例
### 2.3 缺陷
1. 不支持涉及到多个key操作的命令，因为分片时这些key可能会被分到不同的redis节点中去，读操作时，响应时间等于响应最慢的那个redis节点的时间，
    写操作时，可能会同时去写多个redis节点，可能会出现部分key写入成功，部分写入失败，无法保证数据一致性的问题。
    
2. 只要有一个redis实例挂掉，该实例将会从连接池中移除，可能会造成数据丢失。

## 3.哨兵模式
### 3.1 简介
Sentinel模式： 一般包括3个sentinel节点，1个Master节点，1个Slave节点

Sentinel节点，用于监控Master和Slave的运行状态，并在Master节点down机的情况下，从slave中选择新的Master，并修改其他的Slave的Master

主观下线：服务器在给定的毫秒数之内,没有返回 Sentinel 发送的 PING 命令的回复,或者返回一个错误,那么 Sentinel 将这个服务器标记为主观下线;

客观下线：多个Sentinel 实例在对同一个服务器做出下线判断，并且通过 SENTINEL is-master-down-by-addr 命令互相交流之后,得出的服务器下线判断(足够多的哨兵节点数量);
          哨兵之间进行一次投票，选出一个哨兵进行故障切换,切换成功后，通过发布订阅模式，让各个哨兵完成主从切换的监控。

哨兵的定时任务：

  1. 每个 Sentinel 以每秒钟一次的频率向它所知的主服务器、从服务器以及其他 Sentinel 实例发送一个 PING 命令;
  
  2. 在一般情况下， 每个 Sentinel 会以每 10 秒一次的频率向它已知的所有主服务器和从服务器发送 INFO 命令，
    当一个主服务器被 Sentinel 标记为客观下线时， Sentinel 向下线主服务器的所有从服务器发送 INFO 命令的频率会从 10 秒一次改为每秒一次
### 3.2 JedisSentinelPool简介
JedisSentinelPool -> redis.clients.jedis.JedisSentinelPool#initSentinels(1-4) -> redis.clients.jedis.JedisSentinelPool#initPool

1. 遍历sentinel地址，根据sentinel地址和端口，初始化Jedis；

2. 调用SENTINEL get-master-addr-by-name，根据主节点的名称获取主节点的地址和端口号，并初始化成HostAndPort；

3. 如果在任何一个sentinel地址中找到了master，就不再遍历；如果都没有找到，要么是所有的sentinels节点都down掉了，要么是master节点没有被存活的sentinels监控到；

4. 为每个sentinel都启动了一个监听器MasterListener，该监听器本身是一个守护线程，它会去订阅sentinel上关于master节点地址改变的消息;

5. master会与实例变量currentHostMaster(MasterListener发现主节点地址改变后，会改变该值)作比较是否相等，相等则重新设置地址，不等且是第一次调用，则初始化GenericObjectPool。

### 3.3 JedisSentinelPool的总结
1. 仅仅适用于单个master-slave，不能进行数据分片
## 4. cluster模式
### 4.1 简介
  1. 主从和哨兵模式单个节点的存储能力有限，访问能力有限，集群模式具有高可用、可扩展性、分布式、容错等特性。
  
  2. redis集群有2^14=16384个哈希槽，每个key通过CRC16校验后对16384取模来决定放置在哪个槽位，集群中的每个节点负责一部分hash槽；
  
    (注:为什么使用16384个slots？)
    
    2.1 CRC16算法产生的hash值有16bit，可以产生2^16=65536个值。因为redis节点在发送心跳包时需要包含所有的槽位信息，以便让节点知道当前集群的信息，16384=16k，
    在发送心跳包时，进行bitmap压缩，压缩后2*8(bit)*1024(1k)=2k，即可以使用2k的空间创建16K的槽位信息，如果采用65536bit，经压缩后需要8k的空间，浪费带宽。
    
    2.2 redis的集群主节点数量基本不可能超过1000个，集群节点越多，心跳包的消息体内携带的数据就越多，如果超过1000个，会导致网络拥堵，因此不建议节点数据超过1000，
    因此对于节点数在1000以内的redis集群，16384个槽位已经够用，没必要再扩展到65536；
    
    2.3 相对而言，槽位越小，节点少的情况下，压缩比高，因为采用bitmap进行压缩，其填充率为slots/节点数
    
  3. redis集群只允许节点操作自己分片的数据，目前只支持具有相同slot值的key进行批量操作，对于映射为不同slot值的key执行批量操作时可能存在于多个节点上，会抛异常
  
    (注：可以使用键哈希标签(Keys hash tags)在集群稳定、没有做碎片重组的情况下允许多键操作)
    
    3.1 如果键值不包含{,则计算键的哈希值
    
    3.2 如果不是直接计算键的哈希，只有在第一个 { 和它右边第一个 } 之间的内容会被用来计算哈希值，如果两个字符之间没有任何字符，则计算整个键的哈希值。
### 4.2 JedisCluster简介
  1. 初始化：redis.clients.jedis.JedisClusterConnectionHandler#initializeSlotsCache
    
  ![Image_text](https://segmentfault.com/img/bV4Xpd?w=1031&h=471)
  
    在JedisClusterInfoCache中，保存pool的配置，维护了两个hashMap，Jedis <-> JedisPool，集群内所有节点及其对应的线程池，Slot <-> JedisPool，每个槽位对应的节点对象池;
    
    1.1 调用cluster slots命令获取所有槽位信息，List<Object>结构,其中Object表示List<Object>结构，包含每个节点的信息；
    序号0：哈希槽起始编号，序号1：哈希槽结束编号，序号2：主节点信息，包含ip地址、端口号、节点ID,序号3：从节点信息，包含ip地址、端口号、节点id
    
    1.2 维护每个节点的缓存Jedis <-> JedisPool,key是ip:port,vaule是JedisPool,维护主节点的缓存Slot <-> JedisPool
    
  2. 调用方法：redis.clients.jedis.JedisClusterCommand#run(java.lang.String)
 
    2.1 获取连接有三种方式
    第1种：默认null，执行命令发生错误，则根据节点生成缓存Jedis <-> JedisPool,并获取Jedis对象，
    如果键所在的槽位并没有指派给当前节点，节点返回一个MOVED_ERR，指引客户端redirect至正确的节点，说明缓存保存的集群配置信息有误，需要重新更新缓存，
    如果返回的ASK_ERR,表明集群正在重新分片，可能会出现被迁移的槽的一部分键值对保存在源节点里面，另一部分键值对则保存在目标节点里，返回ASK_ERR，指引客户端转向(redirect)正确的节点
    
    第2种：默认fasle，从集群中随机选择一个节点
    
    第3种：默认的方式，根据槽位定位节点，可避免MOVED_ERR。根据key找到对应的槽点，然后在缓存中获取对应的JedisPool，然后根据JedisPool池生成jedis对象(对象池)；
    如果没有找到对应的JedisPool，则清空缓存，重新更新缓存，在获取对应的JedisPool，仍未找到则从集群中随机选择一个节点
    
    (注ASK错误与MOVED错误)
    
    MOVED_ERR：集群中接收命令的节点，会计算出槽位，并检查槽位是否分配给自己，
    如果键所在的槽并没有指派给当前节点，那么节点会向客户端返回一个MOVED错误，指引客户端转向(redirect)至正确的节点，并再次发送之前想要执行的命令
    
    ASK_ERR：集群中的两个节点在迁移槽时，可能会出现被迁移的槽的一部分键值对保存在源节点里面，另一部分键值对则保存在目标节点里的情况，此时节点
    会先查找键key，如果找到了则直接执行客户端的命令，如果没有找到，则发送ASK_ERR，指引客户端转向(redirect)正确的节点。
    
    2.2 根据获取的正确连接(Jedis对象)，执行相应的redis命令，首先将命令写入RedisOutputStream，redis.clients.jedis.Connection#flush，
    将buf数组的命令写入到socket.outputStream流中，即将命令发送给了Redis。
 
    2.3 RedisInputStream装饰了socket.inputStream流，将其读取到缓存中，完成多次读取，根据返回结果前缀的不同，使用不同的方法解析流的信息。
