# Future

## javadoc

Future 表现为异步计算的结果。提供的方法被用来检查计算是否完成、等待计算完成、获取计算的结果。当计算完成时，结果只能使用get方法获得，在必要的情况下阻塞直到计算完成。返回操作通过cancel()方法来执行。提供的额外方法用来确定任务是否正常结束，或者被取消。一旦计算完成，计算就不能被取消。如果您想要使用Future的取消能力，但不能提供一个可用的结果，您可以声明Future的类型并返回null作为基础任务的结果。

使用实例(注意以下的类都是虚构的)

```java
interface ArchiveSearcher { String search(String target); }
 class App {
   ExecutorService executor = ...
   ArchiveSearcher searcher = ...
   void showSearch(final String target)
       throws InterruptedException {
     Future<String> future
       = executor.submit(new Callable<String>() {
         public String call() {
             return searcher.search(target);
         }});
     displayOtherThings(); // do other things while searching
     try {
       displayText(future.get()); // use future
     } catch (ExecutionException ex) { cleanup(); return; }
   }
 }
```

FutureTask是Future的实现，并且实现了Runable，所以可以被Executor执行。比如，已上的构建提交可以替换为：

```java
FutureTask<String> future =
   new FutureTask<String>(new Callable<String>() {
     public String call() {
       return searcher.search(target);
   }});
 executor.execute(future);
```

内存一致性影响：异步计算所才去的操作 发生在 接下来的在另一个线程的Future.get()的响应 之前。

## 重要方法

### boolean cancel(boolean mayInterruptIfRunning)

尝试取消这个任务的执行。这个尝试将会失败，当任务已经完成、已经被取消、或者由于其他原因不能被取消。如果成功，并且当调用cancel()时任务还未开始，这个任务将永远不会得到执行。如果任务已经开始，mayInterruptIfRunning将决定是否尝试中断执行任务的线程以停止任务。

这个方法返回之后，随后的isDone()调用将会一直返回true；随后的isCancelled()将一直返回true如果这个方法返回true。

### boolean isCancelled()

如果任务在它正常完成之前被取消，返回true。

### boolean isDone()

如果任务完成，返回true。完成可能由于正常终止、异常或者返回操作 -- 所有的这些场景，这个方法都会返回true。

### V get()

如果必要会阻塞等待计算完成，然后获取它的结果。

### V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException

如果必要会在给定的时间内阻塞等待计算完成，然后如果能够获取，就获取它的结果。
