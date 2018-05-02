# GIL是什么

GIL并不是Python的特性，它是在实现Python解析器(CPython)时所引入的一个概念，而CPython是大部分环境下默认的Python执行环境。GIL 全称 gloabl interpreter lock (全局解释器锁) ，官方解释：

> In CPython, the global interpreter lock, or GIL, is a mutex that prevents multiple native threads from executing Python bytecodes at once.  This lock is necessary mainly because CPython’s memory management is not thread-safe. (However, since the GIL exists, other features have grown to depend on the guarantees that it enforces.)

这主要是针对cpython解释器来说的，其他解释器不一样。

# GIL的影响

GIL遵循的原则：“一个线程运行 Python ，而其他 N 个睡眠或者等待 I/O.”（即保证同一时刻只有一个线程对共享资源进行存取）。

之前看到一直以为有GIL全局锁的的存在，那在多线程中为什么要自己再加锁呢，后来发现想错了，怎么可能会等一个线程结束了才会执行另一个线程呢，那多线程就没有了存在的必要。

```python
# 有了GIL全局锁，线程间访问全局变量还是需要同步
total = 0
def add():
    global total
    for i in range(1000000):
        total += 1

def desc():
    global total
    for i in range(1000000):
        total -= 1

import threading
thread1 = threading.Thread(target=add)
thread2 = threading.Thread(target=desc)

thread1.start()
thread2.start()

thread1.join()
thread2.join()

print(total)
```

这个例子中，两个线程分别对一个全局变量total进行加和减1000000次，但是结果并不是0！而是每次运行结果都不相同。

造成这个结果的原因是GIL事实上是会释放的，python中有两种多任务处理：

1. 协同式多任务处理：一个线程无论何时开始睡眠或等待网络 I/O，就会释放GIL锁
2. 抢占式多任务处理：如果一个线程不间断地在 Python 2 中运行 1000 字节码指令，或者不间断地在 Python 3 运行15 毫秒，那么它便会放弃 GIL，而其他线程可以运行

这样解释之后感觉GIL没啥影响啊，反正会切换的嘛，那为什么都说由于GIL的存在，导致python的多线程比单线程还慢，按我的理解，在单核CPU下没什么不一样（也有可能有性能损失），但是在多核CPU下问题就大了，不同核心上的线程同一时刻也只能执行一个，所以***不能够利用多核CPU的优势***，反而在不同核心间切换时会造成资源浪费，反而比单核CPU更慢。

# 解决办法

1. 多线程比较低效时，可用multiprocess库替代Thread库，即使用多进程而不是多线程，每个进程有自己的独立的GIL，因此也不会出现进程之间的GIL争抢。但这样的话也会带来很多其他问题，比如进程间数据通讯和同步的困难。
2. 多线程也不是这么一无是处，在IO密集型操作时，用多线程还是有效果的，不会比多进程差，甚至
3. 用其他解析器。像JPython和IronPython这样的解析器由于实现语言的特性，他们不需要GIL的帮助。然而由于用了Java/C#用于解析器实现，他们也失去了利用社区众多C语言模块有用特性的机会。所以这些解析器也因此一直都比较小众。毕竟功能和性能大家在初期都会选择前者。
4. 等待GIL的改进。ython社区也在非常努力的不断改进GIL，甚至是尝试去除GIL。并在各个小版本中有了不少的进步。

# 总结

由于GIL涉及到底层实现，比较复杂，想要完全搞明白还是很困难的。但是只要记住2点：

1. 在IO密集型型操作下，多线程还是可以的。比如在网络通信，time.sleep()延时的时候。
2. 在CPU密集型操作下，多线程性能反而不如单线程，此时只能用多进程。



更多 *为什么GIL导致多线程效率低下*  的解释可参考以下链接

## 参考链接

[1]: http://python.jobbole.com/87743/	"深入理解 GIL：如何写出高性能及线程安全的 Python 代码"
[2]: https://coding.imooc.com/class/200.html  "Python3高级编程和异步IO并发编程"
[3]: http://python.jobbole.com/81822/	"Python的GIL是什么鬼，多线程性能究竟如何"
[4]: https://www.zhihu.com/question/39923765?sort=created	"python下同样代码，多核多线程为什么比单核多线程慢很多？"



