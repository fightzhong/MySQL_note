## 通用命令
```
keys [pattern]: 遍历指定模式的所有key, 如:
        [key *]表示遍历所有的key
        [key h*]表示遍历所有以h开头的key
        [key he[h-l]*]表示遍历所有以he开头, 第三个字符为h到l之间, 之后人一个字符的所有key
        [key he?]表示遍历所有以he开头, 第三个字符为任意字符的key(只有三个字符)
        该命令一般不在生成环境中使用, 因为是O(n)操作

dbsize: 计算key的总数

exists key: 检查key是否存在(1为存在, 0为不存在)

del key...: 删除指定的key-value, 可以同时删除多个

expire key seconds: key在seconds秒后过期
ttl key: 查询当前key的剩余时间, 表示该时间之后key会过期
        (-2表示key已经不存在了, -1表示key存在, 并没有过期时间, 大于0则表示还有多少秒)
persist key: 去掉key的过期时间

type key: 返回key的类型(返回值如果为none则表示不存在该key)
```

## 字符串类型
- get、set、del
```
get key: 获取键为key的值
set key value [nx | xx | ex seconds | px milliseconds]:
    设置键key的值为val, 如果为后面接nx, 则表示添加操作(如果已经存在该键了, 则执行失败), 如果后面为
    xx, 则表示更新操作(如果不存在该键, 则执行失败), 如果后面为ex 5, 则表示该键存在于内存中的时间为
    5秒, px 5则是5毫秒, 即缓存时间
del key: 删除键值对key

例子1:
  set name zhangsan
  get name ("结果为张三")
  set name lisi nx (执行失败, 返回null, 因为nx是添加操作, 已经存在name键了)
  set name lisi xx (执行成功, name的值被更新为lisi)
  del name (删除name键值对)
  set name wangwu xx (执行失败, 不存在该键, 更新失败)
  set name wangwu nx (执行成功, nx为添加操作)

例子2:
  set uuid abcdef ex 30 (设置一个字符串键值对uuid=abcdef, 过期时间为30秒以后)
  ttl uuid (获取键为uuid的剩余时间, 结果为25, 表示25秒后过期)
  persist uuid (使得键为uuid的键值对没有过期时间)
  ttl uuid (结果为-1)
  expire uuid 10 (设置uuid的缓存时间为10秒)

  // 10秒后

  ttl uuid (结果为-2, 表示没有该键值对)
  get uuid (结果为nil)
```

- incr、decr、incrby、decrby(counter计数命令)
```
incr key: 对键为key的整数进行自增, 如果不存在该键, 则默认key = 0, 执行命令后为1
decr key: 对键为key的整数进行自减, 如果不存在该键, 则默认key = 0, 执行命令后为-1
incrby key increment: 对键为key的整数进行加法操作, 增加increment值, 默认key = 0, 执行命令即0 + increment
decrby key decrement: 对键为key的整数进行减法操作, 减少decrement值, 默认key = 0, 执行命令即0 - increment

例子:
  set age "18"
  get age (结果为"18")
  incr age (结果为"19")
  decr age (结果为"18")
  incrby age 10 (结果为"28")
  decrby age 10 (结果为"18")
```

- mget、mset(批量增加, 批量设置)
```
mset key value [key value]...: 批量增加
mget key...: 批量获取

例子:
  mset name zhangsan age 18 province guangdong
  get name (结果为"zhangsan")
  mget name age province (结果为"zhangsan", "18", "guangdong")
```

- 其它命令(getset、append、strlen、incrbyfloat、getrange、setrange)
```
getset key newvalue: 返回键key对应的旧值, 并将key的值设置为newvalue
append key value: 对键做拼接字符串的操作, 将value值拼接到原来的值上
strlen key: 返回键对应的值的字节长度, 如果是中文, 有可能一个中文对应两个或三个字节
incrbyfloat key increment: 对键位key的值增加浮点值, 增加increment
getrange key start end: 获取键为key的值中范围为start到end之间的字符, 包括start和end
setrange key index value: 将键位key的值中索引位index的设置为新值, 如果新值的长度大于1个, 则会将
                          index位置后面的值也进行替换, 新值的长度为多少, 则旧值从index开始数这么
                          个长度, 然后用新值替换
```

- 字符串类型的小知识点
```
<1> 字符串一般用于以下场景: 缓存, 计数器(因为有incr操作, 并且是单线程), 分布式锁等
<2> 字符串的大小最大为512M
<3> 字符串可以存储json字符串, 整型, bit二进制数据
```

## 哈希类型(可以理解为small redis, 也可以理解为MapMap)
- hget、hset、hdel
```
hset key field value: 设置一个键为key的hash, 属性为field, 值为value
hget key field: 获取键为key的hash中属性为field的值
hdel key field: 删除键为key的hash中属性为field的值

例子:
  hset user name zhangsan
  hset user age 18
  hget user name (结果为18)
  hdel user age (删除key为user的hash中的age属性)
```

