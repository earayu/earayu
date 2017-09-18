# Netty



## 组件

* BootStrap	ServerBootStrap
  * Channel 所有方法都是线程安全的，多线程write自动排序
  * ChannelConfig 可以on the fly配置
* Eventloop
* EventLoopGroup
* ChannelInitializer
* ChannelPipeline
* ChannelHandler
* ChannelFuture



### ChannelHandler

ChannelHandlerContext和Channel都有write()方法，区别在于。ctx的write是写入下一个handler；而channel的write是写入最开始的handler，也就是管道的开头。

#### ChannelInboundHandler

#### ChannelOutboundHandler



## 秘密

* 一个EventLoop可以处理多个Channel
* ChannelInitializer本身也是ChannelPipeline中的一个ChannelHandler，在添加完其他handler后会把自己删除
* EventLoopGroup中每个EventLoop都是单线程的，每个Channel都永远地被同一个Eventloop持有，所以不用考虑线程安全问题