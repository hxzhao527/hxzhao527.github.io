title: 由二进制形式发布go package看go如何编译
author: Haoxiang Zhao
tags:
  - golang
categories:
  - 后端
date: 2018-10-22 11:16:00
---
# 前言
平常安装go的依赖, 都是*go get*或者使用一些包管理工具, 比如[dep](https://github.com/golang/dep), [go mod](https://github.com/golang/go/wiki/Modules). 但这些都是将源码拉至本地,然后再编译. 然而特殊情景下, 我们不能对外提供源码,比如为了保密等. 这时候该怎么搞呢? 基于此, 在 Go1.7 版中加入了一个新东西 **//go:binary-only-package**. 这两种方式的go都是怎么编译的呢?


# 正文
## 由main.go到可执行文件
我们本地执行一下golang官方的hello world示例
```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, 世界")
}
```
为了看下编译到底发生了啥,我们加一下额外参数 (build的参数见[Compile](https://golang.org/cmd/go/#hdr-Compile_packages_and_dependencies))
`go clean && go build -x`.

我们发现(go v1.11.1), 
```sh
/usr/lib/go/pkg/tool/linux_amd64/compile ...
/usr/lib/go/pkg/tool/linux_amd64/link ...
```
这不是和*c/c++*一样吗,先*compile*再*link*.(其他版本还有5g, 5c什么的, 可能我安装了假golang)
这两个工具的使用说明, 可以使用*-h*看下帮助文档, 我们注意到里面有这么个参数
```sh
-importcfg file
    	read import configuration from file
```
意思就是我们构建时需要的外部依赖. 在compile部分, 我们发现就只有
```sh
packagefile fmt=/usr/lib/go/pkg/linux_amd64/fmt.a
packagefile runtime=/usr/lib/go/pkg/linux_amd64/runtime.a
```
类比一下c/c++的编译,其实这里就是include包含的头的依赖, 也就是编译依赖(区分链接依赖).

按照c/c++的思路, 我们导入了`fmt`, 它又引入了`runtime`, `strconv`, `unicode/utf8`等, **为啥编译依赖只有两个**? (具体原因后面再看)

编译后再编译, 我们发现*-importcfg*内容变多了, 这次依赖的部分都在了.我们到`$GOROOT/pkg/$GOHOSTOS_$GOARCH`目录下看了下, 所有涉及的`.a`文件都在这里.

既然放在`$GOROOT`的pkg下可以, 那放在`$GOPATH`的pkg下行不行?
是不是我们可以把这个build过程拆分开以达到用静态库进行编译的目的, 比如我们自己提供`.a`文件?

## binary-only-package
在1.7版本的go中, 大牛们引入了这个新东西, 以达到使用二进制库进行构建的目的, 这样就能隐藏起具体源码, 也能起到一定的保护作用. 那具体怎么用呢?
### 1. 构建二进制包
文档中写到,
> When compiling multiple packages or a single non-main package, build compiles the packages but discards the resulting object, serving only as a check that the packages can be built.

也就是只要不是`main`包, 就能编译成二进制包, 而不是执行文件. 我们创建一个临时的目录并加一些简单的内容来生成我们二进制包.
```sh
mkdir -p $GOPATH/src/temp
cat <<EOF > $GOPATH/src/temp/hello.go
package temp

import "fmt"

func Hello() {
	fmt.Println("Hello, 世界")
}
EOF
```
我们生成二进制的包
```sh
go build -o $GOPATH/pkg/`go env GOHOSTOS`_`go env GOARCH`/${PWD##*/}.a -x
```
使用`-o`参数在`$GOPATH/pkg/$GOHOSTOS_$GOARCH/`下生成`temp.a`. 注意文件名, 需要是包的名称
### 2. 运行一下
我们先保留temp文件夹里的源码, 看能否成功.
```sh
mkdir -p $GOPATH/src/study
cat <<EOF > $GOPATH/src/study/main.go
package main

import "temp"

func main() {
	temp.Hello()
}
EOF
go run -x $GOPATH/src/study/main.go
```
成功输出.

现在我们把temp目录源码删掉, `mv $GOPATH/src/temp $GOPATH/src/temp_bak`.结论和常识告诉我们, 不行, 因为在`$GOROOT`和`$GOPATH`都找不到temp包. 那手动编译再链接呢?

前面我们正常执行`go run`时加了`-x`这时直接将前面的输出复制一下就好了. 我们发现一样可行, 成功执行. 也就是, **有二进制包没有源码也能正常编译运行**.

### 3. 能否不要手动编译执行, 还是使用go的工具链呢?
我们前面是直接将temp目录移除, 虽然手动编译能成功, 但是理清`-importcfg`该有哪些内容, 有点太难了, 能和其他go项目一样, 用`go run` 或者 `go build `这种原生的go的工具链吗?
这时就用到了大佬们加入的`//go:binary-only-package`了.

我们文件的内容替换, 保留文件夹, 这里需要注意两点: 1. `//go:binary-only-package`这一行在文件的开头且没有空格. 2. 这行和`package temp`声明之间有一个空行.
```sh
mv $GOPATH/src/temp_bak $GOPATH/src/temp
cat <<EOF > $GOPATH/src/temp/hello.go
//go:binary-only-package

package temp
EOF
```
再次执行`go run -x main.go`, 
```sh
# command-line-arguments
cannot find package fmt (using -importcfg)
/usr/lib/go/pkg/tool/linux_amd64/link: cannot open file : open : no such file or directory
```
提示找不到`fmt`, 啥? 这不是标准库吗? 其实细看输出的`-importcfg`不难发现, 确实缺少`fmt`二进制包的位置, 可看别人教程到这一步都成了呀? 这是go 1.10的变化, 对于`//go:binary-only-package`需要自己在包中指明依赖.
```sh
cat <<EOF > $GOPATH/src/temp/hello.go
//go:binary-only-package

package temp

import "fmt"
EOF
```
再次执行`go run -x main.go`, 完美.
### 4. ide友好项
如果我们既加了`//go:binary-only-package`, 又把前面二进制包里的函数重写了呢?
我们保持前面构建的`.a`文件还在`$GOPATH/pkg/$GOHOSTOS_$GOARCH/`下. 为temp包填上新内容
```sh
cat <<EOF > $GOPATH/src/temp/hello.go
//go:binary-only-package

package temp

import "fmt"

func Hello(){
    fmt.Println("Second Hello")
}
EOF
```
再次执行`go run -x main.go`, 我们发现还是输出的`Hello, 世界`, 也就是一开始的内容, 因此即便源码实现和构建二进制包时的不一致, 也还是以二进制包的为准. 基于此, 二进制包也能用上ide的自动补全和检查了.

前面我们使用二进制包成功构建运行, 但是ide是处理不了这些特殊的包的, 为了代码提示和自动补全, 可以写一个假的实现, 放在源码文件里.
### 5. 总结
我们总结一下, 为了使用二进制包, 我们都做了啥. 
1. 构建二进制包, 这个操作一般都是提供包又不给源码的人做.
2. 将二进制包`.a`文件放到`$GOPATH/pkg/$GOHOSTOS_$GOARCH/`, 注意构建时的系统和cpu架构也必须是这个.
3. 在`$GOPATH/src`下为这个包创建一个目录, 然后加上`//go:binary-only-package`, 再将包的依赖导入声明补上
4. 如果为了ide友好, 再加一个假的实现.

# 参考
1. [使用二进制形式发布go package](https://colobu.com/2018/01/10/use-binary-package-in-go/)
2. [How does the go build command work ?](https://dave.cheney.net/2013/10/15/how-does-the-go-build-command-work)
3. [Compile packages and dependencies](https://golang.org/cmd/go/#hdr-Compile_packages_and_dependencies)
4. [How does Go compile so quickly?](https://stackoverflow.com/questions/2976630/how-does-go-compile-so-quickly)