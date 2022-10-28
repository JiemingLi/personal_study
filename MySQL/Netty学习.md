

# Channel，Channel Pipeline，Channel PipeLineContext，ChannelHandler的关系

创建1个channel就会有1个channelpipeline，1个channelpipeline就绑定多个channel handler，1个channel handler绑定多个channel handler context

一个Channel包含一个ChannelPipeline，创建Channel时会自动创建一个ChannelPipeline，每个Channel都有一个管理它的pipeline，这关联是永久性的。

# ChannelHandler

## 分类

Netty把ChannelHandler分为2类，InboundHandler和OutboundHandler。

## 工作原理

在workGroup接收到io读取事件后进行处理，此处理交给ChannelPipeline进行，更严格地说，是交给ChannelPipeline中的各个ChannelHandler按照一定的顺序进行处理。





![Netty基础招式——ChannelHandler的最佳实践](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c161500f7a6d48e08aafc327cd02290a~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

Netty接收到数据后，经过若干 InboundHandler 处理后接收成功。如果要输出数据，就需要经过若干个 OutboundHandler 处理完成后发送。

在使用Netty时，直接打交道的是ChannelPipeline和ChannelHandler，但是，它们之间有一座“隐形”的桥梁，名字叫做ChannelHandlerContext。ChannelHanderContext就是ChannelHandler的上下文，每个 ChannelHandler 都对应一个 ChannelHandlerContext。

每一个 ChannelPipeline 都包含多个 ChannelHandlerContext，所有 ChannelHandlerContext 之间组成了双向链表。如下图所示。

![Netty基础招式——ChannelHandler的最佳实践](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3bedbdfdc9ba433987b9155d4afbaf96~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)



![img](https://img2018.cnblogs.com/blog/1090617/201901/1090617-20190107182913543-217821213.jpg)****

有两个特殊的ChannelHandlerContext，分别是HeadContext和TailContext，表示双向链表的头尾节点。

![Netty基础招式——ChannelHandler的最佳实践](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/651a92af5ceb4072969e380cdca2ff26~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)



HeadContext同时实现了`ChannelInboundHandler`和`ChannelOutboundHandler`。因此，HeadContext在读取数据时作为头节点，向后传递InBound事件，同时，在写数据时作为尾节点，处理最后的OutBound事件。

TailContext只实现了`ChannelInboundHandler`。它在InBound事件传递的末尾，负责处理一些资源释放的工作。在OutBound事件传递的第一个节点，不做任何处理，仅仅传递OutBound事件给prev节点。

而我们平时自定义的ChannelHandler，就是插在这两个头尾节点之间的。



[理解pipeline](https://www.cnblogs.com/qdhxhz/p/10234908.html)

[channelHandle理解](https://juejin.cn/post/6994307073857044516#heading-0)