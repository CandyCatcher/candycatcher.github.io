# I/O模型

## BIO和NIO的理解?

## 阻塞IO/BIO

在阻塞式IO模型中，Java应用程序从发起IO系统调用开始，一直到系统调用返回，在这段时间内，发起IO请求的Java进程（或者线程）是阻塞的。直到返回成功后，应用进程才能开始处理用户空间的缓存区数据。并且每一个IO请求都需要独立的线程完成数据读写，业务处理的完整操作。

**缺点**：

* 当并发数较大时，需要创建大量线程来处理连接，系统资源占用较大。
* 连接建立后，如果当前线程暂时没有数据可读，则线程就阻塞在read操作上，造成线程资源浪费。

## 非阻塞IO/NIO

在内核缓冲区中没有数据的情况下，系统调用会立即返回，返回一个调用失败的信息。应用程序的线程需要不断地进行IO系统调用，轮询数据是否已经准备好，如果没有准备好，就继续轮询，直到完成IO系统调用为止。

> NIO 解决的地方是从网卡到内核空间部分的阻塞，也就是说应用程序发送一个读取 IO 的请求，如果数据还没有从网卡写入内核空间，直接返回未就绪，这样就做到了不需要程序死等到结果。等到写入内核空间以后，程序继续读取数据，这时候才会阻塞程序
>
> 在内核缓冲区中有数据的情况下，在数据的复制过程中系统调用是阻塞的，直到完成数据从内核缓冲复制到用户缓冲。复制完成后，系统调用返回成功，用户进程（或者线程）可以开始处理用户空间的缓存数据。

**缺点：**不断地轮询内核，这将占用大量的CPU时间，效率低下。

## IO多路复用模型

在IO多路复用模型中通过select/epoll系统调用，单个应用程序的线程，可以不断地轮询成百上千的socket连接的就绪状态，当某个或者某些socket网络连接有IO就绪状态，就返回这些就绪的状态（或者说就绪事件）。

特点：

一个选择器查询线程，可以同时处理成千上万的网络连接，所以，用户程序不必创建大量的线程，也不必维护这些线程，从而大大减小了系统的开销。这是一个线程维护一个连接的阻塞IO模式相比，使用多路IO复用模型的最大优势。

缺点：

本质上，select/epoll系统调用是阻塞式的，属于同步阻塞IO。也就是说这个事件的查询过程是阻塞的。

## 信号驱动IO模型

用户进程预先在内核中设置一个回调函数，当某个事件发生时，内核使用信号（SIGIO）通知进程运行回调函数。 然后用户线程会继续执行，在信号回调函数中调用IO读写操作来进行实际的IO请求操作。

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240706185519843.png" alt="image-20240706185519843" style="zoom:50%;" />

优势：

用户进程在等待数据时，不会被阻塞，能够高用户进程的效率。

缺点：

* 在大量IO事件发生时，可能会由于处理不过来，而导致信号队列溢出。
* 对于处理UDP套接字来讲，对于信号驱动I/O是有用的。可是，对于TCP而言，由于致使SIGIO信号通知的条件为数众多，进行IO信号进一步区分的成本太高，信号驱动的I/O方式近乎无用。
* 信号驱动IO可以看成是一种异步IO，可以简单理解为系统进行用户函数的回调。只是，信号驱动IO的异步特性，又做的不彻底。为什么呢？ <u>信号驱动IO仅仅在IO事件的通知阶段是异步的，而在第二阶段，也就是在将数据从内核缓冲区复制到用户缓冲区这个过程，用户进程是阻塞的、同步的。</u>

## 异步IO模型

用户线程通过系统调用，向内核注册某个IO操作。内核在整个IO操作（包括数据准备、数据复制）完成后，通知用户程序，用户执行后续的业务操作。

优势：

在内核等待数据和复制数据的两个阶段，用户线程都不是阻塞的。

用户线程需要接收内核的IO操作完成的事件，或者用户线程需要注册一个IO操作完成的回调函数。正因为如此，异步IO有的时候也被称为信号驱动IO。

缺点：

应用程序仅需要进行事件的注册与接收，其余的工作都留给了操作系统，也就是说，需要底层内核供支持。

# NIO和BIO到底有什么区别？有什么关系？

1. NIO是通过缓存区和通道的方式处理数据，BIO是通过InputStream和OutputStream流的方式处理数据。
2. NIO的通道是双向的，BIO流的方向只能是单向的。
3. NIO采用的多路复用的同步非阻塞IO模型，BIO采用的是普通的同步阻塞IO模型。
4. NIO的效率比BIO要高，NIO适用于网络IO，BIO适用于文件IO。

# IO多路复用是如何实现同步非阻塞的?

NIO在Linux系统下，socket连接默认是阻塞模式，可以通过设置将socket变成为非阻塞的模式（Non-Blocking）。非阻塞模式下，在内核缓冲区中没有数据的情况下，系统调用会立即返回，返回一个调用失败的信息。

