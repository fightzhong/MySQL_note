## 配置
```
远程连接redis需要在配置文件中设置: protected-mode no
```

## Jedis依赖
```
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.0</version>
</dependency>
```

## Jedis简单使用
```
Jedis jedis = new Jedis( ip地址, 端口 );
jedis.set( "name", "zhangsan" );
System.out.println( jedis.get( "name" ) ); // zhangsan
System.out.println( jedis.dbSize() ); // 1
System.out.println( jedis.keys( "*" ) ); // [name]
```

## Jedis连接池
- 使用方式
```
GenericObjectPoolConfig config = new GenericObjectPoolConfig();
config.setxxx(); // 重新设置连接池配置
JedisPool jedisPool = new JedisPool( config, ip, 端口 );
```

- 常用参数
```
maxTotal: 资源池最大连接数, 默认为8(建议)
maxIdle: 资源池允许最大空闲连接数, 默认为8(建议)
minIdle: 资源池确保最少空闲连接数, 默认为0(建议)
jmxEnabled: 是否开启jmx监控, 可用于监控, 默认为true(建议)

blockWhenExhausted: 当资源耗尽时, 调用者是否需要等待, 只有为true时, maxWaitMillis参数才生效, 默认true(建议)
maxWaitMillis: 当资源池连接用尽时, 调用者的最大等待时间(毫秒), 默认为-1, 表示永不等待, 即直接报错(不建议默认)
testOnBorrow: 向资源池借用连接时是否做连接有效性测试(ping), 无效连接会被移除, 默认false(建议)
testOnReturn: 向资源池归还连接时是否做有效性检测(ping), 无效链接会被移除, 默认false(建议)
```

- 推荐写法
```
Jedis jedis = null;
try {
  jedis = jedisPool.getResource();
  // 执行一系列的jedis操作
} catch (Exception e) {
  logger.error( e.getMessage(), e );
} fianlly {
  if ( jedis != null )
    jedis.close();
}
```






