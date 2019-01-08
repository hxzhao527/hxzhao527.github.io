title: '[翻译]The Go scheduler'
author: haoxiang zhao
tags:
  - golang
categories:
  - 后端
date: 2018-10-18 17:37:00
---
# 前言
写了这么久的*Go*的基础代码，还不知道它到底是怎么运行的，因此想通过翻译此文开开眼界。原文来自 [The Go scheduler](http://morsmachine.dk/go-scheduler)。虽然其实已有前辈翻译过了，但是自己翻译一下强化一下理解，顺便学下周边知识，还有就是试试`hexo`到底好用不好用。
# 简介
大佬[Dmitry Vyukov](https://github.com/dvyukov)为1.1版的`Go`贡献了一个新的调度器，这为*GO*的并行编程带来了显著的性能提升。这个新的调度器可以说是无出其右了。
其实这篇Blog中的大部分内容在[官方的设计文档](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw)中已经阐述得很全面了，不过文档就是文档，太学术了。这篇blog增加了一些图示来让它更加直观，更便于理解。
# 正文
## Go到底需要一个怎样的调度器？
在我们分析这个新的调度器之前，我们需要先搞明白一个问题，那就是操作系统明明已经为开发者提供了调度线程的能力，为啥go还需要在[用户空间](http://www.linfo.org/user_space.html)再自己搞一个调度器出来。

已有的[pthread](https://en.wikipedia.org/wiki/POSIX_Threads)线程模型，更多的是对Unix进程模型的扩展，*pthread*和进程一样有着诸如[信号掩码（signal mask）](https://blog.csdn.net/budory/article/details/46803863)，[CPU 亲和性（affinity）](https://www.ibm.com/developerworks/cn/linux/l-affinity.html)，甚至还能交由[cgroups](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)管理。这些对于goroutine来说无疑太重了。如果我们一个进程下运行着1000000个线程的，这些进程的管理乘以这些额外的特性想想都可怕（译：抛给开发者自己代码管理的话，啧啧啧，美味）。这样看来，pthread不单没有必要，甚至可能是累赘。

还有一个问题就是基于当前的[go内存模型](https://golang.org/ref/mem)，操作系统是不能做出一个周全的调度决策的。以[GC](Tracing garbage collection)为例，在GC时，*stop-the-world*要求所有的“线程”（go中对应的为goroutine）暂停，以求内存处于一致状态（译：这个不知新的GC实现还是不是这样）。这就引出了一个新的问题，什么时候内存处于一致状态。如果调度由操作系统管理，在有大量“线程”在运行，为了达到内存的一致态，就必须等，等到这些可能处于任意状态的“线程”一致，这样的话，就会有大量的“线程”需要停止工作。

如果自己开发调度，何时内存一致不再是问题了就，我们可以只在内存一致时再调度。而对于GC，我们只需要等待那些此时正跑在cpu上的线程就好了。
## 主角登场
目前有这么三种常用的线程模型。一是*N:1*，即N个用户级别的“线程”跑在一个系统线程之上。这种模型的最大优势就是“线程”间快速的上下文切换（[Context Switches](https://en.wikipedia.org/wiki/Context_switch)），但是这种模型不能充分的发挥多核cpu的优势。另一种是*1:1*，即一个“线程”对因一个系统线程，这种倒是可以充分发挥多核的优势，但是上下文的切换的代价却太过高昂。

而*Go*则尝试着两者的折中版，也就是*M:N*，即用若干（N）个系统线程调度若干（M）个用户级别的“线程”，也就是*goroutines*。在充分利用多核cpu的同时还兼具上下文快速切换。唯一的缺点就是，还得自己写调度器代码～～:D

在go的调度器中，主要有着这么三个角色：
![scheduler-main-entry](/images/scheduler-main-entry.jpg)

这个三角形状的*[M](https://github.com/golang/go/blob/master/src/runtime/runtime2.go#L404)*， 可以理解为 *machine* 的缩写，是由系统管理的进程，也就是上面的N个系统进程的缩影，是真正干活的。
这个圆圈的*[G](https://github.com/golang/go/blob/master/src/runtime/runtime2.go#L339)*, 则代表着*goroutines*， 它有自己的栈，计数器（[instruction pointer](https://en.wikipedia.org/wiki/Program_counter)）,以及其他调度需要的重要数据。
方块*[P](https://github.com/golang/go/blob/master/src/runtime/runtime2.go#L473)*， processor， 并不是说处理器，而是调度的上下文，可以理解问在一个*M*（os线程）上的调度器，即局部调度器。它是实现从*N:1*到*M:N*的关键。

![scheduler-two-m](/images/scheduler-two-m.jpg)
如上图所示，两个系统进程（M），各自有着自己的上下文（P）， 和一个正在运行的G（浅蓝色的）。而系统进程（M）为了执行*goroutines*（G），必须拿到上下文（P）。**也就是说上下文P的数量，决定了最多有多少系统进程同时在运行**。P的数量可以通过环境变量`GOMAXPROCS`设置， 也可以通过runtime包的 [GOMAXPROCS](https://golang.org/pkg/runtime/#GOMAXPROCS) 函数设置（译：如果这个值比cpu核心数多怎么办？）。

图中的灰色*G*，是没有在执行的*goroutines*，但是是准备好去执行的就绪状态。和图里看到的一样，它们在一个队列中，称之为`runqueues`。实际上，我们在代码里写的`go func(){...}()`，在调度器这边其实就是在队列尾部增加一个*G*。当P判定自己需要进行调度时，就从队列中取出一个G，设置好栈和计数器就开始跑了。

为了减小对上面这个准备队列的互斥竞争，每个P都会有自己的队列。在之前版本的调度器中，只有一个全局的队列，M经常为了互斥锁等待。如果这样的话，在32乃至更多核的机器上，性能会受到影响，更不用说提升。

如果一切都按照预想的进行，P总能拿到新的G，调度器正常运行，那就再好不过了。可总有调皮的～～

## P存在的意义
调度器为啥需要这个*P*呢？直接将`runqueues`放在*M*上，不是就挺好？事实并非如此，比如我们其中一个系统线程因为某种原因就是被blocked了，该怎么办呢？需要对队列中所有G进行转移，而如果有P这一层的存在，只需要P寻求一个新的M就好了，这样是不是更高效，更合理。

例如，我们有一个系统调用[阻塞](https://stackoverflow.com/questions/36112445/golang-blocking-and-non-blocking)了一个系统进程，也就是一个M被阻塞了，为了G还能正常被调度，这时需要把P从这个M上转移走。
![scheduler-syscall](/images/scheduler-syscall.jpg)
如上图中左半部分，原本我们有一个系统线程M0，运行G0时调用了一个系统API（比如读取文件）将进程阻塞了，为了不影响后续G的执行，上下文P被转移到M1，也就是上图的右半部分。这里的M1，可能是为了针对这个阻塞的系统调用新创建的，也可能是从线程池（译：原文为thread cache，后续找到合适的解释再补上）中取的。总之，调度器要确保有足够的M去挂载（译：挂载这个词是个人理解，就是说P能结合一个M），去执行G。而G0则会保持在M0上（译：没有P了吗？是的），毕竟这个线程实际还是活的，只是当前操作阻塞了。

当这个系统调用返回时（完成或失败），M0自然也就不是阻塞状态了，这时M0为了能继续执行G0必须尝试去获取一个上下文P。常规操作是去别的M上“偷”一个P过来。但是如果偷失败了，那么只能将G0放到全局的`runqueues`中，把自身放到线程池中等待被使用。

每个P在执行完自己的队列后都会去全局的队列中拉新的G下来执行。当然，每个P其实也是会定期检查一下全局队列是否有G， 不然放到全局的这些G可能就卡死了，甚至由于没有P可用，永远不能执行了。

正是因为调度器对系统调用的处理方式，即便将`GOMAXPROCS`设置为1,也不会有问题。

## 调度的事怎么能说是偷呢
前面我们说的是M阻塞，P转移。那如果一切顺利，P把自己队列中的的G全执行完了呢？如果G在各个`runqueues`分布不均匀的话还是有可能的。这样就会造成一个P处理完了它的执行队列，但是整体还在运行的局面。为了让程序继续执行下去，可以去全局的队列拉G过来，然而如果全局的队列是空的呢？这时P就不得不去别的P中“偷”一些过来～～（译：歇着不好吗，233）
![scheduler-p-steal](/images/scheduler-p-steal.jpg)
于是，当一个P处理完自己的G，就会去别的P偷一些过来处理。这样就保证了每个P都有事做，换句话说也就是让M一直最大限度地在工作。

# 尾语
其实关于调度器还有很多东西，比如cgo线程，`LockOSThread()`函数，与网络poller的整合。那些已经远远的超出了这篇blog的范畴，不过却是值得去探究的。或许我后面的blog会涉及这些。
# 参考
1. [Context Switches - MSDN](https://docs.microsoft.com/en-us/windows/desktop/procthread/context-switches)
2. [The Go Memory Model](https://golang.org/ref/mem)
3. [Go's work-stealing scheduler](https://rakyll.org/scheduler/)
4. [Scalable Go Scheduler Design Doc](https://golang.org/s/go11sched)
5. [The Go runtime scheduler source](https://github.com/golang/go/blob/master/src/runtime/proc.go)
# 推荐
1. [dotGo 2017 - JBD - Go's work stealing scheduler](https://youtu.be/Yx6FBsGNOp4)