而在IO多路复用模型中，引入了一种新的系统调用，查询IO的就绪状态。在Linux系统中，对应的系统调用为select/epoll系统调用。通过该系统调用，一个进程可以监视多个文件描述符（包括socket连接），一旦某个述符就绪（一般是内核缓冲区可读/可写），内核能够将就绪的状态返回给应用程序。随后，应用程序根据就绪的状态，进行相应的IO系统调用。

一个线程使用一个选择器Selector监听多个通道 Channel 上的IO事件，从而让一个线程就可以处理多个IO事件。通过配置监听的通道Channel为非阻塞，那么当通道上的IO事件还未到达时，线程会在select方法被挂起，让出CPU资源。直到监听到Channel有IO事件发生时，才会进行相应的响应和处理。

Selector只有在通道上有真正的IO事件发生时，才会进行相应的处理，这就不必为每个连接都创建一个线程，避免线程资源的浪费和多线程之间的上下文切换导致的开销。

# BIO和NIO应用场景

1. BIO 方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4以前的唯一选择。
2. NIO 方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，弹幕系统，服务器间通讯等。JDK1.4 开始支持。

# 阻塞/非阻塞区别

阻塞是指内核IO操作时，用户进程（或者线程）一直在等待，而不能干别的事情；非阻塞是指用户进程（或者线程）拿到内核返回的状态值就返回自己的空间，可以去干别的事情。

# 同步/异步区别

同步与异步可以看成是发起IO请求的两种方式。同步IO是指用户空间（进程或者线程）是主动发起IO请求的一方，系统内核是被动接受方。异步IO则反过来，系统内核主动发起IO请求的一方，用户空间是被动接受方。

# 主要的IO线程模型有哪些？

## Connection Per Thread（一个线程处理一个连接）线程模型

![image-20240706193646447](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240706193646447.png)

这种方法的最大问题是：如果前一个网络连接的handle（socket）没有处理完，那么后面的新连接没法被服务端接收，于是后面的请求就会被阻塞住，这样就导致服务器的吞吐量太低。

![image-20240706193552524](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240706193552524.png)

每个线程都独自处理自己负责的socket连接的输入和输出，不会阻塞到后面新socket连接的监听和建立。这样，服务器的吞吐量就得到了提升。

但是这个模式应于大量的连接，需要耗费大量的线程资源，对线程资源要求太高。在系统中，线程是比较昂贵的系统资源。如果线程的数量太多，系统无法承受。而且，线程的反复创建、销毁、线程的切换也需要代价。

## Reactor线程模型

Reactor模型，是指通过一个或多个输入同时传递给服务器，并将它们同步分派给请求对应的处理线程的事件驱动处理模式。 

Reactor模型中有2个关键组成：

* Reactor Reactor在一个单独的线程中运行，负责监听和分发事件，分发给适当的处理程序来对IO事件做出反应。
* Handlers 处理程序执行I/O事件要完成的实际事件。

取决于Reactor的数量和Hanndler线程数量的不同，Reactor模型有3个变种：

* 单Reactor单线程。Reactor反应器和Handers处理器处于一个线程中执行

  1. 一个NIO线程同时处理成百上千的链路，性能上无法支撑。即便NIO线程的CPU负荷达到100%，也无法满足海量消息的编码、解码、读取和发送
  2. 可靠性问题。在单线程反应器模式中， Reactor反应器和Handler处理器都执行在同一条线程上。在这种场景下，被阻塞的Handler不仅仅负责输入和输出处理的传输处理器，还包括负责新连接的监听处理器，这就可能导致服务器无响应。

* 单Reactor多线程

  Reactor 多线程模型将业务逻辑交给多个线程进行处理。除此之外，多线程模型其他的操作与单线程模型是类似的，比如连接建立、IO事件读写以及事件分发等都是由一个线程来完成。

  缺点：

  连接建立、IO事件读取以及事件分发完全有单线程处理。比如当某个连接通过系统调用正在读取数据，此时相对于其他事件来说，完全是阻塞状态，新连接无法处理、其他连接的IO查询/IO读写以及事件分发都无法完成。

* 主从Reactor多线程

  在多线程模型中，我们提到，其主要缺陷在于同一时间无法处理**大量新连接**、**IO就绪事件**；因此，将主从模式应用到这一块，就可以解决这个问题。

  主从 Reactor 模式中，分为了主 Reactor 和 从 Reactor，分别处理 新建立的连接、IO读写事件/事件分发。

  - 主 Reactor 可以解决同一时间大量新连接，将其注册到从 Reactor 上进行IO事件监听处理
  - IO事件监听相对新连接处理更加耗时，此处我们可以考虑使用线程池来处理。这样能充分利用多核 CPU 的特性，能使更多就绪的IO事件及时处理。

* Netty线程模型

  Netty主要基于主从Reactors多线程模型（如下图）做了一定的修改，其中主从Reactor多线程模型有

  多个Reactor：MainReactor和SubReactor：

  MainReactor负责客户端的连接请求，并将请求转交给SubReactor

  SubReactor负责相应通道的IO读写请求

  非IO请求（具体逻辑处理）的任务则会直接写入队列，等待worker threads进行处理

# Netty 高性能表现在哪些方面？

