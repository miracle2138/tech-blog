# defer

## 使用方式
defer关键字后跟一个函数。
在函数退出前被调用，常用于资源清理，异常处理，锁释放等场景。

## 原理
defer关键字的实现是由go编译器和运行时共同完成的。
简单来说，go编译器遇到defer语句，会插入deferproc函数调用。在函数返回前会加入deferreturn调用。
- deferproc：将defer调用的函数以及函数参数等信息封装到_defer结构体中。一个函数可能会有多个defer，这些_defer结构体形成一个链表。链表总是从头结点插入，所以后声明的defer会被优先执行。
从这里看到，函数的参数也是在声明defer时确定的，defer之后的修改不会生效。defer链表会被当前goroutine持有，所以defer不会垮goroutine执行。
- deferreturn：在函数返回前，会执行deferreturn函数。该函数负责遍历_defer链表，依次取出defer参数并执行defer函数。需要注意与return的关系。

## 参数求值
defer函数的参数是由编译器插入的deferproc函数处理的，其内部会将所用到的参数一并存入到_defer结构体中，是参数的**快照值**。
几个实例，理解下参数求值：
示例一：
```go
func Test22(t *testing.T) {
	start := time.Now()
	defer fmt.Printf("exec time is %+v", time.Since(start).Seconds())

	func () {
		time.Sleep(5 * time.Second)
	}()
}
```
这段代码通过defer计算函数执行时间。defer函数是`Printf`，参数是字符串和时间。其中时间参数是在声明defer时确定的，此时`time.Since(start).Seconds()`的结果是0，而非函数执行完毕时的5.
可以使用函数闭包达到目的。
```go
func Test23(t *testing.T) {
	start := time.Now()
	defer func() {
		fmt.Printf("exec time is %+v", time.Since(start).Seconds())
	}()

	func () {
		time.Sleep(5 * time.Second)
	}()
}
```
此时defer的函数是闭包，没有参数，而闭包函数内的`time.Since(start).Seconds()`会在函数真正执行时才确定。

示例二：
```go

func Test21(t *testing.T) {
	calc := func (index string, a, b int) int {
		ret := a + b
		fmt.Println(index, a, b, ret)
		return ret
	}

	x := 1
	y := 2
	defer calc("A", x, calc("B", x, y)) // (1)B 1 2 3 (4) A 1 3 4
	x = 3
	defer calc("C", x, calc("D", x, y)) // (2)D 3 2 5 (3) C 3 5 8
	y = 4
}
```
输出：
B 1 2 3
D 3 2 5
C 3 5 8
A 1 3 4

先执行`calc("B"`和`calc("D"`，用于deferproc确定函数调用参数。再执行`calc("C"`和`calc("A"`。

## return
defer和return的关系。return可理解为两条指令：设置返回值和返回被调方。而defer执行是插入到这两步之间的。这使得defer可以修改返回值。

示例一：
```go
func Test24(t *testing.T) {
	ret := func () int{
		t := 10
		defer func(){
			t++
		}()
		return t
	}()

	fmt.Println(ret)
}
```
输出：10
`return t`表示将t的值赋值给返回值变量，该函数没有明确声明返回值变量名，但返回值是真正存在的。此时defer里修改t并不会影响已经赋值的返回值变量。

稍微修改一下：
```go
func Test25(t *testing.T) {
	ret := func () (r int) {
		t := 10
		defer func(){
			r++
		}()
		return t
	}()

	fmt.Println(ret)
}
```
输出：11
所以`return x`的含义是**将x的值赋值给返回值变量（这个变量可能没有命名），再执行ret**。如果想在defer里修改返回值，必须对返回值命名。

## 作用域
defer的作用发生在defer声明的函数中，与语句块没有关系。
```go
func Test26(t *testing.T) {
	{
		defer fmt.Println("defer")
	}

	fmt.Println("main")
}
```
输出：
main
defer

## 参考
- [5.3 defer #](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-defer/#531-%E7%8E%B0%E8%B1%A1)
- [Go - 使用 defer 函数 要注意的几个点](https://segmentfault.com/a/1190000021362837)
- [3.4 defer关键字](https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.4.html)
- [深入理解defer（下）defer实现机制](https://zhuanlan.zhihu.com/p/69455275)