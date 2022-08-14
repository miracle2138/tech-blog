# go 同步原语

## sync.Mutex
用于在并发编程中对共享变量的保护。例如
```go
func Test43(t *testing.T) {
var c int64
wg := sync.WaitGroup{}
for i := 0; i < 10000; i++ {
wg.Add(1)
go func() {
c++
wg.Done()
}()
}
wg.Wait()
fmt.Println(c)
}
```
++并不是一个原子操作，因此这段代码并不一定输出10000.需要加锁才行：

```go
func Test44(t *testing.T) {
var c int64
wg := sync.WaitGroup{}
m := sync.Mutex{}
for i := 0; i < 10000; i++ {
wg.Add(1)
go func() {
m.Lock()
defer m.Unlock()
c++
wg.Done()
}()
}
wg.Wait()
fmt.Println(c)
}
```

go语言的锁有个特性：释放锁和加锁的协程可以不是同一个。因此在使用锁时，应该遵循一个原则：加锁和解锁操作成对出现。
```go
func Test42(t *testing.T) {
m := sync.Mutex{}
go func() {
m.Lock()
defer m.Unlock()

time.Sleep(time.Second * 10)
}()

time.Sleep(time.Second)
m.Unlock()
fmt.Println(m.TryLock())
}
```
最终输出true，说明主协程确实把子协程的锁释放了。

除此之外，go语言也支持读写锁，一种细粒度的锁。读读操作并不会被阻塞。

## WaitGroup
用于等待一组协程的结束。用法是在协程外调用Add方法，协程内执行结束后，调用Done方法。在主协程调用Wait方法等待。
在写并发例子时为了防止主协程退出可以使用，或者等待一组并发请求，如http、rpc等。
```go
func Test45(t *testing.T) {
	wg := sync.WaitGroup{}
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func(ii int) {
			time.Sleep(time.Second * time.Duration(ii))
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```

## Once
once会保证传入的代码只执行一次。例如：
```go
func Test46(t *testing.T) {
	once := sync.Once{}
	for i := 0; i < 10; i++ {
		once.Do(func() {
			fmt.Println("process")
		})
	}
}
```
实现也很简单，先判断状态，如果通过再用mutex互斥执行。

