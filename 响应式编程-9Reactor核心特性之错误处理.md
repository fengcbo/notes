# 响应式编程-Reactor核心特性之错误处理

在Reactive Streams中，错误是个终止事件。当错误发生时，它会终止序列并向操作符的下游传播直至最后一步(你定义的Subscriber以及它的onError方法)。



这些错误仍然应该在应用程序级别处理。例如，您可以在UI中显示错误通知，或者在REST端点中发送有意义的错误负载。因此，应该始终定义订阅者的onError方法。



> 如果没有定义，onError抛出UnsupportedOperationException异常。您可以进一步通过Exceptions.isErrorCallbackNotImplemented方法检测和排除异常。



反应器还提供了处理链中间错误的替代方法，作为错误处理操作符。



>在学习错误处理操作符之前，必须记住，反应序列中的任何错误都是终止事件。即使使用了错误处理操作符，也不允许原始序列继续。相反，它将onError信号转换为新序列(回退序列)的开始。换句话说，它替换了上游终止的序列。



现在我们可以考虑每种处理错误的方法。如果需要的话，我们将与命令式编程的try模式进行并行处理。



## 1. 错误处理操作符

您可能熟悉在try-catch块中处理异常的几种方法。最值得注意的是，其中包括:

1. catch并返回静态默认值
2. catch并执行一个可选拥有回退方法的路径
3. catch并动态计算回退值。
4. catch，然后包装成一个BusinessException并抛出
5. catch，然后记录特殊的错误消息，然后抛出
6. 使用finally块清除资源或者使用java7提供的try-with-resource结构



所有这些在Reactor中都有等价语法，以错误处理操作符的形式存在。



在介绍这些操作符之前，我们首先在反应链和try-catch块之间建立一个平行线。



当订阅时，链末端的onError回调类似于catch块。在这里，当Exception被抛出，执行流程就跳转到catch：

```java
Flux<String> s = Flux.range(1, 10)
    .map(v -> doSomethingDangerous(v)) ①
    .map(v -> doSecondTransform(v)); ②
s.subscribe(value -> System.out.println("RECEIVED " + value), ③
            error -> System.err.println("CAUGHT " + error) ④
);
```



①执行一个可能会抛出异常的转换

②如果运行正常，第二个转换将会执行

③每一个转换成功的值都会被打印出来

④如果出现错误，序列将终止并且错误信息将被打印出来



这是在概念上类似于下面的try / catch块：

```java
try {
    for (int i = 1; i < 11; i++) {
        String v1 = doSomethingDangerous(i); ①
        String v2 = doSecondTransform(v1); ②
        System.out.println("RECEIVED " + v2);
    }
} catch (Throwable t) {
    System.err.println("CAUGHT " + t); ③
}
```



①如果这里抛出异常

②剩余的循环将跳过

③执行流程跳转到这里



现在我们建立了一条平行线，然后我们看一下不同的错误处理案例和与之相同操作符。



###  静态返回值

与catch并返回静态默认值的方式功能相同的是onErrorReturn：

```java
Flux.just(10)
    .map(this::doSomethingDangerous)
    .onErrorReturn("RECOVERED");
```



根据发生的异常，你还可以选择使用filter(choose)来决定何时使用默认值回复；

```java
Flux.just(10)
    .map(this::doSomethingDangerous)
    .onErrorReturn(e -> e.getMessage().equals("boom10"), "recovered10");
```



### 回退方法

如果您想要一个以上的默认值，并且您有一种更安全的方法来处理数据，您可以使用onErrorResume。这相当于(2)(用回退方法捕获并执行另一个路径)。



例如，如果你的进程正在从一个外部的不可靠的服务中获取数据，但是您还保留了一个本地缓存的相同数据，这些数据可能有点过时，但更可靠，您可以执行以下操作:

```java
Flux.just("key1", "key2")
    .flatMap(k -> callExternalService(k)) ①
    .onErrorResume(e -> getFromCache(k)); ②
```



①对于每个键，我们异步调用外部服务。

②如果外部调用失败，我们就调用那个key的缓存。注意我们总是请求同一个回退方法不管错误源e是什么。



与onErrorReturn一样，onErrorResume也有一些变体，允许您根据异常的类或Predicate过滤哪些异常以进行回退。它需要一个函数，这也允许您根据遇到的错误切换到不同的回退序列:

```java
Flux.just("timeout1", "unknown", "key2")
    .flatMap(k -> callExternalService(k))
    .onErrorResume(error -> { ①
        if (error instanceof TimeoutException) ②
            return getFromCache(k);
        else if (error instanceof UnknownKeyException) ③ 
            return registerNewEntry(k, "DEFAULT");
        else
            return Flux.error(error); ④
    });
```



①这个函数总是动态选择如何继续

②如果是超时，获取本地缓存

③如果key未知，创建一个新的entry

④其他所有情况，重新抛出异常



### 动态回退值

