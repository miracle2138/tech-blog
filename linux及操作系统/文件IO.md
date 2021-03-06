# 文件IO

## 文件IO类型
关于文件IO的API有两套。
一套是Unix风格的，这类API不带缓冲，直接与操作系统进行交互。常见的函数有open、write、read、close以及fsync。每一次函数调用都涉及操作系统调用。
另一套是ISO C标准的，这套API是带缓冲的，会在用户空间申请缓冲区，每一次调用是对缓冲区的操作，待到一定时机才会调用read、write函数与操作系统交互。
其函数一般带有f前缀，比如fread和fwrite。

## 不带缓冲区API
### fsync
我们调用write函数写入数据，其实是将数据写入到操作系统的缓冲区，并不会直接写入磁盘。操作系统写数据都是针对内存page块来写，不会直接和IO设备交互。
流程大致是先看要写的数据page块是不是在内存中了，如果在直接写，如果不在，就从磁盘中加载page块到内存中，再写。
此时的page块和磁盘中的数据不一致，这样的页成为脏页，最终需要刷回磁盘。
操作系统什么时机会刷脏页？
- 1）进程调用fsync函数；
- 2）脏页太多放不下；
- 3）脏页太久；

所以，进程将数据写入操作系统缓冲，只要操作系统不挂，数据就不会丢失。但如果希望更高的可靠性，就调用fsync强制刷盘，所以在很多数据库系统中会经常见到fsync。

那操作系统为什么不直接写磁盘？主要是考虑效率。
直接操作磁盘很耗时，所以操作系统希望攒一批操作，最终一起刷回磁盘。程序读写数据都有局部性特征，也就是相邻的几次IO，可能都在同一块磁盘page上，所以先在内存中将这些操作暂存着，
等到合适的时机统一刷新回磁盘，效率更高。不过随之而来的问题就是可能丢数据。

越底层的设备代价越高，提升性能的方式就是使用层层的缓冲区，进行批量处理，提升性能。

## 带缓冲区API
如前面所介绍的，C标准IO使用缓冲区减少了对read、write函数的调用次数，提升了性能。
默认系统会自动分配缓冲，可以使用setbuf等函数手动设置缓冲区。
缓冲区也可以分为两种：全缓冲区和行缓冲。全缓冲区填满后才会调用read、write，行缓冲还会判断是否有换行。
可以使用fflush函数强制将数据刷新至操作系统。

## 直接IO
在操作系统层面，IO操作又可以分为缓冲IO和直接IO。上面所讲的page cache其实就是缓冲IO机制。可以绕过操作系统缓冲直接操作磁盘设备，被称为直接IO。
直接IO可以通过设置打开文件的访问模式实现。直接IO的好处在于减少数据的拷贝次数，不需要从内核拷贝至用户区。
如果应用程序能够自己管理好缓冲策略，决定IO调用时机，直接IO也是不错的选择。通常情况还是将缓冲管理交给OS或者C lib为好。

## 小结
所以，如果我们调用C标准的IO库，至少涉及2层缓冲：应用程序缓冲以及操作系统缓冲。

## 参考
- [Linux中的文件I/O缓冲](https://www.litreily.top/2018/10/25/io-cache/)
- [一文看懂 | 什么是页缓存](https://cloud.tencent.com/developer/article/1848933)
- [Page Cache的落地问题](https://www.jianshu.com/p/ed5900d31f1f)
- [直接IO](https://sccarterrans.github.io/IO%E6%A8%A1%E5%9E%8B-%E7%BC%93%E5%AD%98IO%E4%B8%8E%E7%9B%B4%E6%8E%A5IO/)