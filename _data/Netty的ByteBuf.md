netty的内存类别有哪些？堆内内存 堆外内存

如何保证多线程内存分配的安全性？

不同大小的内存分配策略是不同的？

# 内存与内存管理器的抽象

## ByteBuf结构

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240521235714434.png" alt="image-20240521235714434" style="zoom:50%;" />

在netty里面，有几个重要的指针：

1. readerIndex，<u>0到readerindex这里就是不读的数据，也就是抛弃的数据</u>；**从readerIndex开始读数据。**
2. writerIndex，<u>readerIndex到writerIndex这里就是没有读的数据，也就是可读的内容</u>；**从writeIndex开始写数据。**
3. capacity，<u>writerIndex到capacity这段数据就是我们可以往里面写的空间</u>。
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

# ByteBuf分类



## AbstactByteBuf

先看ByteBuf的抽象实现类AbstractByteBuf，对buffer也就是ByteBuf的一个骨架式的实现。阅读源码多了，我们都知道这个套路，abstract一般就是对接口的骨架式实现。

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522004521532.png" alt="image-20240522004521532" style="zoom:50%;" />

前面介绍的那几个指针，都在这里体现出来了。

`readableBytes()`的实现：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522005017463.png" alt="image-20240522005017463" style="zoom:50%;" />

保证不越界，并且附上读指针。从这里我们开始留意一下，ByteBuf很多都是返回this指针，方便链式调用。

看`readByte()`:

它的操作就是先获取当前的写指针并mark，获取当前的字节，然后将当前的指针+1。

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522005256387.png" alt="image-20240522005256387" style="zoom:50%;" />

`_getByte(i)`方法是一个抽象方法 。别的`readBool()`，`readInt()`等这些方法，都是一个道理。在这里，由于byte是一个字节，所以readerIndex是i+1；

如果是`readInt()`:

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522005556569.png" alt="image-20240522005556569" style="zoom:50%;" />

int 是个字节，那么就是加4。

至于下划线方法的实现，会委托给每种ByteBuf最终子类的实现，有各自的方法，这就关系到各个ByteBuf实现的区别了，后续的文章会继续讲解。（`readableBytes()`，`writableBytes()`，`maxWritableBytes()`，`markReaderIndex()`,`resetReaderIndex()`这些方法能够在这里实现是因为它们和底层的读写没有关系）

AbstractByteBuf通过对这些接口对外能够提供丰富的功能，这里是不是对抽象有了深一点的理解呢？抽象出接口，并实现一部分功能，具体的实现由方法里面的子类实现的方法去实现。

## Pooled和Unpooled（池化 非池化）

这两个的区别是使用的内存是从预先分配好的内存还是未分配的内存取，**第一个从预先分配好的内存去取一段连续的内存封装成byteBuffer**，**第二种调用系统api直接去申请一块内存**。

在分配bytebuf的时候，会有一个分配器：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522215653908.png" alt="image-20240522215653908" style="zoom: 50%;" />

分配器的两个子类实现UnpooledByteBufAllocator和PooledByteBufAllocator就能够实现pool和unpool内存的分配。后面我们会继续深入。

 说得更加直白一点：**pooled的ByteBuf基于内存池可以重复利用**。

## Unsafe和非Unsafe

（通过指针）直接操作对象的内存地址还是通过安全的方式操作内存（直接操作物理内存），**如果是unsafe就可以拿到bytebuf在jvm的内存调用jdk的unsafe直接去操作（通过jdk的API）**。

我们之前知道，AbstractByteBuf会有个最终去执行getByte的方法  `protected abstract byte _getByte(int index); `这些下划线开头的方法，最终都由他的子类去实现。我们先看一个子类的实现`PooledUnsafeHeapByteBuf`，他**是一个unsafe的实现，看他实现的`_getByte()`方法**：

![image-20240610235952439](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240610235952439.png)

他是通过内存地址去拿到的，用的`UnsafeByteBufUtil`去实现的，我们后面会知道，所有的`unsafebytebuf`都会通过这个工具类去操作底层的unsafe，继续一直深入下去，就会看到是用底层的unsafe去实现的：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522221643673.png" alt="image-20240522221643673" style="zoom:50%;" />