* IO 线程模型：通过多线程Reactor反应器模式，在应用层实现异步非阻塞（异步事件驱动）架构，用最少的资源做更多的事。
* 内存零拷贝：尽量减少不必要的内存拷贝，实现了更高效率的传输。
* 内存池设计：申请的内存可以重用，主要指直接内存。内部实现是用一颗二叉查找树管理内存分配情况。 
* 对象池设计：Java对象可以重用，主要指Minior GC非常频繁的对象，如ByteBuﬀer。并且，对象池使用无锁架构，性能非常高。 
* mpsc无锁编程：串形化处理读写, 避免使用锁带来的性能开销。
* 高性能序列化协议：支持 protobuf 等高性能序列化协议。

# Netty服务端启动的步骤？

1. 创建服务端channel。这个过程就是调用jdk底层的逻辑创建jdk底层的channel，然后netty将其包装成自己的channel，同时创建一些基本组件绑定在此channel上面，比如newId、newUnsafe、newChannelPipeline(就是新建了头和尾俩节点)，最后设置configureBlocking为非阻塞。

2. 初始化服务端channel。创建完channel后，netty会做一些初始化的工作，即初始化启动程序设置的属性、将初始化的一个handler(ServerBootstrapAcceptor)添加到pipeLine中。

   > 什么时候将用户的handler加进去的？
   >
   > 初始化`ChannelPipeline`的时机是当`Channel`向对应的`Reactor`注册成功后，在`handlerAdded事件回调`中利用`ChannelInitializer`进行初始化。
   >
   > <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240708005102954.png" alt="image-20240708005102954" style="zoom:33%;" />

3. 将服务端`NioServerSocketChannel`注册到主Reactor线程组中。同时注册Selector。netty将jdk底层的channel注册到事件轮询器selector上面，并把netty的服务端channel作为一个attachMent绑定到对应的jdk底层的channel上(但是并没有指定关心的事件)。

4. 端口绑定。最后调用`doBind()`方法来实现对本地端口的监听后，**netty会重新向selector注册一个Accept事件**，这样netty就可以接受新的连接了。

# Netty服务端的socket在哪里初始化？

创建服务端channel是从用户代码的bind方法进入的，在ServerBootstrap这个类里面进行服务端channel的初始化。

# Netty在哪里进行accept连接？

在客户端连接建立好后，初始化客户端`NioSocketChannel`，在`从Reactor线程组中`选取一个`Sub Reactor`，将客户端`NioSocketChannel` 注册到`Sub Reactor`中的`selector`上。

# Netty的主Reactor线程是何时启动的?

`Reactor线程`的启动是在向`Reactor`提交第一个异步任务的时候启动的。

Netty中的主Reactor线程组`NioEventLoopGroup`中的Main Reactor`NioEventLoop`是在用户程序`Main线程`向`Main Reactor`提交用于注册`NioServerSocketChannel`的异步任务时开始启动。

# NioEventLoop的创建过程？

Netty为主从Reactor线程模式，`EventLoop`就是Netty中的`Reactor`，可以说它就是Netty的引擎，负责Channel上**IO就绪事件的监听**、**IO就绪事件的处理**、**异步任务的执行**驱动着整个Netty的运转。不同的模式下，`EventLoop`有着不同的实现，我们只需要切换不同的实现类就可以完成对Netty IO模型的切换。

|            BIO            |     NIO      |     AIO      |
| :-----------------------: | :----------: | :----------: |
| ThreadPerChannelEventLoop | NioEventLoop | AioEventLoop |

1. 创建用于启动Reactor线程的ThreadPerTaskExecutor。它的工作方式，就是来一个任务就创建一个线程执行。而创建的这个线程正是netty的核心引擎Reactor线程。
2. 创建NioEventLoop。netty除了让`Reactor`轮询注册其上的所有`Channel`上的IO就绪事件，还会让它做一些异步任务的执行工作，这样需要创建一个队列来保存任务。然后创建一个selector，轮询注册到nioEventLoop上面的连接。
3. 创建线程选择器selector。由于Hash冲突这种情况的存在，所以导致HashSet的插入和遍历操作的性能不如数组，所以Netty用数组这种数据结构实现了`SelectedSelectionKeySet`，用它来替换原来的`HashSet`实现。
4. 创建channel到reactor上的绑定策略。

当然这是初始化，程序运行到现在，依然只有一条主线程，EventLoop的Thread还没`start()`干活，但是起码已经有能力准备启动了。

# NioEventLoop的执行逻辑？

1. Reactor线程在Selector上阻塞获取IO就绪事件。在这个模块中首先会去检查当前是否有异步任务需要执行，如果有异步需要执行，那么不管当前有没有IO就绪事件都不能阻塞在Selector上，随后会去非阻塞的轮询一下Selector上是否有IO就绪事件，如果有，正好可以和异步任务一起执行。优先处理IO就绪事件，在执行异步任务。
2. 如果当前没有异步任务需要执行，那么Reactor线程会接着查看是否有定时任务需要执行，如果有则在Selector上阻塞直到定时任务的到期时间deadline，或者满足其他唤醒条件被唤醒。如果没有定时任务需要执行，Reactor线程则会在Selector上一直阻塞直到满足唤醒条件。
3. 当Reactor线程满足唤醒条件被唤醒后，首先会去判断当前是因为有IO就绪事件被唤醒还是因为有异步任务需要执行被唤醒或者是两者都有。随后Reactor线程就会去处理IO就绪事件和执行异步任务。
4. 最后Reactor线程返回循环起点不断的重复上述三个步骤。

