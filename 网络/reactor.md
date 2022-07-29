# reactor网络模型
reactor是一种基于多路复用的异步网络编程模型，底层使用的仍然是同步网络IO接口。
netty等网络框架很好的实现了reactor的编程模式。

## 单线程单reactor
基本实现了reactor的模式，但是由于单线程，效率不高。
- 新建server channel以及selector，注册con事件
- acceptor：负责接受connection，并注册read事件以及相应的handler
- handler：负责处理read事件，封装成业务层数据结构，完成一定的业务逻辑，生成并暂存返回值。注册write事件，在下一个select循环中处理write事件，写入返回值。**可以看到读写操作会在两个select周期内完成**

## 多线程单reactor
`单线程单reactor`模式无法充分利用多核模式。所以升级版是加入线程池。网上的文章关于线程池的描述是不准确的，需要明确线程池到底干了啥。
**线程池仅仅做处理业务逻辑，IO读写仍然在reactor主线程里**
这里线程池是在handler里加入的，仅用于处理业务逻辑，即拿到完整的request，生成response。但是从channel里读request以及往channel里写response的IO操作并不是在线程池里处理的，是在reactor主线程，所以这里有锁竞争。
很多博客把这里写成了`per connection per thread`模式了，即accept一个con后，直接丢进线程池里处理。这是不对的，原因在于select模式下，读和写是在两个周期内完成的，无法在一个线程中执行。
每一次的handler对象都是new出来的，handler对象需要暂存业务逻辑生成的response。
附录里的链接是正确的。

## 多线程多reactor
在`多线程单reactor`下，虽然让业务逻辑的并发度高了，但是所有的网络IO都是在主线程的。`多线程多reactor`模式将原先的reactor进行了拆分，main reactor负责处理accept。后续的read/write IO操作交给sub reactor处理。
sub reactor可以有多个。

更多内容参见附录

## 参考
- [传说中神一样的Reactor反应器模式](https://www.cnblogs.com/crazymakercircle/p/9833847.html)
- [Java 多线程Reactor模式的实现](http://www.codebaoku.com/it-java/it-java-231013.html)