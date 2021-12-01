# mysql中的redo log和bin log

## redo log简介

### WAL机制
我们知道mysql具有持久性的特点，是因为写入到mysql的数据可以被持久化到磁盘中。但是随机写磁盘是性能很低的操作，在mysql中使用WAL机制来提升写性能的。
WAL全程是WRITE AHEAD LOG。
简单来讲WAL的思想是：将写请求记录到日志中，也就是这里的redo log。这样等价于将随机写磁盘转化为顺序写磁盘，二者性能差了几个数量级。
再具体一些：当mysql收到一个写请求，出于性能原因，不会直接写磁盘。而是写内存的buffer数据，这样后续读请求就能读到新数据。但是光写buffer数据无法满足持久性，数据可能由于系统crash丢失。
所以mysql就额外写了redo log，用于crash之后的数据恢复。另外即使数据不丢失，内存buffer满了以后也需要借助redo log将改动刷到磁盘，以腾出buffer空间。所以redo log更像是一个持久化的缓冲区，
将写请求临时记录下来，保证不丢失，之后在合适的时机将改动写入到磁盘。

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

## binlog 简介


