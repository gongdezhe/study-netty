本篇文章主要讲述Netty是如何绑定接口，启动服务。通过Demo来了解Netty的各大核心组件，本片侧重讲解各大组件是如何协同来组建Netty核心

下面实现一个简单的服务端启动实例

```java
public final class SimpleServer {

    public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .handler(new SimpleServerHandler())
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                        }
                    });

            ChannelFuture f = b.bind(8888).sync();

            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    private static class SimpleServerHandler extends ChannelInboundHandlerAdapter {
        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            System.out.println("channelActive");
        }

        @Override
        public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
            System.out.println("channelRegistered");
        }

        @Override
        public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
            System.out.println("handlerAdded");
        }
    }
}
```

以上就是服务端启动的简单代码，下面分析每一步步骤细节

> EventLoopGroup Netty最核心的是reactor线程，对应项目中使用广泛的NIOEventLoop，说白了：就是一个死循环，不停地检测IO时间，处理IO事件，执行任务
>
> _ServerBootstrap_是服务端的一个启动辅助类，通过给他设置一系列的参数来绑定端口启动服务
>
> `group(bossGroup, workerGroup) bossGroup`的作用就是不断地接受到新的连接，将的新连接丢给`workerGroup`来处理
>
> `.channel(NioServerSocketChannel.class)`表示服务端启动的是NIO相关的信道，信道在网状里面是一大核心概念，可以理解为一条信道就是一个连接或者一个服务端绑定动作
>
> `.handler(new SimpleServerHandler()`表示服务器启动过程中，需要经过哪些流程，这里`SimpleServerHandler`名单最终的顶层接口为`ChannelHander`，是网状的一大核心概念，表示数据流经过的处理器，可以理解为流水线上的每一道关卡
>
> childHandler\(new ChannelInitializer&lt;SocketChannel&gt;\)...表示一条新的连接进来之后，该怎么处理，也就是上面所说的，boss如何给work分配
>
> `ChannelFuture f = b.bind(8888).sync();`这里就是真正的启动过程了，绑定8888端口，等待服务器启动完毕，才会进入下行代码
>
> `f.channel().closeFuture().sync();`等待服务端关闭套接字
>
> `bossGroup.shutdownGracefully(); workerGroup.shutdownGracefully();`关闭两组死循环

服务端的整个流程如上所示。

