# Go语言学习——工作区以及常用go命令

## 环境变量及目录
go语言经历了几个版本，最初版本，开发环境围绕着几个目录以及环境变量展开：
- GOROOT：存放go语言开发的工具包，安置后默认在`/usr/local/go`路径下
- GOPATH：开发工作区，本地开发的代码以及依赖的代码，这个目录下又会分为`pkg`、`bin`以及`src`三个子目录
    - bin：存放可执行的二进制文件
    - pkg：存放编译后的文件，防止重复编译
    - src：存放源码文件

我们新写的代码通常放在src目录下，内部引用了其他代码，系统会默认从上述两个环境
变量下找依赖。如果我们依赖了远程的代码包，则需要使用go get命令将代码把下载下来
下面我看一个例子（不适用goModule）

使用`go env`命令查看是否设置了`GO111MODULE`的值，这里我们先将其关闭：
```
go env -w GO111MODULE=off
```
同时设置GOPATH：
```go
go env -w GO111MODULE=/Users/miracle/programming/go/proj5
```
进入项目目录：
```
/Users/miracle/programming/go/proj5
```
先建立lib文件夹:
```shell
mkdir "lib"
```
`cd lib`
建立一个普通的go函数文件`lib.go`
```go
package lib

func F()  {

}

```
再退出lib文件夹，创建main.go文件:
```go
package main

import "fmt"
import "lib"

func main()  {
	lib.F()
	fmt.Println("haha")
}
```
此时运行`go run main.go`，会提示：
```go
main.go:4:8: cannot find package "lib" in any of:
	/usr/local/go/src/lib (from $GOROOT)
	/Users/miracle/programming/go/proj5/src/lib (from $GOPATH)
```
可以看到，这里报错信息为找不到lib包，系统从`GROOT`和`GAPATH`两个变量指向的目录查找
由于我们没有按照最开始的文件格式创建`GOPATH`目录，也即没有src目录，所以系统无法在
`GOPATH/src`下找到`lib`包

我们做一个修改，再添加一层`src`目录，将`main.go`和`lib`移至该目录下，再运行命令`go run src/main.go`，就可以运行了：
```
⇒  go run src/main.go
haha
```

## 如何在IDE里运行
首先在设置里关闭Enable Go Modules Integration选项，然后设置GOPATH，直接run即可。

从这个例子，我们需要理解GROOT和GOPATH的作用：go默认会在这俩目录下寻找源文件和命令

## 如何加入第三方依赖包
可以使用go get domain/module 命令下载源码包至src目录下，再执行go run命令

## go 命令
刚才使用了go get命令来获取第三方依赖，还有两个很常用的命令：
- go build：用于构建可执行程序，生成与当前目录下
- go install：可以构建可执行程序或者代码库，生成与bin或者pkg目录下。
这两个目录均可以在项目目录下运行，后面跟包名或者文件名即可
