# Netty为什么受欢迎？

netty是一款受到大公司青睐的框架。个人认为，他有以下几大优势：

1. 并发高
2. 传输快

## Netty优势之一：并发高

Netty是一款基于NIO\(Nonblocking I/O:非阻塞IO\)开发的网络通信框架，相比较于NIO（Blocking I/O：阻塞IO），他的并发性能有了很大提高。普及一下：BIO和NIO的区别

![](/assets/BIO1.png)

_**阻塞IO的通信方式**_

> 在BIO中，等待客户端发送数据的过程是阻塞的，这样就造成了一个线程只能处理一个请求的情况，而机器能支持的最大线程数是有限的，这就是BIO不能支持高并发的原因

![](/assets/1089449-9eebe781fba495fd.png)

_**非阻塞IO的通信方式**_

> 在NIO中，当一个Socket建立好连接之后，Thread并不会阻塞去接受这个Socket，而是将这个请求交给Selector，Selector会不断的去遍历所有的Socket，一旦有一个Socket建立完成，他会通知Thread，然后Thread处理完成数据在返回给客服端--这个过程是阻塞的，这样就能让一个Thread处理更多的请求了

下面2张图基于BIO的处理流程和NIO的处理流程，辅助理解两种方式的差别

![](/assets/1089449-6377fd47256970ef.png)

_**BIO的处理流程**_

![](/assets/1089449-78814cbb3acc30bd.png)

_**NIO的处理流程**_

**从上面NIO和BIO的学习，NIO的单线程处理连接的数量比BIO要高很多，单线程为什么能处理这么多的连接呢？**

主要在NIO中引入的Selector

当一个客户端的网络请求建立连接之后，服务端有2个步骤要做：

1. 接受客户端发送过来的全部数据
2. 服务端处理完请求业务之后返回response给客户端

BIO和NIO的区别主要在第一步

除了BIO和NIO外，还有一些其他的IO模型，下面图更加形象的介绍的5种IO模型的处理流程：

![](/assets/1089449-6ff240c3900e0aaa.png)

_**5种IO模型**_

> BIO：同步阻塞IO，阻塞整个步骤，如果连接少，他的延迟最低，因为一个线程只处理一个连接，适用于少连接且延迟低的场景。如：数据库连接
>
> NIO：同步非阻塞IO，阻塞业务处理但不阻塞数据接收，适用于高并发且处理简单的场景。如：聊天软件
>
> 多路复用IO：他的数据接收和业务处理2个步骤是分开的，也就是说：一个连接可能他的数据接收是线程A完成的，数据处理是线程B完成的，他比BIO能处理的请求多，但是比不了NIO，但是他的处理性能又比BIO更差，因为一个连接他需要2次系统调用，而BIO他只需要一次，所以这种IO模型应用的不错
>
> 信号驱动IO：这种IO主要是嵌入式开发，不予深究
>
> 异步IO：他的数据请求和数据处理都是异步的，数据请求一次返回一次，试用于长连接的业务场景

以上摘自[Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)

## Netty优势之一：传输快

Netty的传输快依赖NIO的一个特性--零拷贝

我们知道，在Java内存模型中：内存存在栈内存、堆内存和字方法区等等，其中堆内存是占用内存空间最大的一块，也是Java对象存放的地方，一般我们的数据需要从IO读取到堆内存，网络数据交互中需要经过Socket缓存，也就是说一个数据会被拷贝2次才能到达他的终点，如果数据量大，不仅数据拷贝耗时长而且还造成资源浪费

Netty针对这种情况，使用了NIO的一大特性--零拷贝，当他需接收数据时候，他会在堆内存之外开辟一块内存，数据就直接从IO中读取到那块内存中，在Netty中通过ByteBuf可以对这些数据进行直接操作，从而加快传输速度

下面2图介绍两种拷贝方式的区别：

摘自[Linux 中的零拷贝技术，第 1 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-zerocopy1/index.html)

![](/assets/1089449-014c9e07d56e4be5.png)

_**传统数据拷贝**_

![](/assets/1089449-50e5aa5eec7e86cc.png)

_**零拷贝**_

上文介绍的ByteBuf是Netty的一个重要概念，他是Netty数据处理的容器

