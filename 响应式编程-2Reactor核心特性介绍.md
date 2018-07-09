# 响应式编程-Reactor核心特性介绍

Reactor的核心模块是`reactor-core`，它是基于Java8并聚焦于Reactive Streams规范。

Reactor引入了复合的响应式类型并实现了Reactive Streams，同时还提供了丰富的操作符，尤其是Flux和Mono。Flux代表拥有0...n个元素的响应式序列，而Mono代表拥有一个或者空(0..1)的值。

这种区别在类型中包含一些语义信息，用于指示异步处理的粗略基数。例如，HTTP请求只产生一个响应，因此进行计数操作没有多大意义。将HTTP调用的结果表示为Mono<HttpResponse>比Flux<HttpResponse>更有意义，因为它只提供与零项或一项内容相关的操作符。

操作符能够更改最大的处理基数同时将其转化成相应的类型。例如，count操作符存在于Flux中，但它返回Mono<Long>。