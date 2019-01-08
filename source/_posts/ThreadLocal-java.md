title: ThreadLocal
author: Haoxiang Zhao
tags:
  - java
categories:
  - 后端
date: 2018-12-29 09:49:00
---
# 前言
为什么会有这种操作? 有个变量非[线程安全](https://stackoverflow.com/questions/261683/what-is-meant-by-thread-safe-code), 又不想加锁,, 那既然抢, 不如每人一个~~
# 正文
## Java
```java
public class Foo
{
    // SimpleDateFormat is not thread-safe, so give one to each thread
        private static final ThreadLocal<SimpleDateFormat> formatter = ThreadLocal.withInitial(()->{
       return  new SimpleDateFormat("yyyyMMdd HHmm");
    });

    public String formatIt(Date date)
    {
        return formatter.get().format(date);
    }
}
```
### 怎么实现的呢? 
可以瞅瞅源码[Thread.java](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/lang/Thread.java), [ThreadLocal.java](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/lang/ThreadLocal.java). 

简单来说, `ThreadLocal`是 `Thread` 对象的一个属性, 各个线程都有自己的, 自然不会打起来. **可用的时候并没有给这个属性赋值呀, 为啥?**

以`ThreadLocal.get()`为例:
```java
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```
把代码捋直就是`Thread.currentThread().threadLocals.getEntry(this)`
`this`就是自己定义并实例的那个`ThreadLocal`对象.

这里的`threadLocals` 是`ThreadLocal.ThreadLocalMap`的实例. 其中的`entry`继承自`WeakReference`
```java
static class Entry extends WeakReference<ThreadLocal<?>> {

    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```
作为`Thread`的属性, `Thread`运行完被gc, `ThreadLocal`自然也被清理了. 也就不存在内存泄漏了.

至于使用线程池的, 就需要自己或者`Pool`去调用`ThreadLocal.remove`做清理工作.



# 附录
## 留坑
1. `ThreadLocal.ThreadLocalMap` 是如何散列的, 与一般的`HashMap`有什么不同?

## 参考
1. [When and how should I use a ThreadLocal variable?](https://stackoverflow.com/questions/817856/when-and-how-should-i-use-a-threadlocal-variable)
2. [How is Java's ThreadLocal implemented under the hood?](https://stackoverflow.com/questions/1202444/how-is-javas-threadlocal-implemented-under-the-hood)