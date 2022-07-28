# select、poll以及epoll

## 多路复用
传统的阻塞IO模式在多并发场景下，只能采用per thread per connection的模式进行开发。换句话说一个线程只能处理一个socket。一旦IO发生阻塞，线程也会阻塞。
IO多路复用技术提供了一种可以在一个线程中同时监听多个socket的能力。
Linux OS提供了响应的API接口，分别是select、poll以及epoll。学习网络线程模型必定绕不开这几个名词。

## select
select是最原始的多路复用接口。只包含一个select函数
```go
int select (int __nfds, fd_set *__restrict __readfds,
           fd_set *__restrict __writefds,
           fd_set *__restrict __exceptfds,
           struct timeval *__restrict __timeout);
```
### 主要有三类参数：
- fd_set：是一个整形数组。数组在逻辑上是一个位图，每一个bit代表一个文件描述符。总共有3类fd_set：readfds、writefds以及exceptfds。分别代表可读、可写以及异常事件。
- ndfs：select最大支持1024个文件描述符，默认遍历这么多。如果不需要，可以通过该参数指定上限。
- timeout：超时时间。

### 使用
每一次使用前需要先初始化好各个fds数组，当select执行后，会根据就绪的socket fd改写fds数组，将就绪的bit置为1。所以下一次select需要重新初始化fds结构。

### 缺点
select函数fds结构最多只能支持1024个文件描述符。

## poll
poll整体和select类似，只是将数组改为了指针，去掉了文件描述符上限。
```go
struct pollfd {
    int fd;                    // poll的文件描述符
    short int events;        // poll关心的事件类型
    short int revents;        // 发生的事件类型
  };
```
将每一个需要监听的socket封装进一个pollfd结构中。

### 缺点
类似select，每一次调用都需要将关注的文件fd以及事件类型传入，涉及到用户/内核态的数据拷贝。如果是高qps的场景，这个开销不小。

## epoll
epoll设计了3个API，将高频以及低频的操作分开。

### epoll_create
```go
int epoll_create (int __size);
```
用于创建一个epoll对象。

### epoll_ctl
```go
int epoll_ctl (int __epfd, int __op, int __fd, struct epoll_event *__event);
```
用于在epoll对象上给某一个fd添加/删除某一种监听事件

### epoll_wait
```go
int epoll_wait (int __epfd, struct epoll_event *__events, int __maxevents, int __timeout);
```
用于询问操作系统IO事件是否就绪。将就绪的事件使用events参数返回。

### 优势
可以看到，epoll将事件注册以及事件的就绪查看解耦，拆分成两个接口。在注册部分将fd以及感兴趣的事件告诉epoll，属于低频部分。在高频的wait操作中就不需要再传递这些信息了。

### ET和LT
- ET（Edge Trigger）模式，指的是如果一个socket就绪，epoll_wait只会触发一次，不管这些数据是分几次读取的；
- LT（Level Trigger）模式，指的是如果一个socket就绪，只要有数据，epoll_wait就会一直触发。

举个例子，比如一份数据分成5个数据包过来。
如果是ET模式，只会在一次epoll_wait中返回就绪。如果这次处理只读了两个数据包就报错了，那么后续的数据包就不会再触发了，意味着可能丢数据。
如果是LT模式，即使这次epoll_wait的逻辑报错没有读到全部的数据包，下一次调用epoll_wait仍然会返回就绪，可以继续读取。
所以ET模式效率更高，不会重复触发，但是容错率较低，一旦发生bug可能数据丢失。而LT模式容错更高，但牺牲了一定的效率。

## 参考
- [IO多路复用（一）-- Select、Poll、Epoll](https://segmentfault.com/a/1190000016400053)