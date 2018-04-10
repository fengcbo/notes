# NIO

## 简介

java.io 使用阻塞的方式处理输入输出

```
java.io中的最核心的概念是流(Stream)，面向流(分为输入流和输出流)的编程。java中一个流要么是输出流要不是输入流，不能既是输出流又是输入流。
```

java.nio JDK1.4后推出的，建议以非阻塞的方式处理输入输出。

```
java.nio中有三个核心概念：Selector、Channel与Buffer。在java.nio中，我们是面向块(block)或者缓冲区(Buffer)来编程。

一个Selector关联一个Thread，每个Selector会监听多个Channel，每个Channel会对应一个Buffer(缓冲)。每个线程对应一个Selelctor，线程操作哪个Channel是通过Selector的事件来控制的。

Buffer就是一块内存，java底层实质就是一个数组，数据的读和写都是通过Buffer来操作的。同一个Buffer既可以用来读也可以用来写。除了数据之外，Buffer还提供了对于数据的结构化访问方式，并且可以追踪到系统的读写过程。java中的7中原生数据类型都有对应的Buffer类型，如IntBuffer, LongBuffer, ByteBuffer, CharBuffer，但是没有boolean类型的。

Channel指的是可以向其写入或读取的对象，类似java.io中年的Stream，与Stream不用的是，所有的数据的读写都是用过Buffer来进行的，永远不会出现直接向Channel中写入或读取数据的情况，而且Channel是双向的。由于Channel是双向的，所以它能更好的反应出操作系统的真实情况，在Linux系统中，底层操作系统的通道就是双向的。
```

## Buffer

Buffer包含四个重要属性和五个重要方法

四个属性分别是：capacity，position，limit，mark

capacity：capacity代表Buffer包含的元素的数量，它是不可变的。
position：position代表当前Buffer下一个可读或者可写的索引。
limit：limit代表当前Buffer第一个不可读或不可写的索引。
mark：mark是一个索引，在调用Buffer的reset的时候将position设为mark，mark不是总是被定义的，未定义为-1




