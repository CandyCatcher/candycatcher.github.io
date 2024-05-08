常见的netty启动代码：

``` java
private static final EventLoopGroup bossGroup = new NioEventLoopGroup();
private static final EventLoopGroup workerGroup = new NioEventLoopGroup();

public void nettyInit() {
    log.info("tcp open port ：{}", port);

    try {
        // ServerBootstrap是一个用来创建服务端Channel的工具类，创建出来的Channel用来接收进来的请求
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap.group(bossGroup, workerGroup)
            // 使用NioSocketChannel 作为服务器的通道实现
            .channel(NioServerSocketChannel.class)
            // 设置线程队列得到连接个数(也可以说是并发数)
            .option(ChannelOption.SO_BACKLOG, 2048)
            //设置保持活动连接状态
            .childOption(ChannelOption.SO_KEEPALIVE, true)
            //设置NoDelay禁用Nagel,消息会立即发送出去,不用等到一定数量才发送出去
            //.childOption(ChannelOption.TCP_NODELAY, true)
            //这里的log是boss group的日志级别
            .handler(new LoggingHandler(LogLevel.DEBUG))
            // 用来指定具体channel中消息的处理逻辑
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel socketChannel) throws Exception {
                    // 注册handler
                    socketChannel.pipeline()
                        //                                    .addLast(new IdleStateHandler(10, 0,0, TimeUnit.SECONDS))
                        // 解码器
                        //.addLast(new MessageDecoder())
                        // 处理器
                        .addLast(new SimpleServerHandler());
                }
            });
        // 绑定端口，开始接收进来的连接
        //            InetSocketAddress address = new InetSocketAddress(host, por
        // 启动服务器(并绑定端口)
        final ChannelFuture channelFuture = serverBootstrap.bind(port).sync();
        // 给channelFuture 注册监听器，监控我们关心的事件
        channelFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (channelFuture.isSuccess()) {
                    log.info("listening port [{}] success!", port);
                } else {
                    log.error("listening port [{}] fail!", port);
                }
            }
        });
        log.info("======Startup Netty Server Success!=========");
        channelFuture.channel().closeFuture().sync();
    } catch (Exception e) {
        log.error(" netty service startup exception:{}", e.getMessage());
        throw new RuntimeException(e.getMessage());
    }
}
```

* 服务端的socket在哪里初始化？
* 在哪里进行accept连接？

netty服务端启动的步骤：

* 创建服务端channel。这个过程就是调用jdk底层的逻辑创建jdk底层的channel，然后netty将其包装成自己的channel，同时创建一些基本组件绑定在此channel上面。
* 初始化服务端channel。创建完channel后，netty会做一些初始化的工作，比如初始化一些基本属性以及添加一些逻辑处理器。
* 注册Selector。netty将jdk底层的channel注册到事件轮询器selector上面，并把netty的服务端channel作为一个attachMent绑定到对应的jdk底层的channel上，最后调用doBind()方法来实现对本地端口的监听后，netty会重新向selector注册一个Accept事件，这样netty就可以接受新的连接了。
* 端口绑定

## 创建服务端channel

<img src="https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240422112507772.png" alt="image-20240422112507772" style="zoom: 33%;"/>

创建服务端channel是从用户代码的bind方法进入的，一直往下点，就能看到initialAndRegister方法，

![image-20240424102338874](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240424102338874.png)

>  initialAndRegister 方法名清晰明了

可以看到一个channelFactory创建了服务端channel：

![image-20240424110512999](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240424110512999.png)

* ChannelFuture是什么？

channelFactory的实现类为ReflectiveChannelFactory，在这里是通过反射调用构造函数创建channel实例的：

![image-20240424110644447](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240424110644447.png)

clazz是在用户代码里面传过来的，这样的话就能根据用户传来的类创建对应的对象：

![image-20240424110926214](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240424110926214.png)

点进channel方法，可以看到这里把传进来的class类封装成了ReflectiveChannelFactory，并返回：

![image-20240424111115243](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240424111115243.png)

### NioServerSocketChannel的构造函数做了什么？

<img src="https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240424111555441.png" alt="image-20240424111555441" style="zoom:25%;" />

NioServerSocketChannel的构造函数：

![image-20240424111509383](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240424111509383.png)

