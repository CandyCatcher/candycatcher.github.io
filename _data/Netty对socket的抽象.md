![image-20240418231014509](/Users/candyboy/Library/Application Support/typora-user-images/image-20240418231014509.png)

1. 首先服务端监听某一个端口，在一个循环中一直接受新连接的进入。
2. 新用户的连接在java底层是作为一个socket接入的，在netty中把socket封装成channel
3. 服务端接受客户端的数据流的读写都是在ByteBuf中
4. netty把各个业务逻辑的处理作为一个channelHandler

## 基本组件1

### NioEventLoop

在demo里面，线程有两种，server接收的线程以及处理连接信息的线程：

这里的run方法就是对应的服务端的两个while(true)方法：

![image-20240420004027363](/Users/candyboy/Library/Application Support/typora-user-images/image-20240420004027363.png)

然后run方法里面的select方法对应的就是`serverSocket.accept()`方法

拿到用户端的socket之后，通过`processSelectedKeys`处理拿到的连接

![image-20240420004346878](/Users/candyboy/Library/Application Support/typora-user-images/image-20240420004346878.png)

可以看到这里面处理每一个连接：

![image-20240420004737215](/Users/candyboy/Library/Application Support/typora-user-images/image-20240420004737215.png)

### channel

channel对应的就是socket。

`processSelectedKeys`方法里面，如果有`OP_ACCEPT`的事件进来的话，调用read方法。

![image-20240420005508491](/Users/candyboy/Library/Application Support/typora-user-images/image-20240420005508491.png)

read方法的实现有两种类，NioMessageUnsafe指的是有新连接进来的时候

![image-20240420005711411](/Users/candyboy/Library/Application Support/typora-user-images/image-20240420005711411.png)

进来看read方法，通过doReadMessages读取readBuf中的数据

![image-20240420010008198](/Users/candyboy/Library/Application Support/typora-user-images/image-20240420010008198.png)

点击`doReadMessages`方法，可以看到java底层的逻辑了，点击javaChannel()

![image-20240420010204079](/Users/candyboy/Library/Application Support/typora-user-images/image-20240420010204079.png)

可以看到，ServerSocketChannel对应的就是nio中的socketChannel

![image-20240420010536437](/Users/candyboy/Library/Application Support/typora-user-images/image-20240420010536437.png)

然后将ch进行封装成自定义的NioSocketChannel

![image-20240420010724323](/Users/candyboy/Library/Application Support/typora-user-images/image-20240420010724323.png)

![image-20240420010919102](/Users/candyboy/Library/Application Support/typora-user-images/image-20240420010919102.png)

### ByteBuffer

### PipeLine

pipeLine对应的就是这一段代码，处理业务逻辑链：

![image-20240420011321217](/Users/candyboy/Library/Application Support/typora-user-images/image-20240420011321217.png)

 在创建`NioSocketChannel`的时候，往上找父类就能看到`newChannelPipeline`

![image-20240420011733763](/Users/candyboy/Library/Application Support/typora-user-images/image-20240420011733763.png)

### ChannelHandler

ChannelHandler对应的就是具体的逻辑快 ，里面存了一些回调方法：

![image-20240420012651346](/Users/candyboy/Library/Application Support/typora-user-images/image-20240420012651346.png)