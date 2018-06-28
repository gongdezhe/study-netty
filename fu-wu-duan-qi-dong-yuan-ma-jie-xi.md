在《Netty的服务端Demo》中我们写了一个基本的Netty服务端的代码，本节我们深入骑宠细节，一窥服务端的启动过程

_ServerBootstrap_一系列参数使用[方法chaining](https://en.wikipedia.org/wiki/Method_chaining#Java)的方式将启动服务器需要的参数保存提交。这里不做重点介绍

重点看以下这段代码

```java
b.bind(8888).sync();
```

根据代码跟踪查看，确定这行代码就是最终启动服务的入口，我们跟代码继续分析

```java
public ChannelFuture bind(int inetPort) {
    return bind(new InetSocketAddress(inetPort));
}
```

通过端口号创建一个_InetSocketAddress_，然后继续绑定

```java
public ChannelFuture bind(SocketAddress localAddress) {
    validate();
    if (localAddress == null) {
        throw new NullPointerException("localAddress");
    }
    return doBind(localAddress);
}
```

_validate\(\)_验证服务启动参数的合法性，然后调用_doBind\(\)_

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    //...
    final ChannelFuture regFuture = initAndRegister();
    //...
    final Channel channel = regFuture.channel();
    //...
    doBind0(regFuture, channel, localAddress, promise);
    //...
    return promise;
}
```

这里去除掉细枝末节，专注核心方法，2大核心方法:`initAndRegister()和doBind0()`

下面逐一分析：

## `initAndRegister()`

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    // ...
    channel = channelFactory.newChannel();
    //...
    init(channel);
    //...
    ChannelFuture regFuture = config().group().register(channel);
    //...
    return regFuture;
}
```

我们看到`initAndRegister()做了几件事情`

1. 新创建一个channel
2. 初始化这个channel
3. 将这个channel register寄存给某个对象

三个逐一分析

* #### new一个信道

首先要知道信道的定义,官方给出的对信道的描述如下：

> 与网络套接字或能够进行I / O操作（如读取，写入，连接和绑定）的组件的关系

这里的信道，是在服务启动的时候创建，我们可以和普通的网络编程中的ServerSocket对应理解

通过`initAndRegister()的实现我们知道信道是通过一个channelFactory`新创建出来，`channelFactory的接口如下`

```java
public interface ChannelFactory<T extends Channel> extends io.netty.bootstrap.ChannelFactory<T> {
    /**
     * Creates a new channel.
     */
    @Override
    T newChannel();
}
```

就一个方法，我们查看_ChannelFactory_被赋值的地方

> AbstractBootstrap.java

```java
public B channelFactory(ChannelFactory<? extends C> channelFactory) {
    if (channelFactory == null) {
        throw new NullPointerException("channelFactory");
    }
    if (this.channelFactory != null) {
        throw new IllegalStateException("channelFactory set already");
    }

    this.channelFactory = channelFactory;
    return (B) this;
}
```

在这里被赋值，我们层层回溯[^1]，查看函数调用的地方，发现最终函数调用的地方

```java
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    }
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
}
```

这里，我们演示调用程序_channel\(channelClass\)_方法的时候，将_channelClass_作为_ReflectiveChannelFactory_的构造函数创建出一个_ReflectiveChannelFactory_

演示端的代码如下：

```java
.channel(NioServerSocketChannel.class);
```

在本节`initAndRegister()的时候我们看到：`

```java
channelFactory.newChannel();
```

可以推断，最终调用到`ReflectiveChannelFactory.newChannel()`方法

```java
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {

    private final Class<? extends T> clazz;

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        if (clazz == null) {
            throw new NullPointerException("clazz");
        }
        this.clazz = clazz;
    }

    @Override
    public T newChannel() {
        try {
            return clazz.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + clazz, t);
        }
    }
}
```

看到_clazz.newInstance\(\);_，我们明白，原来是通过反射的方式来创建的一个对象，而这个类我们是在 _ServerBootstrap_ 中传入的_NioServerSocketChannel.class_

通过上面可知，最终创建信道相当于调用默认构造函数创建一个`NioServerSocketChannel`对象





* #### 初始化这个信道
* #### 将这个信道寄存给某个对象



读源码细节，有两种读的方式，一种是回溯，比如用到某个对象的时候可以逐层追溯，一定会找到该对象的最开始被创建的代码区块，还有一种方式就是自顶向下，逐层分析，一般用在分析某个具体的方法，庖丁解牛，最后拼接出完整的流程