# `Reactor线程`正阻塞在`selector.select()`调用上等待`IO就绪事件`的到来，如果此时正好有`异步任务`被提交到`Reactor`中需要执行，并且此时无任何`IO就绪事件`，而`Reactor线程`由于没有`IO就绪事件`到来，会继续在这里阻塞，那么如何去执行`异步任务`呢？？

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240708012244862.png" alt="image-20240708012244862" style="zoom: 50%;" />

# 默认情况下，Netty服务端起多少线程？何时启动？

默认情况下，neet服务端是起cpu核数的2倍个线程。

# NIO空轮询BUG以及Netty是如何解决jdk空轮训bug的？

JDK NIO的空轮询BUG其实是JDK NIO在Linux系统下的epoll空轮询问题。<u>就是即使是关注的select轮询事件返回数量为0，NIO照样不断的从select本应该阻塞的`Selector.select()`/`Selector.select(timeout)`中wake up出来，导致CPU飙到100%问题。</u>

Netty就是通过记录空轮询次数来判断是否发生了空轮询（Netty默认是512次），若发生空轮询则重建Selector。

# `NioEventLoop`是如何在千百条channel中,精确获取出现指定感兴趣事件的channel的?

一个Selector中可能被绑定上了成千上万个Channel，通过selectionKey+attachment 的手段，精确的取出发生指定事件的channel，进而获取channel中的unsafe类进行下一步处理。

# Netty是如何保证异步串行无锁化的？

# Netty是在哪里检测有新连接接入的？

检测新连接从`processSelectedKeysOptimized(SelectionKey[] selectedKeys)`开始的。具体是nioEventLoop的线程里做的：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240708125609560.png" alt="image-20240708125609560" style="zoom:50%;" />

# 新连接是怎样注册到NioEventLoop线程的？

在服务器accept新连接的时候，通过调用doReadMessage创建客户端的channel。

根据轮询算法获取一个eventloop，接着拿到客户端 NioSocketChanenl的channel这个对象，调用channel的regist方法将jdk原生的channel注册进原生的selector，最后传播channelActive挨个回调他们的状态给这条客户端的新连接注册上netty能处理的感兴趣的事件。

<img src="/Users/candyboy/Library/Application%20Support/typora-user-images/image-20240708140942348.png" alt="image-20240708140942348" style="zoom:33%;" />

# 服务端的channel需要反射创建，而客户的的channel直接new?

netty不仅可以做 NIO编程模型的服务器, 传统的阻塞式IO,或者其他类型的服务器他也可以做, 我们传递进入的服务端Chanel的类型决定了他可以成为的服务器的类型, netty的设计者是不知道,用户想用netty做些什么的,于是设计成通过反射创建。

但是,一旦服务端的channel类型确定了,对应的客户端的channel也一定知道了,直接new 就好了

# Netty异常的传播

netty中如果发生了异常的话,异常事件的传播和当前的节点是 入站和出站处理器是没关系的,一直往下一个节点传播,如果一直没有handler处理异常,最终由tail节点处理。既然异常的传播和入站和出站类型的处理器没关系,那么我们就在pipeline的最后,也就是tail之前,添加我们的统一异常处理器就好了。

# netty的内存类别有哪些？

分配堆内内存和堆外内存。

# 如何保证多线程内存分配的安全性？

一个NioEventLoop里面有一个私有的Arena（其实就是ThreadLocal的原理）

池化分配缓存需要注意内存安全性：

1. 分配时是拿到线程局部缓存的PoolThreadCache。因为newDirectBuffer可能会多线程调用，这里通过threadCache拿到当前线程的cache进行分配。而这里的threadCache就是PoolThreadLocalCach。
2. 在线程局部缓存的Area上进行内存分配。



内存分配释放不可避免地会遇到多线程并发场景，PoolChunk 的完全平衡树标记以及 PoolSubpage 的 bitmap 标记都是多线程不安全的，都是需要加锁同步的。为了减少线程间的竞争，Netty 会提前创建多个 PoolArena（默认数量为 2 * CPU 核心数），当线程首次请求池化内存分配，会找被最少线程持有的 PoolArena，并保存线程局部变量 PoolThreadCache 中，实现线程与 PoolArena 的关联绑定

# 如何保证channel的线程安全性？

1.一个EventLoopGroup当中包含一个或多个EventLoop
2.一个EventLoop在其生命周期内只和唯一的一个Thread线程绑定（即一个io线程）
3.所有由EventLoop处理的各种io事件都将在其所关联的io线程上执行，因为是单线程保证了线程安全
4.一个Channel在其生命周期内只会注册在一个EventLoop（selector）上
5.运行期间，一个EventLoop会被分配给一个或多个channel



