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

> _EventLoopGroup_ Netty最核心的是reactor线程，对应项目中使用广泛的NIOEventLoop，说白了：就是一个死循环，不停地检测IO时间，处理IO事件，执行任务
>
> _ServerBootstrap_是服务端的一个启动辅助类，通过给他设置一系列的参数来绑定端口启动服务
>
> _group\(bossGroup, workerGroup\) _bossGroup的作用是不断地接收新的连接，workerGroup负责处理bossGroup接收的请求



