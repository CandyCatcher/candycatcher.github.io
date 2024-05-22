# 内存与内存管理器的抽象

## ByteBuf结构

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240521235714434.png" alt="image-20240521235714434" style="zoom:50%;" />

在netty里面，有几个重要的指针：

1. readerIndex，0到readerindex这里就是不读的数据，也就是抛弃的数据；从readerIndex开始读数据。
2. writerIndex，readerIndex到writerIndex这里就是没有读的数据，也就是可读的内容；从writeIndex开始写数据。
3. capacity，writerIndex到capacity这段数据就是我们可以往里面写的空间。
4. maxCapacity，其实这里应该还有个maxCapacity（可以看做是在capacity后面），capacity到maxCapacity这里，是这个Byte还可以扩展的空间。 

## read、write、set方法

ByteBuf的read方法有超多种实现：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240521235916034.png" alt="image-20240521235916034" style="zoom: 50%;" />

同样的，write、set也有很多种方法的实现。

## mark和reset方法

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522000401453.png" alt="image-20240522000401453" style="zoom:50%;" />

markReaderIndex的作用为：标记此缓冲区中的当前 readerIndex。可以通过调用resetReaderIndex()将当前readerIndex重新定位到标记的readerIndex。 标记的readerIndex的初始值为0。

resetReaderIndex的作用为：将当前 readerIndex 重新定位为此缓冲区中标记的 readerIndex。

writer同理。

## AbstactByteBuf

先看ByteBuf的抽象实现类AbstractByteBuf，对buffer也就是ByteBuf的一个骨架式的实现。阅读源码多了，我们都知道这个套路，abstract一般就是对接口的骨架式实现。

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522004521532.png" alt="image-20240522004521532" style="zoom:50%;" />

前面介绍的那几个指针，都在这里体现出来了。

`readableBytes()`的实现：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522005017463.png" alt="image-20240522005017463" style="zoom:50%;" />

保证不越界，并且附上读指针。从这里我们开始留意一下，ByteBuf很多都是返回this指针，方便链式调用。

看`readByte()`:

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522005256387.png" alt="image-20240522005256387" style="zoom:50%;" />

`getByte(i)`方法里面，别的`readBool()`，`readInt()`等这些方法，都是一个道理。在这里，由于byte是一个字节，所以readerIndex是i+1；

如果是`readInt()`:

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522005556569.png" alt="image-20240522005556569" style="zoom:50%;" />

int 是个字节，那么就是加4。

至于下划线方法的实现，会委托给每种ByteBuf最终子类的实现，有各自的方法，这就关系到各个ByteBuf实现的区别了，后续的文章会继续讲解。（`readableBytes()`，`writableBytes()`，`maxWritableBytes()`，`markReaderIndex()`,`resetReaderIndex()`这些方法能够在这里实现是因为它们和底层的读写没有关系）

AbstractByteBuf通过对这些接口对外能够提供丰富的功能，这里是不是对抽象有了深一点的理解呢？抽象出接口，并实现一部分功能，具体的实现由方法里面的子类实现的方法去实现。

# ByteBuf分类

## Pooled和Unpooled

这两个的区别是使用的内存是从预先分配好的内存还是未分配的内存取，第一个从预先分配好的内存去取一段连续的内存，第二种调用系统api直接去申请一块内存。

在分配bytebuf的时候，会有一个分配器：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522215653908.png" alt="image-20240522215653908" style="zoom: 50%;" />

分配器的两个子类实现UnpooledByteBufAllocator和PooledByteBufAllocator就能够实现pool和unpool内存的分配。后面我们会继续深入。

说得更加直白一点：**pooled的ByteBuf基于内存池可以重复利用**。

## Unsafe和非Unsafe

（通过指针）直接操作对象的内存地址还是通过安全的方式操作内存（直接操作物理内存），如果是unsafe就可以拿到bytebuf在jvm的内存调用jdk的unsafe直接去操作（通过jdk的API）。

