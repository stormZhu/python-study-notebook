# 为什么要通信

如果各个线程之间各干各的，确实不需要通信，这样的代码也十分的简单。但这一般是不可能的，至少线程要和主线程进行通信，不然计算结果等内容无法取回。而实际情况中要复杂的多，多个线程间需要交换数据，才能得到正确的执行结果。

# 全局变量

最简单的想法是建立一个全局变量。几个子线程共同操作这个全局变量(几个线程写变量，几个线程读变量）。

同样举爬虫的例子，假设需要爬取博客网站的所有文章详情，先要通过文章列表页爬取所有文章的`url`，再根据文章的`url`，爬取文章的具体内容。一般来说爬去文章的`url`速度比较快，为文章内容数据量相对更大，速度要慢一些，所以可以用一个线程（文章列表爬取线程）爬取文章`url`列表，多个线程（文章内容爬取线程）根据`url`访问文章具体内容并爬取。此时两个线程之间有交互，文章内容爬取线程需要得到文章列表爬取线程的具体数据，而文章列表爬取线程无需文章内容爬取线程的数据。

```python
import threading # 导入线程包
import time

detail_url_list = []
# 爬取文章详情页
def get_detail_html(detail_url_list, id):
    while True:
        if len(detail_url_list)==0: # 列表中为空，则等待另一个线程放入数据
            continue
        url = detail_url_list.pop()
        time.sleep(2)  # 延时2s，模拟网络请求
        print("thread {id}: get {url} detail finished".format(id=id,url=url))

# 爬取文章列表页
def get_detail_url(detail_url_list):
    for i in range(10000):
        time.sleep(1) # 延时1s，模拟比爬取文章详情要快
        detail_url_list.append("http://projectedu.com/{id}".format(id=i))
        print("get detail url {id} end".format(id=i))

if __name__ == "__main__":
    # 创建读取列表页的线程
    thread = threading.Thread(target=get_detail_url, args=(detail_url_list,))
    # 创建读取详情页的线程
    html_thread= []
    for i in range(4):
        thread2 = threading.Thread(target=get_detail_html, args=(detail_url_list,i))
        html_thread.append(thread2)
    start_time = time.time()
    # 启动两个线程
    thread.start()
    for i in range(4):
        html_thread[i].start()
    # 等待所有线程结束
    thread.join()
    for i in range(4):
        html_thread[i].join()

    print("last time: {} s".format(time.time()-start_time))
```

结果如下：

```python
get detail url 0 end
get detail url 1 end
thread 2: get http://projectedu.com/0 detail finished
get detail url 2 end
thread 3: get http://projectedu.com/1 detail finished
get detail url 3 end
thread 1: get http://projectedu.com/2 detail finished
get detail url 4 end
thread 3: get http://projectedu.com/3 detail finished
get detail url 5 end
get detail url 6 end
thread 2: get http://projectedu.com/4 detail finished
thread 0: get http://projectedu.com/5 detail finished
get detail url 7 end
get detail url 8 end
thread 3: get http://projectedu.com/6 detail finished
thread 2: get http://projectedu.com/7 detail finished
get detail url 9 end
get detail url 10 end
thread 0: get http://projectedu.com/8 detail finished
```

看起来结果很完美，但是，存在着一定的隐患，虽然一般很慢暴露出来。

有两个问题：

1. `python中`的`List`不是线程安全的，可能`pop()`函数执行到了一半，另一个线程同时执行`pop()`，或者另一个线程执行`append()`，这个时候`detail_url_list`中的数据就会发生错误，导致程序挂掉或者得到不正确的结果。
2. 假设`detail_url_list`中只有一个元素，当一个线程判断列表不为空，还没有`pop()`出数据时，时间片被另一个线程抢走，同样列表中还有元素，同样不为空，成功的把数据取出来，这时列表就为空了，这时时间片又让给了上一个线程，上一个线程执行`pop()`，导致`pop from empty list`的异常！在`url = detail_url_list.pop()`语句前加上`time.sleep(1)`可以暴露出这个问题。

解决方法是在判断全局变量是否为空之前，从全局变量取值之后加锁，具体在之后讨论。

# 消息队列--queue模块

使用消息队列的过程和上面一样，只不过`queue`进行了很好的封装，在放值和取值的时候时线程安全的。

`queue`模块实现了多生产者，多消费者的队列。当 要求信息必须在多线程间安全交换，这个模块在**线程编程**时非常有用 。里面主要实现了3中队列。

```python
1. class queue.Queue(maxsize = 0): 构造一个FIFO队列，maxsize可以限制队列的大小。如果队列的大小达到了队列的上限，就会加锁，加入就会阻塞，直到队列的内容被消费掉。maxsize的值小于等于0，那么队列的尺寸就是无限制的
2. class queue.LifoQueue(maxsize = 0): 构造一个LIFO（Last In First Out）队列
3. class PriorityQueue(maxsize = 0)：优先级最低的先出去，优先级最低的一般使用sorted(list(entries))[0]。典型加入的元素是一个元祖(优先级, 数据) 
```

下面主要说明的是`queue.Queue()`

1. 构造方法

   ```python
   import queue
   detail_url_queue = queue.Queue(maxsize=1000)
   ```

   `Queue`是构造方法，函数签名是`Queue(maxsize=0)` ，其中`maxsize`设置队列的大小。

2. 实例方法

   > 1. Queue.qsize(): 返回queue的近似值。注意：qsize>0 不保证(get)取元素不阻塞。qsize<maxsize不保证(put)存元素不会阻塞
   > 2. Queue.empty(): 判断队列是否为空。和上面一样注意
   > 3. Queue.full(): 判断是否满了。和上面一样注意
   > 4. Queue.put(item, block=True, timeout=None): 往队列里放数据。如果满了的话，blocking = False 直接报 Full异常。如果blocking = True，就是等一会，timeout必须为 0 或正数。None为一直等下去，0为不等，正数n为等待n秒还不能存入，报Full异常。
   > 5. Queue.put_nowait(item): 往队列里存放元素，不等待
   > 6. Queue.get(item, block=True, timeout=None): 从队列里取数据。如果为空的话，blocking = False 直接报 empty异常。如果blocking = True，就是等一会，timeout必须为 0 或正数。None为一直等下去，0为不等，正数n为等待n秒还不能读取，报empty异常
   > 7. Queue.get_nowait(item): 从队列里取元素，不等待两个方法跟踪入队的任务是否被消费者daemon进程完全消费
   > 8. Queue.task_done(): 表示队列中某个元素呗消费进程使用，消费结束发送的信息。每个get()方法会拿到一个任务，其随后调用task_done()表示这个队列，这个队列的线程的任务完成。就是发送消息，告诉完成啦！
   >        如果当前的join()当前处于阻塞状态，当前的所有元素执行后都会重启（意味着收到加入queue的每一个对象task_done()调用的信息）
   >        如果调用的次数操作放入队列的items的个数多的话，会触发ValueError异常
   > 9. Queue.join(): 一直阻塞直到队列中的所有元素都被取出和执行未完成的个数，只要有元素添加到queue中就会增加。未完成的个数，只要消费者线程调用task_done()表明其被取走，其调用结束。当未完成任务的计数等于0，join()就会不阻塞

使用queue重写之前的代码：

```python
import threading # 导入线程包
from queue import Queue
import time

# 爬取文章详情页
def get_detail_html(detail_url_list, id):
    while True:
        url = detail_url_list.get()
        time.sleep(2)  # 延时2s，模拟网络请求
        print("thread {id}: get {url} detail finished".format(id=id,url=url))

# 爬取文章列表页
def get_detail_url(queue):
    for i in range(10000):
        time.sleep(1) # 延时1s，模拟比爬取文章详情要快
        queue.put("http://projectedu.com/{id}".format(id=i))
        print("get detail url {id} end".format(id=i))

if __name__ == "__main__":
    detail_url_queue = Queue(maxsize=1000)
    # 先创造两个线程
    thread = threading.Thread(target=get_detail_url, args=(detail_url_queue,))
    html_thread= []
    for i in range(3):
        thread2 = threading.Thread(target=get_detail_html, args=(detail_url_queue,i))
        html_thread.append(thread2)
    start_time = time.time()
    # 启动两个线程
    thread.start()
    for i in range(3):
        html_thread[i].start()
    # 等待所有线程结束
    thread.join()
    for i in range(3):
        html_thread[i].join()

    print("last time: {} s".format(time.time()-start_time))
```

操作基本一样，只不过`queue.Queue()`保证了线程安全。

# 总结

1. 线程间需要通信，使用全局变量需要加锁。
2. 使用`queue`模块，可在线程间进行通信，并保证了线程安全。

# 参考文章

1. [Python3线程间通信](https://blog.csdn.net/weixin_38125866/article/details/76796655)
2. [Python之queue模块](https://www.cnblogs.com/skiler/p/6977727.html)
3. [Python3高级编程和异步IO并发编程](https://coding.imooc.com/class/200.html )



