## 慢查询
- Redis生命周期
```
对于Jedis客户端连接Redis来说, 可以将Redis的生命周期分为四步:
通过网络发送命令 => 命令队列(因为是单线程) => 执行命令 => 通过网络返回结果

而对于慢查询来说, 其只作用在执行命令这一阶段, 也就是说仅仅只是redis命令执行时间才能够算入慢查询的
判断范围
```

- 慢查询配置及命令
```
当一条命令执行的时间超过了慢查询日志规定的时间阈值后, 该命令就会被记录下来, 并放入到一个慢查询日志
队列中, 这个日志队列是一个固定长度的队列, 其数据会存放在内存中, 一旦redis重启就会被清空, 所以需要
定期的对这些记录进行持久化到数据库或者文件中, 便于查看

config get slowlog-max-len: 获取慢查询日志队列的最大长度(默认为128)
config set slowlog-max-len xx: 设置慢查询日志的最大长度

config get slowlog-log-slower-than: 获取慢查询日志的命令执行时间阈值, 当大于等于该阈值时则被当做
                                    成慢命令, 从而被慢查询日志记录(单位是微秒, 默认是10000, 即
                                    10ms), 10ms在一般情况下是很慢的了, 一般设置为1ms
config set slowlog-log-slower-than xxx: 设置慢查询日志的命令执行时间阈值, 如果为0, 则记录所有,
                                        如果为负数, 则永不记录

slowerlog len: 获取慢查询日志队列中慢查询日志的数量
slowerlog get n: 获取慢查询日志队列中前n条记录
slowerlog reset: 清空慢查询日志队列中的内容
```

## pipeline(流水线)
- 使用场景分析
```
在redis的字符串操作中, 有一个mget命令, 可以一次获取多条记录, 但是却执行一次命令, 在我们进行Jedis
客户端连接Redis的时候, 我们每执行一条命令就会将Redis生命周期执行一遍, 即:
[通过网络发送命令 => 命令队列(因为是单线程) => 执行命令 => 通过网络返回结果]

即一条命令执行的时间 = 网络传输时间 + 执行时间

那么就会导致一个问题, 如果我们执行1000条命令, 那么就等于1000次网络传输时间加上1000次执行时间, 从而
使得网络传输的时间占据了大部分, 从而会导致在用户看来慢的情况, 那么就需要一种措施, 类似于mget一样,
将1000条命令集中在一起, 然后一起发送到redis中进行执行, 从而使得执行时间变成了1次网络传输时间加上
1000次命令执行时间, 在一定程度上加快了响应速度, 这就是pipeline的功能
```

- 使用方式(配合Jedis连接池)
```java
GenericObjectPoolConfig poolConfig = new GenericObjectPoolConfig();
JedisPool jedisPool = new JedisPool( poolConfig, "149.129.77.144", 6379 );
Jedis jedis = null;

try {
  jedis = jedisPool.getResource();
  Pipeline pipeline = jedis.pipelined();

  pipeline.set( "token", "token" );
  pipeline.keys( "*" );
  pipeline.dbSize();

  List<Object> objects = pipeline.syncAndReturnAll();
  System.out.println( objects );
} catch (Exception e) {
  e.printStackTrace();
} finally {
  if ( jedis != null )
    jedis.close();
}
```

- pipeline于mget等批量操作的对比
```
<1> mget等redis内置命令的批量操作是原子性的, 也就是说mget获取多条数据要么都获取, 要么都不获取
<2> pipeline打包的多条命令不是原子性的, 这些操作会打包发送到Redis服务器, 但是放入redis命令队列是
    一条条放进去的, 有可能放入两条之间会有其它redis客户端发送来的命令
<3> pipeline只能作用在一个节点上
```

## 发布订阅
```
publish channel message: 向频道channel发送信息
subscribe channel...: 订阅指定的频道(可以订阅多个)
unsubscribe channel...: 取消订阅(可以多个)

psubscribe [pattern]: 订阅满足pattern的频道
pubsub channels: 列出至少有一个订阅者的频道
pubsub numsub [channel]: 列出给定频道的订阅者数量
```


