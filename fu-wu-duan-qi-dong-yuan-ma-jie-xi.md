在《Netty的服务端Demo》中我们写了一个基本的Netty服务端的代码，本节我们深入骑宠细节，一窥服务端的启动过程

ServerBootstrap一系列参数使用[方法chaining](https://en.wikipedia.org/wiki/Method_chaining#Java)的方式将启动服务器需要的参数保存提交。这里不做重点介绍

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

通过端口号创建一个InetSocketAddress，然后继续绑定

```java
public ChannelFuture bind(SocketAddress localAddress) {
    validate();
    if (localAddress == null) {
        throw new NullPointerException("localAddress");
    }
    return doBind(localAddress);
}
```

validate\(\)验证服务启动参数的合法性，然后调用doBind\(\)

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

就一个方法，我们查看ChannelFactory被赋值的地方

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

在这里被赋值，我们层层回溯，查看函数调用的地方，发现最终函数调用的地方









* #### 初始化这个信道
* #### 将这个信道寄存给某个对象



