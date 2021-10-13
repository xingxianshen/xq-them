# Redis.config
>网络 NETWORK

```bash
bind 127.0.0.1 #绑定的ip
protected-mode yes #保护模式
port 6379 # 默认端口
```
> 通用 GENERAL

```bash
daemonize yes # 以守护进程的方式运行，需要自己开启为yes
pidfile /var/run/redis.pid  #如果以后台方式运行，我们就需要指定一个pid文件
# 日志
# Specify the server verbosity level.
# This can be one of:
# debug (a lot of information, useful for development/testing)
# verbose (many rarely useful info, but not a mess like the debug level)
# notice (moderately verbose, what you want in production probably)
# warning (only very important / critical messages are logged)
loglevel notice
logfile "" #日志文件目录
databases 16  # 数据库数量， 默认是16个数据库
```
> 快照

持久化，在单位时间内，执行了多少次命令，则会持久化到文件 .rdb  .aof
Redis 是内存数据库，如果没有持久化，数据断电即失
```bash
# 如果900s内 有一个key及进行了修改就会持久化
save 900 1
save 300 10
save 60 10000
# 以后会自己定义

stop-writes-on-bgsave-error yes # 持久化出错后，redis是否继续工作
rdbcompression yes  #是否需要压缩rdb文件
rdbchecksum yes  #保存rdb时进行错误校验
dir ./ #.rdb文件保存的目录
```
> 安全 ECURITY

```bash
requirepass foobared #设置密码
config set requirepass #通过命令行去设置密码
```
> 限制

```bash
maxmemory <bytes>  #最大内存容量
maxmemory-policy noeviction #内存超出上限后策略
```
> APPEND ONLY 模式 aof配置

```bash
appendonly no   # 默认不开启，使用rdb
appendfilename "appendonly.aof"  # 持久化文件的名字
# appendfsync always
appendfsync everysec  # 每秒执行一次，sync
# appendfsync no
```





# Redis 持久化
> rdb

用来做主从复制
RDB持久化是指在指定的时间间隔内将内存中的数据集快照写入磁盘，实际操作过程是fork一个子进程，先将数据集写入临时文件，写入成功后，再替换之前的文件，用二进制压缩存储。
<img alt="MyRedisNode-1d6ee42e.png" src="assets/MyRedisNode-1d6ee42e.png" width="" height="" >
优点
1、适合大规模的数据回复
2、对数据的完整性要求不高
缺点
1、需要进行一定时间间隔的操作，如果redis意外宕机，最后一次修改的记录就会丢失
2、fork进程的时候，会占用一定的内存空间
> aof

保存所有的写入操作，每次重新启动读取操作
如果每次启动时.aof配置文件错误时，无法启动,redis提供了一个自动修复功能`redis-check-aof --fix`
优点：
数据完整性好
缺点：
速度慢
+ 总结：
1 RDB持久化方式能够在指定的时间间隔内对你的数据进行快照存储
2 AOF持久化方式记录每次对服务器写的操作，当服务器重启的时候会重新执行这些命令来恢复原始的数据，AOF命令以Redis协议追加保存每次写的操作到文件末尾，Redis还能对AOF文件进行后台重写，使得AOF文件的体积不至于过大。
3 只做缓存，如果你只希望你的数据在服务器运行的时候存在，你也可以不使用任何持久化
4、同时开启两种持久化方式
    - 在这种情况下，当redis重启的时候会优先载入AOF文件来恢复原始的数据，因为在通常情况下AOF文件保存的数据集要比RDB文件保存的数据集要完整
    - ·RDB的数据不实时，同时使用两者时服务器重启也只会找AOF文件，那要不要只使用AOF呢?作者建议不要，因为RDB更适合用于备份数据库（AOF在不断变化不好备份），快速重启，而且不会有AOF可能潜在的Bug，留着作为一个万一的手段
