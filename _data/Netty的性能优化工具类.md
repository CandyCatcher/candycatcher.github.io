# FastThreadLocal

## FastThreadLocal的创建

创建的话直接从FastThreadLocal的构造方法进入：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623231621101.png" alt="image-20240623231621101" style="zoom: 67%;" />

在构造方法里面只是创建了index，而这个index就是每个线程里面FastThreadLocal唯一的标识。

看这个`nextVariableIndex()`方法，是InternalThreadLocalMap的一个静态方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623232256499.png" alt="image-20240623232256499" style="zoom:50%;" />

nextIndex是什么呢？

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623232828349.png" alt="image-20240623232828349" style="zoom:50%;" />

nextIndex是一个静态原子类，每次新建一个FastThreadLocal线程里面的index都会加1，这个index作为这个新建FastThreadLocal的标识，之后要找到这个FastThreadLocal就通过这个标识。

## get方法的实现

### 获取ThreadLocalMap

从FastThreadLocal的get方法进入：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623233404301.png" alt="image-20240623233404301" style="zoom: 50%;" />

先看下`InternalThreadLocalMap.get()`:

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623233749872.png" alt="image-20240623233749872" style="zoom:50%;" />

可以看到有fastGet和slowGet，slowGet就是线程不为FastThreadLocalThread的时候，我们先看slowGet方法的实现：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623234027262.png" alt="image-20240623234027262" style="zoom:50%;" />

通过slowThreadLocalMap的get方法找出InternalThreadLocalMap，然后如果不存在就创建一个并放到slowThreadLocalMap里面，那么这个slowThreadLocalMap是什么的？

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623235103972.png" alt="image-20240623235103972" style="zoom:50%;" />

这个 slowThreadLocalMap是jdk原生的ThreadLocal。接着通过ThreadLocal的get方法里面把InternalThreadLocalMap取出来的，相对来说更加"slow"，那么fast呢？

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623235430604.png" alt="image-20240623235430604" style="zoom:50%;" />

这一看我们就能知道它为什么fast了，因为它的InternalThreadLocalMap直接是这个thread的一个成员变量，直接取就可以。如果取不到就直接新建一个给它。

### 直接通过索引找到对象

得到map之后，把map传进去get，看源码：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624001825830.png" alt="image-20240624001825830" style="zoom:50%;" />

先看`indexedVariable(index)`方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624001957942.png" alt="image-20240624001957942" style="zoom:50%;" />

就是在indexedVariables数组里面取出特定index的值。

看一下下面这张图：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624002637251.png" alt="image-20240624002637251" style="zoom: 33%;" />

假设在jvm中存在两个ThreadLocal对象，index分别是0、1。然后对于每一个线程，都维护了一个threadLocalMap对象，在每一个threadLocalMap对象中都有一个数组，通过threadLocal对象的get方法拿到当前线程的对象的时候，先拿到当前线程的threadLocalMap，由于线程的threadLocalMap是数组实现的，每一个fastThreadLocal有全局唯一的一个index，所以通过index直接找到fastThreadLocal在当前线程的对象。 Thread2同理。 

### 如果threadLocalMap中index对应的对象为空，那就初始化对象

如果对象没有拿到，那么就进行初始化的操作：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624005607687.png" alt="image-20240624005607687" style="zoom:50%;" />

首先看下indexedVariable是在哪里初始化的：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624005906087.png" alt="image-20240624005906087" style="zoom:50%;" />

接着讲所有的元素设置为unset：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624005955685.png" alt="image-20240624005955685" style="zoom:50%;" />

然后看一下initialize方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624010114957.png" alt="image-20240624010114957" style="zoom:50%;" />

initializeValue默认实现的方法是返回null，我们当然也可以覆盖这个方法去实现自己的逻辑。

然后有个threadLocalMap.setIndexedVariable(index, v);

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624010246090.png" alt="image-20240624010246090" style="zoom:50%;" />

这里就是把我们初始化的值设置到map里面，下次我们要取得时候就能取到了。如果不够就扩容并且添加进去，如果旧值是UNSET就返回true。

最后还有一个`addToVariablesToRemove(threadLocalMap, this);`我们先通过名字记下来：

因为是初始化的值，所以以后是需要移除的。

## set方法的实现

FastThreadLocal的set方法的实现原理和get方法有些类似

### 获取threadLocalMap对象

同样的，获取ThreadLocalMap的方法还是和之前的一样调用方法`InternalThreadLocalMap.get()`

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624214344885.png" alt="image-20240624214344885" style="zoom:50%;" />

### 通过索引set对象

接着看set方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624214447023.png" alt="image-20240624214447023" style="zoom:50%;" />

如果设置进来的值不是UNSET那么就把值设置进去，也就是将index对应的值设置为value：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624215238486.png" alt="image-20240624215238486" style="zoom:50%;" />

设置值还是只要把对象放到特定的索引位置就可以了，注意我们还是返回了`oldValue==UNSET`。也就是，我们返回了之前的值是否为unset，返回之后，如果为unset，那么就要调用`addToVariablesToRemove`。

### 调用remove方法

如果是unset对象，那么就调用remove方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624215521918.png" alt="image-20240624215521918" style="zoom:50%;" />

看这个`removeIndexedVariable`方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624215745820.png" alt="image-20240624215745820" style="zoom:50%;" />

其实很简单，就是把索引所在位置的对象设置为UNSET对象。

还有一个`removeFromVariablesToRemove(threadLocalMap, this);`和前面的`addToVariablesToRemove`相对应。

最后有一个判断，如果移除的对象是用户设置进去的，那么就调用onRemoval方法，这个方法可以在用户代码里面覆盖的。

# 轻量级对象池Recycler

先看一下Recycler的结构。在Recycler中有一个默认的fastThreadLocal：

![image-20240624222535355](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624222535355.png)

这个fasThreadLocal存储的是一个stack，这个栈里面存放的是handler，然后每一个handler对应的是一个对象。

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624222258000.png" alt="image-20240624222258000" style="zoom: 33%;" />

然后看一下stack的成员变量的作用：

* thread：当前的thread

* ratioMask：每次通过调用recycle把对象进行回收的时候，它并不是每次都会把对象进行回收的，这个参数用来控制对象回收的频率

* maxCapacity：这个stack的最大容量，也就是池的大小，也就是一个线程里面最多可以放多少个对象

  Recycler的构造函数传进来一个默认值`DEFAULT_INITIAL_MAX_CAPACITY_PER_THREAD`:

  ![image-20240624224815672](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624224815672.png)

  这个默认值为：

  ![image-20240624224855541](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624224855541.png)

  继续看Recycler的构造函数：

  <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624225102465.png" alt="image-20240624225102465" style="zoom:50%;" />

  在这里面

  <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240624225217460.png" alt="image-20240624225217460" style="zoom:50%;" />

* maxDelayedQueues：我们的对象可能在一个线程里面创建，在另外一个线程那里释放，这个对象就不会放在第一个线程的数组里，而是放在另外一个线程的WeakOrderQueue里面。那么一个线程创建的对象可以在多少个不同的线程里面缓存呢？就是这个值了

* cursor、prev、head：这个就是存放了其他线程的链表的指针，也就是WeakOrderQueue的指针

* availableSharedCapacity：因为可以在不同的线程释放缓存对象，这个参数就是一个线程里面可以缓存别的线程对象的最大的个数