先创建jdk底层的channel：

![image-20240424111643708](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240424111643708.png)

* ServerSocketChannel
* SelectorProvider



![image-20240424112641863](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240424112641863.png)

> channel 和 javaChannel()是一个对象

调用NioServerSocketChannelConfig的构造函数，tcp参数配置类

![image-20240424140659151](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240424140659151.png)

赋值创建的ServerSocketChannel给成员变量ch，然后配置ch的属性为非阻塞的。如果不使用netty进行网络编程的话，那么这一步是必不可少的？

![image-20240424140758892](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240424140758892.png)

> 反正要抛出异常停止程序了，在catch中还有异常catch就打印一个错误日志

## 初始化Channel

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424212939907.png" alt="image-20240424212939907" style="zoom:40%;" />

在这里可以看到新建channel之后，就进行初始化的操作：

![image-20240424213257088](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424213257088.png)

进入init，首先将用户自定义的channelOptions，ChannelAttrs分别绑定到channel上面。分别通过options0()和attrs0()拿到用户配置的属性，然后通过set方法绑定到channel上面：

![image-20240424213651360](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424213651360.png)

这里也是将配置childOptions的放进去：

![image-20240424214148287](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424214148287.png)

接着就是将用户自定的handler添加到pipeLine中（pipeLine是在创建channel的时候创建的）：

![image-20240424214709663](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424214709663.png)

 用户创建的handler是在AbstractBootstrap中的一个成员变量：

![image-20240424215454719](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424215454719.png)

> 虽然是自己的成员变量，但是还是通过创建一个config的对象用来管理自己的变量：
>
> ![image-20240424215720147](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424215720147.png)

ServerBootstrapAcceptor是一个特殊的handler，netty默认添加的，这是把一个新连接绑定到一个新线程上面：

![image-20240424215905778](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424215905778.png)

## 注册Selector

把channel放在事件轮训器上面：

![image-20240424222839402](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424222839402.png)

还是从initAndRegister方法开始：

![image-20240424222956077](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424222956077.png)

> register方法是在AbstractBootstrap类中的，是怎么从register方法从AbstractBootstrap到AbstractChannel的？(明天看一下)
>
> 

点进register方法：

`AbstractChannel.this.eventLoop = eventLoop`这行的含义就是告诉channel之后的所有的操作都交给eventLoop 

![image-20240424223708977](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424223708977.png)

然后就是通过register0进行处理：

![image-20240424225044969](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424225044969.png)

来到具体的执行地方，可以看到还是调用了jdk底层的方法将channel绑定到了selector上面  

![image-20240424224152014](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424224152014.png)

> jdk原生的这个方法是什么含义？
>
> selector当轮训到这个channel上面有的io操作的时候，可以直接将attachment拿出来，针对netty的niochannel做一些事件的传播处理 

接着就是执行doRegister中的两个回调方法：

![image-20240424225254314](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424225254314.png)

实际上就是对应着用户自定义类中的这两个方法：

![image-20240424225545439](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424225545439.png)

## 端口绑定

![image-20240424225900813](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424225900813.png)

还是回到doBind方法：

![image-20240424230138984](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424230138984.png)

一直跟进去，来到AbstractChannel：

![image-20240424230324365](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424230324365.png)

> 怎么一步步调用进来的？

doBind方法点进来之后，进入实际执行方法，可以看到调用了jdk方法进行了端口的绑定：

![image-20240424230538186](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424230538186.png)

然后就有这么一个逻辑：

```java
if (!wasActive && isActive()) { // 之前没有绑定，现在绑定了，那么就执行这个方法
  invokeLater(new Runnable() {
    @Override
    public void run() {
      pipeline.fireChannelActive();
    }
  });
}
```

> 好多地方都是new 一个任务执行方法的

在这里可以看到，fireChannelActive实际是在端口绑定完成之后触发并传播给用户代码的：

![image-20240424231321293](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424231321293.png)

然后第二个方法，点进进去之后：  

![image-20240424231638496](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424231638496.png)

继续点进去，传播到tail：

![image-20240424231712542](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424231712542.png)

> 为什么传播到了最末尾的节点？

然后会回到这里来？

![image-20240424232323661](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240424232323661.png)
