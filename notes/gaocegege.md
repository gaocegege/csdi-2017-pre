# Note

这篇文章是关于长尾延迟的研究。文章选取了三种不同的 Linux Server:

* Null RPC Server 一个什么也不做的 RPC 服务器
* Memcached
* Nginx

他们找到了一些原因：

* interference from other processes, including background
processes that run even on a system seemingly dedicated
to a single server;
* request re-ordering caused by scheduling policies that are
not designed with tail latency in mind;
* application-level design choices involving how transport
connections are bound to processes or threads;
* multi-core issues such as how NIC interrupts and server
processes are mapped to cores;
* and, CPU power saving mechanisms.

然后定量地评估了解决这些问题的技术带来的好处。

## 模型

文章把一个 server 建模成一个 A/S/c queue system，A 的意思是 Arrvial distribution, 就是请求到达的分布。S 是 Service time distribution，服务的时间分布，c 是独立的 worker 的数量。对于一个面向网络服务的 server 来说，往往不是就一个线程在处理请求的，这里的 worker 就是广义上的用来处理用户请求的进程/线程/CPU。这个模型建立在一个前提上，就是请求来的比处理的慢，不然就是无穷大的延迟了。

### 模型模拟

随后 2.1 和 2.2 以及 2.3 的实验是属于幼儿园小朋友也猜得到结果的。2.1 证明了最理想的状态就是服务一个一个来，每一个都等前一个做完正好来。 2.2 说 server 的利用率越低，长尾延迟情况越小。2.3 说明在共享一个请求队列的前提下，worker 越多越好。论文只是把这些幼儿园常识用模型实验的方式量化了出来。

2.4 是关于调度策略的，论文提出了四种调度策略：

* FIFO: Requests are put in a single global FIFO queue at
the server and workers pull requests from it. The earliest
request is always processed next.
* LIFO: The opposite of FIFO; the most recently arriving
request is processed next.
* Random worker: When requests arrive, they are assigned
to a random worker. Each worker has its own FIFO queue.
* Random request: All requests arriving at the server are
put in a single queue, and workers pull requests from the
queue in random order. Each queued request is equally
likely to be processed next, regardless of its arrival time.

结果图可以看论文，FIFO 有着最好的长尾延迟，LIFO 有着最恐怖的长尾延迟。Random request 有着最好的平均延迟，但是有的请求会在 queue 里呆一辈子。

在整个第二节，论文把所有常识性的东西拿来说了一说，对真实服务器的 measurement 都在后面

## 真 实验 无双

首先说那个 Null RPC Server，是最经典的多线程的服务器模型：

>the server has a main accept thread that spawns new
worker threads to handle each arriving client connection. The
worker threads enter a blocking loop, reading requests using
read and immediately writing a response back using write.
Worker threads do not access any shared data structures or
locks, and they do not use any other system calls.

做的事情是读 TCP 请求然后打印出来发回给 client，线程调度啊请求队列啊之类的都是 OS 负责的。

Memcached 是一个分布式缓存中间件（分布式是基于一致性哈希），是基于 libevent 的。libevent 在 Linux 下是使用 epoll 来做 IO 多路复用的，参考 [Memcached 网络模型](http://xiaobaoqiu.github.io/blog/2014/11/03/memcachedwang-luo-mo-xing/)，是多线程的半同步半异步的模型。

![](http://xiaobaoqiu.github.io/images/memcached/half-sync-half-async.jpg)

(1).异步模块接收可能会异步到来的各种事件(I/O,信号等),然后将它们放入队列中;
(2).同步模块一般只有一种动作,就是不停的从队列中取出消息进行处理;

最后是 Nginx，Nginx 是自己封装了 epoll，而且是多进程的，参考[初探nginx架构](http://tengine.taobao.org/book/chapter_02.html)。master 进程先建好需要 listen 的 socket 后，然后再 fork 出多个 woker 进程，这样每个 worker 进程都可以去 accept 这个 socket。当一个 client 连接到来时，所有 accept 的 worker 进程都会受到通知，但只有一个进程可以 accept 成功，其它的则会 accept 失败。Nginx 提供了一把共享锁 accept_mutex 来保证同一时刻只有一个 worker 进程在 accept 连接，从而解决惊群问题。当一个 worker 进程 accept 这个连接后，就开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后才断开连接，这样一个完成的请求就结束了。

### 总结问题

#### Background process

## Comments

Background process 的问题可以用 Unikernel 来解决