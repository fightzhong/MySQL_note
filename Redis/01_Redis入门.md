## Redis的安装(Linux)
```
进入opt目录: cd /opt
下载redis: wget http://download.redis.io/releases/redis-5.0.5.tar.gz
解压redis: tar xzf redis-5.0.5.tar.gz
创建redis软链接: ln -s redis-5.0.5 redis
进入redis文件夹: cd redis
编译并安装: make && make install
```

## Redis可执行文件说明(redis/src目录下)
```
redis-server: Redis服务器
redis-cli: Redis命令行客户端
redis-benchmark: Redis性能测试工具
redis-check-aof: AOF文件修复工具
redis-check-dump/redis-check-rdb: RDB文件修复工具
redis-sentinel: Sentinel服务器(2.8以后)
```

## Redis启动/关闭
```
最简启动: redis-server(使用redis默认配置, 默认为6379端口)
动态参数启动: redis-server --port 6380
配置文件启动: redis-server configpath(配置文件路径)

建议生产环境选择配置文件启动, 单机多实例(一台机器开启多个Redis)配置文件可以用端口区分开

关闭:
  redis-cli -p xxx shutdown
```

## 验证Redis启动
```
ps -ef | grep redis
netstat -tulnp | grep redis
redis-cli -h IP地址 -p 端口号(客户端连接, 然后用ping命令测试)
```

## Redis常用配置
```
daemonize: 是否以守护进程的方式启动(default no, suggest yes)
port: 对外端口号(默认为6379)
logfile: Redis系统日志(日志文件名, 配合dir来指定日志文件的位置)
dir: Redis工作目录(表示持久化文件和日志文件存储的地方)
```

## Redis单线程为什么快
```
1、纯内存
2、非阻塞I/O
3、避免线程切换和竞态消耗
```

## Redis单线程需要注意什么
```
1、由于是单线程, 所以一次只运行一次命令
2、拒绝长(慢)的命令:
  keys, flushall, flushdb, slow lua script, mutil/exec, operate big value(collecion)
```
