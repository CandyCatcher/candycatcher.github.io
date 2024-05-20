pipeline我们将分以下几点分析源码：

1、pipeline的初始化 

2、pipeline的handler的添加和删除 

3、inbound事件、outbound事件和异常的传播

研读过pipeline源码，需要能够回答以下三个问题：

1、netty是如何判断ChannelHandler类型的

答：通过传入是inbound还是outbound

2、对于ChannelHandler的添加应该遵循什么样的顺序

答：inbound传播是从Head结点开始的，outbound传播是从Tail节点开始的，所以需要根据需要从这两个方面选择

3、用户手动触发事件传播，不同的触发方式有什么样的区别？

可以这里回答：ctx.channel().write()和ctx.write()的区别，ctx.channel().write()从tail节点开始传播（对于outBound事件），ctx.write()从当前节点开始传播。

# pipeLine在哪里初始化的

无论我们是通过反射创建出服务端的channel也好，还是直接new创建客户端的channel也好，随着父类构造函数的逐层调用，最终我们都会在Channel体系的顶级抽象类`AbstractChannel`中，创建出Channel的一大组件 `channelPipeline`：

![image-20240518020226912](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240518020226912.png)

跟进去,看到,它创建了一个`DefaultChannelPipeline(thisChannel)`。
`DefaultChannelPipeline`是channelPipeline的默认实现,他有着举足轻重的作用,我们看一下下面的 `Channel` `ChannelContext` `ChannelPipeline`的继承体系图,我们可以看到图中两个类,其实都很重要,

![image-20240518020320391](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240518020320391.png)

那么从newChannelPipeline()进去(把自己channel当做参数传递进去)，一直进去：

![image-20240518021106399](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240518021106399.png)

## pipeline的节点的数据结构：ChannelHandlerContext

我们看head和tail，他们最终都继承自ChannelHandlerContext这个接口，不仅如此，pipeline的每一个节点都是这样的数据结构：

![image-20240519225356827](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519225356827.png)

AttributeMap这个很明显，就是保持了key和value的数据类型。

然后我们看ChannelInboundInvoker，

![image-20240519225458483](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519225458483.png)

这里inbound事件，一般就是指读事件、注册事件或者active等；

![image-20240519225911874](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519225911874.png)

里面的这些方法，主要靠这个ChannelOutboundInvoker处理

然后再看channelHandlerContext本身：

![image-20240519230049154](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519230049154.png)

自己的一些方法主要就是上面的几个方法：

* `channel()`说明pipeline的每个节点都要了解自己是哪一个channel的；
* `executor()`说明每个节点都需要知道现在是和哪一个NioEventLoop绑定在一起的；
* `handler()`说明真正在做事情的就是这个handler；
* 还有一个ByteBufAllocator，内存的分配器，这就说明在这个节点这里，如果有数据读写要分配ByteBuf的时候，需要使用哪一个内存分配器去分配。

AbstractChannelHandlerContext是ChannelHandlerContext这个类的实现，实现了大部分的功能，里面主要有个next和prev的指针，把这些handler连起来。

## pipeline中的两大哨兵

### TailContext

TailContext是在DefaultChannelPipeline的内部类：

![image-20240519231048761](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519231048761.png)

TailContext的构造函数，inbound为true，outbound为false（说明他是一个inbound处理器）。handler就是它自己this（说明他不仅是一个节点而且业务处理器也是它自己）。总体来说，只处理以下两个事件：

![image-20240519231132763](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519231132763.png)

发生了一些异常（或者读事件）了，就要处理，但是最终处理异常（或者读事件）信息的时候，只是把它打印出来（如果是读事件的话，会把message看看是否能释放，可以释放就释放了）：

![image-20240519231236706](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519231236706.png)

可以看到tail主要做一些收尾的信息，如果你有一些事件没处理，它就帮你处理了。

### HeadContext

它和tailContext的区别就之一多实现了一个ChannelOutboundHandler接口：

![image-20240519231448222](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519231448222.png)

然后还多了一个成员变量Unsafe，如果是客户端的有客户端的实现（NioByteUnsafe），服务端的有服务端的实现（NioMessageUnsafe），用于实现底层数据的读写。

注意到它有的构造函数inbound是false，outbound是true，和TailContext相反（当时和我们直觉相反）。