判断当前线程是否是EventLoop中所绑定的那个唯一的io线程
如果是，则直接执行相应的channelHandler，处理该io事件
若不是，则将需要执行的channelHandler封装成一个任务交给EventLoop中io线程去执行，EventLoop中具体保存任务的是一个FIFO队列，而且又是单线程执行，所以在保证线程安全的同时也保证了任务的有序性。

# 不同大小的内存分配策略是不同的？

# netty多对一的请求如何响应

# TCP的粘包拆包，为什么会发生粘包拆包，怎么解决？

**TCP分包的原因：**

TCP是以流的方式来处理数据，一个完整的包可能会被TCP拆分成多个包进行发送，也可能把小的封装成一个大的数据包发送。

TCP是以段（Segment）为单位发送数据的，建立TCP链接后，有一个最大消息长度（MSS）。如果应用层数据包超过MSS，就会把应用层数据包拆分，分成两个段来发送。这个时候接收端的应用层就要拼接这两个TCP包，才能正确处理数据。另外，路由器（或以太网）有一个MTU（ 最大传输单元），一般是在46－1500字节，除去IP头部20字节，留给TCP的就只有MTU-20字节。所以一般TCP的MSS为MTU-20=1460字节。当应用层数据超过1460字节时，TCP会分多个数据包来发送。

**TCP粘包的原因：**

TCP为了提高网络的利用率，会使用一个叫做Nagle的算法。该算法是指，发送端即使有要发送的数据，如果很少的话，会延迟发送。如果应用层给TCP传送数据很快的话，就会把两个应用层数据包“粘”在一起，TCP最后只发一个TCP数据包给接收端。

**Netty中提供了多个 Decoder 解析类 用于解决上述问题：**

* FixedLengthFrameDecoder 、LengthFieldBasedFrameDecoder ，固定长度是消息头指定消息长度的一种形式，进行粘包拆包处理的。
* LineBasedFrameDecoder 、DelimiterBasedFrameDecoder ，换行是于指定消息边界方式的一种形式，进行消息粘
* 我们可以在Netty的ChannelPipeline中添加一个自定义的解码器，该解码器负责将接收到的字节流按照我们定义的规则进行拆分和组合。具体的处理方式可以根据应用的业务需求来定制。

> 以太网帧的最大帧长为1500，最小帧长为64字节，为什么？
>
> 以太网最小帧长必须大于整个网络的最大时延位，这样以太网帧最小值为64字节时才能保证数据发送期间进行有效的冲突检测。如果帧长度太小，就可能出现同时有两个帧在信道上传播，产生的冲突无法有效的通知到对方，造成信道无法传输数据。
>
> 信道是所有主机共享的，如果某主机发送的数据帧太长，就会长时间占据信道，影响其他主机通信。同时，太长的帧需要花费足够的缓冲区来缓存，甚至会超出接收方缓冲区的大小，造成缓冲溢出。为避免某一主机长时间占用信道，因此规定了以太网的最大帧长为1500字节。

为什么用Netty（也就是技术选型，同时说下Netty的优势，比如线程模型、IO设计、零拷贝、缓存命中等） 

# netty连接断开怎么办

# Netty怎么使用少线程解决并发请求的问题的

# NiO的线程模型/Netty反应堆模式

Netty主要基于主从Reactors多线程模型。

Netty中IO事件的处理流程，大致分为4步，具体如下：

* 第1步：通道注册。Netty封装了NIO的Selector组件和Thread线程实例，设计了自己的Reactor角色，名称叫做EventLoop（事件循环）；并且封装了NIO的Channel组件，设计了自己的传输通道组件，名字仍然叫做Channel，只是所处的包不同。通道注册，指的是将Netty的Channel注册到EventLoop上，对应到底层就是NIO的Channel注册到NIO的Selector上。
* 第2步：查询事件。在Netty反应器模式中，一个线程会负责一个反应器（或者SubReactor子反应器），EventLoop和Thread也是这种一对一的模式。一个反应器负责一个Selector的查询，EventLoop内部Thread不断地轮询，查询选择器Selector中的IO事件，并记录在选择键上面。
* 第3步：事件内部分发、数据读取和发射。这里和经典的Reactor模式有细微的区别：在经典Reactor模式中事件分发和数据读取是分开的，Reactor负责IO事件的分发，Handler负责数据的读取；而在Netty的Reactor模式中，反应器EventLoop把事件分发和数据读取两个操作一起负责了。具体来说，EventLoop能访问到通道的Unsafe成员，当IO事件发生时，直接通过Unsafe成员完成NIO底层的数据读取。 EventLoop读取到的数据后， 会把数据发射到Channel内部的Pipeline流水线通道。
* 第4步：流水线传播和业务处理。数据在通道的Pipeline流水线上传播，通道的流水线由Handler构成，由Handler业务处理器负责，处理完成之后，再把结果传播或者传递到下一个Handler。 为啥需要Pipeline流水线呢？主要是由于同一个NIO事件，可能会有多个业务处理，比如数据的解码、数据的校验、业务的处理，所以Netty通过责任链模式将多个业务处理器组织起来，成为一个pipeline（流水线）。Pipeline流水线由通道负责管理，属于通道的一部分。数据可以在流水线上传播，再交给流水线上的Handler来处理。Handler业务处理器放置的是具体的业务逻辑，这是Java工程师们需要负责开发的部分。

