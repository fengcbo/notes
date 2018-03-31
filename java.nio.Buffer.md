# NIO Buffer

## class doc

Buffer是原生类型数据的容器

Buffer是基于特定原生数据类型的线性的, 有序序列。除了它的内容之外，Buffer的基本属性有capacity，limit和position：

> Buffer的capacity代表它所包含的元素数量。Buffer的capacity是非负的, 不可变的。

> Buffer的limit是第一个不可读或写的元素的索引。Buffer的limit是非负的并且永远不会大于它的capacity。

> Buffer的position是下一个读或写的元素索引。Buffer的position是非负的并且永远不会大于它的limit。

除了boolean类型，每一个原生类型都有一个Buffer的子类。

### 数据传输

Buffer类的每一个子类都定义了两种类型的get和put操作：

```
读写一个或多个元素的相关操作从当前的position开始，然后会 postion 会增加传输元素的个数。如果请求转移超过了limit，则相对的get操作抛出 BufferUnderflowException，相对的put操作抛出 BufferOverflowException；不管哪种情况，都没有数据传输。
绝对操作采用显示元素索引的方式，不会影响position。如果索引参数超过了limit，则绝对get和put操作会抛出IndexOutOfBoundsException。
当然，数据也可以通过一个适当的通道的I/O操作被转移到缓冲区中，这是相对于当前位置的。
```

### 标记(Marking)和重置(resetting)

Buffer的标记(mark)是一个索引，在reset方法被执行时会被重置。标记(mark)不会总被定义，但是当它被定义了，它就是非负的并且不会大于position。如果标记(mark)被定义了，当position或者limit调整后小于了标记(mark)，则标记(mark)会被抛弃。如果标记(mark)没有被定义，则执行reset方法会抛出 InvalidMarkException。

### 不可变性(Invariants)

下面的不可变性适用于mark, position, limit, capacity:

> 0 <= mark <= position <= limit <= capacity

一个新创建的Buffer总是会有值为0的position和为定义的标记(mark)。初始的limit可能为0，或者其他值，这依赖于buffer的类型以及它所创建的方式。新建的buffer的每个元素初始值均为0。


### Clearing(清理), flipping(翻转), and rewinding(重置)

除了访问position, limit, capacity和重置标记(mark)的方法外，这个类还定义了一下的操作：

- clear() 使缓冲区为新的读通道或相对put操作做好准备：它设置limit为capacity，设置position为0。
- flip() 使缓冲区为新的写通道或相对get操作做好准备：它设置limit为当前position，设置position为0。
- rewind() 使缓冲区为重新读取它包含的数据做好准备：它保持limit不变，并将position设置为0。

### 只读Buffer

每一个Buffer都可以读，但不是所有的Buffer都可以写。每个缓冲类的突变方法被指定为可选操作，当在只读缓冲区上调用时将抛出ReadOnlyBufferException。一个只读的buffer不允许它的内容改变，但是它的mark, position和limit是可变的。一个buffer是否是只读的取决于它的isReadOnly方法。

### 线程安全

Buffers不够能被多线程安全的使用。如果一个buffer需要被超过一个线程使用，就需要做相应的同步控制。

### 调用链
除了需要返回值的方法，在本类的方法将会返回执行它们的buffer对象。这就允许方法的链式执行；如下面的代码片段：

```
b.flip();
b.position(23);
b.limit(42);
```

就可以替换为一个简洁语句

```
b.flip().position(23).limit(42);
```
