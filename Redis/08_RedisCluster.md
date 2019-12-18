## 搭建集群

- 原生安装
```
一、编辑redis节点的配置(配置redis-7000.conf - redis-7005.conf)
	port 7000 # 端口
	daemonize yes # 守护模式
	dir "/opt/redis/data/"
	dbfilename "dump-7000.rdb"
	logfile "7000.log"

	cluster-enabled yes # 开启cluster
	cluster-config-file node-7000.conf # 开启cluster节点后, 会自动生成配置文件, 放于dir下, 该字段指定配置文件的名称
	cluster-node-timeout 15000 # 在该参数规定的时间内如果没有收到有效的回复, 则单方面的认为主节点已经宕机了, 即主观下线
	cluster-require-full-coverage no # 如果设置为yes则表示当前集群中只要出现了一个节点有问题, 则认为当前集群是存在问题的

二、启动redis, redis-server xxx.conf, 由于cluster是基于槽的模式进行数据的存放的, 而此时没有分配, 所以此时这些节点都是
	 被认为下线状态, 执行redis-cli -p 7000 set hello world会显示节点是不可用的

三、查看此时节点的配置信息
	redis-cli -p port cluster nodes
	redis-cli -p port cluster info
	cat /opt/data/node-7000.conf(直接查看启动后生成的配置文件)

四、meet操作, 使得这些cluster集群节点相互关联起来(7000端口主动其他节点进行通信, 从而建立起所有集群节点的通信)
	redis-cli -p 7000 cluster meet 192.168.220.128 7001
	redis-cli -p 7000 cluster meet 192.168.220.128 7002
	redis-cli -p 7000 cluster meet 192.168.220.128 7003
	redis-cli -p 7000 cluster meet 192.168.220.128 7004
	redis-cli -p 7000 cluster meet 192.168.220.128 7005

五、分配槽
	redis-cli -p 7000 cluster addslots 0: 为7000端口的节点分配指定的槽
	以上面开启的6个节点为例, 我们需要实现三主三从的情况, 即7000, 7001, 7002作为主节点, 后面的节点依次作为这三个节点的从
	节点, 我们要为主节点分配槽, 对于16383个槽来说, 7000端口分配了(0-5461), 7001端口分配了(5462-10922), 7002端口分配了
	(10923-16383), 下面是一个shell脚本来实现槽的分配:
		#!/bin/bash
		start=$1
		end=$2
		port=$3

		for (( i=$start; i <= $end; i=i+1  ))
			do 
				#echo $i
				redis-cli -h 192.168.220.128 -p $port cluster addslots $i
			done

	调用方式:
		./a.sh 0 5461 7000
		./a.sh 5462 10922 7001
		./a.sh 10923 16383 7002

六、指定主从关系
	redis-cli cluster replicate node-id(节点的id, 可以通过cluster nodes来获取)
		redis-cli -h 127.0.0.1 -p 7003 cluster replicate ${node-id-7000}
		redis-cli -h 127.0.0.1 -p 7004 cluster replicate ${node-id-7001}
		redis-cli -h 127.0.0.1 -p 7005 cluster replicate ${node-id-7002}
```

- 利用自带的redis-cli --cluster工具安装( 在旧的版本中使用的是redis-trib.rb工具 )
```
注意: 
	<1> 如果使用redis-trib.rb命令则需要安装ruby
	<2> 可以通过redis-cli --cluster help来获取命令的帮助信息

一、依次启动7000-7006配置对应的redis服务

二、利用redis-cli --cluster来实现meet和主从关系分配
	redis-cli --cluster create 192.168.220.128:7000 192.168.220.128:7001 192.168.220.128:7002 
				192.168.220.128:7003 192.168.220.128:7004 192.168.220.128:7005 --cluster-replicas 1
	该命令表示每个主节点有一个从节点, 一共6个节点, 前三个为主节点, 后三个依次为前三个节点的从节点
	通过redis-cli --cluster create命令后, 会自动进行主从关系的维护, 以及集群中槽的平均分配
```



