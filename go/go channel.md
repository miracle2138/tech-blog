# channel

## 多线程编程范式
多线程编程有两个范式：
- 共享内存：多线程操作同一块内存实现通信，为了防止并发问题，需要加锁实现互斥。
- 消息通信：多线程之间通过通道消息机制实现通信，无需共享内存，无需加锁互斥。

Java语言主要基于共享内存方式进行通信，而Go语言则通过channel的方式实现了消息通信机制，并不鼓励共享内存的使用，这也是Go语言设计的一大哲学。将需要共享的变量交给通道，通道可以保证同步语义。

## 为什么不鼓励共享内存？
- 共享内存需要加锁互斥，在复杂的业务下，锁操作实现繁琐，容易造成性能下降、死锁等问题
- 消息通信是一种更高层次的抽象（虽然其底层也是基于共享内存+锁的方式实现的），但是在业务层面，我们可以将多个线程隔离开，不用关注内存共享问题，线程与线程之间通过channel通信，更容易写出清晰的程序。
  详见[为什么使用通信来共享内存](https://draveness.me/whys-the-design-communication-shared-memory/)

## 使用
channel类似于消息队列模型，所有消费者共享同一个group，也即一份数据只能有一个消费者线程拿到。同步机制由channel保证。
```go
func produce() int64 {
	r := rand.Int63()
	fmt.Printf("produce value: %d\n", r)
	return r
}

func consume(i int64) {
	time.Sleep(time.Second)
	fmt.Printf("consume value: %d\n", i)
}

func Test47(t *testing.T) {
	buffer := make(chan int64, 10)
	go func() {
		for {
			buffer <- produce()
		}
	}()

	for i := 0; i < 5; i++ {
		go func() {
			for {
				consume(<-buffer)
			}
		}()
	}
	time.Sleep(time.Second * 1000)
}
```
- 如上是一个简单的生产者-消费者模型。
- channel可以分为带buffer的和不带buffer的，不带buffer的默认容量为0。此时相当于一个协程同步工具，任何读写都将被阻塞，除非有后续的写读。

```go
func Test48(t *testing.T) {
	ch := make(chan int64)

	select {
	case v := <-ch:
		fmt.Printf("value: %d", v)
	default:
		fmt.Printf("no value")
	}
}
```
正常来说channel的操作均是阻塞式的，配合select可以实现非阻塞式读取。

```go
func Test51(t *testing.T) {
	ch := make(chan int64, 1)
	ch <- 1
	time.Sleep(time.Second)

	wg := sync.WaitGroup{}
	wg.Add(1)
	for i := 0; i < 2; i++ {
		go func() {
			v, ok := <-ch
			if ok {
				fmt.Printf("real value: %d\n", v)
			} else {
				fmt.Printf("empty value when close: %d\n", v)
			}
			wg.Done()
		}()
	}

	close(ch)
	wg.Wait()
}
```
输出
```
real value: 1
empty value when close: 0
```
channel的操作可以返回双值，第二个bool代表第一个值是否是真正从channel上读取的值，还是一个channel关闭后返回的零值。


## 原理
- channel是关键字，编译器会将其编译为相应的函数和结构体
- channel结构体的关键字段：锁、数据buffer、读取和写入指针、读取和写入队列等
- 写入时：如果有等待的消费者，直接唤醒，写入；否则如果有剩余空间，写入空间；否则挂起；
- 读取时：如果有等待的生产者，直接唤醒；否则如果空间有数据，直接读取；否则挂起；