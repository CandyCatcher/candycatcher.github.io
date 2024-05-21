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

## Unsafe和非Unsafe

## Heap和Direct

# 不同规格大小和不同类别的内存的分配策略

# 内存的回收过程