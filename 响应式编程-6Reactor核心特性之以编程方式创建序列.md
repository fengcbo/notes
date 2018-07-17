# 响应式编程-Reactor核心特性之以编程方式创建序列

在本文中，我们将介绍Flux和Mono的创建，并通过编程的方式定义它关联的事件(onNext, onError 以及 onComplete)。所有的这些方法都揭露了一个事实：即它们公开了API来触发我们称之为sink(水槽)的事件。

## 1. Generate

Flux 编程式创建的最简单的方式是通过generate()方法，这个方法需要一个生成器函数。

这是同步的、一个接一个生产，这也就意味着sink是SynchronousSink(同步sink)并且每一次回调执行，它的next()只能被调用一次。那你可以额外的调用error(Throwable) 或者 complete()，但这是可选的。

最有用的变体可能是允许您保持一个状态，您可以在使用sink时引用这个状态来决定接下来要发出什么。生成器函数变成了拥有状态对象类型<S>的 BiFunction<S, SynchronousSink<T>, S>。你必须提供一个Supplier<S>来指定初始状态，现在你的生成器函数回在每次循环时返回新的状态。

例如，你可以使用int作为状态:

基于状态的generate示例
```java
Flux<String> flux = Flux.generate(
    () -> 0, ①
    (state, sink) -> {
      sink.next("3 x " + state + " = " + 3*state); ②
      if (state == 10) sink.complete(); ③
      return state + 1; ④
    });
flux.subscribe(System.out::println);
```

①提供状态的初始值：0
②使用状态来选择产生什么(状态 * 3)
③使用状态判断什么时候停止
④返回下次使用的新的状态值(除非在本次调用时序列停止)

上面的代码输出如下：

```
3 x 0 = 0
3 x 1 = 3
3 x 2 = 6
3 x 3 = 9
3 x 4 = 12
3 x 5 = 15
3 x 6 = 18
3 x 7 = 21
3 x 8 = 24
3 x 9 = 27
3 x 10 = 30
```

你也可以使用可变的<S>状态。上面的示例可以使用一个单例的AtomicLong来作为状态，并在每次循环时修改它：


可变状态示例
```java
Flux<String> flux = Flux.generate(
    AtomicLong::new, ①
    (state, sink) -> {
      long i = state.getAndIncrement(); ②
      sink.next("3 x " + i + " = " + 3*i);
      if (i == 10) sink.complete();
      return state; ③
    });
flux.subscribe(System.out::println);
```

①这次我们使用可变对象作为状态
②在这里修改状态
③返回想用的对象作为新的状态


> 如果状态对象需要清理资源，使用generate(Supplier<S>, BiFunction, Consumer<S>)


下面是包含Consumer的generate的使用示例：
```
Flux<String> flux = Flux.generate(
                AtomicLong::new,
                (state, sink) -> {①
                    long i = state.getAndIncrement();②
                    sink.next("3 x " + i + " = " + 3*i);
                    if (i == 10) sink.complete();
                    return state;③
                }, (state) -> System.out.println("state: " + state));④
flux.subscribe(System.out::println);
```


①我们依然使用可变对象作为状态
②在这里修改状态
③返回想用的对象作为新的状态
④可以看到在Consumer的lambda中state的值为：11

对于包含数据库连接或在流程结束时需要处理的其他资源的状态，消费者lambda可以关闭连接或处理在流程结束时应该完成的任何任务。



## 2. Create

通过编程创建Flux的更高级形式create可以异步或同步地工作，并且适用于每轮的多个产出。

它公开了FluxSink及其next、error以及complete方法。与generate相反，它不包含状态。也就是说，它可能在回调中触发多个事件(甚至在任何线程任何时间点)。

> create对于现存API与reactive的连接非常有用 - 比如异步的基于监听的API。



想想你使用基于监听的API。它通过块处理数据，毕竟且有两个事件：(1) 已经准备好的数据块 (2)完成事件处理；如MyEventListener接口：

```java
interface MyEventListener<T> {
    void onDataChunk(List<T> chunk);
    void processComplete();
}
```



你可以使用create将它连接到Flux<T>：

```java
Flux<String> bridge = Flux.create(sink -> {
    myEventProcessor.register( ④
      new MyEventListener<String>() { ①

        public void onDataChunk(List<String> chunk) {
          for(String s : chunk) {
            sink.next(s); ②
          }
        }

        public void processComplete() {
            sink.complete(); ③
        }
    });
});
```

①连接到MyEventListener

②块中的每个元素成为Flux的一个元素

③processComplete被转换成了onComplete

④所有的这些都是异步完成的，不管myEventProcessor何时执行



