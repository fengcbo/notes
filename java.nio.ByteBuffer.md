# ByteBuffer

## class doc

ByteBuffer定义了六种类型的操作：

> 读写单个字节的相对或绝对的get和put方法
> 将buffer的连续字节序列转移到一个数组里面的相对的块get操作
> 将一个数组或其他类型的连续字节序列转移到当前buffer的相对的块put操作
> 绝对和相对get和put方法，它们读取和写入其他基本类型的值，将它们转换成特定字节顺序的字节序列
> 创建视图缓冲区的方法，它允许将字节缓冲区视为包含其他原始类型值的缓冲区;
> 用于压缩、复制和分割字节缓冲区的方法。

可以通过为buffer的内容申请空间的allocation方法 或者 通过包装一个已存在buffer的wrapping方法 创建Byte缓冲区。

### 直接和非直接缓冲区

缓冲区可以是直接缓冲，也可以是非直接缓冲。对于给定的直接缓冲，Java虚拟机将尽最大努力直接执行本机I/O操作。也就是说，它将尝试在每次调用底层操作系统的本地I/O操作时，避免将缓冲区的内容复制到(或从)一个中间缓冲区中。

直接缓冲可以通过 allocateDirect工厂方法来创建。与非直接缓冲区相比，这个方法返回的缓冲区通常会有更高的申请和回收消耗。直接缓冲区的内容可能驻留在正常的垃圾收集堆之外，因此它们对应用程序内存占用的影响可能不太明显。因此，建议将直接缓冲区主要分配给那些被底层系统的本地I/O操作所控制的大型、长寿命的缓冲区。通常，当他们能产生可预测的程序性能增幅时，最好申请直接缓冲。

buffer也可以通过mapping来创建，它直接把文件的一部分映射到内存。Java平台的实现可以选择支持通过JNI从本机代码创建直接字节缓冲。如果这种类型的buffer示例被映射到不可访问的内存区域，并尝试访问这块区域，这并不会改变buffer的内容，并会导致未知的异常在访问时或访问后被抛出。

buffer是直接缓冲还是非直接缓冲由isDirect方法来决定。提供了这种方法，以便在性能关键代码中实现显式的缓冲区管理。

### 访问二进制数据

ByteBuffer提供了读写除了boolean之外的其他原生类型的方法。原始值根据缓冲区的当前字节顺序被转换为(或从)字节序列，这些字节顺序可以通过order方法检索和修改。

为了访问不同类型的二进制数据，ByteBuffer为每种类型定义了一组绝对和相对的get和put方法。比如，ByteBuffer为32位的float定义了如下方法：

```
float  getFloat()
float  getFloat(int index)
void  putFloat(float f)
void  putFloat(int index, float f)
```

ByteBuffer为char、short、int、long和double 也定义了一直的方法。绝对的get和put方法的index参数依据bytes而不是被读写的类型。

为了访问同类的二进制数据，即相同类型的值序列，该类定义了可以创建给定字节缓冲区的视图的方法。缓冲的视图简单来书是另一个缓冲，它的内容依托于byte buffer。改变byte buffer的内容将反映到缓冲视图，反之亦然；这两个buffer的position、limit和mark是相互独立的。例如，asFloatBuffer方法创建一个FloatBuffer类的实例，该实例由调用该方法的字节缓冲区支持。为char、short、int、long和double类型定义了相应的视图创建方法。

视图缓冲区对于特定于类型的get和put方法有三个重要的优点:

- 视图缓冲区的索引不是以字节为单位的，而是根据其值的类型特定大小来进行索引的;
- 视图缓冲区提供了相对批量get和put方法，这些方法可以在缓冲区和数组或同一类型的其他缓冲区之间传递相邻的值序列
- 视图缓冲区可能更高效，因为它将是直接的，当且仅当它的支持字节缓冲区是直接的情况下。

The byte order of a view buffer is fixed to be that of its byte buffer at the time that the view is created.

### 调用链

这个类中没有返回值的方法会返回当前buffer的调用者。这就允许方法调用可以是链式的。比如，代码片段

```
bb.putInt(0xCAFEBABE);
bb.putShort(3);
bb.putShort(45);
```
可以被替换成单行的语句：

```
bb.putInt(0xCAFEBABE).putShort(3).putShort(45);
```
