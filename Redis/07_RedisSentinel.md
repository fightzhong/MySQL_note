## Redis主从复制环境搭建
```
说明: 一主三从, 主服务器为7000端口, 从服务器为7001, 7002, 7003端口

共同配置: 
    daemonize yes
    dir "/opt/redis-5.0.5/data"

主服务器(7000端口):
    port 7000
    pidfile "/var/run/redis_7000.pid"
    logfile "7000.log"
    dbfilename "dump-7000.rdb"
    

从服务器(只展示一台, 其他两台仅仅只是端口不一样):
    port 7001
    pidfile "/var/run/redis_7001.pid"
    logfile "7001.log"
    dbfilename "dump-7001.rdb"
    replica-read-only yes
    replicaof 192.168.220.128 7000
```

## RedisSentinel环境搭建
```
说明: 三台Sentinel哨兵服务器, 端口分别是26379, 26380, 26381, 配置如下(除了端口不一样之外都一样)

配置:
    port 26379
    pidfile "/var/run/redis-sentinel-26379.pid"
    logfile "26379.log"
    dir "/opt/redis-5.0.5/data"
    protected-mode no
    sentinel monitor mymaster 192.168.220.128 7000 2
    sentinel parallel-syncs mymaster 1
    sentinel down-after-milliseconds mymaster 30000
```

## 故障转移演示
```
<1> 启动主从复制的服务器和哨兵服务器
    redis-server /opt/redis/config/redis-7000.conf
    redis-server /opt/redis/config/redis-7001.conf
    redis-server /opt/redis/config/redis-7002.conf
    redis-server /opt/redis/config/redis-7003.conf

    redis-sentinel /opt/redis/config/sentinel-26379.conf
    redis-sentinel /opt/redis/config/sentinel-26380.conf
    redis-sentinel /opt/redis/config/sentinel-26381.conf

<2> 查看主服务器的主从复制状态[redis-cli -p 7000 info replicaion]:
    # Replication
    role:master
    connected_slaves:3
    slave0:ip=192.168.220.128,port=7001,state=online,offset=741840,lag=0
    slave1:ip=192.168.220.128,port=7002,state=online,offset=741840,lag=1
    slave1:ip=192.168.220.128,port=7003,state=online,offset=741840,lag=2


<3> 查询其中一台从服务器的主从复制状态[redis-cli -p 7001 info replication]:
    # Replication
    role:slave
    master_host:192.168.220.128
    master_port:7001
    master_link_status:up
    master_last_io_seconds_ago:1
    master_sync_in_progress:0
    slave_repl_offset:759497
    slave_priority:100
    slave_read_only:1

<3> 查询哨兵服务器的sentinel状态[redis-cli -p 26379 info sentinel]
    # Sentinel
    sentinel_masters:1
    sentinel_tilt:0
    sentinel_running_scripts:0
    sentinel_scripts_queue_length:0
    sentinel_simulate_failure_flags:0
    master0:name=mymaster,status=ok,address=192.168.220.128:7000,slaves=3,sentinels=3
```

## sentinel的三个定时任务
```
<1> 每10秒每个sentinel会对所有的master和slave服务器发送info消息, 主要作用是:
    - 发现slave节点
    - 确定主从关系
<2> master节点有一个发布订阅的频道, 每2秒sentinel会利用该频道与主节点进行消息的交换(对节点的看法和自身的信息)
<3> 每个sentinel每秒会对其他节点和其他sentinel发送ping命令, 即心跳检测
```

## 主观下线&客观下线
```
对于一个sentinel哨兵, 如果在参数sentinel down-after-milliseconds中规定的时间中没有获取到主服务器的正确响应的话,
那么就主观认定这个master已经挂掉了, 但是这只是主观认为而已, 然后其此时会向其他sentinel发送[sentinel is-master-down-by-addr]
来与其他sentinel进行交流, 从而得出该master是否是真正的挂掉了, 如果超过了有quorum台sentinel认为该master为主观下线, 那么
该master就被认定为客观下线, 此时会转向故障转移的操作
```

## 领导者选举
```
一个redis服务被判断为客观下线时，多个监视该服务的sentinel协商，选举一个领头sentinel，对该redis服务进行故障转移操作, 选举是公平
的, 每个sentinel在发现主服务器主观下线后都会要求其他sentinel将自己设置为领头, 如果超过一半的sentinel同意了, 则该sentinel就成为
领头, 如果在限定的时间内没有选出合适的, 则暂停一段时间后再重新选定, 




```





## 参数描述
```
sentinel monitor <mastername> <ip> <port> <quorum>: sentinel哨兵监控特定ip和端口的master, quorum字段的意思是当投票的
                                                    个数大于该值时则当前哨兵认定主服务器为客观下线

sentinel parallel-syncs <mastername> num: 在确定新的主服务器后, 从服务器会开始与新的主服务器进行数据同步, num值更大则表示
                                          并行进行数据同步的服务器就更多, 值为1则表明只能有一台同时进行同步

sentinel down-after-milliseconds <mastername> <millisends>: 主观下线的依据, 如果一台哨兵在millisends毫秒之内没有收到主服
                                                            务器的有效回复, 那么该时间之后就会判定该主服务器为主观下线状态,
                                                            进而开始下一步的选举操作

is-master-down-by-addr ip port current_epoch runid: ip和port字段为主观下线的服务ip和端口, current_epoch为sentinel的纪元,
                                                    可以认为是一个版本号, runid如果是*则表示这是用来sentinel之间进行判断
                                                    master是否为下线状态, 如果是sentinelid则表示用于选举领导者

sentinel failover-timeout mymaster 180000: failover过期时间，当failover开始后，在此时间内仍然没有触发任何failover操作，当前
                                           sentinel将会认为此次failoer失败, 执行故障迁移超时时间，即在指定时间内没有大多数的
                                           sentinel 反馈master下线，该故障迁移计划则失效
```




## 注意点
```
<1> 只有sentinel集群中大多数的sentinel主观认为master为下线的情况下才能真正的触发故障转移, 所以即使满足了quorum值指定个数的sentinel
    时也不一定会触发故障转移, 此时只是开启了故障转移的计划而已, 结合failover-timeout的配置, 如果在该配置指定的时间内仍然没有达到大多
    数sentinel主观认为master下线的话, 就会认为此次的主观下线是失败的
<2> 在进行主从和sentinel集群配置的时候, 采用真正的ip地址, 而不要采用127.0.0.1回环地址
<3> sentinel进行故障转移后, 转移后的配置会持久化到主从和sentinel配置文件中, 从而使得即使重启也可以保持变更后的状态
```