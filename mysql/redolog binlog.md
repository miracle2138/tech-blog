# mysql中的redo log和bin log

## redo log简介

### WAL机制
我们知道mysql具有持久性的特点，是因为写入到mysql的数据可以被持久化到磁盘中。但是随机写磁盘是性能很低的操作，在mysql中使用WAL机制来提升写性能的。
WAL全程是WRITE AHEAD LOG。
简单来讲WAL的思想是：将写请求记录到日志中，也就是这里的redo log。这样等价于将随机写磁盘转化为顺序写磁盘，二者性能差了几个数量级。
再具体一些：当mysql收到一个写请求，出于性能原因，不会直接写磁盘。而是写内存的buffer数据，这样后续读请求就能读到新数据。但是光写buffer数据无法满足持久性，数据可能由于系统crash丢失。
所以mysql就额外写了redo log，用于crash之后的数据恢复。另外即使数据不丢失，内存buffer满了以后也需要借助redo log将改动刷到磁盘，以腾出buffer空间。所以redo log更像是一个持久化的缓冲区，
将写请求临时记录下来，保证不丢失，之后在合适的时机将改动写入到磁盘。

### 刷盘时机
- 1)redo log满了
- 2)内存buffer满了，需要淘汰最早的内存页，如果是脏页需要刷盘
- 3)系统空闲，异步刷盘
- 4)mysql正常关闭

1)会导致系统写入停止，系统需要将redo log刷盘腾出空间才能接受新的写请求；2)内存空间buffer区满了，需要淘汰老的数据页，如果被淘汰的数据页是脏页，也需要刷盘，此时也会影响系统使用。
所以1）和2）的出现都会导致系统性能下降。mysql会使用异步机制提前刷盘防止1）和2）的出现。衡量的标准就是：
- F1：系统内脏页比例
- F2：redo log写盘速度，也就是write pos - check point值

大致思路是根据这两个指标计算出每秒异步刷脏页的次数。简单理解就是这两个值越大，刷脏页速度也越快。
mysql有一个参数innodb_io_capacity，用于配置系统每秒刷盘次数的容量。最终刷盘次数由F1和F2计算出的一个百分比P * innodb_io_capacity来决定。

另外mysql还有一个特性也会导致性能下降：如果mysql想要淘汰/刷新一个脏页，会额外check下临近的page块，如果也是脏页，会一并处理。这个特性会导致刷脏页数量变多。

### redo log逻辑结构
redo log可以理解为一个环形结构，write pos标识下一个可写入的位置。check point标识已经刷回到磁盘的位置，所以check point之前的数据可以删除。
check point到write pos之间的数据是需要刷新到磁盘的。redo log是循环的，所以不可能无尽得写入，当write pos重新回到check point时，就需要将数据刷回磁盘后才能写入新数据。

### redo log格式
redo log是物理日志，记录了磁盘page块的修改内容

### redo log buffer与刷盘策略
redo log本身也有内存buffer，是为了进一步提升写入性能的。控制redo log刷盘的参数是innodb_flush_log_at_trx_commit。该值有3种值：
- 0：每次写入操作仅仅是写入redo log的内存buffer，每秒定时将buffer数据fsync到磁盘中。如果mysql或者os crash会丢失1s内的数据；
- 1：每次写入操作fsync到磁盘中，不会丢数据，可靠性最高，但是性能较低；
- 2：每次写入os的cache中，每秒定时fsync到磁盘中，如果os crash，会丢失1s数据，mysql crash不影响，是一种兼顾可靠性与性能的折中方案；

### redo log作用
保证mysql持久性，使得mysql具备crash-safe能力

## binlog 简介

### binlog作用
归档日志，可以用于数据恢复和主从同步。redo log也具备数据恢复的能力，有什么区别？redo log的恢复指的是crash-safe，只能恢复check-point之后的数据，数据量有限，因为redo log是不能无限写入的。

有了redo log为什么还需要binlog？因为redo log是innodb存储引擎层的日志，换言之其他存储引擎是没有redo log的。但是主从同步是server层的需求，需要有一种独立于存储引擎的日志，
这就是需要binlog日志的原因。

### 二者的区别
- redo log是innodb引擎特有的，binlog是server层日志
- binlog可以无限写，redo log空间有限
- redo log是物理日志，binlog是逻辑日志

### binlog模式
- row模式：只记录修改行的物理位置。可以保持精确性，但是文件较大。
- statement模式：记录修改的语句，进行重放。如果语句中有上下文信息如当前时间等，可能导致不一致。
- mixed模式：如果不影响精确性，就用statement模式，否则用row模式。

### 刷盘策略
参数sync_binlog用于控制写入binlog的策略：
- 0：不写盘，由操作系统控制
- 1：每次写入都fsync写盘
- n：每n个事务fsync写盘

双"1"配置：指的是redo log和binlog写盘参数都配置为1，这样可靠性最高，但是性能也最低

## 两段式提交
写入流程：存储引擎将数据写入redo log和内存，此时redo log处于prepared状态。之后写binlog。再提交redo log。
可以看到，这里写redo log分成了两步。为什么？
一次写操作需要同时写redo log和binlog，必须保证同时成功，很显然这是一个分布式事务问题。先来看下如果不成功会有什么问题？
- 只有binlog成功：数据会同步到从库，但是主库没有commit，数据未提交，导致主从不一致。
- 只有redo log成功：主库数据提交，从库无法同步，导致主从不一致。
所以，一次写操作必须让两种日志都写成功。mysql采用2PC机制。
所以redo log写完后处于prepared状态，只有binlog也成功才会commit。其实这里的2PC和TA的2PC略有区别，在TA中，每一个事务都有prepared状态，但是在mysql中binlog没有prepared状态。
这与mysql的崩溃恢复机制有关，只要binlog写成功了，数据就算commit了，即便redo log没有写入commit。所以binlog可以没有prepared状态。

## 恢复机制
mysql启动后会从checkpoint点扫描redo log恢复数据
- 如果binlog没写入，回滚；
- 如果binlog写入，redo log没有没有commit，提交redo log；

## 参考
- [MySQL 的 crash-safe 原理解析](http://yun.jinre.com/newsinfo/382775.html)
- [02 | 日志系统：一条SQL更新语句是如何执行的？](https://time.geekbang.org/column/article/68633)
- [12 | 为什么我的MySQL会“抖”一下？](https://time.geekbang.org/column/article/71806)
- [MySQL之Redo Log](https://zhuanlan.zhihu.com/p/35355751)