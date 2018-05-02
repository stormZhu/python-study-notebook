# 背景

对于`IO操作`来说，多线程和多进程差别不大，甚至多线程比多进程效率更高，因为对于操作系统来说，线程的调度比多进程更加轻量。



下面从简单的爬虫例子对多线程进行说明，比如要爬取一个网站的所有文章，网站主要分为文章列表页和文章详情页。方法有两种：

1. 先爬取文章列表页，得到所有文章的`url`，再对所有的`url`进行爬取，得到详情页。
2. 多个线程同时爬取。

毫无疑问，方法二效率更高。这是因为网络通信是`IO操作`，一个线程请求阻塞的时候，不会影响其他线程发起网络请求，几个网络请求可以同时进行。而单线程串行运行的时候，需要等待到了网络的返回后才能发起下一个网络的请求，这样效率会很慢。

# threading库简介

threading库用于提供线程相关的操作，先介绍线程库的几个关键函数：

1. 构造方法

   ```python
   import threading # 导入线程包
   thread1 = threading.Thread(target=get_detail_html, args=("",))
   ```

   `threading.Thread`是构造方法，函数签名是`Thread(group=None, target=None, name=None, args=(), kwargs={})` ，其中

       group: 线程组，目前还没有实现，库引用中提示必须是None；
       target: 要执行的方法； 
       name: 线程名；
       args/kwargs: 要传入方法的参数。

2. 实例方法

   ```python
   isAlive(): 返回线程是否在运行。正在运行指启动后、终止前。 
   get/setName(name): 获取/设置线程名。 
   start():  线程准备就绪，等待CPU调度
   is/setDaemon(bool): 获取/设置是后台线程（默认前台线程（False））。（在start之前设置）
       如果是后台线程，主线程执行过程中，后台线程也在进行，主线程执行完毕后，后台线程不论成功与否，主线程和后台线程均停止；
       如果是前台线程，主线程执行过程中，前台线程也在进行，主线程执行完毕后，等待前台线程也执行完成后，程序停止。
   start(): 启动线程。 
   join([timeout]): 阻塞当前上下文环境的线程，直到调用此方法的线程终止或到达指定的timeout（可选参数）。
   ```



# 代码实例

下面编写代码模拟爬虫的执行过程：

```python
import threading # 导入线程包
import time
def get_detail_html(url):
    print("get detail html started")
    time.sleep(3) # 延时3s，模拟网络请求
    print("get detail html end")

def get_detail_url(url):
    print("get detail url started")
    time.sleep(2)
    print("get detail url end")

if __name__ == "__main__":
    # 先创造两个线程
    thread1 = threading.Thread(target=get_detail_html, args=("",))
    thread2 = threading.Thread(target=get_detail_url, args=("",))
    start_time = time.time()
    # 启动两个线程
    thread1.start()
    thread2.start()
    print("last time: {} s".format(time.time()-start_time))
```

结果如下：

```python
get detail html started
get detail url started
last time: 0.0009822845458984375 s
get detail url end
get detail html end
```

可以看到，花费的时间不是2s，不是3s，也不是5s，而是一个接近0s的数。这是由于主程序已经在执行完`thread2.start()`之后，就立刻执行完了`print()`函数，所以计时结果不能表明程序真实运行时间。

上面的代码其实有三个线程在运行，主线程`main`，和子线程`thread1`和`thread2`，由运行结果可知，主线程代码已经执行完成（按我的理解，主线程还没有结束！），两个子线程还在继续运行（如果子线程也退出了就不可能打印出`get detail html end`和`get detail url end`），这涉及到守护线程的概念，稍后再做解释。那么如何正确的计算运行时间呢？需要调用`join([timeout])`方法，阻塞调用此方法的线程到运行结束或者达到指定的超时时间。

正确代码如下：

```python
import threading # 导入线程包
import time
def get_detail_html(url):
    print("get detail html started")
    time.sleep(3) # 延时3s，模拟网络请求
    print("get detail html end")

def get_detail_url(url):
    print("get detail url started")
    time.sleep(2)
    print("get detail url end")

if __name__ == "__main__":
    # 先创造两个线程
    thread1 = threading.Thread(target=get_detail_html, args=("",))
    thread2 = threading.Thread(target=get_detail_url, args=("",))
    start_time = time.time()
    # 启动两个线程
    thread1.start()
    thread2.start()
    # 等待两个子线程的结束
    thread1.join()
    thread2.join()
    print("use time: {} s".format(time.time()-start_time))
```

可以得到正确的运行结果：

