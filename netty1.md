# netty源码分析

我们先看一个server的启动示例：

```
public class MyServer {

    public static void main(String[] args) {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        ServerBootstrap bootstrap = new ServerBootstrap();

        bootstrap.group(bossGroup,workerGroup).channel(NioServerSocketChannel.class).childHandler(new MyServerInitializer());

        try {
            ChannelFuture channelFuture = bootstrap.bind(8899).sync();
            channelFuture.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

这段程序是我们使用netty的模板，先创建bossGroup和workerGroup两个EventLoopGroup(事件循环组)，然后创建ServerBootstrap启动器，再向ServerBootstrap设置我们的bossGroup和workerGroup，以及我们使用的Channel类型(本例中使用NioServerSocketChannel)，还有workGroup使用的handler(通过childHandler注册我们定义的Initializer)，最后我们将启动器班内绑定到固定的端口上，然后在finally优雅关闭线程组。

以上是我们使用netty的基本流程，下面我们分析下EventLoopGroup。

## EventLoopGroup

什么是EventLoopGroup呢？看下EventLoopGroup的javadoc
```
Special EventExecutorGroup which allows registering Channels that get processed for later selection during the event loop.
```

翻译过来的意思就是：EventLoopGroup是特殊的EventExecutorGroup，它允许注册多个通道，这些通道会在事件循环期间得到后续的selection操作的处理。

这里的selection操作指的是NIO中的Selector的select操作。

EventExecutorGroup除了包含父类的方法外，它还包含额外的四个方法：

### 1. EventLoop next()
EventExecutorGroup见名知意，它是事件循环组，那么它肯定包含多个事件循环(EventLoop)。next方法返回下一个要使用的事件循环(EventLoop)

### 2. ChannelFuture register(Channel channel);
向事件循环(EventLoop)注册一个通道。返回一个ChannelFuture对象，当注册完成时会ChannelFuture将会得到一个通知。

### 3. ChannelFuture register(ChannelPromise promise);
使用一个ChannelFuture向事件循环(EventLoop)注册一个通道。这里的ChannelFuture指的是ChannelPromise promise，这是由于ChannelPromise实现了ChannelFuture接口。当注册完成时，传递的ChannelFuture对象(即ChannelPromise promise)得到通知，并且得到返回值。

### 4. ChannelFuture register(Channel channel, ChannelPromise promise);
这个方法已经被netty标为过时，netty建议使用第3个方法，在此不再赘述。