## 集群的伸缩
- 扩张集群
```
一、扩张两个集群节点7006, 7007节点, 创建两个对应的配置文件, 然后用redis-server启动
二、利用redis-cli --cluster来增加节点到集群中(将7006和7007加入到集群中)
	 增加主节点: redis-cli --cluster add-node 192.168.220.128:7006 192.168.220.128:7000
	 增加从节点: redis-cli --cluster add-node 192.168.220.128:7007 192.168.220.128:7000 --cluster-slave
	 				--cluster-master-id <主节点的id>(通过cluster nodes查看)
三、迁移数据(随便指定一个集群中的节点即可, 会自动找到其他节点)
	开启集群的数据迁移: redis-cli --cluster reshard 192.168.220.128:7000
	How many slots do you want to move (from 1 to 16384): 表示需要从之前的节点中移除多少个槽, 之前为三个主服务器, 加
														 入7006后为4个, 16384平均下来就是一个节点为4096个槽, 所以
														 需要从之前的所有槽中一共移除4096个
	What is the receiving node ID: 指定接收这些槽的节点id, 即7006节点对应的id
	Please enter all the source node IDs: 指定源节点的id, 即从哪里移除4096个槽, 指定为all表示从所有的槽中移除4096个,
										  在这些槽中平均移除1365个
```

- 缩容集群
```
<1> 将需要下线的节点的槽迁移到指定的节点中:
	redis-cli --cluster reshard <ip:port> --cluster-from <source-node-id> --cluster-to <target-node-id>
	--cluster-slots <slots-count>
	
	<ip:port>: 该集群中任意节点的ip和端口都可以, 通过这个ip和端口可以了解到其他的节点
	<source-node-id>: 源节点的nodeid, 即需要下线的节点的id, 通过cluster nodes命令可以查看到各个节点的nodeid
	<target-node-id>: 目标节点的nodeid, 即接收这些槽的目标节点
	<slots-count>: 从源节点移除的槽的个数, 假设源节点有4096个槽, 以及除源节点外还有三个主节点, 则每个节点可以获得1365
				   个槽, 所以需要移除三次, 分别移到不同的节点

<2> 执行上述命令移除7006节点的槽到7000, 7001, 7002节点
<3> 移除节点:
	redis-cli --cluster del-node <ip:port> <node-id>

	<ip:port>: 该集群中任意节点的ip和端口都可以, 通过这个ip和端口可以了解到其他的节点
	<node-id>: 需要从集群中移除的节点, 如果有从节点, 一定要先移除从节点, 否则会出现故障转移
	节点移除后会自动关闭
```


## 客户端路由
- move重定向
```
<1> 利用cluster keyslot <key>来计算键对应的槽, <key>表示键名

<2> 下面我们以一个例子先进行演示(以上面的集群缩容后的环境作为演示的环境):
	
	redis-cli -p 7000 cluster keyslot username: 计算username应该存放在哪个槽中
	redis-cli -p 7000: 登录7000端口所在的redis服务器
	127.0.0.1:7000> get username
	(error) MOVED 14315 192.168.220.128:7002
	在7000端口执行get username, 触发了moved异常, 因为7000端口只有0-5461槽, 我们需要开启集群模式才能获取到值
	redis-cli -c -p 7000 get username

<3> -c这个参数表示使用集群模式(cluster)

描述:
	<1> 客户端发送命令到任意的节点
	<2> 根据这个命令的内容计算槽和对应的节点( 利用cluster keyslot hello来计算键对应的槽 )
	<3> 客户端重定向发送命令到目标节点(需要自己编写该逻辑, Java中的JedisCluster已经实现了这个功能)
```

- ask重定向
```
在客户端获取到key对应的槽的位置对应的节点时, 还没有进行修改, 如果此时发生了扩容或者缩容, 那么槽对应的节点就有可能发生了
变化, 这就是ask重定向出现的原因, 在进行扩容或者缩容的时候, 是需要对槽进行迁移的, 如果在迁移的过程中, 我们对源节点进行访问
时, 数据还在槽内则正常返回, 如果不在槽内了就返回一个ASK错误, 对于JedisCluster来说, 当遇到ASK错误的时候, 其会刷新槽-节点
之前的信息缓存的
```

- move重定向和ask重定向的区别
```
<1> 两者都是客户端重定向
<2> moved重定向是在槽已经确定迁移完毕后的
<3> ask表示槽还在迁移中
```

## 集群cluster的常用命令
```
cluster info: 查看集群的信息
cluster nodes: 列出当前集群中所有节点以及相关信息
cluster slots: 查看槽的分配情况
cluster keyslot <key>: 查询key被放置的槽
cluster meet <ip> <port>: 将ip和port所指定的节点添加到集群中
cluster replicate <node-id>: 将node-id对应的节点作为当前节点(执行该命令的节点)的主节点
```
