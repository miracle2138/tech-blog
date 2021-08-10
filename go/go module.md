# 使用go module

之前介绍了基于GOPATH的工作模式，但是随着go语言的发展，我们发现基于GOPATH
的模式会有很多问题：
- 只能同时有一个工作目录
- 同一个依赖的包只能有一个版本，且是最新版本

所以从go 11开始，go推出了基于go module的工作模式，go module弱化了GOPATH的
地位，仅用其存公共的依赖，类似maven的m2目录。工作区不再局限于GOPATH，而可以是
任意目录

我们可以使用如下命令初始化一个go工作目录
```shell
go mod init
```
这条命令将产生一个go.mod文件，可以在该文件内声明本模块的模块名、依赖等信息

### 一些go mod的命令
只声明依赖不会下载依赖包，可以使用`go mod download` 或者`go get -u`命令下载
使用`go mod tidy`可以整理依赖，去掉没有使用的依赖项
使用`go mod graph`可以打印依赖树

### 与go module相关的环境变量
- GO111MODULE：go的命令会检测这个变量，因为基于go module和GOPATH的工作模式是截然不同的
    可以设置为off、on和auto，auto会检测项目下是否有go.mod文件。变量里的111代表推出该变量
    的go语言版本，也就是1.11
- GOPROXY：go的依赖不同于java，是直接依赖的源码文件，所以依赖都是存在代码库中的。
    可以从代码版本控制系统中下载依赖，比如github等。go对于代码库有一个协议，实现了这个协议的
    平台都可以被go下载。如果想要创建私有代码库，就需要设置该变量，默认值为`https://proxy.golang.org,direct`
    国内是无法访问到的，所以会这么设置：
    ```
    go env -w GOPROXY=https://goproxy.cn,direct
    ```
    这里的direct是指，如果指定的proxy没有搜到，就从源站下载
- GOSUMDB：拉取源码库文件的sum值，用于比对是否有修改

### 关于go.mod文件
go.mod文件主要做这么两件事：1.声明模块名称；2.声明模块依赖
- module：用于定义当前项目的模块路径。其他模块依赖或者本模块本地依赖，import需要加模块名。
- go：用于标识当前模块的 Go 语言版本，值为初始化模块时的版本，目前来看还只是个标识作用。
- require：用于设置一个特定的模块版本。
- exclude：用于从使用中排除一个特定的模块版本。
- replace：用于将一个模块版本替换为另外一个模块版本。

### 关于go.sum文件
一旦我们下载了依赖，就会在当前目录下生成go.sum文件，该文件用于记录本地依赖文件的校验和，以检验文件
的完整性。

```
github.com/miracle2138/myGoLib v1.0.0 h1:U0RgiYl7o9Z1F0Vs+a7C2uK+FEIzC7K1FSgQfH13NnU=
github.com/miracle2138/myGoLib v1.0.0/go.mod h1:tpKg9ip0N3cYVQYtsKM+YTxvjNW5Q1Eb5ng1tEIJag8=
```
每一个依赖至多两行，格式为：`依赖名 版本 h1: hash值`

最终下载的依赖位于`$GOPATH/pkg/mod`目录下

### 关于版本号
能用tag就用tag。go的依赖版本是有git的tag标识的，并且有一套规范：
`v${x}.${y}.${z}`，x:major，y:minor，z:patch。如果发生不可兼容的版本改动，升级major。向下兼容
的改动升级minor，如果是对改动的修正升级patch
在go.mod中声明的依赖必须指定版本，可以有多种格式。推荐的是tag格式。当然如果没有tag，也可以指定
分支名，提交id等。有些时候go会给我们生成一个默认的版本，比如：
`v0.0.0-20210807121333-1882fbe82372`
v0.0.0代表没有tag版本，第二列是时间，第三列是提交hash

### 导入依赖的路径
如果是v0或者v1，那么声明依赖时，不用带版本。如果主版本是2及以上，则声明依赖时也必须带上版本
例如：
```
github.com/miracle2138/myGoLib/v2 v2.0.0
```
代码中import时也要带

### 如果发布依赖
类似java的maven打包jar，发布到nexus上。由于go是基于源码依赖，所以直接将源码打上git tag，推送
到代码仓库就算发布。
我们可以用自己的github试试。
先在github上建一个repo，然后本地生成公私钥，并在github上配置公钥。然后创建一个项目，本地clone。
写一个简单的go代码lib：
```go
package lib

func Liyao03Func() string {
	return "1"
}
```
go.mod
```
module github.com/miracle2138/myGoLib

go 1.16
```
git提交，并打上tag `git tag v1.0.0`
git push即可。

### 使用发布的依赖
在另一个项目里使用
```go
module com.liyao/proj1

go 1.16

require (
	github.com/miracle2138/myGoLib v1.0.0
)
```

```go
package main

import (
	"fmt"
	"github.com/miracle2138/myGoLib/lib"
)

func main()  {
	r := lib.Liyao03Func()
	fmt.Println(r)
}
```
即可

遗留问题：当依赖升级到2.0是，无法拉到依赖，待排查