- hmset、hmget(批量增加、批量获取)
```
hmset key field1 value1 [field value]....: 批量对键为key的hash进行属性和值的设置
hmget key field1 field2...: 批量获取键为key的hash中的属性值

例子:
  hmset user name zhangsan age 14
  hget user name age
  结果:
    "zhangsan"
    "14"
```

- hexists、hlen、hincrby、hincrbyfloat
```
hexists key field: 判断键为key的hash中是否存在属性field
hlen key: 获取键为key的hash中的属性个数
hincrby key field increment: 对键为key的hash的field属性值进行增加整数, 增加的值为increment
hincrbyfloat key field increment: 对键为key的hash的field属性值进行增加浮点数, 增加的值为increment
```

- hgetall、hkeys、hvals
```
hgetall key: 获取键为key的hash中所有的属性和值
hkeys key: 获取键为key的hash中所有的属性
hvals key: 获取键为key的hash中所有的属性值
```


## 列表类型
- lpush、rpush、linsert(增加)
```
lpush key value...: 向列表key中的左边添加value(可以添加多个)
rpush key value...: 向列表key中的右边添加value(可以添加多个)
linsert key before|after target value: 向列表key中指定的元素target的前面或者后面添加一个value
                                       需要注意的是, 这个指定的元素target是列表中的第一个元素
```

- lpop、rpop、lrem、ltrim(删除)
```
lpop key: 从列表key中的左边删除一个元素
rpop key: 从列表key中的右边删除一个元素
lrem key count value: 从列表key中删除指定个数即count个的value, 如果count > 0表示从左边算count个,
                      如果count < 0表示从右边算count个, 如果count = 0表示删除全部value
ltrim key start end: 对列表进行截断, 只保留start到end中的元素(包括这两个位置的元素)
```

- lset(改)
```
lset key index value: 将列表中索引为index位置设置值为value
```

- lindex(查)
```
lindex key index: 获取列表key中的第index位置下的元素(index从0开始)
llen key: 获取列表key的长度
```

- 其它命令(blpop、brpop)
```
blpop key timeout: 从列表key中左边删除一个元素, 如果列表为空, 则阻塞timeout秒
brpop key timeout: 从列表key中右边删除一个元素, 如果列表为空, 则阻塞timeout秒

如果在timeout时间内有新的元素增加进来, 则退出阻塞, 如果timeout设置为0, 则表示一直阻塞下去直到有元素
进入列表
```

## 集合类型
- 集合内操作
```
sadd key member...: 向集合key中增加元素, 可以同时增加多个
srem key member...: 从集合key中删除元素, 可以同时删除多个
spop key count: 随机从集合key中删除count个元素

scard key: 获取集合key中元素的个数
sismember key member: 判断集合key中是否含有元素member
smembers key: 获取集合key中的所有元素
srandmember key count: 随机从集合key中获取count个元素(不会破坏集合中元素)
```

- 集合间操作
```
sdiff set1 set2...: 取集合set1, set2的差集, 即set1有但是set2没有的, 如果将set1和set2交换位置, 则是
                取set2有但是set1没有的, 可以传入多个集合, 即取得第一个集合中有但是后面集合中没有的
sinter set1 set2...: 取交集, set1和set2同时有的元素(支持多个集合操作)
sunion set1 set2...: 取并集, 将set1和set2进行合并(支持多个集合操作)

sdiffstore dest set1 set2: 取集合set1, set2的差集, 将结果存储在dest集合中
sinterstore dest set1 set2: 取集合set1, set2的交集, 将结果存储在dest集合中
suinonstore dest set1 set2: 取集合set1, set2的并集, 将结果存储在dest集合中
```

## 有序集合类型
- zadd、zincrby(增加修改操作)
```
zadd key [score member]...: 往集合key中增加元素, score表示分数, member表示值
zincrby key increment member: 对集合key中的元素member的分数增加increment(增加分数)
```

- 删除操作(zrem、zremrangebyscore、zremrangebyrank)
```
zrem key member...: 删除集合key中的元素member, 可以删除多个
zremrangebyscore key min max: 删除集合key中分数在min到max之间的的元素
zremrangebyrank key start end: 删除集合key中排名在start到end之间的元素
```

- 查询操作(zscore、zcard、zcount、zrank、zrange、zrangebyscore)
```
zscore key member: 获取集合key中member的分数
zrank key member: 获取集合key中member的排名, 从0开始, 分数小的排前面
zcard key: 获取集合中元素的个数

zcount key min max: 获取集合key中分数在min到max之间的元素个数(包括临界值)
zrange key start end [withscore]: 获取集合key中排名在start到end之间的元素, 如果加上了withscore
                                  则表示将分数也会获取到
zrangebyscore key min max [withscore]: 获取集合key中分数在start到end之间的元素, 如果加上了withscore
                                       则表示将分数也会获取到
```

## 数据结构和内部编码
```
key:
  数据结构            内部编码
  string              raw, int, embstr

  hash                hashtable, ziplist

  list                linkedlist, ziplist

  set                 hashtable, intset

  zset                skiplist, ziplist
```
