我们之前知道，AbstractByteBuf会有个最终去执行getByte的方法  `protected abstract byte _getByte(int index); `这些下划线开头的方法，最终都由他的子类去实现。我们先看一个子类的实现`UnpooledUnsafeHeapByteBuf`，他是一个unsafe的实现，看他实现的`_getByte()`方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522221430789.png" alt="image-20240522221430789" style="zoom:50%;" />

他是用的`UnsafeByteBufUtil`去实现的，我们后面会知道，所有的`unsafebytebuf`都会通过这个工具类去操作底层的unsafe，继续一直深入下去，就会看到是用底层的unsafe去实现的：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522221643673.png" alt="image-20240522221643673" style="zoom:50%;" />

然后我们看`UnpooledHeapByteBuf`的`_getByte()`的实现：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522222020918.png" alt="image-20240522222020918" style="zoom:50%;" />

他是通过HeapByteBufUtil去实现的，继续深入：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522222252206.png" alt="image-20240522222252206" style="zoom:50%;" />

对比他们两种实现，我们总结：**unsafe通过操作底层unsafe的offset+index的方式去操作数据，非unsafe直接通过一个数组的下标（或者jdk底层的buffer）去操作数据**。

unsafe需要依赖到jdk底层的unsafe对象，非unsafe不需要。

## Heap和Direct

操作jvm的堆内存还是直接内存，区别就是如果是直接内存不受jvm控制，所以也不会回收，需要自己去回收。

我们以`UnpooledHeapByteBuf`和`UnpooledDirectByteBuf`为例，介绍他们之间的区别。

首先看`UnpooledHeapByteBuf`源码

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522223829785.png" alt="image-20240522223829785" style="zoom:50%;" />

看到array，我们知道heap的话，它的数据就存放在array里面，所有内存相关的操作就在这个array上进行。上一部分我们知道，`_getByte`的时候，就是操作一个数组（通过数组下标的方式），这个数组就是这个array。

> 堆怎么实现的？

然后看UnpooledDirectByteBuf源码：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522224016967.png" alt="image-20240522224016967" style="zoom:50%;" />

他是通过一个ByteBuffer来存储数据，ByteBuffer是什么？

我们查看它的实现，能够看到有一个：`DirectByteBuffer`，这个是jdk的nio底层分配的buffer，用于操作堆外内存。

我们_getByte的时候，就是通过这个buffer去实现的：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522224238617.png" alt="image-20240522224238617" style="zoom:50%;" />

内存分配就是通过Unpooled实现的。

现在我们分析unpooled，我们找到这个父类,分配内存就是通过这个去实现的。

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522224847439.png" alt="image-20240522224847439" style="zoom:50%;" />

继续看ALLOC的AbstractByteBufAllocator的实现：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522225038814.png" alt="image-20240522225038814" style="zoom:50%;" />

newDirectBuffer方法有两个实现，UnpooledByteBufAllocator的newDirectBuffer方法的实现：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522225240644.png" alt="image-20240522225240644" style="zoom:50%;" />

> 我们看到是否unsafe不是我们自己决定的，而是由jdk实现的，如果能够获取到unsafe对象，就使用unsafe，反之亦然。

我们分析的是UnpooledDirectByteBuf，继续：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522225441694.png" alt="image-20240522225441694" style="zoom:50%;" />

我们这个看到一个allocateDirect(initialCapacity)，一层层进去，就能看到jdk的nio底层创建ByteBuffer的方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522225553563.png" alt="image-20240522225553563" style="zoom:50%;" />

然后通过setByteBuffer方法，把分配的内存设置到buffer变量里面去。

# 不同规格大小和不同类别的内存的分配策略

## 内存分配器ByteBufAllocator

### ByteBufAllocator功能

查看ByteBufAllocator源码：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522230301869.png" alt="image-20240522230301869" style="zoom:50%;" />

前面几个方法buffer和ioBuffer就是分配buffer或者ioBuffer。至于分配什么buffer，direct的还是heap的，由具体实现决定。