以上4步，就是整个Netty的IO处理器流程。Netty的Reactor模式，和经典Reactor模式实现区别很小，主要的区别是在第3步、第4步。

# 讲一下你理解的Netty网络通信框架



# netty中的boss和worker如何分配

1. 负责新连接的监听和接收的EventLoopGroup轮询组中的反应器（Reactor），完成查询通道的新连接IO事件查询，因此该轮询组可以形象地称为Boss轮询组。
2. 另一个轮询组中的反应器（Reactor），完成查询所有子通道的IO事件，并且执行对应的Handler处理器完成IO处理——例如数据的输入和输出（有点儿像搬砖），这个轮询组为Worker轮询组。

# Netty核心组件介绍

我把Netty的核心组件分为三层，分别是网络通信层、事件调度层和服务编排层。

1. 网络通信层。在网络通信层有三个核心组件：Bootstrap、ServerBootStrap、Channel
   1. Bootstrap：负责客户端启动并用来链接远程Netty Server；
   2. ServerBootStrap：负责服务端监听，用来监听指定端口；
   3. Channel：相当于完成网络通信的载体。
2. 事件调度层。事件调度器有两个核心组件：EventLoopGroup与EventLoop
   1. EventLoopGroup：本质上是一个线程池，主要负责接收I/O请求，并分配线程执行处理请求。
   2. EventLoop：相当于线程池中的线程
3. 服务编排层。在服务编排层有三个核心组件ChannelPipeline、ChannelHandler、ChannelHandlerContext
   1. ChannelPipeline：负责将多个ChannelHandler链接在一起
   2. ChannelHandler：针对I/O的数据处理器，数据接收后，通过指定的Handler进行处理。
   3. ChannelHandlerContext：用来保存ChannelHandler的上下文信息

# netty的ByteBuf和java的区别

### ByteBuf产生原因

当我们进行数据传输的时候，往往需要使用到缓冲区，常用的缓冲区就是JDK NIO类库提供的java.nio.Buffer。

7种基础类型（Boolean除外）都有自己的缓冲区实现，对于NIO编程而言，我们主要使用的是ByteBuffer。从功能角度而言，ByteBuffer完全可以满足NIO编程的需要，但是由于NIO编程的复杂性，ByteBuffer也有其局限性，它的主要缺点如下。

1. ByteBuffer长度固定，一旦分配完成，它的容量不能动态扩展和收缩，当需要编码的对象大于ByteBuffer的容量时，会发生索引越界异常；
2. ByteBuffer只有一个标识位置的指针position，读写的时候需要手工调用flip()和rewind()等，使用者必须小心谨慎地处理这些API，否则很容易导致程序处理失败；
3. ByteBuffer的API功能有限，一些高级和实用的特性它不支持，需要使用者自己编程实现。

为了弥补这些不足，Netty提供了自己的ByteBuffer实现——ByteBuf。

### 结构

![在这里插入图片描述](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgwatermark%2Ctype_ZmFuZ3poZW5naGVpdGk%2Cshadow_10%2Ctext_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xlbW9uX01Z%2Csize_16%2Ccolor_FFFFFF%2Ct_70.png)
从上面这幅图可以看到，ByteBuf 是一个字节容器，容器里面的的数据分为三个部分：

- 第一个部分是已经丢弃的字节，这部分数据是无效的；
- 第二部分是可读字节，这部分数据是 ByteBuf 的主体数据， 从 ByteBuf 里面读取的数据都来自这一部分;
- 最后一部分的数据是可写字节，所有写到 ByteBuf 的数据都会写到这一段。最后一部分虚线表示的是该 ByteBuf 最多还能扩容多少容量;

### ByteBuf和ByteBuffer的区别

**1.扩容**

ByteBuffer：ByteBuffer缓冲区的长度固定，分多了会浪费内存，分少了存放大的数据时会索引越界，所以使用ByteBuffer时，为了解决这个问题，我们一般每次put操作时，都会对可用空间进行校检，如果剩余空间不足，需要重新创建一个新的ByteBuffer，然后将旧的ByteBuffer复制到新的ByteBuffer中去。

ByteBuf：而ByteBuf则对其进行了改进，它会自动扩展，具体的做法是，写入数据时，会调用ensureWritable方法，传入我们需要写的字节长度，判断是否需要扩容：

注意：

> 1.当申请的新空间大于阀值时，采用每次步进4MB的方式进行扩张内存，而不是倍增，因为这会造成内存膨胀和浪费
> 2.而但申请的新空间小于阀值时，则以64为基数进行倍增而不是步进，因为当内存比较小的时候，倍增是可以接受的（64 -> 128 和 10Mb -> 20Mb相比）

**2.位置指针**

ByteBuffer：ByteBuffer中只有一个位置指针position（ByteBuf有两个），所以需要我们手动得调用flip等方法。ByteBuffer中会有三个下标，初始位置0，当前位置positon，limit位置。