此外，由于create可以异步执行并且可以管理背压，所以您可以通过指示一个溢出策略(OverflowStrategy)来改进如何表现回压。



- IGNORE：完全忽略下游的背压请求。当下游的队列满时，会产生一个IllegalStateException异常。
- ERROR：但下游不能保持时，发送一个IllegalStateException异常。
- DROP：如果下游不能接受了，就丢弃掉收到的信号。
- LATEST：使下游只获取来自上游的最新信号。
- BUFFER：(默认)如果下游不能保持，就缓存所有的信号。(如果使用无界缓存可能导致OutOfMemoryError)。



> Mono也有一个create生成器。Mono create的MonoSink不允许生产多个。在生产完第一个后，它会丢弃所有的信号。



### Push模型

push是create的一个变种，它适合通过单个生产者处理事件。和create类似，push也是异步的并可以通过create支持的溢出策略来管理背压。只有一个生产者线程在同一时刻可以执行next、complete或者error方法。



```java
Flux<String> bridge = Flux.push(sink -> {
    myEventProcessor.register(
      new SingleThreadEventListener<String>() { ①

        public void onDataChunk(List<String> chunk) {
          for(String s : chunk) {
            sink.next(s); ②
          }
        }

        public void processComplete() {
            sink.complete(); ③
        }

        public void processError(Throwable e) {
            sink.error(e); ④
        }
    });
});

```



①连接到SingleThreadEventListener API 

②使用next方式，事件从的单一的监听线程被push到sink(水槽)

③complete事件也在同一个监听线程被执行

④error事件也在同一个监听线程被执行



### 混合 push/pull模型

与push不同，create可以在push或pull模式中使用，使之适合于使用基于监听器的api进行桥接，在这些api中，数据可以随时异步地交付。onRequest回调可以被注册到FluxSink上以追踪请求。这个回调被用来从源头请求更多的数据，而且可以在请求等待时通过分发数据到sink(水槽)来管理背压。他启用了一种混合push/pull模型，在这种模型中，下游可以从上游和上游拉出已经可用的数据，并在稍后数据可用时将数据推到下游。

```java
Flux<String> bridge = Flux.create(sink -> {
    myMessageProcessor.register(
      new MyMessageListener<String>() {

        public void onMessage(List<String> messages) {
          for(String s : messages) {
            sink.next(s); ③
          }
        }
    });
    sink.onRequest(n -> {
        List<String> messages = myMessageProcessor.request(n); ①
        for(String s : message) {
           sink.next(s); ②
        }
    });
```



①在发出请求时轮询消息。

②如果消息立即可用，push消息到sink(水槽)。

③随后异步到达的其余消息也将被分发。



### 清理

onDispose和onCancel是两个回调函数，在取消或者终止时执行任何清理工作。onDispose可以被用来执行清理操作，当Flux完成、出错或被取消。onCancel可用于在使用onDispose清除之前执行任何特定于取消的操作。



```java
Flux<String> bridge = Flux.create(sink -> {
    sink.onRequest(n -> channel.poll(n))
        .onCancel(() -> channel.cancel()) ①
        .onDispose(() -> channel.close())  ②
    });
```

①onCancel用于取消信号。

②onDispose用于完成、错误或取消信号。



## 3. Handle



handle方法稍微有点不同。它在Mono和Flux都存在。此外，它是一个实例方法，这意味着它被链接到一个现有的源(类似常见的操作符)。



在使用SynchronousSink以及只允许顺序生产的场景下，它与generate类似。但是，handle可以用于从每个源元素中生成任意值，可能会跳过一些元素。通过这种方式，它可以作为map和filter的组合提供服务。handle方法的签名如下：

```java
handle(BiConsumer<T, SynchronousSink<R>>)
```



下面我们看个示例。reactive streams规范不允许序列中出现null。假如你想要执行一个map操作，但是你想使用已存在的方法来作为map函数，但是这个函数有时返回null怎么办？



比如，下面的方法可以安全的提供一个integer数字的源：

```java
public String alphabet(int letterNumber) {
        if (letterNumber < 1 || letterNumber > 26) {
                return null;
        }
        int letterIndexAscii = 'A' + letterNumber - 1;
        return "" + (char) letterIndexAscii;
}
```



然后我们可以使用handle来删除null：



使用handle来进行map操作并消除null

```java
Flux<String> alphabet = Flux.just(-1, 30, 13, 9, 20)
    .handle((i, sink) -> {
        String letter = alphabet(i); ①
        if (letter != null) ②
            sink.next(letter); ③
    });

alphabet.subscribe(System.out::println);
```



①转成字母

②如果map函数(alphabet)返回null...

③通过不调用sink.next来过滤掉它



输出的结果

```text
M
I
T
```

