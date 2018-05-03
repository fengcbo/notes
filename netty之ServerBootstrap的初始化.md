# netty 源码分析

[上次](./netty之NioEventLoopGroup的构建.md)分析了NioEventLoopGroup的构建过程，本节将依然沿用上次的例子分析下ServerBootstrap的初始化过程。

## ServerBootstrap

ServerBootstrap的无参构造方法没有做任何事，不在赘述。下面分析下它的几个方法，首先看group方法：

![ServerBootstrap_group](./img/ServerBootstrap_group.png)

这个方法首先调用了父类(AbstractBootstrap)的group方法，看下：

![AbstractBootstrap_group](./img/AbstractBootstrap_group.png)

可以看到parentGroup也就是我们命名的bossGroup被设置给了AbstractBootstrap的成员变量，返回的B其实是继承了AbstractBootstrap的子类也就是ServerBootstrap。

同样childGroup也就是我们命名的workerGroup设置给了ServerBootstrap的成员变量childGroup。继续看下一个方法channel()

![AbstractBootstrap_channel](./img/AbstractBootstrap_channel.png)

channel()方法是在AbstractBootstrap中定义的，它的javadoc说：传入的channelClass(我们传入的是NioServerSocketChannel.class)是用来创建Channel实例的。你可以选择使用本方法，或者如果Channel的实现没有无参的构造方法，则调用channelFactory(io.netty.channel.ChannelFactory)。其实本方法也是在调用channelFactory(io.netty.channel.ChannelFactory)的方法，默认传入的io.netty.channel.ChannelFactory是io.netty.channel.ReflectiveChannelFactory。ReflectiveChannelFactory其实是利用反射调用无参的构造方法创建Channel实例，我们看下ReflectiveChannelFactory的定义：

![ReflectiveChannelFactory](./img/ReflectiveChannelFactory.png)

ReflectiveChannelFactory实现了ChannelFactory接口，见名知意ChannelFactory是创建Channel的工厂类，比较简单只定义了一个newChannel()方法，这里不再详述。ReflectiveChannelFactory实现了newChannel()方法，它是通过反射创建传入的class对象的实例，本例中是创建NioServerSocketChannel对象。

再来看channelFactory()方法：

![channelFactory01](./img/channelFactory01.png)

翻译一下javadoc：当调用bind()方法时，io.netty.channel.ChannelFactory被用来创建Channel的实例。这个方法通常在由于复杂的需求导致channel(Class)方法不再适用时才会被使用。如果你的Channel实现拥有无参的构造方法，强烈推荐使用channel(Class)来简化你的代码。

这个方法的javadoc给出了我们两个信息：
1. Channel的创建是在bind()方法调用时。
2. 如果Channel的实现类拥有无参的构造方法，那么推荐使用channel(Class)方法，而不是本方法。


这个方法调用了重载的方法，所以继续跟下去：

![channelFactory02](./img/channelFactory02.png)

注意这里的入参是io.netty.bootstrap.ChannelFactory <? extends C> channelFactory，上面的入参io.netty.channel.ChannelFactory<? extends C> channelFactory，其中io.netty.channel.ChannelFactory是io.netty.bootstrap.ChannelFactory的子接口。

这个方法所做的就是将传入的channelFacctory负值给成员变量，在bind方法调用时，使用该变量创建Channel对象。

channel()方法到此分析结束，它所做的工作就是用过传入的Channel Class，创建一个ChannelFactory赋值给AbstractBootstrap的成员变量private volatile ChannelFactory<? extends C> channelFactory，在调用bind()方法是通过该变量创建Channel对象。下面分析下handler和childHandler方法：

![handler](./img/handler.png)

![childHandler](./img/childHandler.png)

handler位于AbstractBootstrap类中，childHandler位于ServerBootstrap类中；handler用于处理请求作用于bossGroup，childHandler用于任务处理作用于workerGroup。这两个方法只是简单的成员变量负值。

下面分析bind()方法的实现：

![bind01](./img/bind01.png)

bind()方法接受一个int类型的端口号，返回一个 [ChannelFuture](./netty之ChannelFuture.md)。它调用了重载的方法，传入了通过port创建的InetSocketAddress。看下重载方法：

![bind02](./img/bind02.png)

这个方法首先校验group和channelFactory成员变量，确保他们非空，然后真正执行绑定操作(doBind)。

![doBind](./img/doBind.png)


