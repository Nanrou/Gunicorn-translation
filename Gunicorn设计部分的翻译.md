# Design

关于Gunicorn架构的简要描述。

## Server Model

Gunicorn是基于`pre-fork`（预启动，提前fork）的工作模式。这就意味着Gunicorn是由一个主进程来管理这些worker进程的。主进程不会知道那些客户端的任何事情，所有的请求和响应都是由worker进程来处理。

### Master

主进程要做的就是监听各种子进程的信号和做出相应动作，它通过监听信号，如`TTIN`，`TTOU`，`CHLD`来管理这些运行中的worker。`TTIN`和`TTOU`信号告诉主信号去增加或减少运行worker的数量。`CHLD`则是表明了一个子程序被终止了，这个时候主程序就会自动重启这个失败掉的worker。

### Sync Workers

默认的worker工作模式就是同步worker，一次只能处理一个请求。这个模式是最简单的，因为无论出现任何错误，都只会影响那一个请求。尽管前文提出一次只执行一个请求，但我们在编写应用程序的时候要提出一些假设。（最后一句我暂时没有理解）

### Async Workers

异步worker模式是基于`Greenlets`（通过`Eventlet`和`Gevent`实现）。`Greenlets`是由python的多线程协作实现的。一般情况下，我们的应用可以直接使用这个worker模式而不需要做其他事情。

### Tornado Workers

Gunicorn也支持Tornado worker模式，你还可以用Tornado的框架来写相应的应用。不过尽管Tornado worker能够用来处理WSGI应用，但文档并不推荐使用这个模式。

### AsyncIO Workers

这个模式是兼容python3的，有两种worker。

`gthread`模式是线程worker，它从主循环中接收连接，将这个连接作为一个任务放到线程池中。在长连接的情况下，连接会被放在循环中等待事件的发生，如果长连接超时还没有事件发生的话，连接会被关闭。

`gaiohttp`就是直接用`aiohttp`这个库。（仅支持python3）

## Choosing a Worker Type

默认的同步工作模式是假设你的应用是受到CPU和网络带宽的限制，一般情况下，就是说你的应用不会处理大量不可预见的访问。举个例子，假如你的应用突然收到很多请求，那么有可能会有很多连接堆积在服务器上，从而导致一些对外服务失效。在这种情况下，异步工作模式会更好一些。

文档解释推荐设置一个缓存代理在Gunicorn前面的主要原因，是因为假设了硬件资源不足。如果直接将这些暴露到网络中，dos攻击就会笑出声。文档推荐了用`Hey`来做负载测试。

***以下行为要求异步模式***

* 应用中有长阻塞
* 面向网络的服务请求（个人理解为就是API）
* 流请求和流响应
* 长时间轮询
* 网络套接字
* Comet