即使您没有一种更安全的方法来处理数据，您也可能希望从您收到的异常中计算回退值。这相当于(3)(捕获并动态计算回退值)。



例如，如果你的返回类型持有异常的变体(想想Future.complete(T success) 以及Future.completeExceptionally(Throwable error))，你可能会传递异常并实例化错误处理的变体。



这可以与使用onErrorResume的回退方法解决方案相同。你需要一些样板文件:

```java
erroringFlux.onErrorResume(error -> Mono.just( ①
        myWrapper.fromError(error) ②
));
```



①这个样板通过Mono.just以及onErrorResume创建了一个Mono

②然后将异常包装到特定的类中，或者以其他方式计算异常值。



### 捕获并重新抛出

在“回退方法”示例中，flatMap中的最后一行给出了如何实现第(4)项(捕获、包装到BusinessException和重新抛出)的提示:

```java
Flux.just("timeout1")
    .flatMap(k -> callExternalService(k))
    .onErrorResume(original -> Flux.error(
        new BusinessException("oops, SLA exceeded", original)
    );
```



然而，有一种更直接的方法可以实现onErrorMap:

```java
Flux.just("timeout1")
    .flatMap(k -> callExternalService(k))
    .onErrorMap(original -> new BusinessException("oops, SLA exceeded", original));
```



### 记录或反应

在有些情况下，你可能想要继续传播错误但是你仍然想响应它而不修改序列(比如，记录错误)，此时可以使用doOnError操作符。这相当于(5)(捕获、记录特定错误的消息并重新抛出)。这个操作符，以及所有以doOn为前缀的操作符，有时被称为“副作用”。它们允许您在不修改序列的情况下查看序列的事件。



下面的示例确保，当我们返回到缓存时，至少会记录外部服务出现故障。我们也可以想象我们有统计计数器作为一个错误副作用增加。

```java
LongAdder failureStat = new LongAdder();
Flux<String> flux =
Flux.just("unknown")
    .flatMap(k -> callExternalService(k)) ①
    .doOnError(e -> {
        failureStat.increment();
        log("uh oh, falling back, service failed for key " + k); ②
    })
    .onErrorResume(e -> getFromCache(k)); ③
```



①可能会失败的外部服务调用

②记录日志

③调用缓存



### 使用资源和finally代码块

最后一个与强制编程并行的方法是通过Java 7“try-with-resources”构造或finally代码块的使用来完成清理工作。这相当于(6)(使用finally块清理资源或Java 7“try-with-resource”构造)。两者在Reactor都有等价操作，分别是using和doFinally：

```java
AtomicBoolean isDisposed = new AtomicBoolean();
Disposable disposableInstance = new Disposable() {
    @Override
    public void dispose() {
        isDisposed.set(true); ④
    }

    @Override
    public String toString() {
        return "DISPOSABLE";
    }
};

Flux<String> flux =
Flux.using(
        () -> disposableInstance, ①
        disposable -> Flux.just(disposable.toString()), ②
        Disposable::dispose ③
);
```

①第一个lambda生成资源。这里我们使用Disposable模拟。

②第二个lambda处理资源，返回一个Flux<T>。

③第三个lambda在从2)开始的流量终止或取消时调用，以清理资源。

④在订阅和执行序列之后，isdispose原子布尔值将变为true。



另一方面，doFinally是关于当序列终止时希望执行的操作，无论是onComplete、onError还是cancellation。它给了你一个暗示，什么终止触发了副作用：

```java
LongAdder statsCancel = new LongAdder(); ①

Flux<String> flux =
Flux.just("foo", "bar")
    .doFinally(type -> {
        if (type == SignalType.CANCEL) ②
          statsCancel.increment(); ③
    })
    .take(1); ④
```



①我们假设我们想收集统计数据。这里我们用的是LongAdder。

②doFinally为终止类型使用一个信号类型。

③这里我们只在取消时增加统计信息。

④take(1)将在发出1项后取消。



### 演示onError的终止方面

为了证明所有这些操作符在错误发生时导致上游原始序列终止，我们可以使用一个具有Flux.interval来更直观的展示。区间运算符每隔x个时间单位进行滴答，并增加一个Long值：

```java
Flux<String> flux =
Flux.interval(Duration.ofMillis(250))
    .map(input -> {
        if (input < 3) return "tick " + input;
        throw new RuntimeException("boom");
    })
    .onErrorReturn("Uh oh");

flux.subscribe(System.out::println);
Thread.sleep(2100); ①
```



①注意，默认情况下，interval在**timer** `Scheduler` 上执行。假设我们想在一个主类中运行这个示例，我们在这里添加一个sleep调用，这样应用程序就不会在不产生任何值的情况下立即退出。



每250ms打印一行如下:

```text
tick 0
tick 1
tick 2
Uh oh
```



即使有额外的一秒运行时，也不会从间隔中产生更多的滴答声。这个序列确实被错误终止了。



### 重试



























