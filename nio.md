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

> capacity：capacity代表Buffer包含的元素的数量，它是不可变的。

> position：position代表当前Buffer下一个可读或者可写的索引。

> limit：limit代表当前Buffer第一个不可读或不可写的索引。

> mark：mark是一个索引，在调用Buffer的reset的时候将position设为mark，mark不是总是被定义的，未定义为-1


五个重要方法有是：clear()，flip()，rewind()，mark()，reset()

> clear()：在从通道读数据或者往buffer写数据之前调用，这个方法会把position设为0，limit设为capacity，mark设为-1(即至为失效)。clear方法不会真正清空buffer的数据，但它通常在清空数据的场景下使用。

> flip()：翻转buffer。在往通道写数据或者从buffer读数据之前调用，这个方法会把limit设为position，然后把position设为0，mark设为-1(即至为失效)。

> rewind(): 重读(倒带)缓冲区。在往通道写数据或者从buffer读数据前调用，这个方法把position设为0，mark设为-1(即至为失效)，limit不改变。相当于重读缓冲区的数据。

> mark()：在buffer的当前position设置标记。即将mark设置为position，与reset()搭配使用

> reset()：重置buffer的position为上次标记(mark)的值。将position设置为mark，即重读标记后的数据。

buffer的get和put方法分为相对方法和绝对方法：
1. 相对方法：limit值与positioin值会在操作时被考虑到
2. 绝对方法：完全忽略limit和position的值

### ByteBuffer
创建ByteBuffer的方法如下：

```
ByteBuffer buffer = ByteBuffer.allocate(1024);
```

allocate()方法需要一个capacity参数，返回的类型是：java.nio.HeapByteBuffer

java.nio.HeapByteBuffer是ByteBuffer的子类，它是非直接缓冲，和它的名字意思类似，它是在JVM的堆上分配内存。

ByteBuffer有五个子类：java.nio.HeapByteBuffer、java.nio.HeapByteBufferR、java.nio.MappedByteBuffer、java.nio.DirectByteBuffer、java.nio.DirectByteBufferR

其中java.nio.HeapByteBufferR继承自java.nio.HeapByteBuffer，它是个只读缓冲；
java.nio.MappedByteBuffer直接继承自ByteBuffer；java.nio.DirectByteBuffer继承自java.nio.MappedByteBuffer，它是直接缓冲；java.nio.DirectByteBufferR继承自java.nio.DirectByteBuffer，它是直接缓冲并且是只读的。

DirectByteBuffer在java中称为直接缓冲，它的内容是存储在JVM堆内存之外的，是由操作系统来管理的。而HeapByteBuffer在Java中称为非直接缓冲，它的内容存储在堆内存中；当HeapByteBuffer从Channel中读取数据时，会先把数据从Channel copy到操作系统，然后再把数据从操作系统的缓冲copy到堆缓冲，这里有两次数据copy，而对于直接缓冲只会发生第一次的copy。这里还涉及到一个概念：zero-copy

zero-copy见名知意：零拷贝。其实zero-copy所谓的零拷贝是相对操作系统内核来说的，并没真正实现零拷贝。通常数据从存储设备到应用程序涉及到了内核态和应用态的转换。当读取数据时，数据会先被copy到操作系统内核，这里数据到了内核态；然后内核态的数据会被copy到应用程序，这样数据就到了用户态，这里发生了一次上下文切换，发生了两次copy。而zero-copy并不会发生内核态到用户态的copy和上下文切换，程序只需要通过句柄直接操作内核态数据。数据从

## Channel

通过NIO读取文件的三个步骤：

1. 从FileInputStream获取到FileChannel对象
2. 创建Buffer
3. 将数据从Channel写到Buffer中

```
FileInputStream fis = new FileInputStream("input.txt");
FileOutputStream fos = new FileOutputStream("output.txt");

FileChannel inputChannel = fis.getChannel();
FileChannel outputChannel = fos.getChannel();

ByteBuffer buffer = ByteBuffer.allocate(1024);
while (true){
    // limit=capacity position=0  => limit=10 position=0
    buffer.clear();
    // limit=capacity position=21
    int read = inputChannel.read(buffer);
    System.out.println(read);
    if (-1 == read) {
        break;
    }
    // limit=position=21 position=0
    buffer.flip();

    // limit=21 position=21
    outputChannel.write(buffer);
}

inputChannel.close();
outputChannel.close();
```

