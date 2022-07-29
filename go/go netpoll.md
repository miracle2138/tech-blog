# go netpoll 网络模型

## java的网络模型
网络模型指的是编程语言层面实现的基于tcp协议的网络服务器。
在go之前，先回顾下老大哥java的网络模型。

最著名的模型是基于BIO的`connection per thread`模式。即一个线程一个连接，因为IO调用是阻塞的，所以并发只能用多线程来抗。
缺点就是线程的代价比较高，并发能力有限。

之后基于epoll那一套多路复用的网络接口，衍生出了类似reactor的异步编程模型，netty就是这类模式的实现。

简单聊一下BIO和NIO的这两种模式。
早期的web server例如tomcat确实是采用`connection per thread`模式的，而且也能在一些情况下运作良好。但是前面也介绍了这个模式最大的问题在于，并发度取决于线程的数量，是硬性限制。
而NIO的响应式编程则取消了这个限制，采用epoll等多路复用模式使得一个线程就能处理多个连接。
在大量短连接为主的应用场景下，二者区别不大，例如http web server，甚至BIO的编程模式成本更低，bug率也更低。
但是在一些长连接为主的应用下，NIO更占优势，因为不用花费太多线程资源就能支持大量的连接。这是NIO server的优势，即**连接数和线程解耦**。

## go的网络模型
go的网络模型十分精巧。在编程层面，go提供了`connection per goroutine`的模式，使得tcp层的网络编程十分便捷。
例子：
```go
package main

import (
 "log"
 "net"
)

func main() {
 listen, err := net.Listen("tcp", ":8888")
 if err != nil {
  log.Println("listen error: ", err)
  return
 }

 for {
  conn, err := listen.Accept()
  if err != nil {
   log.Println("accept error: ", err)
   break
  }

  // start a new goroutine to handle the new connection.
  go HandleConn(conn)
 }
}

func HandleConn(conn net.Conn) {
 defer conn.Close()
 packet := make([]byte, 1024)
 for {
  // block here if socket is not available for reading data.
  n, err := conn.Read(packet)
  if err != nil {
   log.Println("read socket error: ", err)
   return
  }

  // same as above, block here if socket is not available for writing.
  _, _ = conn.Write(packet[:n])
 }
}
```
但是在实现层，`connection per thread`模式因为线程调度和阻塞式IO都是os层面的机制，不需要用户态过多关注细节。
但是对于go的`connection per goroutine`而言，goroutine是用户态实现的，数据包就绪与否是内核态实现的，如何做到二者的无缝相连？
更近一步说，需要实现：
- 协程里的IO阻塞（上述例子的Read和Write）时，即没有数据包或者不可写时，G可以被挂起
- 协程里的IO就绪时，即有数据包或者可写时，G可以被恢复执行

go在实现时，底层网络借助了多路复用的模式。并且：
- 首先，**go的网络IO调用（accept、read、write、select）均是非阻塞的**，否则一旦陷入阻塞，啥也做了。
- 其次，**go会在调度goroutine时候执行epoll_wait系统调用，即`runtime.netpoll`**，检查是否有状态发生改变的fd，有的话就把他取出，唤醒对应的goroutine去处理

展开来看
- accept：负责监听端口并创建连接。一旦有连接，就会创建一个G取处理。如果没有则park住。
- read：执行在子G上。go对read做了封装，首先调用了系统的非阻塞的read方法，如果没有数据则会将当前的G park住，同时将G和con绑定。具体逻辑在`netpollblock`
- write：执行在子G上。与read的逻辑基本一样
- netpoller：在read和write中被挂起的G，会在netpoller中被唤醒。Go scheduler 会在循环调度的 runtime.schedule() 函数以及 sysmon 监控线程中调用 runtime.nepoll。该函数会
调用epoll检测各个con上的就绪情况，如果有就绪，则将con以及相应的G唤醒，重新调度。具体逻辑在`netpollunblock`

综上，Go 借助于 epoll 和 runtime scheduler 等的帮助，设计出了自己的 I/O 多路复用 netpoller，成功地让 Listener.Accept / conn.Read / conn.Write 等方法从开发者的角度看来是同步模式。

## 参考
- [Go netpoller 原生网络模型之源码全面揭秘](https://mp.weixin.qq.com/s?__biz=MzAxMTA4Njc0OQ==&mid=2651443085&idx=3&sn=2c1ed8474bc7fed68b519ce9e5f5e0b0&scene=21#wechat_redirect)
- [epoll在Golang中的应用](https://segmentfault.com/a/1190000038994423)