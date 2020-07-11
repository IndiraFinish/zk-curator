# zk-curator
curator客户端使用

## Session
经过测试发现超过SessionTimeout的时候会创建新的session，如果超时时间过短，会创建过多的session，导致服务端压力增加，
不超过SessionTimeout则不会创建新的连接（sessionId一样）

![超过会话超时间](https://user-gold-cdn.xitu.io/2020/7/3/173131f9fe45f118?w=1610&h=186&f=png&s=140991)

如上图所示超过会话超时时间导致创建三个session。

经过测试发现，重连成功后临时节点不会失效。


## 在大规模服务治理情况下zk为什么不适合做注册中心

### 注册中心应该满足AP而不是CP

当发生网络分区的时候zk为了满足数据一致新的情况会导致zk的实例无法写入，这个时候会导致新的provider无法注册，影响服务的连通性。
这个时候需要依赖rpc框架本身的容灾能力或者zk客户端的容灾能力来弥补。

### 横向扩展困难

zk只有Leader节点可以写入数据，无法通过横向扩展提升写数据能力，在服务数量多，频繁的发布服务和增加服务的情况下，存在写瓶颈。
并且随着集群规模的变大，集群处理写入的性能反而下降，原因是zk集群在处理事务性请求操作时，zk集群中对该事务性的请求发起投票，
只有超过半数的Follow服务器投票一致，才会执行该条写入操作。随着集群中Follow服务器的数量越来越多，一次写入等相关操作的投票也就变得越来越复杂，
并且Follow服务器之间彼此的网络通信也变得越来越耗时，导致随着Follow服务器数量的逐步增加，事务性的处理性能反而变得越来越低。

### 使用长连接的压力

随着服务规模的扩大，健康检查、服务注册、节点订阅的写请求都使用长连接都会带来巨大的压力。

### 持久化和事务日志压力

zk为了集群中的事务一致性，Leader服务器会向zk集群中的其他角色服务发送数据同步信息，在接收到数据同步信息后，
zk集群中的Follow和Observer服务器就会进行数据同步，每个写请求都会在每个zk的节点记录一条事务日志用于数据同步（类似于redis的aof）
并且每隔一段时间都会对内存数据进行快照，以此保证数据的一致性和持久性，以及宕机之后的数据可恢复。但是注册中心并不关注历史版本的服务地址列表和
健康状态

### session的状态机原理难以掌握
