# Context

go语言提供了context对象，允许我们在goroutine之间传递信息。包括：k-v值，cancel func以及deadline。

## Context接口
Context接口定义如下：
```go
type Context interface {
	// 该方法返回context的deadline信息，如果没有设置deadline，ok返回false
    Deadline() (deadline time.Time, ok bool)
	// 如果context设置了deadline或者cancel func，一旦生效，done返回的通道就会有值。如果没有设置取消，返回nil
    Done() <-chan struct{}
	// 如果context被取消，返回取消的原因。DeadlineExceeded或者Canceled
    Err() error
	// 返回context里的k-v值
	Value(key any) any
}
```
context接口声明了其可能具备的功能，但不是说任意一个context实例都具备这些功能，取决于具体context的实现类型或者构建方式。

## context包
context包提供了context对象相关的工具以及工厂方法。
`context.Background`、`context.TODO`、`context.WithDeadline` 和 `context.WithValue`这几个方法可以用于创建context。
`context.TODO`是一个空实现的context接口类型。
```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
return
}

func (*emptyCtx) Done() <-chan struct{} {
return nil
}

func (*emptyCtx) Err() error {
return nil
}

func (*emptyCtx) Value(key any) any {
return nil
}
```
可以看到emptyCtx本质是一个int，其实现的context接口的4个方法都是空方法。可以认为其不具备任何功能，除非调用WithXXX方法构建特殊的context。
`context.Background`和`context.TODO`没有区别，二者只是命名不同而已。按照规范，`context.Background`应该是默认的context，或者最起始的context，所有context都应该从它衍生而来。

## context之WithCancel
```go
func Test39(t *testing.T) {
	ctx := context.Background()
	fmt.Printf("context.Background() 的Done()方法返回: %v\n", ctx.Done())
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	fmt.Printf("使用WithCancel后的Done()方法返回: %v\n", ctx.Done())
	go func(ctx2 context.Context, duration time.Duration) {
		select {
		case <-ctx2.Done():
			fmt.Println("goroutine is canceled.")
			return
		case <-time.After(duration):
			fmt.Println("goroutine is finished.")
		}
	}(ctx, time.Second*3)

	time.Sleep(time.Second * 2)
	cancel()
	fmt.Println(ctx.Err())
}
```
输出：
```
context.Background() 的Done()方法返回: <nil>
使用WithCancel后的Done()方法返回: 0xc000102360
context canceled
goroutine is canceled.
```
协程需要运行3s，我们在第2s时调用了cancel，协程提前终止。
除此之外，还需要注意：
- defer cancel的使用，防止context泄漏。也即如果不调用cancel，这个context将会一直存在。**WithCancel必须调用cancel!!!**
- `Done()`方法只有可以被取消的context才会返回非nil
- `Err()`在调用Cancel之后，返回了`context canceled`错误。

通过源码我们可以看到，WithCancel方法最终构建了一个cancelCtx实例：
```go
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```
这个struct是context接口的另一个实现。内部维护了一个channel，通过Done方法返回。一旦调用了cancel方法，这个channel就有值了。

## context之dWithTimeout/WithDeadline
```go
func Test40(t *testing.T) {
	ctx := context.Background()
	ctx, cancel := context.WithTimeout(ctx, time.Second*2)
	defer cancel()

	go func(ctx2 context.Context, duration time.Duration) {
		select {
		case <-ctx2.Done():
			fmt.Println("goroutine is canceled.")
		case <-time.After(duration):
			fmt.Println("goroutine is finished.")
		}
	}(ctx, time.Second*3)

	time.Sleep(time.Second * 5)
	fmt.Println(ctx.Err())
}
```
输出：
```
goroutine is canceled.
context deadline exceeded
```
创建了一个2s超时的context，协程运行3s，可以看到协程被提前终止。另外，`WithTimeout`创建的context除了具备超时功能，还具有主动cancel的能力，因此需要defer cancel调用。
通过源代码，超时类的context是由`timerCtx`类型实现的。`timerCtx`会继承`cancelCtx`，所以具备了主动取消的功能。超时功能则是通过构建了一个`time.Timer`的定时器，定时器到点后回调cancel func来实现的。

## context之WithValue
```go
func Test41(t *testing.T) {
	ctx := context.Background()
	ctx = context.WithValue(ctx, "K_ENV", "prod")

	go func(ctx2 context.Context) {
		s, ok := ctx2.Value("K_ENV").(string)
		if ok {
			fmt.Printf("value is %s", s)
		}
	}(ctx)
	time.Sleep(time.Second)
}
```
WithValue构建了一个`valueCtx`类型的context。struct内部会持有k和value两个字段，也就是说一个`valueCtx`实例只能持有一对k-v。每一次WithValue的调用都会新建一个context实例，指向parent context。
通过Value方法检索值也是递归调用context链来查找的，是一个O(n)级别的操作！

## 参考
- [6.1 上下文 Context](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-context)