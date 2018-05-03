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