然后我们看`UnpooledHeapByteBuf`的`_getByte()`的实现，也就是**非unSafe的实现**：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522222020918.png" alt="image-20240522222020918" style="zoom:50%;" />

他是通过HeapByteBufUtil去实现的，继续深入：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522222252206.png" alt="image-20240522222252206" style="zoom:50%;" />

对比他们两种实现，我们总结：**unsafe通过操作底层unsafe的offset+index的方式去操作数据，非unsafe直接通过一个数组的下标（或者jdk底层的buffer）去操作数据**。

 

## Heap和Direct

**操作jvm的堆内存还是直接内存**，区别就是如果是直接内存不受jvm控制，所以也不会回收，需要自己去回收。

我们以`UnpooledHeapByteBuf`和`UnpooledDirectByteBuf`为例，介绍他们之间的区别。

首先看`UnpooledHeapByteBuf`源码

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522223829785.png" alt="image-20240522223829785" style="zoom:50%;" />

看到array，我们知道heap的话，它的数据就存放在array里面，所有内存相关的操作就在这个array上进行。上一部分我们知道，`_getByte`的时候，就是操作一个数组（通过数组下标的方式），这个数组就是这个array。

> 堆怎么实现的？

然后看UnpooledDirectByteBuf源码：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522224016967.png" alt="image-20240522224016967" style="zoom:50%;" />

他是通过一个ByteBuffer（jdk的byteBuffer）来存储数据，ByteBuffer是什么？

为什么是DirectByteBuffer呢？怎么赋值的呢？

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

前面几个方法buffer和ioBuffer就是分配buffer，<u>至于分配什么buffer，direct的还是heap的，由具体实现决定；</u>或者ioBuffer。

后面就有heapBuffer和directBuffer就是分配堆内内存和堆外内存了。既然如此，为什么还要上面的buffer方法？看到后面的AbstractByteBufAllocator就真相大白，其实先调用buffer方法，再在这个方法里面地调用heapBufffer和directBuffer。

> 这里我们先留意一下，netty如何区分heapBuffer还是directBuffer，就是通过这里区分的。第三点的时候回继续分析别的。 

最后的compositeBuffer那就是分配堆内内存和堆外内存都有的buffer，用的不多。

### AbstractByteBufAllocator

ByteBufAllocator的骨架式实现AbstractByteBufAllocator。

> 都是这个套路，先用一个抽象类进行骨架的实现

AbstractByteBufAllocator 暴露两个接口让子类去实现：newHeapBuffer、newDirectBuffer。可以看它实现的buffer方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522230906418.png" alt="image-20240522230906418" style="zoom:50%;" />

随便看一个**directBuffer**，一直进去：

![image-20240522231029496](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522231029496.png)

实际去new一个**directBuffer**留给它的子类去实现。同理，new一个**HeapBuffer**也留给它的子类去实现。

实际上它有两个子类：

**PooledByteBufAllocator**和**UnpooledByteBufAllocator**

* PooledByteBufAllocator相当于在预先分配好的内存里面取一段
* UnpooledByteBufAllocator相当于直接调用系统API去分配内存

### ByteBufAllocator两大子类

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522215653908.png" alt="image-20240522215653908" style="zoom: 50%;" />

`PooledByteBufAllocator`和`UnpooledByteBufAllocator`，我们现在知道:

1. 从上一节我们知道netty区分`unpooled`还是`pooled`的buffer是通过allocator的子类`PooledByteBufAllocator`和`UnpooledByteBufAllocator`去实现的。

2. 而通过上一小节我们知道，区分heap还是direct的buffer是通过内存分配器ByteBufAllocator自己的方法去实现的

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522235401329.png" alt="image-20240522235401329" style="zoom:50%;" />

3. 从上一节最后我们也知道是否unsafe是由jdk底层去实现的，如果能够获取到unsafe对象，就使用unsafe。

   ![image-20240522235826809](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimgimage-20240522235826809.png)



## 内存分配器UnpooledByteBufAllocator

### heap内存的分配

上一章其实在最后一部分我们分析过了，heap的的内存分配和读取都是在array数组上面。分配的堆内存源码就是下面UnpooledByteBufAllocator的这一段：

