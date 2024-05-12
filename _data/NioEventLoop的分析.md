# 三个问题

1. 默认情况下，Netty服务端起多少线程？何时启动？
2. Netty是如何解决jdk空轮训bug的？
3. Netty是如何保证异步串行无锁化的？

# NioEventLoop的创建

NioEventLoop是怎么创建的？在创建过程中做了什么？

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425213021774.png" alt="image-20240425213021774" style="zoom:40%;" />

如果构造函数没有传参数，那么默认的参数是cpu核数的2倍：

![image-20240425213502927](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425213502927.png)

继续跟进去，可以看到`DefaultEventExecutorChooserFactory.INSTANCE`是一个默认的线程选择器：

![image-20240425213708167](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425213708167.png)

继续点进去就能看到具体创建的逻辑了：

ThreadPerTaskExecutor  意味为当前的事件循环组创建Executor , 用于 针对每一个任务的Executor线程的执行器

newDefaultThreadFactory根据它的特性,可以给线程加名字等。比传统的好处是 把创建线程和 定义线程需要做的任务分开, 我们只关心任务,  两者解耦

![image-20240425215427500](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425215427500.png)

> Executor执行线程的工具，并不是一个线程，它的execute方法是交给了一个线程工厂new线程执行任务

## 创建线程执行器

**每次执行任务都会通过`threadFactory`创建一个线程实体**

![image-20240506204617894](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240506204617894.png)

NioEventLoop的线程命名规则是nioEventLoop-1-xx：

看一下newDefaultThreadFactory方法：

![image-20240506205424767](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240506205424767.png)

getClass方法是NioEventLoop，**为什么呢？**

![image-20240506205954866](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240506205954866.png)

先看toPoolName方法：

![image-20240506205927111](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240506205927111.png)

一直点构造方法会来到这里：

![image-20240506210253701](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240506210253701.png)

> ![image-20240506210546112](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimgimage-20240506210546112.png)
>
> new的线程不是原生的线程，是netty封装了threadLocal的

## 创建NioEventLoop

这一步做了三件事情：

1. 保存上面创建的线程执行器ThreadPerTaskExecutor
2. 创建一个MpscQueue（任务队列）
3. 创建一个selector（轮询注册到它上面的连接）

newChild方法传进去一个executor，用来创建NioEventLoop：

![image-20240425215641460](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425215641460.png)

上面的`newChild(executor, args);`方法其实是抽象方法,真正运行时会执行子类`NioEventLoopGroup`的实现：

![image-20240425215726481](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425215726481.png)

newChild出来的就是NioEventLoop，它继承自SingleThreadEventExecutor，点击newChild的NioEventLoop的实现，然后一直点，super构造方法的SingleThreadEventExecutor：

![image-20240425215852230](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425215852230.png)

可以看到讲一个线程执行器绑定起来了，另外new了一个taskQueue，外部线程在执行netty任务的时候，如果不是在nioeventloop对应的线程中 ，那么会讲这个任务保存到任务队列里，然后让nioeventloop对应的线程执行。

newTaskQueue创建的是mpscQueue，这个队列的作用是当外部线程执行netty的一些任务的时候，如果判断到不是当前NioEventLoop对应的线程去执行的时候，就把它放到队列里面，然后由NioEventLoop对应的线程去执行：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimgimage-20240425220307686.png" alt="image-20240425220307686" style="zoom:67%;" />

然后创建一个selector，轮询注册到nioEventLoop上面的连接：

`selector = openSelector();`

> **队列有啥用?**
>
> 我们知道,Netty中的线程可不止一个, 多个EventLoop意味着多个线程, 任务队列的作用就是当其他线程拿到CPU的执行权时,却得到了其他线程的IO请求,这时当前线程就把这个请求以任务的方式提交到对应线程的任务队列里面

## 创建线程选择器

**chooser的作用为新连接绑定nioeventLoop**

在这里，它把刚刚传经来的executor绑定进去了，后面创建NioEventLoop底层线程的时候需要用到。

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425221046559.png" alt="image-20240425221046559" style="zoom:67%;" />

这里判断nioeventLoop的数组长度是否是2的幂，是的话执行效率更高：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425221213084.png" alt="image-20240425221213084" style="zoom: 33%;" />

解释更优的原因：计算机底层，&比%计算速度更快，&直接通过二进制操作可以实现，但是%计算机底层没有二进制简单的实现，需要通过复杂实现。

正确的原因：实现目的就是轮询到最后一个，就开始重头轮询（循环取下标，也可以说是取模），普通方式的实现是可以的，我们现在解析PowerOfTwoEventExecutorChooser：

假设现在有16个线程，二进制为10000，length-1为1111，线程索引从0开始，idx也从0开始

