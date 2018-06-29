这里介绍一下Netty的一些核心组件概念，让大家对Netty有更深一步的了解

* **Channel**

Channel  称为信道，也叫数据信息流，与Channel有关的概念有以下4个![](/assets/1089449-afd9e14197e1ef11.png)_**channel**_

1. Channel：表示一个连接，可以理解为一次请求，就是一个channel
2. ChannelHandler:核心处理业务逻辑在这里进行处理，用于处理业务请求
3. ChannelHandlerContext:用于传输业务数据
4. ChannelPipeline:y用于保存处理过程中需要用到的ChannelHandler和ChannelHandlerContext

* **ByteBuf**

ByteBuf是一个存储字节的容器，最大特点就是**使用方便**，它既有自己的读索引和写索引，方便对整段字节缓存进行读写，也支持get/set,方便你对其中的每个字节进行读写，数据结构如下：

* **Codec**