+ 性能建议
  - 因为RDB文件只用作后备用途，建议只在Slave上持久化RDB文件，而且只要15分钟备份一次就够了，只保留save 9001这条规则
  - 如果Enable AOF，好处是在最恶劣情况下也只会丢失不超过两秒数据，启动脚本较简单只load自己的AOF文件就可以了，代价一是带来了持续的lO，二是AOF rewrite的最后将rewrite过程中产生的新数据写到新文件造成的阻塞几乎是不可避免的。只要硬盘许可，应该尽量减少AOF rewrite的频率，AOF重写的基础大小默认值64M太小了，可以设到5G以上，默认超过原大小100%大小重写可以改到适当的数值。
  - 如果不Enable AOF，仅靠Master-Slave Repllcation 实现高可用性也可以，能省掉一大笔IO，也减少了rewrite时带来的系统波动。代价是如果Master/Slave同时倒掉，会丢失十几分钟的数据，启动脚本也要比较两个Master/Slave 中的RDB文件，载入较新的那个，微博就是这种架构

### 发布订阅
Redis发布订阅（pub/sub）是一种消息通信模式，发送者（pub）订阅者（sub）
订阅/发布消息图：
<img alt="MyRedisNode-8fbfe291.png" src="assets/MyRedisNode-8fbfe291.png" width="" height="" >
> 命令

<img alt="MyRedisNode-c5aae29e.png" src="assets/MyRedisNode-c5aae29e.png" width="" height="" >
> 原理

+ Redis是使用C实现的，通过分析Redis源码里的pubsgb.c文件，了解发布和订阅机制的底层实现，籍此加深对Redis的理解。Redis通过PUBLISH 、SUBSCRIBE 和PSUBSCRIBE等命令实现发布和订阅功能。
+ 通过SUBSCRIBE命令订阅某频道后，redis-server里维护了一个字典，字典的键就是一个个channel，而字典的值则是一个链表，链表中保存了所有订阅这个channel的客户端。SUBSCRIBE 命令的关键，就是将客户端添加到给定 channel的订阅链表中。通过PUBLISH命令向订阅者发送消息，redis-server 会使用给定的频道作为键，在它所维护的channel字典中查找记录了订阅这个频道的所有客户端的链表，遍历这个链表，将消息发布给所有订阅者。
+ Pub/Sub从字面上理解就是发布( Publish)与订阅( Subscribe )，在Redis中，你可以设定对某一个key值进行消息发布及消息订阅，当一个key值上进行了消息发布后，所有订阅它的客户端都会收到相应的消息。这一功能最明显的用法就是用作实时消息系统，比如普通的即时聊天，群聊等功能。
使用场景：
1、实时消息系统
2、实时聊天

### Redis 主从复制
#### 概念
1 主从复制，是指将一台Redis服务器的数据，复制到其他的Redis服务器。前者称为主节点(master/leader)，后者称为从节点(slave/follower);数据的复制是单向的，只能由主节点到从节点。Master以写为主，Slave以读为主。
2 默认情况下，每台Redis服务器都是主节点;且一个主节点可以有多个从节点(或没有从节点)，但一个从节点只能有一个主节点。
主从复制的作用主要包括:
1、数据冗余︰主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式。
2、故障恢复∶当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复;实际上是一种服务的冗余。
3、负载均衡︰在主从复制的基础上，配合读写分离，可以由主节点提供写服务，由从节点提供读服务(即写Redis数据时应用连接主节点，读Redis数据时应用连接从节点），分担服务器负载;尤其是在写少读多的场景下，通过多个从节点分担读负载，可以大大提高Redis服务器的并发量。
4、高可用基石∶除了上述作用以外，主从复制还是哨兵和集群能够实施的基础，因此说主从复制是Redis高可用的基础。
> 环境配置


```bash
info replication  查看当前库的信息
# Rep1ication
role :master #角色master
connected_s1aves : O  #没有从机
```
搭建集群
1、修改端口
2、修改pid名字
3、log文件名字
4、dump.db文件名字
> 配置从机

```bash
通过命令行配置，是临时的
SLACEOF 域名 端口
通过配置文件replication修改
replicaof <masterip> <masterport>
masterautn <master-password>  # 配置主机的密码
```
> 同步原理

Slave启动成功连接到master后会发送一个sync同步命令
Master接到命令，启动后台的存盘进程，同时收集所有接收到的用于修改数据集命令，在后台进程执行完毕之后，master将传送整个数据文件到slave，并完成一次完全同步。
全量复制︰而slave服务在接收到数据库文件数据后，将其存盘并加载到内存中。增量复制:Master继续将新的所有收集到的修改命令依次传给slave，完成同步但是只要是重新连接master，一次完全同步（全量复制)将被自动执行

### 哨兵模式
