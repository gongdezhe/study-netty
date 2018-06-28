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



