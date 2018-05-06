# Future

Future是异步处理的结果。它继承了java.util.concurrent.Future，扩展了[java.util.concurrent.Future](./java.util.concurrent.Future.md)的功能，解决了在某些情况下java.util.concurrent.Future的局限性。

[java.util.concurrent.Future](./java.util.concurrent.Future.md)只有一个isDone()来判断方法是否完成，但是future的完成会分为几种状态：成功结束、异常失败、调用返回。[java.util.concurrent.Future](./java.util.concurrent.Future.md)并不能很好的分辨这几种状态，所以netty扩展了[java.util.concurrent.Future](./java.util.concurrent.Future.md)，为它提供了isSuccess()来判断是否成功、以及cause()来获取异常信息；同时提供了 getNow()以非阻塞的方式获取返回值。

## Future增加的方法

### boolean isSuccess()

当且仅当I/O操作成功完成时返回true。

### boolean isCancellable()

当且仅当操作可以通过cancel(boolean)方法取消时返回true。

### Throwable cause()

当I/O操作失败时，返回I/O操作失败的原因。

如果操作成功或者future尚未完成，返回null。

### Future<V> addListener(GenericFutureListener<? extends Future<? super V>> listener)

向future添加指定的listener。当future完成(isDone)，指定的listener将被通知。如果future已经完成，指定的listener将立即被通知。

### Future<V> addListeners(GenericFutureListener<? extends Future<? super V>>... listeners)

向future添加指定的listener。当future完成(isDone)，指定的listener将被通知。如果future已经完成，指定的listener将立即被通知。

### Future<V> removeListener(GenericFutureListener<? extends Future<? super V>> listener)

从future中移出第一个出现的指定listener。当future完成(isDone)，指定的listeners将不会被通知。如果指定的listener没有与future关联，方法将什么都不坐并默默的返回。

### Future<V> removeListeners(GenericFutureListener<? extends Future<? super V>>... listeners)

从future中移出每个指定listeners的元素中在future中出现的第一个。当future完成(isDone)，指定的listeners将不会被通知。如果指定的listeners没有与future关联，方法将什么都不坐并默默的返回。


### Future<V> sync() throws InterruptedException

一直阻塞直到future完成，并且抛出失败的原因如果future失败。

### Future<V> syncUninterruptibly()

一直阻塞直到future完成，并且抛出失败的原因如果future失败。

### Future<V> await() throws InterruptedException

阻塞直到future完成。如果当前线程被中断，抛出InterruptedException。

### Future<V> awaitUninterruptibly()

阻塞直到future完成不抛出InterruptedException。这个方法捕获了InterruptedException，并且默默地丢弃了。

### boolean await(long timeout, TimeUnit unit) throws InterruptedException

在指定的时间内阻塞直到future完成。当且仅当future在给定的时间内完成，返回true。如果当前线程被中断，抛出InterruptedException。

### boolean await(long timeoutMillis) throws InterruptedException

在指定的时间内阻塞直到future完成。当且仅当future在给定的时间内完成，返回true。如果当前线程被中断，抛出InterruptedException。

### boolean awaitUninterruptibly(long timeout, TimeUnit unit)

在指定的时间内阻塞直到future完成，不会抛出InterruptedException。这个方法捕获了InterruptedException，并且默默地丢弃了。当且仅当future在给定的时间内完成，返回true。

### boolean awaitUninterruptibly(long timeoutMillis)

在指定的时间内阻塞直到future完成，不会抛出InterruptedException。这个方法捕获了InterruptedException，并且默默地丢弃了。当且仅当future在给定的时间内完成，返回true。

### V getNow()

非阻塞的获取结果。如果future尚未完成，将会返回null。

由于null可能被用来标注future成功了，你需要通过isDone()来检查future是否已经完成并且不要依赖返回的null。

### boolean cancel(boolean mayInterruptIfRunning)

如果返回操作成功了，它会标注future为失败，并抛出CancellationException。