1、轮询到第15次，也就是idx为1110，结果为1110，由于线程索引从0开始，就是第15个线程，正确

2、轮询到第16次，也就是idx为1111，结果为1111，由于线程索引从0开始，就是第16个线程，正确

3、关键是第17次，idx就是10000，结果为0000，由于线程索引从0开始，也就是第1个线程，正确

4、轮询到第18次，idx就是10001，结果为0001，由于线程索引从0开始，也就是第2个线程，正确

是不是很神奇？

因为2次幂的length减一，所有的位都是1。&的时候，在它前面的位置都会置0，也就是又可以重新开始了。

这一点，其实和hashMap1.7hash取余的方式很像。

> nioEventLoop的chooser和selector有什么不同？

到现在为止, **NioEventLoopGroup和NioEventLoop就都初始化完成了**,当然这是初始化,程序运行到现在,依然只有一条主线程, EventLoop的Thread还没`start()`干活,但是起码已经有能力准备启动了

就像下面的体系一样, 五脏俱全

- NioEventLoopGroup
  - NIoEventLoop
    - excutor(线程执行器) , 执行IO任务/非IO任务
    - selector 选择器
  - Chooser

# NioEventLoop的启动

NioEventLoop的两个触发器：

1. 服务端启动绑定端口
2. 新连接接入通过chooser绑定一个NioEventLoop（之后看）

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425223201493.png" alt="image-20240425223201493" style="zoom:33%;" />

从bind一直点进去。先是一个通过channel的eventLoop()方法获取NioEventLoop，这个是在registre将channel和NioEventLoop绑定起来的。

![image-20240425223337602](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425223337602.png)

点进execute方法,如果我们直接使用鼠标点击进去,会进入`java.util.concurrent`包下的`Executor`接口, 原因是因为,它是NioEventLoop继承体系的超顶级接口,我们进入它的实现类,`SingleThreadEventExcutor`, 也就是`NioEventLoop`的间接父类, 源码如下:

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425223752101.png" alt="image-20240425223752101" style="zoom:50%;" />

首先判断是否是NioEventLoop的线程：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425223904436.png" alt="image-20240425223904436" style="zoom:50%;" />

由于是通过main的线程，所以这里肯定不一样的，所以返回fasle

> 这里是cas来判断状态是否对的
>
> <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425224412817.png" alt="image-20240425224412817" style="zoom:50%;" />
>
> 同时可以看到好多`exector.execute`方法，异步任务方法

主要做了两件事: 1. 调用了NioEventLoop的线程执行器的`execute`,execute就是在创建线程, 线程创建完成后,立即把新创建出来的线程当作是`NioEventLoop`相伴终生的线程;

![image-20240506231837933](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240506231837933.png)

创建/绑定完成了新的线程后, `SingleThreadEventExecutor.this.run();` 这行代码的意思是,调用本类的`run()`方法,这个`run()`方法就是真正在干活的事件循环,但是呢, 在本类中,`run()`是一个抽象方法,因此我们要去找他的子类,那么是谁重写的这个`run()`呢? 就是NioEventLoop, 它根据自己需求,重写了这个方法

# NioEventLoop的执行逻辑

在NioEventLoop启动这里，有一个这个方法，就是最终执行的任务：

`SingleThreadEventExecutor.this.run();`

下面为run方法具体的执行逻辑

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425225210261.png" alt="image-20240425225210261" style="zoom:50%;" />

select方法就是轮训io事件，由于一个nioEventLoop对应一个selector，这个selector就是轮训 注册到selector上面的io事件

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425225308734.png" alt="image-20240425225308734" style="zoom:50%;" />

然后

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240425230208650.png" alt="image-20240425230208650" style="zoom:67%;" />

由于ioRatio是50，那么runAllTasks就是ioTime

## select方法逻辑

> 这里得看一下java的nio逻辑：https://www.cnblogs.com/robothy/p/14242971.html

deadLine以及任务穿插逻辑处理

 ![image-20240506214726361](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240506214726361.png)

> 第二步为什么要检查任务队列是不是空，以及wakeUp的状态？
>
> 如果在wakenUp值为true时提交任务，则该任务没有机会调用Selector#wakeup。 所以在执行select操作之前我们需要再次检查任务队列。如果不这样做，任务可能会被挂起，直到select操作超时。如果管道中存在 IdleStateHandler，则任务可能会被挂起，直到空闲超时。

如果上面的两个判断，即截止时间没到以及任务队列也是空的话，就进行阻塞式的select操作

![image-20240506215841539](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240506215841539.png)

解决jdk空轮训的bug

![image-20240506220209335](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240506220209335.png)