然后它也实现了绑定连接等操作，绑定的话我们在服务端绑定端口的已经说过了：

![image-20240519231804300](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519231804300.png)

HeadContext和TailContext区别很大，HeadContext主要是把事件原模原样地往下传播，传播事件的时候，都会从head开始；在一些进行读写操作的时候，都会委托到head的unsafe这里操作，也就是HeadContext负责channel具体协议的实现。

而TailContext负责终止事件和异常传播的作用。

# 添加channelHandler



添加[handler](https://so.csdn.net/so/search?q=handler&spm=1001.2101.3001.7020)由用户代码addLast进入，最终会来到DefaultChannelPipeline。

![image-20240519232459401](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519232459401.png)

添加channelHandler分为以下几个步骤：

1. 判断是否重复添加
2. 创建节点并添加至链表
3. 回调添加完成事件

## 判断是否重复添加

从这个方法进入`checkMultiplicity(handler);`

![image-20240519233119413](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519233119413.png)

首先判断是否ChannelHandlerAdapter的实例，是，强转，

> 如果不是的话，不做处理，说明handle都是Adapter的？

判断如果不是分享的，并且添加过，就报错。如果添加成功之后，h.added=true,表示已经添加过。

> 看一下isSharable方法

## 创建节点并添加至链表

回到一开始的代码，创建节点和添加链表：

![image-20240519233434204](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519233434204.png)

我们可以大概的想象到，这个newContext把handler包装成context，然后添加到这个pipeline的是这个context也就是newCtx。

首先我们看看filterName做了什么，表面上就是过滤名字的意思。如果名字为空，那就创建名字；否则，就判断名字是否重复：

![image-20240519233722653](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519233722653.png)

newContext看看怎么包装吧：

![image-20240519233943209](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519233943209.png)

然后是添加addLast0(newCtx);

![image-20240519234235266](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519234235266.png)

这就是双向链表在尾部添加节点的操作，但是这个节点必须在tail节点之前，也就是保证tail必须在最后面。

## 回调添加完成事件

![image-20240519234353568](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519234353568.png)

从`callHandlerAdded0(newCtx)`进入：

![image-20240519224722689](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimgimage-20240519224722689.png)

因为我们实现的是ChannelInitializer类，所以会回调到这里来：

![image-20240519235412308](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519235412308.png)

然后进入本类的

![image-20240519235546477](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519235546477.png)

然后这个方法`initChannel((C) ctx.channel());`里面就调用到用户代码实现的匿名方法initChannel：

![image-20240520000719094](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240520000719094.png)

最后通过setAddComplete设置添加完成状态，借助cas的操作：

![image-20240520000933413](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240520000933413.png)

# 删除channelHandler

删除channelHandler主要分为以下几个步骤：

1. 找到节点
2. 链表里面删除找到的那个节点
3. 回调删除handler事件

假设有这么一个handle，如果通过了验证，就把这个authHandler删除掉（如果没有通过验证，就关闭这个连接）：

![image-20240520003312994](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240520003312994.png)

点进去remove开始分析：

![image-20240520004145049](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240520004145049.png)

## 找到节点

handler是被handleContext包装的。

上面的getContextOrDie名字取得有意思，通过handler找到context，没有找到就抛出异常

![image-20240520004302879](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240520004302879.png)

看看怎么找到context

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240520004339846.png" alt="image-20240520004339846" style="zoom:67%;" />

粗暴得很，居然是从head的下一个节点开始找，找ctx的head和当前的一样，就返回。

## 删除找到的那个节点

![image-20240520004617551](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240520004617551.png)

首先需要`assert ctx != head && ctx != tail;`也就是保证不是head或者tail，在netty里面，这两个节点是不能删除的。

同步代码块里面，remove0()是执行真正的逻辑删除：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240520004703331.png" alt="image-20240520004703331" style="zoom:67%;" />

至于callHandlerRemoved0方法，就是回调删除事件。

![image-20240520005755699](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240520005755699.png)

## 回调删除handler事件

![image-20240520005255052](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240520005255052.png)

# inBound事件传播

## 何为inBound事件

先看ChannelHandler的源码：

<img src="https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240520092514099.png" alt="image-20240520092514099" style="zoom:67%;" />

接口就是定义规范，这里定义了最基本的，add，remove，exception发生各自的回调。

ChannelHandlerAdapter就是实现了isSharable方法。如果实现是shareable，则返回true，因此可以添加到不同的link ChannelPipeline。

![image-20240520093142277](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240520093142277.png)

然后我们看ChannelInboundHandler，这个类的作用主要是当channel状态更改的时候回调到用户。

<img src="https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240520094827163.png" alt="image-20240520094827163" style="zoom:67%;" />

它为ChannelHandler状态更改添加回调。这使用户可以轻松地挂钩到状态更改。

## ChannelRead事件的传播

我们以ChannelRead事件为例来分析inBound事件：

``` java
public final class Server {

    public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childOption(ChannelOption.TCP_NODELAY, true)
                .childAttr(AttributeKey.newInstance("childAttr"), "childAttrValue")
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    public void initChannel(SocketChannel ch) {
                        ch.pipeline().addLast(new InBoundHandlerA());
                        ch.pipeline().addLast(new InBoundHandlerB());
                        ch.pipeline().addLast(new InBoundHandlerC());
                    }
                });

            ChannelFuture f = b.bind(8888).sync();

            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

每个InBoundHandlerX()，X代表A或者B或者C，里面都是这样实现的：

``` java
public class InBoundHandlerA extends ChannelInboundHandlerAdapter {
 
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("InBoundHandlerA: " + msg);
        ctx.fireChannelRead(msg);
    }
}
```

运行之后，我们telnet一下本地端口8888，能得到一下结果：

``` java
InBoundHandlerA: hello world
InBoundHandlerB: hello world
InBoundHandlerC: hello world
```


这说明，InBound事件的传播，是按照添加顺序来的。

也就是Head-->A-->B-->C--->Tail

那么我们进入HeadContext的ChannelRead，有新连接进来之后先调用这个方法：



# OutBound事件传播

为什么会从用户代码的addLast进来呢？

是在 jdk原生的channel注册进EventLoop中的Selector后紧接着回调的,源码如下：

![image-20240518022347554](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240518022347554.png)

回调函数在`pipeline.invokeHandlerAddedIfNeeded();`, **看它的命名, 如果需要的话，执行handler添加操作** 哈哈,我们现在当然需要,刚添加了个`ServerBootstrapAcceptor`。

方法是pipeline调用的, 哪个pipeline呢? 就是上面我们说的`DefaultChannelPipeline`, ok,跟进源码,进入 `DefaultChannelPipeline`

![image-20240519223457935](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519223457935.png)

> 那handler是什么时候注册的？

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519223714286.png" alt="image-20240519223714286" style="zoom: 67%;" />

其中的task是谁? `pendingHandlerCallbackHead` 这是`DefaultChannelPipeline`的内部类, 它的作用就是辅助完成添加handler之后的回调, 源码如下:

![image-20240519223955991](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519223955991.png)

我们跟进上一步的`task.execute()`就会看到它的抽象方法，那么是谁实现的呢？实现类是`PendingHandlerAddedTask`同样是`DefaultChannelPipeline`的内部类,，既然不是抽象类了，就得同时实现他父类`PendingHandlerCallback`的抽象方法，其实有两个一是个`excute()`；另一个是`run()` 

![image-20240519224154967](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519224154967.png)

我们往下追踪, 调用类本类方法`callHandlerAdded0(ctx);` 源码如下:

![image-20240519224722689](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240519224722689.png)

`ctx.handler()` 获取到了当前的channel，然后调用channel的`handlerAdded()`方法。

这个`handlerAdded()`是定义在ChannelHandler中的回调方法， **什么时候回调呢?** 当handler添加后回调，因为我们知道，当服务端的channel在启动时，会通过 channelInitializer 添加那个`ServerBootstrapAcceptor`，所以`ServerBootstrapAcceptor`的`handlerAdded（）`的回调时机就在上面代码中的`ctx.handler().handlerAdded(ctx);`

如果直接点击去这个函数,肯定就是`ChannelHandler`接口中去; 那么 新的问题来了,谁是实现类? **答案是抽象类 ChannelInitializer**`` 就在上面我们添加`ServerBootstrapAcceptor`就创建了一个`ChannelInitializer`的匿名对象
