在《Netty的服务端Demo》中我们写了一个基本的Netty服务端的代码，本节我们深入探究细节，一窥服务端的启动过程

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

在这里被赋值，我们层层回溯，查看函数调用的地方，发现最终函数调用的地方

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

> 读源码细节有两种读的方式：
>
> 一种是回溯，比如用到某个对象的时候可以逐层追溯，一定会找到该对象的最开始被创建的代码区块
>
> 还有一种方式就是自顶向下，逐层分析，一般用在分析某个具体的方法，庖丁解牛，最后拼接出完整的流程

下面重心就放在_NioServerSocketChannel_的默认构造函数上

```java
private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();
public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
```

```java
private static ServerSocketChannel newSocket(SelectorProvider provider) {
    //...
    return provider.openServerSocketChannel();
}
```

通过_SelectorProvider.openServerSocketChannel\(\)_创建一条服务端信道，然后进入以下方法

```java
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

第一行代码调用父类的方法，第二行new出来一个NioServerSocketChannelConfig，其顶层接口为ChannelConfig，netty官方的描述如下

> A set of configuration properties of a Channel.

基本可以判定：_ChannelConfig_ 也是netty里面的一大核心模块：该对象在创建_NioServerSocketChannel_对象的时候被创建

我们继续追踪到_NioServerSocketChannel_的父类

> AbstractNioMessageChannel.java

```java
protected AbstractNioMessageChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent, ch, readInterestOp);
}
```

继续往上追

> AbstractNioChannel.java

```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    //...
    ch.configureBlocking(false);
    //...
}
```

这里简单地将前面的 _provider.openServerSocketChannel\(\);_ 创建出来，_ServerSocketChannel _保存到成员变量，然后调用  
_ch.configureBlocking\(false\);_设置该_channel_为非阻塞模式，标准的jdk nio编程的玩法

这里的_readInterestOp_及前面层层传入的_SelectionKey.OP\_ACCEPT_，接下重点分析_super\(parent\);_（这里面的parent其实是null，由前面写死传入）

> AbstractChannel.java

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
```

这里又new出来了三大组件，赋值到成员变量，分别是

```java
id = newId();
protected ChannelId newId() {
    return DefaultChannelId.newInstance();
}
```

id是netty中每条channel的唯一标识

```java
unsafe = newUnsafe();
protected abstract AbstractUnsafe newUnsafe();
```

查看unsafe的定义

> Unsafe operations that should never be called from user-code. These methods are only provided to implement the actual transport, and must be invoked from an I/O thread

成功捕捉netty的又一大组件，先不管是什么，只要知道_newUnsafe_方法最终属于类_NioServerSocketChannel_中

最后

```java
pipeline = newChannelPipeline();

protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}

protected DefaultChannelPipeline(Channel channel) {
        this.channel = ObjectUtil.checkNotNull(channel, "channel");
        succeededFuture = new SucceededChannelFuture(channel, null);
        voidPromise =  new VoidChannelPromise(channel, true);

        tail = new TailContext(this);
        head = new HeadContext(this);

        head.next = tail;
        tail.prev = head;
}
```

要知道_DefaultChannelPipeline_是干嘛的？通过顶层接口_ChannelPipeline_的定义

> A list of ChannelHandlers which handles or intercepts inbound events and outbound operations of a Channel

从该类的文档中可以看出，该接口基本又是netty的一大核心模块

到了这里，我们将服务端channel创建完成，将这些细节串起来的时候，我们顺带提出netty的几大基本组件，总结如下：

* Channel
* ChannelConfig
* ChannelId
* Unsafe
* Pipeline
* ChannelHander

本片侧重服务端启动过程，这些组件后期深入每个组件

**总结一下，用户调用 Bootstrap.bind\(port\) 第一步就是通过反射的方式new一个NioServerSocketChannel 对象，并且在new的过程创建了一系列的核心组件**

* #### 初始化这个信道

到这里，回忆一下开始吗，第一步newChannel完毕，这里对这个channel做init，init方法如下：

```java
@Override
void init(Channel channel) throws Exception {
    final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        channel.config().setOptions(options);
    }

    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }

    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
    }
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
    }

    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

下面我们拆解步骤，一一分析

1.设置option和attr

```java
final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        channel.config().setOptions(options);
    }

    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }
```

通过这里我们可以看到，这里先调用options0\(\)以及attrs0\(\)，然后将得到的options和attrs注入到channelConfig或者channel中

2.设置新接入的channel的option和attr

```java
final EventLoopGroup currentChildGroup = childGroup;
final ChannelHandler currentChildHandler = childHandler;
final Entry<ChannelOption<?>, Object>[] currentChildOptions;
final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
synchronized (childOptions) {
    currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
}
synchronized (childAttrs) {
    currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
}
```

这里，和上面类似，只不过不是设置当前channel的这两个属性，而是对应到新进来连接对应的channel

3.加入新连接处理器

```java
p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
```

到了最后一步，`p.addLast()`向serverChannel的流水线处理器中加入了一个 `ServerBootstrapAcceptor`，从名字上就可以看出来，这是一个接入器，专门接受新请求，把新的请求扔给某个事件循环器，我们先不做过多分析

来，我们总结一下，我们发现其实init也没有启动服务，只是初始化了一些基本的配置和属性，以及在pipeline上加入了一个接入器，用来专门接受新连接，我们还得继续往下跟

* #### 将这个信道寄存给某个对象

这一步，我们是分析如下方法

```java
ChannelFuture regFuture = config().group().register(channel);
```

调用到NioEventLoop中的register

```java
@Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
```

```java
@Override
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```





### summary

最后，我们来做下总结，netty启动一个服务所经过的流程  
 1.设置启动类参数，最重要的就是设置channel  
 2.创建server对应的channel，创建各大组件，包括ChannelConfig,ChannelId,ChannelPipeline,ChannelHandler,Unsafe等  
 3.初始化server对应的channel，设置一些attr，option，以及设置子channel的attr，option，给server的channel添加新channel接入器，并出发addHandler,register等事件  
 4.调用到jdk底层做端口绑定，并触发active事件，active触发的时候，真正做服务端口绑定

[^1]: [转载自：**netty源码分析之服务端启动全解析**](https://www.jianshu.com/p/c5068caab217) 