## Bitmap(位图)
```
注意: 下面所说的字节和位数都是从0开始

getbit key offset: 获取key对应的字符串中第offset位的二进制数, 如key为str值为a, 而a对应的0110 0001,
                   则如果offset为3(offset从0开始), 则getbit str 3的值为0, 即第四位二进制数

setbit key offset value: 将键为key的值中二进制位为offset(从0开始计算)位置的设置为指定的bit值(0或1),
                         举个例子, 我们重新定义一个str2, 为了使得str2的值为a, 即0110 0001, 我们
                         需要将第1, 2, 7位的值设为1, 则其它位就会自动补0, 从而得到str2为a了,
                         如:
                            setbit str2 1 1
                            setbit str2 2 1
                            setbit str2 7 1
                            get str2 // 结果为a

bitcount key [start end]: 获取键为key的字符串中二进制为1的个数, start end为可选参数, 表示获取第
                          start-end个字节中二进制为1的个数, 不设置则获取全部, 例如:
                          set str3 abc // 由此可以得出其二进制为: 0110 0001 0110 0010 0110 0011
                          bitcount str3 // 结果为10
                          bitcount str3 0 0 // 表示获取第0个字节中二进制数中1的个数, 结果为3
                          bitcount str3 0 1 // 表示获取第0-1个字节中二进制数中1的个数, 结果为6

bitpos key bit start end: 获取键为key的字符串中二进制数bit的第一次出现的位置, start end为可选参数,
                          表示获取第start-end个字节中二进制数bit的第一次出现的位置, 例如:
                          set str4 abc // 由此可以得出其二进制为: 0110 0001 0110 0010 0110 0011
                          bitpos str4 1 // 获取所有当中二进制数1出现的位置, 即第1位(从0开始, 第0位为0)

                          // 获取字节数start-end中二进制数1出现的位置, 下面表示获取第1-1个字节,
                          // 即第1个字节中(从0开始计算, 第一个字节即b表示的二进制)二进制数1出现
                           的位置, 即第9位
                          bitpos str4 1 1 1

bitop operation destkey key1 key2...: 对键key1和key2的二进制数进行operation操作, 将结果放入destkey
                                      中, operation可以是and(交集), or(并集), not(非), xor(异或)
                              例如:
                                set a a // a的二进制表示为0110 0001
                                set b b // b的二进制表示为0110 0010
                                bitop or c a b // 对a和b进行并集, 则结果为0110 0011, 即结果是c
```


## HyperLogLog(用于完成独立数量的统计, 类似于set, 其本质是字符串)
- 使用方式
```
pfadd key element...: 向键为key的HyperLogLog中增加元素(可以理解为类似于向set中增加元素)
pfcount key...: 获取key中的独立数据的个数, 即不同的元素个数, 可以传入多个key, 表示获取多个key中
                合并在一起后总的独立数据的个数
pfmerge destkey srckey...: 将多个srckey合并在一起后将结果放在destkey中, 类似于集合set的合并

例子:
  pfadd set1 a b c c
  pfadd set2 c d e f g

  pfcount set1 // 独立元素个数为3个(只有三个不同的数据)
  pfcount set2 // 独立元素个数为5个
  pfcount set1 set2 // 独立元素个数为7个, 两个set合并后有7个独立元素

  pfmerge set3 set1 set2 // 将set1、set2进行合并, 将合并结果放入set3
  pfcount set3 // 独立元素个数为7个
```
- HyperLogLog 与 set的对比
```
HyperLogLog存在错误率, 当数据量大的时候, 获取的独立数据的个数不一定是准确的, 并且不支持向集合一样
获取单个数据, 不支持删除单个数据
```

## GEO(一种地理定位的数据结构, 简单的说下用法)
```
geoadd key longitude latitude member: 增加地理信息, 例如:
                                  geoadd location 100.5 110.6 beijing, 则表示增加一个地理信息为
                                  北京的数据, 经度为100.5, 纬度为110.6
geopos key member: 获取地理信息, geopos location beijing, 表示获取location中北京的地理信息经纬度

geodist key member1 member2 [unit]: 获取两个地理位置的距离, unit表示单位[m(米), km(千米), mi(英里), ft(尺)]
例如: geodist location beijing guangdong km, 表示获取北京到广东的距离, 单位是千米

georadius: 获取指定位置范围内的地理位置信息集合

注意: GEO的类型是zset, 没有删除位置的API, 可以通过zrem key member来进行删除
```

















