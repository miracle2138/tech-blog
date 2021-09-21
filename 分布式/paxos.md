## PAXOS
解决分布式系统一致性的思路有两种：
- 主从架构，即数据的写入/读取全部路由到master节点，slave节点被动接受数据，也即进行日志复制，通过master节点来保证数据一致性。如果master节点挂掉，重新进行选举。需要注意日志覆盖
  的情况。常见的算法有raft和zab协议。
- 去中心化架构，集群中没有主从架构，各个节点的作用地位平等。节点之间采用抢锁的机制保持互斥，常见的有paxos和gossip协议。
- 主从架构的实现相对简单，工程友好。

### 角色
- proposer：负责提交客户端请求，针对某一个变更发起提案
- acceptor：负责接受proposer的提案，确定某项提案的最终值
- leaner：负责复制已经被接受的提案值

### 过程
- prepare：
  - proposer生成新的epoch值n，发起提案：prepare(n)
  - acceptor响应提案：记录最新的提案latest_epoch，并且如果之前已经接受了提案值，将提案值的相应的epoch返回，否则返回nil。 如果遇到的提案epoch小于latest_epoch则返回error
- accept：
  - proposer如果收到了半数以上的响应，则进行accept，否则发起新一轮提案
  - 如果acceptor均返回了nil，说明之前未接受值，proposer可以按照自己的值进行提案
  - 如果有acceptor返回了接收值，则以其中epoch值最大的为准，进行提案
  - acceptor收到提案，看是否是过期的提案，如果是则返回error，否则接受提案值

### 小结
整个协议略微晦涩。关键点：
- prepare阶段本质上是一种抢占式锁机制，proposer想要提案，必须拿到acceptor的锁，后续有持更大epoch的proposer可以抢占锁，抢占后acceptor将拒绝为之前的proposer响应
- accept阶段决定值遵循一个原则，即后来者遵循之前的提议。如果acceptor已经接受过提案值了，那么后续的提案应该遵守这个值，如果完全没有提案过，则按照proposer的提案值来