先从UnpooledByteBufAllocator上一段没有结束的newHeapBuffer进入，进入实现的源码：

![image-20240522235826809](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240522235826809.png)

由于支持unsafe，从 new UnpooledUnsafeHeapByteBuf()进入，直接调用的是父类，父类默认是一个unsafe的：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523000020249.png" alt="image-20240523000020249" style="zoom:50%;" />

我们可以知道，heap数据都是存放在byte数组里面的

> `UnpooledUnsafeHeapByteBuf`的父类是`UnpooledHeapByteBuf`(而且我们可以知道`UnpooledUnsafeHeapByteBuf`是直接调用父类UnpooledHeapByteBuf的方法创建的，UnpooledUnsafeHeapByteBuf创建的时候相当于创建了一个UnpooledHeapByteBuf

可以看到在这里创建了数组：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523000901568.png" alt="image-20240523000901568" style="zoom:50%;" />

继续，下面就是一些初始化的操作了：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001029827.png" alt="image-20240523001029827" style="zoom:50%;" />

`setArray(initialArray)`:

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001124529.png" alt="image-20240523001124529" style="zoom:67%;" />

`setIndex(readerIndex,writerIndex)`:

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001211095.png" alt="image-20240523001211095" style="zoom:50%;" />

继续跟进去：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001255566.png" alt="image-20240523001255566" style="zoom:50%;" />

我们现在是unsafe的，那个刚刚在setArray方法设置进去的array就在_getByte(int index)的时候传入进去了：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001447812.png" alt="image-20240523001447812" style="zoom:50%;" />

先在问题来了，unsafe初始化构造函数的时候，直接调用的是父类的构造函数，那UnpooledHeapByteBuf和UnpooledUnsafeHeapByteBuf有啥区别呢？

UnpooledHeapByteBuf和UnpooledUnsafeHeapByteBuf存储的时候，都是通过一个array。但是_getByte(array,index)的时候，UnpooledHeapByteBuf底层是直接操作array：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001447812.png" alt="image-20240523001447812" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523001852542.png" alt="image-20240523001852542" style="zoom:50%;" />

UnpooledUnsafeHeapByteBuf是通过UNSAFE操作这个数组：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523002008091.png" alt="image-20240523002008091" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240523002347982.png" alt="image-20240523002347982" style="zoom:50%;" />

### direct内存的分配

和heap不同，direct内存分配放在buffer里面，然后unsafe的direct buffer是通过底层的unsafe类去操作的。（其实看源码到最终，direct的buffer都是通过unsafe去操作的，也就是jdk底层的allocateDirect最终还是调用到unsafe，这里）

从`UnpooledByteBufAllocator`分配`direct`内存的这个源码开始看起：

![image-20240526001945223](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240526001945223.png)

先从unsafe进去；

![image-20240526002514497](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240526002514497.png)

不管nocleaner先，从最后UnpooledUnsafeDirectByteBuf这个函数进去：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240526002615713.png" alt="image-20240526002615713" style="zoom:50%;" />

最后这个allocateDirect(initialCapacity):

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240526002946765.png" alt="image-20240526002946765" style="zoom:50%;" />

这里是**调用jdk底层去创建directbytebuffer：**

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240526003131585.png" alt="image-20240526003131585" style="zoom:50%;" />

回去setByteBuffer：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimgimage-20240526003259658.png" alt="image-20240526003259658" style="zoom:50%;" />

把刚刚调用allocateDirect分配的buffer赋值到this.buffer里面去。

注意memoryAddress，从这PlatformDependent.directBufferAddress(buffer);一直进去：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240526004216159.png" alt="image-20240526004216159" style="zoom: 67%;" />

也就是，我们获取到内存地址，加上offset，就是我们能用的开始的地址（通过buffer算出内存地址保存到memoryAddress，这样就可以知道buffer在内存里面的地址是多少了），getLong继续就是：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240526004315265.png" alt="image-20240526004315265" style="zoom:67%;" />

调用了jdk底层unsafe加上offset为我们获取我们能用的direct地址，最终回来保存到memoryAddress里面。

memoryAddress有什么用呢？回到UnpooledUnsafeDirectByteBuf的_getByte方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240526005216454.png" alt="image-20240526005216454" style="zoom:67%;" />

看addr(index):

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240526005242491.png" alt="image-20240526005242491" style="zoom:67%;" />

这个memoryAddress就是上面那个，也就是通过memoryAddress就能操作unsafedirect buffer内存了（通过memoryAddress+index当做内存地址）。 

好，这就是UnpooledByteBufAllocator分配和操作direct unsafe内存的分析了。

 看非unsafe，从newDirectBuffer的UnpooledDirectByteBuf方法进入：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240526005954838.png" alt="image-20240526005954838" style="zoom: 50%;" />

从setByteBuffer进入：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240526010026669.png" alt="image-20240526010026669" style="zoom:50%;" />

可以看到没有memoryAddress了，只有buffer。在UnpooledDirectByteBuf的_getByte方法里面，就是直接读取buffer里面的内容了：

![](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimgimage-20240526010218234.png)

再总结一下，

UnpooledHeapByteBuf和UnpooledUnsafeHeapByteBuf存储的时候，都是通过一个array。但是_getByte(array,index)的时候，UnpooledHeapByteBuf底层是直接操作array；而 UnpooledUnsafeHeapByteBuf是通过UNSAFE操作这个数组。

`UnpooledDirectByteBuf`和`UnpooledUnsafeDirectByteBuf`存储的时候，都通过了一个buffer存储（而且UnpooledUnsafeDirectByteBuf还会计算这个buffer在内存的地址memoryAddress，以后通过操作这个memoryAddress+index操作buffer）。_getByte(index)的时候UnpooledDirectByteBuf通过操作这个buffer操作数据；而UnpooledUnsafeDirectByteBuf通过操作unsafe操作index'（这个index'是memoryAddress+index得来的）来操作数据。（但是其实操作jdk底层buffer的时候都是通过unsafe操作的，所以direct的数据最终都是通过unsafe操作的）

## 内存分配器pooledByteBufAllocator

pooledByteBufAllocator相对于复杂一些。

通过他实现内存分配的两个抽象方法来看：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527224027239.png" alt="image-20240527224027239" style="zoom: 50%;" />

这两个方法大体结构上是差不多的，所以就拿directBuffer进行分析。

首先他是通过`threadCache.get()`拿到了PoolThreadCache这个对象。然后拿到这个对象的directArena对象，最后通过directArena对象的allocate方法分配内存。

实际上pooledByteBufAllocator进行内存分配就两个步骤：

1. 拿到线程局部缓存的PoolThreadCache。因为newDirectBuffer可能会多线程调用，这里通过threadCache拿到当前线程的cache进行分配。而这里的threadCache就是PoolThreadLocalCach。
2. 在线程局部缓存的Area上进行内存分配。

### PoolThreadLocalCache

首先看看`threadCache`的定义`PoolThreadLocalCache`：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527211045806.png" alt="image-20240527211045806" style="zoom: 50%;" />

它是 继承自`FastThreadLocal`，`FastThreadLocal`我们可以简单的理解为`ThreadLocal`的快速版。在调用`initialValue`方法的时候，会创建`heapArena`和`directArena`，并且通过这两个对象返回一个`PoolThreadCache`供一个线程使用。(在`PoolThreadLocalCache`第一次调用get的时候，没有获取到对象，就会调用`initialValue`方法产生`PoolThreadCache`)

继续看`PoolThreadCache`的构造函数：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527212123148.png" alt="image-20240527212123148" style="zoom:50%;" />

`PoolThreadCache`的`heapArena`是`PoolThreadLocalCache`创建它的时候传入进去的`heapArena`。

那我们再看看`PoolThreadLocalCache`的`heapArena`是从哪里来:

heapArena是一个数组：

![image-20240527213024532](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527213024532.png)

并且是在PooledByteBufAllocator的构造函数进行初始化的：

![image-20240527213143053](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527213143053.png)

也就是在创建PooledByteBufAllocator的时候，就创建了这个heapArenas，通过newArenaArray创建数组：

![image-20240527213544354](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527213544354.png)

> PoolArena是啥？

nHeapArena是定义这个数组的大小，找到调用它的方法：

![image-20240527213823402](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimgimage-20240527213823402.png)

一直往上找构造函数：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527213926108.png" alt="image-20240527213926108" style="zoom:50%;" />

这个DEFAULT_NUM_HEAP_ARENA就是我们要找的东西，找到定义：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527214008787.png" alt="image-20240527214008787" style="zoom:50%;" />

如果没有定义io.netty.allocator.numHeapArenas这个属性，那么就取`defaultMinNumArena和runtime.maxMemory() / defaultChunkSize / 2 / 3`的最小值，一般是defaultMinNumArena，也就是**CPU核心线程的2倍**。

需要注意的是，就是默认情况下，NioEventloop的线程数量，和arena的数量是相同的，可以保证每一个线程都有一个独享的arena，对于每一个arena，分配内存的时候是不用加锁的 。

> threadLocalCache和Arena?

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527225945659.png" alt="image-20240527225945659" style="zoom: 33%;" />

然后我们总结一下，一个PooledByteBufAllocator创建的时候，会创建两个Arean（一个heap和一个direct）数组。一个PooledByteBufAllocator里面会有一个PoolThreadLocalCache，在获取bytebuffer的时候，这个PoolThreadCache就是通过ThreadLocal的方式将内存分配器其中的一个Area塞到成员变量里面 ，当线程通过他拿到他底层的Area。这样就把线程和Area进行绑定。

poolThreadCache除了在arena进行分配内存，还可以在poolThreadCache维护的byteBuffer缓存列表里面进行分配内存。举一个例子，当我们用pooledByteBufAllocator创建了一个1024个字节的byteBuffer， 用完了释放之后我们有可能在别的地方去继续分配1024个大小的内存，这个时候其实不需要进行area分配，而是直接在PoolThreadCache维护的buffer缓存列表分配。buffer里面，根据分配内容的大小，分为tinyCacheSize、smallCacheSize、 normalCacheSize（存的值为能缓存多少个）。我们从PooledByteBufAllocator的成员变量开始看起：

![image-20240527214517586](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527214517586.png)

这写成员变量，通过PoolThreadLocalCache的initalValue方法传入，PoolThreadCache类的构造器，创建初始值：

![image-20240527214757896](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527214757896.png)

那这个buffer缓存列表是在什么地方创建的：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527231415749.png" alt="image-20240527231415749" style="zoom:50%;" />

点击进入createSubPageCaches：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527231515990.png" alt="image-20240527231515990" style="zoom:50%;" />

### directArena分配direct内存

directArena分配direct内存的流程分为以下几个流程：

1. 从对象池里面拿到PooledByteBuf进行复用
2. 从缓存上面进行分配
3. 从内存堆里面进行内存分配（如果在缓存上分配不成功，就在内存堆里面分配）

从上面的newDirectBuffer继续分析allocate方法

![image-20240527233923483](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527233923483.png)

点进allocate方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527234052330.png" alt="image-20240527234052330" style="zoom: 67%;" />

#### 从对象池里面拿到PooledByteBuf进行复用

找到DirectArena的newByteBuf实现：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527234213577.png" alt="image-20240527234213577" style="zoom: 67%;" />

一般都是支持unsafe的所以进入`PooledUnsafeDirectByteBuf.newInstance(maxCapacity);`

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240527234411161.png" alt="image-20240527234411161" style="zoom: 67%;" />

第一行代码的意思是，如果对象池里面有这个buffer就直接拿出来否则的话就new一个buffer

![image-20240528224648398](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240528224648398.png)

这个newObject方法的意 思就是当回收站没有`PooledUnsafeDirectByteBuf`，就会新建一个`PooledUnsafeDirectByteBuf`。这个方法入参handler的作用就是负责`PooledUnsafeDirectByteBuf`的回收。点进去看这个接口：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240528225617036.png" alt="image-20240528225617036" style="zoom:50%;" />

只有这一个方法进行回收对象。

> 这个对象Recycler可以研究一下

拿到bytebuffer了，就对它进行复用：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240528230302295.png" alt="image-20240528230302295" style="zoom:50%;" />

这个方法就是设置byteBuffer的一些变量：重新设置最大值、设置byteBuffer的引用数量（默认为1）、设置index为默认值、从回收站里面拿出来的与标记相关的值进行重置（discardMarks）。

看一下discardMarks：

![image-20240528230856530](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240528230856530.png)

这样我们就能拿到一个纯净的byteBuffer了。

#### 从缓存上进行内存分配

进入allocate方法：

![image-20240528231513243](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240528231513243.png)

可以看到不管进入哪个分支，都会先在缓存上进行内存分配，如果分配成功直接return，没有return说明没有分配成功，需要进行实际的内存分配。

有一个特例，如果大于chunkSize，会分配一个特殊的：

![image-20240528231834677](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240528231834677.png)

#### 内存规格

netty中的内存规格分为这几种：

![image-20240528232457147](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240528232457147.png)

我们向操作系统请求分配内存都是以Chunk大小申请的，而Chunk的大小为16M。比如我们申请一个1M的内存，先去操作系统申请一个16M的chunk，然后再在这个chunk分配1M的大小放在byteBuffer里面。

8K是一个page，netty会将chunk进行8K大小进行分配。所以一个chunk一共有16m/8k=2的11次方个page。

如果内存分配是8K为page进行内存分配的话，会造成内存浪费。所以就用512B的大小。

netty中与缓存相关的数据结构为MemoryRegionCache，由三部分组成：

![image-20240528233951125](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240528233951125.png)

首先是队列，然后队列中的元素是一个个的entity，entity中都有一个chunk和handler。 每一个handler都指向唯一一段内存。所以一段chunk以及指向chunk的一段连续的内存就能确定一个entity的内存位置和内存大小。这些实体组合起来就成了一个队列。

然后是内存规格，netty里面有三种类型的内存规格。tiny、small和normal

最后是size，一个MemoryRegionCache的缓存的byteBuffer大小是固定的，也就是说MemoryRegionCache缓存的byteBuffez是1K的大小，那么queue存储的元素都是1K为大小的bytebuffer。 那么这些固定大小有哪些？tiny的，有32个，每个的size是16B的倍数；small的，有4个，每个的size分别为512B、1K、2K、4K。如果要分配的byteBuffer是3K，那么往前取4K分配3K；如果是normal的，有3个，每个的size分别为8K、16K、32K。

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240529001619200.png" alt="image-20240529001619200" style="zoom: 33%;" />

对于每一个节点都是一个memoryRegionCache，每一个memoryRegionCache都有一个queue，如果我们需要一个16字节的bytebuffer，那么就从直接定位到16B的节点，然后从16B的队列里取一个bytebuffer，这样就不需要从chunk里面划分内存了。

下面看一下代码：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611211356427.png" alt="image-20240611211356427" style="zoom:50%;" />

一个sizeClass，一个queue，意思就是某一个size类型，对应的queue里面有多少个byteBuffer。

memoryRegionCache是在哪里维护的？

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611213156766.png" alt="image-20240611213156766" style="zoom:50%;" />

memoryRegionCache分为了六种，因为他也分为了Heap和Direct。另外存储的都是memoryRegionCache数组，为什么存储的是数组呢？因为tiny有32种、small有4种、normal有3种。

看一下创建的参数：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611214833848.png" alt="image-20240611214833848" style="zoom:67%;" />

tinyCacheSize的大小为DEFAULT_TINY_CACHE_SIZE，也就是512。这个值是在创建PooledByteBufAllocator类的时候，将三个值分别传入的。

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611220904808.png" alt="image-20240611220904808" style="zoom:50%;" />

也就是512/16=32，也就是tiny的默认是32个。

然后就是tiny的枚举了。

#### 命中缓存的分配逻辑

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611223353727.png" alt="image-20240611223353727" style="zoom:50%;" />

首先对数据进行规格化：

点击进入

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611223443284.png" alt="image-20240611223443284" style="zoom:50%;" />

1.  如果大于chunkSize，直接返回
2. 如果不是tiny，进行double。它不直接进行double，而是找第一个大于等于capacity的2的幂次方的值。 
3. 如果是tiny，那么返回capaticy+16

然后回来看isTinyOrSmall

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611224603965.png" alt="image-20240611224603965" style="zoom:50%;" />

如果需要分配的内存小于pagesize，那么就是tiny或者small。

如果是tiny，也就是：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611224757596.png" alt="image-20240611224757596" style="zoom:50%;" />

512=2^9，如果相与0XFFFFFE00为0，那肯定小于512了。

接下来就是缓存分配了，缓存分配有三种类型：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611225006115.png" alt="image-20240611225006115" style="zoom:50%;" />

其实每一种的逻辑是大差不差的，以allocateTiny为例：

1. 找到对应size的MemoryRegionCache

   cacheForTiny(area, normCapacity)这个就是找到对应的cache，进入：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611225328635.png" alt="image-20240611225328635" style="zoom:50%;" />

   先看tinyIdx方法，这个方法找到对应的节点：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611225410648.png" alt="image-20240611225410648" style="zoom:50%;" />

   是直接将normal/16，看数组，如果要分配的是16B的，那么16/16=1:

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611225503707.png" alt="image-20240611225503707" style="zoom:50%;" />

   然后进入cache方法：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611230001518.png" alt="image-20240611230001518" style="zoom:50%;" />

   根据idx直接拿到第一个memoryRegionCache。

2. 从queue种弹出一个entry给ByteBuf初始化

   拿到节点之后，进入allocate方法：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611230310633.png" alt="image-20240611230310633" style="zoom:50%;" />

   直接从`cache.allocate()`方法进入：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611231100586.png" alt="image-20240611231100586" style="zoom:50%;" />

    首先我们可以看到从对垒中直接poll一个对象。我们看看这个Entry对象：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611231227198.png" alt="image-20240611231227198" style="zoom:50%;" />

   这里面有一个chunk的成员变量，通过chunk可以找到内存一段连续的内存。

   然后初始化，一直往下点进去：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611231711811.png" alt="image-20240611231711811" style="zoom:50%;" />

3. 将弹出的entry扔到对象池进行复用

   拿到entry定位到一段内存，新建了一个byteBuffer，它的作用就完事了。为了防止被回收消耗资源，专门做了一个对象池，不需要的对象可以回收，下次使用，看`recycle()`函数：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611232811903.png" alt="image-20240611232811903" style="zoom:50%;" />

   看一下`recycle()`方法

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611232946288.png" alt="image-20240611232946288" style="zoom:50%;" />

   stac就是对象池。

   那没有缓存命中怎么分配呢？

## 内存分配

### arena

在每一个线程去分配对应的内存的时候，首先通过threadLocal的方式拿到一个PollThreadCache，然后通过PollThreadCache调用allocate方法，PollThreadCache里面分为两部分，一部分是不同规格大小的cache，这个cache怎么分配内存我们已经分析过了。另一部分就是arena。

在代码里看一下：

![image-20240611234559708](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611234559708.png)

除了下面的caches以外还有arena。arena的意思就是从内存里面直接开辟一块内存使用。而缓存是从缓存中分配的内存使用。

看一下arena的数据结构：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611234929202.png" alt="image-20240611234929202" style="zoom: 33%;" />

为什么要把chunklisk双向列表的形式连接，并且chunk之间也是双向列表的形式进行连接呢？

每个ChunkList里面的Chunk的使用率是不同的，Chunk使用率相同的归位一个ChunList，这样netty就能通过一定的算法找到合适的Chunk，提高内存分配的效率。

点进去看一下arena：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611235147168.png" alt="image-20240611235147168" style="zoom:50%;" />

然后看一下q050的初始化：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611235351397.png" alt="image-20240611235351397" style="zoom: 33%;" />

从上面我们可以知道，q100里面的Chunk的内存已经使用100%以上，q075里面的Chunk的内存已经使用75%到100%之间，q050代表它里面的Chunk的内存已经使用50%到100%之间，以此类推。所以netty就可以通过这种方式找到需要的特定的内存，提高效率。

最终这些Chunlist之间是这样连接的双向链表：`qInit<=>q00<=>q025<=>q050<=>q075<=>q100`

如果我们要分配的内存小于Chunk也就是16M，我们用chunk来分配不划算，浪费空间，所以就有了page。一个page的大小是8k，如果我们要分配内存是10k，那么我们用chunk里面的两个page来分配就很划算了。但是如果只要2k呢，一个page也显得不划算，所有又有了subpage。**subpage**的大小介于0~8k之间：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611235511578.png" alt="image-20240611235511578" style="zoom: 50%;" />

看一下subPage，这个也在Arena里面：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240611235934347.png" alt="image-20240611235934347" style="zoom:50%;" />

看一下PoolSubpage的数据结构：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240612000016914.png" alt="image-20240612000016914" style="zoom:50%;" />

第一个成员变量final PoolChunk<T> chunk，说明了这个subpage属于哪个chunk。

elemSize，这个就是subpage的大小。可以为0~8k之间的值（并不一定是2k）。

bitmap里面如果值为1，就说明被分配了，如果为0就未分配，后面还会细讲。

prev和next指针，说明subpage之间也就通过双向链表链接的。

#### page级别的内存分配

还是看`allocate`方法，跳过前面的代码，直接来到这里，if里面是对缓存的分配。如果缓存找不到的话，就调用`allocateNoraml`方法进行page级别内存的分配。

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240612223447741.png" alt="image-20240612223447741" style="zoom:50%;" />

进入`allocateNoraml`方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240612223720477.png" alt="image-20240612223720477" style="zoom: 50%;" />

内存分配的步骤大概有三步：

1. 尝试在现有的chunk上面进行分配

   PoolArena里面有很多不同使用率的chunk的集合。在if条件里面，我们可以看到先在q050、q025、q000等chunklist上面进行内存分配。

   假设能在q050上面进行分配，点进allocate方法：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240612225422084.png" alt="image-20240612225422084" style="zoom:50%;" />

   这里面是一个循环，遍历每一个的chunk查看能否在上面进行分配。如果handle小于0说明没有分配到，那么就继续往下进行分配，如果到了末尾还没有分配那么就直接返回false，说明这个chunklist是没有进行内存分配的。如果handle是大于0的，那么就调用chunk的initBuf方法(可以认为并到第二个步骤中)，如果这个chunk的使用率大于最大的使用率，那么就将当前的chunk删除，并加入到下一个chunk里面

   但是第一次的时候这些chunklist都是空的，所以跳过，进入第二步。

2. 创建一个chunk进行内存分配

   通过newChunk去创建，然后chunk进行分配，返回一个handler，这个handler指向这个chunk的连续内存。

   点进newChunk方法：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240612233036757.png" alt="image-20240612233036757" style="zoom:50%;" />

   点击allocateDirect方法，可以看到是通过jdk的方法分配了一块内存：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240612233146655.png" alt="image-20240612233146655" style="zoom:50%;" />

   回到newChunk方法，后面的参数：

   pageSize：pagesize=8192，也就是我们刚刚要分配的8k内存

   maxOrder: maxOrder=11，代表一共有11层，稍后我们会分析。

   pageShifts: pageShifts=13，2^13=8192，稍后我们分析

   chunkSize: chunkSize=15777216，也就是16M，也就是一个chunkSize的大小

   在进入PoolChunk方法：

   ![image-20240612233534352](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240612233534352.png)

   1右移11位，maxSubpageAllocs是2048；maxSubpageAllocs右移1位是4096，所以整个memoryMap和depthMap的大小为4096大小。

   接着看后面的双重for循环。

   ![image-20240612233907327](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240612233907327.png)

   memoryMap就是一个完全二叉树，里面有11层，对应maxOrder；memoryMapIndex就是从上往下、从左往右数下去的第几个节点；d就是树的深度，p就是树的每一层的数量。depthMap和memoryMap一样分析。通过这个双重for循环，最终我们会分配到一个memoryMap，一共有4096个节点，每个节点的值为树的深度，也就是memoryMap={0,1,1,2,2,2,2,3,3,3,3,3,3,3,3，...，...11}。

3. 初始化PooledByteBuf

   也就是说拿到chunk一块连续内存之后，需要将对应的一个标记打到PolledByteBuf上面。`c.initBuf`就是来做这个事情。

# 内存的回收过程