```
get detail html started
get detail url started
get detail url end
get detail html end
last time: 3.001929521560669 s
```



## 守护(daemon)线程

上面提到在主线程执行完成之后，子线程居然还在背后默默执行，在很多情况下，这都是不好的。解决办法就是使用守护线程。[守护线程](https://baike.baidu.com/item/%E5%AE%88%E6%8A%A4%E7%BA%BF%E7%A8%8B)是特殊的线程，一般用于在后台为其他线程提供服务。

设置一个线程是守护线程，就说明这不是一个很重要的线程，对于这样的线程，只要主线程运行结束，就会直接退出。而如果一个线程是非守护线程的话，即使主线程运行结束也不会退出，而是等待所有的非守护线程运行结束，再退出 。

需要强调一下，**并不是主线程的最后一行代码执行完了，主线程就真的结束了**。

> 对主线程来说，运行完毕指的是主线程所在的进程内的所有非守护线程统统运行完毕，主线程才算运行完毕。因为主线程的结束意味着进程的结束，进程整体的资源都将被回收，而进程必须保证非守护线程都运行完毕后才能结束。

在python中，可以通过设置`daemon 属性`来设定守护与非守护。即在线程开始`thread.start()`之前，调用`setDeamon()`函数，`thread.setDaemon(True)`就表示这个线程为守护线程，“不重要” 。`python`默认每个线程都是非守护线程，即默认执行`thread.setDaemon(False) `

还是以上面同样的例子举例，修改主函数如下:

```python
if __name__ == "__main__":
    thread1 = threading.Thread(target=get_detail_html, args=("",))
    thread2 = threading.Thread(target=get_detail_url, args=("",))
    start_time = time.time()
	# 将thread1设置为守护线程
    thread1.setDaemon(True)
    thread1.start()
    thread2.start()
    print("last time: {} s".format(time.time()-start_time))
```

运行结果:

```python
get detail html started
get detail url started
last time: 0.0009391307830810547 s
get detail url end
```

可以看到，同样由于没有写`join()`函数，时间计算还是不对，最主要的是，`get detail html end `并没有打印出来,也就是说，`thread1`没有运行结束，而`thread2`运行结束了，他们的区别就是`thread1`是守护线程，而`thread2`是非守护线程，在`thread2`运行结束之后，整个程序就退出了，守护线程`thread1`也就提前结束了。这里也验证了**并不是说主线程执行完最后一行代码就结束了**，如果这样的话，`thread2`也不会成功运行完成！

同理，如果设置`thread1`为非守护线程，`thread2`为守护线程呢，`thread1`和`thread2`都能正常运行完，这是因为`thread1`延时3s，在`thread1`运行结束前，`thread2`已经运行完了。

如果将`thread1`和`thread2`同时设置为守护线程，两个子线程都会在运行完`print()`之后直接被销毁。

# 继承threading.Thread

上面使用多线程的代码比较散，在代码量比较小的情况下比较方便，但是当代码量或者内部逻辑比较复杂的时候，还是使用利用面向对象的思想进行编程比较好。python提供了继承threading.Thread的方式来实现多线程。

将上面的代码改写如下：

```python
import threading
import time

class GetDetailHtml(threading.Thread):
    def __init__(self, name):
        super().__init__(name=name) # 调用父类的init方法
    def run(self):
        print("get detail html started")
        time.sleep(3)  # 延时3s，模拟网络请求
        print("get detail html end")


class GetDetailUrl(threading.Thread):
    def __init__(self, name):
        super().__init__(name=name)
    def run(self):
        print("get detail html started")
        time.sleep(2)  # 延时3s，模拟网络请求
        print("get detail html end")

if __name__ == "__main__":
    # 先创造两个线程实例
    thread1 = GetDetailHtml("get_detail_html")
    thread2 = GetDetailUrl("get_detail_url")
    start_time = time.time()
    # 启动两个线程
    thread1.start()
    thread2.start()
	# 等待两个子线程的结束
    thread1.join()
    thread2.join()
    print("last time: {} s".format(time.time()-start_time))
```

运行结果和第一种方法是一样的，主要是重载线程类的的`run()`方法。

# 总结

`threading库`的基本用法还是比较简单，主要是守护线程的概念没怎么接触过。其实守护线程并不是`python`独有的，而是操作系统的概念。不管是`C++`，`java`还是其他编程语言，都会接触这个概念。

线程的建立有两种方式，在逻辑简单的时候，可以直接调用构造函数构造子线程，在逻辑比较复杂的时候，可以通过继承`threading.Thread`的当时，重写`run()`方法。

