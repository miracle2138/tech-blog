# recover && panic

## panic
panic语句会导致当前进程终止执行，除非使用defer recover语句捕获panic。
编译器遇到panic关键字时会编译为gopanic函数。
如果没有recover，gopanic执行过程：获取当前goroutine的defer链表依次执行，调用fatalpanic终止进程。

## recover
recover语句可以使程序从panic中恢复。
编译器遇到recover时，会编译为gorecover函数。该函数会检测当前goroutine里是否发生了panic，如果有，则会在goroutine里设置recover标记。gorecover不会做恢复处理，只是打一个标记。
gopanic调用完defer后，会检测recover标记是否存在，如果存在，则启动恢复流程。
所以，recover只能用在defer中！！！
示例一：
```go
func Test27(t *testing.T) {
	if err := recover(); err != nil {
		fmt.Println("recover panic")
	}

	panic("panic")
}
```
recover执行的时候还没有panic，所以不会打recover标记，程序不会恢复。
```go
func Test28(t *testing.T) {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("recover panic")
		}
	}()

	panic("panic")
}
```
将recover加到defer中，就可以正常恢复了。

## 协程
defer recover只对当前协程的panic有效，不能垮协程。这个很好理解，因为defer链表是被当前goroutine持有的。
```go
func Test29(t *testing.T) {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("recover panic")
		}
	}()

	go func() {
		panic("panic")
	}()

	time.Sleep(time.Second)
	fmt.Println("run")
}
```
协程内的panic，外层无法recover。

## 应用
go的error异常处理机制导致if横飞，广受诟病，如果函数里涉及太多的error，可以考虑使用panic。

## 参考
- [5.4 panic 和 recover](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/)