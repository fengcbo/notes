# 响应式编程-Reactor核心特性之Schedulers

和RxJava类似，Reactor是与并发无关的。也就是说，它不强制执行并发模型。相反，它让开发人员掌握主动权。但是，这并不妨碍它帮助你实现并发。



在Reactor中，执行模型和执行发生的位置由使用的调度器Scheduler决定。调度器是一个接口，可以抽象广泛的实现。Schedulers类提供了很多静态方法来访问下面的执行上下文：

- 当前线程(Schedulers.immediate())
- 单独的可重用的线程(Schedulers.single())。注意这个方法对所有的调用者重用相同的线程，直到Scheduler被回收。如果你想每次调用都专门用个线程，每个调用都使用`Schedulers.newSingle()` 
- 弹性的线程池(Schedulers.elastic())。如果需要它会创建新的worker池，并重用闲置的worker。闲置时间太长(默认值是60s)的工作池被回收。例如，对于I/O阻塞工作，这是一个很好的选择。elastic()是一种方便的方法，可以为阻塞进程提供自己的线程，这样它就不会占用其他资源。
- 为并行工作而调优的固定worker池(Schedulers.parallel())。它会创建和你的CPU核心数一样的worker。

> 虽然 elastic 可以帮助处理遗留的阻塞代码(如果无法避免的话)，但是single和parallel则不能。因此，在默认的 single 和 parallel 调度器中使用阻塞api (block()、blockFirst()、blockLast()、toIterable()和toStream()将导致抛出一个IllegalStateException。
>
> ​    
>
> 自定义调度器还可以通过创建实现 NonBlocking 标记接口的 Thread 实例来标记为“non blocking only”。



此外，您可以通过使用Schedulers.fromexecutorservice (ExecutorService)从任何已存在的ExecutorService中创建Scheduler。(你也可以从Executor创建Scheduler，尽管不鼓励这样做。) 你也可以通过newXXX方法创建各种Scheduler的新实例。比如，Schedulers.newElastic(yourScheduleName)创建一个新的elastic scheduler，名字为yourScheduleName。



> 操作符是通过调优的非阻塞算法实现的，以方便在某些调度程序中进行的工作窃取。



一些操作默认会使用指定的来自Schedulers的Scheduler(通常会提供一个选项选择其他Scheduler)。比如，调用工厂方法 `Flux.interval(Duration.ofMillis(300))`会返回一个没300ms发送元素的Flux<Long>对象。这个方法默认启用Schedulers.parallel()。下面的代码更改了Scheduler，使用了一个类似Schedulers.single()的新实例：

```java
Flux.interval(Duration.ofMillis(300), Schedulers.newSingle("test"))
```



在响应式链中Reactor提供了两种切换执行上下文的方式(Scheduler)：publishOn和subscribeOn。两个方法都需要传入Scheduler并切换执行上下文到该scheduler。但是，publishOn在链中的位置很重要，而订阅的位置则无关紧要。理解他们之间的不同，你首先需要记住在subscribe()之前不会发生任何事。



在Reactor中，当你连接操作时，你可以按需包装Flux和Mono到另一个。一旦您订阅了，就会创建一个订阅服务器对象(Subscriber)链，并向后(向上)创建到第一个发布服务器。这实际上是隐藏的。您可以看到的是Flux(或Mono)和Subscription的外层，但是真正的工作是由这些特定于操作符的中间订阅者完成的。



有了这些知识，我们可以更深入地了解publishOn和subscribeOn 操作符:

- publishOn的应用方式与订阅者链中间的任何其他操作符相同。它接受来自上游的信号，并在执行来自相关调度器Scheduler的worker回调时在下游重放它们。因此，它会影响后续操作符执行的位置(直到另一个publishOn被链接进来)。
- 当构造后向链时，subscribeOn将应用于订阅流程。因此，无论您将订阅方放在链中的哪个位置，它总是会影响源发射的上下文。然而，这并不影响后续调用publishOn的行为。在它们之后，它们仍然为链的一部分切换执行上下文。

> 在链中，实际上只考虑了最早的subscribeOn调用。











































