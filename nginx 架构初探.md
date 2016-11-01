# nginx 架构初探

本文参考： http://tengine.taobao.org/book/chapter_02.html

todo : listenfd socket

nginx在启动后，会有一个master进程和多个worker进程。
基本的网络事件，是放在worker进程中来处理，master进程主要用来管理worker进程,worker进程之间是平等的，每个进程，处理请求的机会也是一样的。
- master进程里面，先建立好需要listen的socket
- 每个worker进程都是从master进程fork过来
- 所有worker进程在注册listenfd读事件前抢accept_mutex
- 抢到互斥锁的那个进程注册listenfd读事件，在读事件里调用accept接受该连接
- accept连接之后开始读取请求，解析请求，产生数据再返回给客户端


## nginx 为什么可以处理高并发？
nginx采用了异步非阻塞的方式来处理请求，也就是说，nginx是可以同时处理成千上万个请求的
对比: apache 每个请求会独占一个工作线程，当并发数上到几千时，就同时有几千的线程在处理请求了。这对操作系统来说，是个不小的挑战，线程带来的内存占用非常大，线程的上下文切换带来的cpu开销很大

## 异步非阻塞到底是怎么回事
阻塞调用会进入内核等待，cpu就会让出去给别人用了
非阻塞就是，事件没有准备好，马上返回EAGAIN，告诉你，事件还没准备好呢，你慌什么，过会再来吧。好吧，你过一会，再来检查一下事件，直到事件准备好了为止，在这期间，你就可以先去做其它事情，然后再来看看事件好了没。(虽然不阻塞了，但你得不时地过来检查一下事件的状态,所以带来的开销也不小)。

 - 异步非阻塞的事件处理机制，具体到系统调用就是像select/poll/epoll/kqueue这样的系统调用。它们提供了一种机制，让你可以同时监控多个事件，调用他们是阻塞的，但可以设置超时时间，在超时时间之内，如果有事件准备好了，就返回。
 - 拿epoll为例，当事件没准备好时，放到epoll里面，事件准备好了，我们就去读写，当读写返回EAGAIN时，我们将它再次加入到epoll里面。
 - 这里的并发请求，是指未处理完的请求，线程只有一个，所以同时能处理的请求当然只有一个了，只是在请求间进行不断地切换而已，切换也是因为异步事件未准备好，而`主动让出`的。这里的切换是没有任何代价，你可以理解为循环处理多个准备好的事件。
 - 与多线程相比，这种事件处理方式是有很大的优势的，不需要创建线程，每个请求占用的内存也很少，没有上下文切换，事件处理非常的轻量级。并发数再多也不会导致无谓的资源浪费（上下文切换）。更多的并发数，只是会占用更多的内存而已。 
 - 之前有对连接数进行过测试，在24G内存的机器上，处理的并发请求数达到过200万。

 >推荐设置worker的个数为cpu的核数

## nginx 如何处理信号与定时器
对于一个基本的web服务器来说，事件通常有三种类型，网络事件、信号、定时器。
信号： 对于nginx来说，如果nginx正在等待事件（epoll_wait时），如果程序收到信号，在信号处理函数处理完后，epoll_wait会返回错误，然后程序可再次进入epoll_wait调用。
定时器： epoll_wait等函数在调用的时候是可以设置一个超时时间的，所以**nginx借助这个超时时间来实现定时器**nginx里面的定时器事件是放在一颗维护定时器的红黑树里面，每次在进入epoll_wait前，先从该红黑树里面拿到所有定时器事件的最小时间，在计算出epoll_wait的超时时间后进入epoll_wait。所以，当没有事件产生，也没有中断信号时，epoll_wait会超时，也就是说，定时器事件到了。这时，nginx会检查所有的超时事件，将他们的状态设置为超时，然后再去处理网络事件。由此可以看出，当我们写nginx代码时，在处理网络事件的回调函数时，通常做的第一个事情就是判断超时，然后再去处理网络事件。


## 伪代码
``` c
ngx_accept_disabled = ngx_cycle->connection_n / 8
    - ngx_cycle->free_connection_n;

if (ngx_accept_disabled > 0) {
    ngx_accept_disabled--;

} else {
    if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
        return;
    }

    if (ngx_accept_mutex_held) {
        flags |= NGX_POST_EVENTS;

    } else {
        if (timer == NGX_TIMER_INFINITE
                || timer > ngx_accept_mutex_delay)
        {
            timer = ngx_accept_mutex_delay;
        }
    }
}
```
#### more to read
How to use epoll? A complete example in C:
https://banu.com/blog/2/how-to-use-epoll-a-complete-example-in-c/