后面就有heapBuffer和directBuffer就是分配堆内内存和堆外内存了。既然如此，为什么还要上面的buffer方法？看到后面的AbstractByteBufAllocator就真相大白，其实先调用buffer方法，再在这个方法里面地调用heapBufffer和directBuffer。

> 这里我们先留意一下，netty如何区分heapBuffer还是directBuffer，就是通过这里区分的。第三点的时候回继续分析别的。 

最后的compositeBuffer那就是分配堆内内存和堆外内存都有的buffer了。

### AbstractByteBufAllocator

ByteBufAllocator的骨架式实现AbstractByteBufAllocator。

> 都是这个套路，先用一个抽象类进行骨架的实现

AbstractByteBufAllocator 暴露两个接口让子类去实现：newHeapBuffer、newDirectBuffer。可以看它实现的buffer方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522230906418.png" alt="image-20240522230906418" style="zoom:50%;" />

随便看一个directBuffer，一直进去：

![image-20240522231029496](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522231029496.png)

留给它的子类去实现。

同理，newHeapBuffer也留给它的子类去实现，它有两个子类：

PooledByteBufAllocator和UnpooledByteBufAllocator

* PooledByteBufAllocator相当于在预先分配好的内存里面取一段
* UnpooledByteBufAllocator相当于直接调用系统API去分配内存

### ByteBufAllocator两大子类

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522215653908.png" alt="image-20240522215653908" style="zoom: 50%;" />

`PooledByteBufAllocator`和`UnpooledByteBufAllocator`，我们现在知道:

1. 从上一节我们知道netty区分`unpooled`还是`pooled`的buffer是通过allocator的子类`PooledByteBufAllocator`和`UnpooledByteBufAllocator`去实现的。

2. 而通过上一小节我们知道，区分heap还是direct的buffer是通过内存分配器ByteBufAllocator自己的方法去实现的

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522235401329.png" alt="image-20240522235401329" style="zoom:50%;" />

3. 从上一节最后我们也知道是否unsafe是由jdk底层去实现的，如果能够获取到unsafe对象，就使用unsafe。

先从UnpooledByteBufAllocator上一段没有结束的newHeapBuffer进入，进入实现的源码：

![image-20240522235826809](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522235826809.png)

由于支持unsafe，从 new UnpooledUnsafeHeapByteBuf()进入：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523000020249.png" alt="image-20240523000020249" style="zoom:50%;" />

我们可以知道，heap数据都是存放在byte数组里面的

> `UnpooledUnsafeHeapByteBuf`的父类是`UnpooledHeapByteBuf`(而且我们可以知道`UnpooledUnsafeHeapByteBuf`是直接调用父类UnpooledHeapByteBuf的方法创建的，UnpooledUnsafeHeapByteBuf创建的时候相当于创建了一个UnpooledHeapByteBuf

可以看到在这里创建了数组：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523000901568.png" alt="image-20240523000901568" style="zoom:50%;" />

继续：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001029827.png" alt="image-20240523001029827" style="zoom:50%;" />

`setArray(initialArray)`:

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001124529.png" alt="image-20240523001124529" style="zoom:67%;" />

`setIndex(readerIndex,writerIndex)`:

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001211095.png" alt="image-20240523001211095" style="zoom:50%;" />

继续跟进去：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001255566.png" alt="image-20240523001255566" style="zoom:50%;" />

我们现在是unsafe的，那个刚刚在setArray方法设置进去的array就在_getByte(int index)的时候传入进去了：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001447812.png" alt="image-20240523001447812" style="zoom:50%;" />

再次理解一下，UnpooledHeapByteBuf和UnpooledUnsafeHeapByteBuf存储的时候，都是通过一个array。但是_getByte(array,index)的时候，UnpooledHeapByteBuf底层是直接操作array：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001447812.png" alt="image-20240523001447812" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001852542.png" alt="image-20240523001852542" style="zoom:50%;" />

UnpooledUnsafeHeapByteBuf是通过UNSAFE操作这个数组：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523002008091.png" alt="image-20240523002008091" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523002347982.png" alt="image-20240523002347982" style="zoom:50%;" />

# 内存的回收过程