ByteBuf：ByteBuf中使用两个指针，readerIndex，writerIndex来指示位置。

### ByteBufUtil

ByteBufUtil是一个非常有用的工具类，它提供了一系列静态方法用于操作ByteBuf对象。

其中最有用的方法就是对字符串的编码和解码，具体如下。

* encodeString(ByteBufAllocator alloc, CharBuffer src, Charset charset)：对需要编码的字符串src按照指定的字符集charset进行编码，利用指定的ByteBufAllocator生成一个新的ByteBuf；
* decodeString(ByteBuffer src, Charset charset)：使用指定的ByteBuffer和charset进行对ByteBuffer进行解码，获取解码后的字符串。
* hexDump(ByteBuf buffer)：将参数ByteBuf的内容以十六进制字符串的方式打印出来，用于输出日志或者打印码流，方便问题定位，提升系统的可维护性。hexDump包含了一系列的方法，参数不同，输出的结果也不同。

# Netty责任链模式

netty的pipeline设计采用了责任链设计模式, 底层采用双向链表的数据结构, 将链上的各个处理器串联起来。客户端每一个请求的到来,netty都认为,pipeline中的所有的处理器都有机会处理它,因此,对于入栈的请求,全部从头节点开始往后传播,一直传播到尾节点(来到尾节点的msg会被释放掉)。

适用场景:

- 对于一个请求来说,如果有个对象都有机会处理它,而且不明确到底是哪个对象会处理请求时,我们可以考虑使用责任链模式实现它,让请求从链的头部往后移动,直到链上的一个节点成功处理了它为止

优点:

- 发送者不需要知道自己发送的这个请求到底会被哪个对象处理掉,实现了发送者和接受者的解耦
- 简化了发送者对象的设计
- 可以动态的添加节点和删除节点

缺点:

- 所有的请求都从链的头部开始遍历,对性能有损耗
- 极差的情况,不保证请求一定会被处理

netty的责任链模式中的组件

- 责任处理器接口。pipeline中的所有的handler的顶级抽象接口为ChannelHandler，它规定了所有的handler统一要有添加、移除、异常捕获的行为。
- 添加删除责任处理器的接口。netty中所有的处理器最终都在添加在pipeline上，所以添加删除责任处理器的接口的行为netty在channelPipeline中的进行了规定。
- 上下文。pipeline中的handler被封装进了上下文中，通过上下文可以轻松拿到当前节点所属的channel以及它的线程执行器。
- 责任终止机制。
  - 在pipeline中的任意一个节点,只要我们不手动的往下传播下去,这个事件就会终止传播在当前节点
  - 对于入站数据,默认会传递到尾节点,进行回收,如果我们不进行下一步传播,事件就会终止在当前节点,别忘记回收msg
  - 对于出站数据,用header节点的使用unsafe对象,把数据写会客户端也意味着事件的终止

# Netty零拷贝机制

大部分的场景下，Netty的接收和发送ByteBuffer的过程中，一般来说会使用直接内存进行Socket通道读写，使用JVM的堆内存进行业务处理，会涉及到直接内存、堆内存之间的数据复制。但是，内存的数据复制，其实是效率非常低的，Netty供了多种方法，帮助应用程序减少内存的复制。

Netty的零拷贝主要体现在六个方面：

1. Direct Buffer（直接缓冲区）：Netty使用了Direct Buffer（直接字节缓冲区），这是一种特殊类型的字节缓冲区，它使用的内存不是JVM的堆内存，而是操作系统的本地内存。这样，当进行网络读写操作时，数据可以直接在操作系统的本地内存中进行传输，而不需要先复制到JVM的堆内存中，再由JVM的堆内存复制到直接缓冲区中。这样就避免了一次数据复制，提高了数据传输的效率。

2. Netty供CompositeByteBuf组合缓冲区类, 可以将多个ByteBuf合并为一个逻辑上的ByteBuf, 避免了各个ByteBuf之间的拷贝。

   HTTP协议传输消息可能是由多个byteBuf组成的，如果没有这个组合缓冲区类，会有了多次额外的数据拷贝操作。

3. Netty供了ByteBuf的浅层复制操作（slice、duplicate），可以将ByteBuf分解为多个共享同一个存储区域的ByteBuf，避免内存的拷贝。

4. 在进行文件传输时，Netty使用了FileRegion接口，它允许将文件的一部分或者全部直接发送到网络中，而不需要先将文件数据读取到JVM的内存中，再写入到Socket中。这样可以避免数据的多次复制，提高文件传输的性能。

5. 在将一个byte数组转换为一个ByteBuf对象的场景，Netty供了一系列的包装类，避免了转换过程中的内存拷贝。

# Netty心跳机制

Netty 提供了两种方式来实现心跳检测：

1.  使用 TCP 层的 KeepAlive 机制。该机制默认的心跳时间是 2 小时，依赖操作系统实现，不够灵活。 

   TCP在连接没有数据通过后的7200s(tcp_keepalive_time)后会发送keepalive消息，当消息没有被确认后，按75s（tcp_keepalive_intvl）的频率重新发送，一直发送9（tcp_keepalive_probes）个探测包都没有被确认，就认定这个连接失效了。

