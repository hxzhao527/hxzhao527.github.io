title: Golang 的注释能做多少事
author: Haoxiang Zhao
date: 2018-11-08 13:42:59
tags:
---
# 引

# 正文
## 可以运行的
原本想起名叫`build项`,不过发现并不是都是影响构建的注解, 还有一些是为其他诸如`go doc`, `go generate`服务的, 貌似共同点就是都是会影响运行, 再就没了.
### `+build`
go的跨平台和[交叉编译](https://en.wikipedia.org/wiki/Cross_compiler)一直是一大卖点. 但是涉及到平台特性的部分,并不是一份代码对应N多平台, 而是N份代码对应N个平台, 也就是各写各的. 比如[syscall](https://github.com/golang/go/tree/master/src/syscall) 中的
[env_unix.go](https://github.com/golang/go/blob/master/src/syscall/env_unix.go),[env_windows.go](https://github.com/golang/go/blob/master/src/syscall/env_windows.go)分别对应win系和unix系两种不同实现. 

那在编译时编译器怎么指到要使用哪种实现? 根据是什么系统? 那怎么交叉编译呢?
这时就用到了`+build`这个注解.
这个注解用在文件的开头, 且需注意空行.
```go
// +build windows
// +build amd64

package haha
```
这个于`haha_windows_amd64.go`这种下划线后缀的一样, 都是build的[tag](https://golang.org/pkg/go/build/#hdr-Build_Constraints). 不只是系统, arch, 自己的tag也行. 只是自己的tag, 在构建时需要自己用过`-t`额外指定.


### `go:generate`
### `godoc`
## 注解
### Deprecation
# 参考
1. [Deprecation notices in Go](https://rakyll.org/deprecated/)
2. [Build Constraints](https://golang.org/pkg/go/build/#hdr-Build_Constraints)
3. [Command compile](https://golang.org/cmd/compile/)