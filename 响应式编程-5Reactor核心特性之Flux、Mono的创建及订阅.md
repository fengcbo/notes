# 响应式编程-Reactor核心特性之Flux、Mono的创建及订阅

使用Flux和Mono的最简单的方法是使用它们各自的类中的工厂方法。

比如，要创建一个String序列，你可以直接枚举出所有的字符串或者把他们放入集合中，然后通过集合创建Flux，如下：

```java
Flux<String> seq1 = Flux.just("foo", "bar", "foobar");

List<String> iterable = Arrays.asList("foo", "bar", "foobar");
Flux<String> seq2 = Flux.fromIterable(iterable);
```

其他的工厂方法如下：

```java
Mono<String> noData = Mono.empty(); ①

Mono<String> data = Mono.just("foo");

Flux<Integer> numbersFromFiveToSeven = Flux.range(5, 3); ②
```

①泛型类型虽然没什么用，但是工厂方法依然总重泛型
②第一参数是范围的开始值，而第二个参数是生产元素的个数

Flux和Mono利用Java8 lambda表达式订阅。你可以选择不同的.subscribe()变体，这些subscribe方法使用lambda进行不同的组合来进行回调，如下面的方法签名所示：

```java
subscribe(); ①

subscribe(Consumer<? super T> consumer); ②

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer); ③

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer); ④

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer,
          Consumer<? super Subscription> subscriptionConsumer); ⑤
```

①订阅并触发序列
②对生产的每个元素进行处理
③处理元素并响应错误
④处理元素和错误，并在序列成功完成后执行某些操作
⑤处理元素、错误和成功完成，同时对subscribe调用产生的Subscription进行某些操作

>这些方法返回一个subscription的引用，当不再需要更多数据时，你可以使用这个引用取消订阅。取消订阅后，数据源需要停止生产数据并清理它创建的资源。在Reactor中取消和清理操作可以用Disposable接口表示。

## `subscribe`方法示例

本章包含了五个subscribe方法的最简单示例。下面的方法展示了一个无参的基础方法：

```java
Flux<Integer> ints = Flux.range(1, 3); ①
ints.subscribe(); ②
```
①初始化一个Flux，这个Flux在有订阅者连接时会产生三个元素
②最简单的方式订阅

上面的代买没有可视化的输出，但是它能很好的工作。Flux产生了三个值。如果提供一个lambda表达式，我们就可以打印这些值。下个subscribe示例展示了打印这些值的一种方式：

```java
Flux<Integer> ints = Flux.range(1, 3); ①
ints.subscribe(i -> System.out.println(i)); ②
```
①初始化一个Flux，这个Flux在有订阅者连接时会产生三个元素
②通过一个订阅者订阅，这个订阅者会打印这些值

上面的代码会有以下输出：

```text
1
2
3
```

为了展示下一个方法，我们会产生一个error，如下面的示例所示：

```java
Flux<Integer> ints = Flux.range(1, 4) ①
      .map(i -> { ②
        if (i <= 3) return i; ③
        throw new RuntimeException("Got to 4"); ④
      });
ints.subscribe(i -> System.out.println(i), ⑤
      error -> System.err.println("Error: " + error));
```

①初始化一个Flux，这个Flux在有订阅者连接时会产生四个元素
②我们需要映射，这样我们就可以处理不同的值
③大部分值直接返回
④对于某个值，返回一个错误
⑤通过一个订阅者订阅，这个订阅者拥有一个错误处理器

我们使用了两个lambda表达式：一个处理我们期望的内容，另一个处理错误。上面的代码会有以下输出：

```text
1
2
3
Error: java.lang.RuntimeException: Got to 4
```

下一个subscribe方法包含错误处理和完成时间处理器，如下面的示例所示：

```java
Flux<Integer> ints = Flux.range(1, 4); ①
ints.subscribe(i -> System.out.println(i),
    error -> System.err.println("Error " + error),
    () -> {System.out.println("Done");}); ②
```

①初始化一个Flux，这个Flux在有订阅者连接时会产生四个元素
②通过一个订阅者订阅，这个订阅者包含完成事件处理器

错误信号以及完成信号都是终止信号，并且是排他的(你不能同时得到两个信号)。为了使完成消费者正常工作，我们需要小心不能够触发错误。

完成匹配器只是一对空的括号，因为它对应着Runnable接口的run方法，run方法没有参数。上面的代码会有以下输出：

```text
1
2
3
4
Done
```