2.  使用 Netty 的 IdleStateHandler。IdleStateHandler 是 Netty 提供的空闲状态处理器，可以自定义检测间隔时间。通过设置 IdleStateHandler 的构造函数中的参数，可以指定读空闲检测的时间、写空闲检测的时间和读写空闲检测的时间。将它们设置为 0 表示禁用该类型的空闲检测。

可以通过继承SimpleChannelInboundHandler实现一个ChannelHandler，用于处理网络连接中的心跳包。

- `channelRead0`方法：当从通道读取到消息时，该方法将被调用。在这里，它检查接收到的消息是否是"Heartbeat Packet"，如果是，则回复"ok"，否则打印其他信息处理。
- `userEventTriggered`方法：该方法用于处理Netty的超时事件。Netty会定期检查通道是否处于空闲状态，这里的空闲指的是没有读写操作发生。如果有超时事件，Netty将触发此方法。在这个方法中，它统计读空闲的次数，如果超过3次，则发送"idle close"消息并关闭连接。
- `channelActive`方法：当通道激活时，即连接成功建立时，该方法将被调用。在这里，它打印出连接的远程地址。

# Netty长连接如何保持

### 什么是长连接

client向server发起连接，server接受client连接，双方建立连接。Client与server完成一次读写之后，**它们之间的连接并不会主动关闭，后续的读写操作会继续使用这个连接。**

### 生命周期

- 正常情况下，一条TCP长连接建立后，只要双不提出关闭请求&不出现异常情况，这条连接是一直存在.
- 操作系统不会自动去关闭它，甚至经过物理网络拓扑的改变之后仍然可以使用。
- 所以一条连接保持几天、几个月、几年或者更长时间都有可能，只要不出现异常情况或由用户（应用层）主动关闭。客户端和服务单可一直使用该连接进行数据通信。

### 优缺点

#### 优点

- 长连接可以省去较多的TCP建立和关闭的操作，减少网络阻塞的影响，
- 当发生错误时，可以在不关闭连接的情况下进行提示，
- 减少CPU及内存的使用，因为不需要经常的建立及关闭连接。

#### 缺点

- **连接数过多时，影响服务端的性能和并发数量**

<u>通过心跳来实现长连接。</u>

应用层协议大多都有HeartBeat机制，通常是客户端每隔一小段时间向服务器发送一个数据包，通知服务器自己仍然在线。并传输一些可能必要的数据。使用心跳包的典型协议是IM，比如QQ/MSN/飞信等协议。

TCP . SO_KEEPALIVE。系统默认是设置的2小时的心跳频率。但是它检查不到机器断电、网线拔出、防火墙这些断线。而且逻辑层处理断线可能也不是那么好处理。一般，如果只是用于保活还是可以的。

很多网络设备，尤其是NAT路由器，由于其硬件的限制（例如内存、CPU处理能力），无法保持其上的所有连接，因此在必要的时候，会在连接池中选择一些不活跃的连接踢掉。

典型做法是LRU，把最久没有数据的连接给T掉。

# Netty内存模型

![image-20240709204313085](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240709204313085.png)

主要结构分为如下几个部分：

- **PoolThreadCache**, 线程本地缓存，缓存之前申请过的内存对象，初始化时会选择利用率最少的PoolArena对象，基本上每个线程可以拥有自己的PoolArena对象以避免锁竞争。
- **PoolArena**，全局内存分配器，定义了小内存分配的size等级及间隔、一般内存的分配流程，以及超大内存的分配，其主要元素有：
  - **tinySubpagePools**，小内存分配数组，tiny是指小于512B的内存分配，每个数组是`PooSubpage`组成的链表，数组间大小间隔为16B，，所以数组共有32个链表
  - **smallSubpagePools**，小内存分配数组，small是指小于一个page大小（默认8k）的内存分配，每个数组也是由`PooSubpage`组成的链表，数组长度为log(pageSize/512)-1
  - **PoolChunkList**，PoolChunk的列表，可以根据利用率划分多个列表，在内存分配中动态改变
- **PoolChunk**，负责一般内存大小（大于page）的分配。page是内存管理的最小单元，默认8k。chunk是page为单元的集合，默认16M。基于完全平衡二叉树管理chunk，树中的节点是一个page，分配的时候会找到合适层进行page分配。每一层page大小的总和等于chunk大小
- **PoolSubpage**，负责小内存分配（小于page），是从chunk的叶子节点中分配得到。基于bitmap划分page。

# 直接内存为什么性能高 优缺点

  

三种多路复用模式的文件描述符放在那里 → select,poll放在应用程序中，复制到内核中，epoll放到内核中
消费者从发起请求到得到结果，经过的环节

如果让你实现一个类似netty的网络通信框架你会怎么做

channel作用

select poll epoll

水平触发、边缘触发

负载均衡算法：加权随机，加权轮询实现原理

万人群聊如何设计，如何低延迟，如何不利用中间件

Netty在项目里面具体怎么用的

如果有用户连续点击按钮，发送两个相同内容的请求，netty怎么区分并处理的