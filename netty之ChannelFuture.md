# ChannelFutures

ChannelFuture继承了[io.netty.util.concurrent.Future](./netty之Future.md)，而[io.netty.util.concurrent.Future](./netty之Future.md)继承了[java.util.concurrent.Future](./java.util.concurrent.Future.md)

## javadoc

ChannelFuture是异步通道 I/O 操作的结果。

netty所有的I/O操作都是异步的。这就意味着所有的I/O调用都不立即返回，并且不担保I/O操作请求在调用结束时被完成。相反，你将会得到一个ChannelFuture对象，这个对象将会给与你结果的信息以及I/O操作的状态。

ChannelFuture可以使已完成也有可能是未完成。当I/O操作开始，一个新的Future对象就被创建。这个新的Future初始是未完成的 - 它也不是成功的、失败的或者被取消的，因为I/O操作还没有完成呢。如果一个I/O操作已经成功的完成、失败或取消，那么Future将被标记为完成，并且附加了更多的特殊信息，比如失败的原因。请注意甚至失败和返回都属于完成状态。

```text
                                      +---------------------------+
                                      | Completed successfully    |
                                      +---------------------------+
                                 +---->      isDone() = true      |
 +--------------------------+    |    |   isSuccess() = true      |
 |        Uncompleted       |    |    +===========================+
 +--------------------------+    |    | Completed with failure    |
 |      isDone() = false    |    |    +---------------------------+
 |   isSuccess() = false    |----+---->      isDone() = true      |
 | isCancelled() = false    |    |    |       cause() = non-null  |
 |       cause() = null     |    |    +===========================+
 +--------------------------+    |    | Completed by cancellation |
                                 |    +---------------------------+
                                 +---->      isDone() = true      |
                                      | isCancelled() = true      |
                                      +---------------------------+
```

netty为你提供了多种方法允许你检查I/O操作是否完成、等待操作完成以及获取I/O操作的结果。它也允许你添加ChannelFutureListener，这样你就可以在I/O操作完成时获得通知。

### 推荐使用addListener(GenericFutureListener)而不是await()

建议使用addListener(GenericFutureListener)而不是await()，以便在执行I/O操作时获得通知，并执行后续任务。

addListener(GenericFutureListener)是非阻塞的。可以非常简单的向 ChannelFuture添加特定的 ChannelFutureListener，当与future关联的I/O操作完成时，I/O线程将会通知这些listener。ChannelFutureListener 会产生最好的性能和资源使用率，但是实现一个连续的逻辑是非常棘手的，如果你不使用事件驱动编程。

相比之下，await()是一个阻塞操作。一旦调用，调用者线程将会阻塞直到I/O操作完成。使用await()可以很容易实现连续的逻辑，但是调用者线程会出现不必要的阻塞直到I/O操作完成，并且会有相当昂贵的内部线程通知开销。而且，在特殊的情况下会产生死锁，如下面描述的那样：

### 在ChannelHandler中不要调用await()

ChannelHandler中的事件处理方法通常是由I/O线程调用的。如果I/O线程调用的事件处理方法调用了await()方法，它所等待的I/O操作可能永远也不会完成，这是由于await()方法会阻塞住它所等待的I/O操作，这就产生了死锁。

 ```java
 // BAD - NEVER DO THIS
  @Override
 public void channelRead(ChannelHandlerContext ctx, Object msg) {
     ChannelFuture future = ctx.channel().close();
     future.awaitUninterruptibly();
     // Perform post-closure operation
     // ...
 }

 // GOOD
  @Override
 public void channelRead(ChannelHandlerContext ctx, Object msg) {
     ChannelFuture future = ctx.channel().close();
     future.addListener(new ChannelFutureListener() {
         public void operationComplete(ChannelFuture future) {
             // Perform post-closure operation
             // ...
         }
     });
 }
 ```

尽管有上面提到的缺点，仍然会有场景调用 await()更方便。在这些场景下，请确保不是I/O线程调用的await()。否则，为了防止死锁，BlockingOperationException将会被抛出。

### 不要混淆I/O 超时和await超时

你为 Future.await(long), Future.await(long, TimeUnit), Future.awaitUninterruptibly(long), 或者 Future.awaitUninterruptibly(long, TimeUnit) 指定的超时时间与 I/O 超时时间没有任何关系。如果I/O操作超时，future将会如上面所说的那样被标记为完成但是失败了。比如，链接超时需要通过一个传输选项来配置。

```java

// BAD - NEVER DO THIS
 Bootstrap b = ...;
 ChannelFuture f = b.connect(...);
 f.awaitUninterruptibly(10, TimeUnit.SECONDS);
 if (f.isCancelled()) {
     // Connection attempt cancelled by user
 } else if (!f.isSuccess()) {
     // You might get a NullPointerException here because the future
     // might not be completed yet.
     f.cause().printStackTrace();
 } else {
     // Connection established successfully
 }

 // GOOD
 Bootstrap b = ...;
 // Configure the connect timeout option.
 b.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 10000);
 ChannelFuture f = b.connect(...);
 f.awaitUninterruptibly();

 // Now we are sure the future is completed.
 assert f.isDone();

 if (f.isCancelled()) {
     // Connection attempt cancelled by user
 } else if (!f.isSuccess()) {
     f.cause().printStackTrace();
 } else {
     // Connection established successfully
 }
```


## 重要方法

ChannelFuture添加了两个重要方法，channel()和isVoid()

### Channel channel()

channel()返回与此关联的I/O操作的通道。

### boolean isVoid()

isVoid()返回true，如果ChannelFuture是一个void的future，并且不允许调用以下任何一个方法：

- addListener(GenericFutureListener)
- addListeners(GenericFutureListener[])
- await()
- Future.await(long, TimeUnit) ()}
- Future.await(long) ()}
- awaitUninterruptibly()
- sync()
- syncUninterruptibly()