这里介绍一下Netty的一些核心组件概念，让大家对Netty有更深一步的了解

* **Channel**

Channel  称为信道，也叫数据信息流，与Channel有关的概念有以下4个![](/assets/1089449-afd9e14197e1ef11.png)_**channel**_

1. Channel：表示一个连接，可以理解为一次请求，就是一个channel
2. ChannelHandler:核心处理业务逻辑在这里进行处理，用于处理业务请求
3. ChannelHandlerContext:用于传输业务数据
4. ChannelPipeline:y用于保存处理过程中需要用到的ChannelHandler和ChannelHandlerContext

5. **ByteBuf**

ByteBuf是一个存储字节的容器，最大特点就是**使用方便**，它既有自己的读索引和写索引，方便对整段字节缓存进行读写，也支持get/set,方便你对其中的每个字节进行读写，数据结构如下：

![](/assets/1089449-b1ec677f253b692a.png)

_**ByteBuf数据结构**_

ByteBuf有三种使用模式：

**1.Heap Buffer 堆缓冲区**

堆缓冲区是byteBuf最常见的模式，它将数据存储在**堆内存**

**2.Direct Buffer 直接缓冲区**

直接缓冲区是ByteBuf的另一种常用模式，他的内存分配不发生在堆，JDK 1.4引入的NIO的ByteBuffer类允许JVM通过本地方法调用分配内存，这样做有2个好处

* 通过免去中间交换的内存拷贝，提升IO处理速度；直接缓冲区的内存可以留在垃圾回收扫描的堆内存外
* DirectBuffer 在 -XX:MaxDirectMemorySize=xxMd大小限制下，使用Heap之外的内存，GC对此“无能为力”，也就意味着规避了在高负载下频繁的GC过程对应用程序的中断影响

**3.Composite Buffer 复合缓冲区**

复合缓冲区相当于有多个不同的ByteBuf的视图，这是netty提供的，JDK不提供这样的功能

* **Codec**



