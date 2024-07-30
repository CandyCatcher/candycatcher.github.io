# 抽象解码器ByteToMessgeDecoder

ByteMessageDecoder是所有[解码](https://so.csdn.net/so/search?q=解码&spm=1001.2101.3001.7020)器的基类，它主要通过以下步骤进行解码：

1. 累加字节流
2. 调用子类的decode方法进行解析
3. 将解析到的ByteBuf向下传播

代码是从ByteMessageDecoder的channelReader方法开始的：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240616233723930.png" alt="image-20240616233723930" style="zoom:50%;" />

传到这里来的是一个ByteBuffer的message，先判断一下如果message是ByteBuffer类型的，就进行解码器的处理；如果不是ByteBuffer类型的，直接将这个对象向下进行传播。

> channelReader方法是怎么传播过来的？

看message是ByteBuffer类型的处理逻辑。

## 累加字节流

首先判断如果累加器cumulation为空，说明这是第一次从IO流中读取数据， 就直接往累加器赋值刚读进来的ByteBuffer对象； 否则就通过cumulate这个方法去把当前累加器中的数据和读进来的数据累加。

看看这个累加器cumulator的定义：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240616234608398.png" alt="image-20240616234608398" style="zoom:50%;" />

接着看这个MERGE_CUMULATOR：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240616234917675.png" alt="image-20240616234917675" style="zoom:50%;" />

这是一个匿名内部类的实现，看if判断的逻辑，我们把它转换一下其实就是：`cumulation.writerIndex()+ in.readableBytes() > cumulation.maxCapacity()`，意思就是如果原来的数据加上读进来的数据超过了最大容量，那就增加容量。否则直接将累加器cumulation赋值给buffer，然后将字节流写入到buffer中，最后释放这个流。

看一下增加扩容的方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240616235457018.png" alt="image-20240616235457018" style="zoom:50%;" />

看了下就是每次只增加in进来这么多的字节。

## 调用子类的decode方法进行解析

回到channelRead方法，调用方法：

```java
callDecode(ctx, cumulation, out);
```

out的定义是这样的：

```java
CodecOutputList out = CodecOutputList.newInstance();
```

可以讲out简单的理解为一个arrayList。简单来说，cumulation调用callDecode方法解析字节流，将解析完的数据放在out队列中。 

看一下callDecode方法代码：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240617001547221.png" alt="image-20240617001547221" style="zoom:50%;" />

这个方法实际上是一个while循环，只要累加器里面有数据，就会一直循环读取。

看一下while循环里面的逻辑：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240617001917972.png" alt="image-20240617001917972" style="zoom: 50%;" />

第一个if，即判断这个list里面是否已经有对象了，如果有对象的话，那么就调用一个事件传播机制向下传播，然后将list清空。接着是fix一个bug，如果上下文ctx被删除，那就break。

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240617002508715.png" alt="image-20240617002508715" style="zoom:50%;" />

接下来在调用子类的decode方法之前， 把当前可读的字节的长度记录下来。 

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240617002735391.png" alt="image-20240617002735391" style="zoom:67%;" />

然后第二个判断，如果out原先的长度和decode完后out的长度相同，有两种情况分别处理：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240617002856961.png" alt="image-20240617002856961" style="zoom:50%;" />

if的意思是从累加器中并没有读取到数据，那就说明累加器中的数据并不足以拼装成一个完整的数据包，那就直接break，等待下一次数据在等一些数据进来后重新组装。else的意思是虽然没有解析出来数据，但是从in中读取了一点数据，只不过还没有解析到对象，那就继续while循环，下一次子循环说不定就能组装成对象。

接着来到这个if， 来到这里说明已经通过decode方法解析出来数据了。if判断的是有没有从累加器中读取到数据，那么就会抛出异常。 

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240617004128880.png" alt="image-20240617004128880" style="zoom:50%;" />

 最后一个if，如果每次只解析一个对象就够了，那么直接break。

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240617004456526.png" alt="image-20240617004456526" style="zoom:50%;" />

## 将解析到的ByteBuf向下传播

在finally就是把解析到的数据继续往下传播：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240617004938234.png" alt="image-20240617004938234" style="zoom:50%;" />

主要就是`fireChannelRead`方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240617005134955.png" alt="image-20240617005134955" style="zoom: 50%;" />

就是把解析得到的对象一个个往下传播，就有可能传播到业务的解码器中。

# 基于固定长度解码器分析

通过上一节分析，我们已经知道具体的解码是交给子类解码器进行解析的，子类decode方法如果解析出来一个对象，那么Netty底层的抽象解码器就会将解析出来的对象向下进行传播，业务里的channelHandle就会接收到这个对象。

固定长度解码器FixedLengthFrameDecoder比较简单，看下类的注释：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240617010031443.png" alt="image-20240617010031443" style="zoom:50%;" />

如果数据包是`A|BC|DEFG|HI`以3为长度将会解析成`ABC|DEF|GHI`，理解起来可以说是很简单了。继承自ByteToMessageDecoder 。

这个类只有一个成员变量frameLength：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240617010447353.png" alt="image-20240617010447353" style="zoom:50%;" />

如果解析到数据，就放到out里面，解析方法在decode里面：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimgimage-20240617010617792.png" alt="image-20240617010617792" style="zoom:50%;" />

首先判断可读字节数量是否小于frameLength，如果小的话说明不能读出来，返回null，out的size不会变。如果可读的大于或者等于frameLength，那就读数。

# 基于行解码器分析

首先看这个类的一些成员变量： 

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618225117409.png" alt="image-20240618225117409" style="zoom:50%;" />

* maxLength：行解码器最长解析的字符长度，超过这个长度将不解析。
* failFast：当到达maxLength的时候，是否抛出异常，还是等到有换行符的时候才抛出异常。
* stripDelimiter：解析字符串成对象的时候，是否把字符串也加入到对象中。
* discarding：是否进入丢弃模式。
* discardedBytes：当前已经丢弃的字节数。

它重载的decode方法和基于固定长度解码器一样，也是把解析到的对象放到out列表里面：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618225344119.png" alt="image-20240618225344119" style="zoom:50%;" />

最重要的是它实现的`deocode(ctx,in)`方法:

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618225536569.png" alt="image-20240618225536569" style="zoom:50%;" />

首先看一下`findEndOfLine`方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618225653829.png" alt="image-20240618225653829" style="zoom:50%;" />

FIND_LF就是`\n`这个字节，i就是`\n`的index。

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618225744854.png" alt="image-20240618225744854" style="zoom:50%;" />

如果找到了`\n`，并且前一个字节是`\r`，那么就将index-1，重新指向`\r`。

接下来看非丢弃模式。 

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618231530211.png" alt="image-20240618231530211" style="zoom:50%;" />

1. eol就是end of line，也就是读到了换行符

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618232051142.png" alt="image-20240618232051142" style="zoom:50%;" />

   首先会计算一下换行符到可读字节的长度，也就是eol-readIndex：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618232911545.png" alt="image-20240618232911545" style="zoom:67%;" />

   然后再拿到换行符的长度：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618232929477.png" alt="image-20240618232929477" style="zoom:67%;" />

   如果readIndex到eol的这段距离大于maxLength，也就是length>maxLength。 Netty是抛弃length这一段加上delimLength这一段，也就是把readIndex变为eol+delimLength，然后fail来传播异常，并且返回空表示什么也没有解析到：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618233002461.png" alt="image-20240618233002461" style="zoom:67%;" />

   说明这些解析到的数据是正常的了，通过stripDelimiter判断需不需要跳过换行符，然后在返回得到的数据：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618233023770.png" alt="image-20240618233023770" style="zoom:67%;" />

   `buffer.readRetainedSlice(length)`这个操作后index指向换行符，然后再跳过换行符。

2. 如果没有读到eol

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618233154062.png" alt="image-20240618233154062" style="zoom: 50%;" />

   没有读到换行符，也就是readIndex到writeIndex之间没有换行符：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618233244233.png" alt="image-20240618233244233" style="zoom:50%;" />

   首先判断有没有超过最大长度，如果超过最大长度，把丢弃的字节个数记录下来，然后把readIndex移动到witerIndex，也就是readIndex到writeIndex这一段都是要抛弃。并设置为丢弃模式。

3. 如果是丢弃模式

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618233725754.png" alt="image-20240618233725754" style="zoom: 50%;" />

   如果读到了eol，也就是读到了换行符，看一下代码：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618233947648.png" alt="image-20240618233947648" style="zoom:50%;" />

   那么我们就要抛弃readIndex到eol这一段数据再加上之前记录的discardedBytes，接下来直接将readIndex指向eol的下一个可读字节。重置为非丢弃模式。然后如果非快速失败，这个时候就可以执行fail了。上面执行fail和这里执行fail是互斥事件，上面执行这里就不会执行，否则这里不执行上面就执行。

4. 如果没有读到eol，也就是没有读取到换行符

   这种情况比较简单，如果是丢弃模式，并且没有读到换行符，那就继续继续需要失败的字节，并抛弃这些字节，也就是把readIndex移动到wirteIndex这里。

# 基于分隔符解码器分析

DelimiterBasedFrameDecoder是基于换行符`\n、\r\n`将字节流解析成一个一个的数据包：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618224920203.png" alt="image-20240618224920203" style="zoom: 50%;" />

首先我们看一个构造函数：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618234805024.png" alt="image-20240618234805024" style="zoom:50%;" />

构造函数的`maxFrameLength`，这个也就是我们能解析的字符最大多大，大于这个的解析出来的内容都要丢弃。然后`ByteBuf... delimiters`的参数，表示可以有多个分隔符。

可以预见，这个类也会重载`decode`方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618225344119.png" alt="image-20240618225344119" style="zoom:50%;" />

看他真正的实现方法：

<img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618235932383.png" alt="image-20240618235932383" style="zoom:50%;" />

分隔符解码分为下面这几步

1. 行处理器

   如果行处理器不为空，则调用行处理器。

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240618235701328.png" alt="image-20240618235701328" style="zoom: 50%;" />

   他之前判断过分隔符是不是行分隔符，如果是，就给lineBaaseDecoder赋值，到了这里，就会进入这个if。

   lineBasedDecoder在什么地方初始化呢？

   在`DelimiterBasedFrameDecoder`的构造函数里面有这样一个逻辑：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240619000157031.png" alt="image-20240619000157031" style="zoom: 50%;" />

    如果判断是行分隔符那就实例化行处理器，给lineBasedDecoder赋值。看isLineBase方法的源码：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240619000248866.png" alt="image-20240619000248866" style="zoom:50%;" />

   这一段的逻辑很简单， 就是判断分隔符是否是`\r\n\n`，如果是这个分隔符的话，那么就使用行解码器。

2. 找到最小分隔符

   因为分隔符解码器支持多个分隔符进行分隔，Netty在解码的时候会找到最小分隔符来拆分数据包。

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240619001316793.png" alt="image-20240619001316793" style="zoom:50%;" />

   在for循环中，计算每一个分隔符分割的帧长度，并且计算最小帧长度，如果小于最小就交换，并且把当前的分隔符作为最小分隔符。 

   ![image-20240619001942786](https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240619001942786.png) 

   在这个数据包中，最小长度是从readIndex到A的，A也是最小分隔符。

3. 解码

   解码的逻辑是下面这一段：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240619002126762.png" alt="image-20240619002126762" style="zoom:50%;" />

   1. 如果找到了分隔符（这个分隔符是最小分隔符）

      1. 如果是丢弃模式

         也就是`discardingTooLongFrame`为true。 设置为非丢弃模式；丢弃掉之前记录的数据；如果不是快速失败，就在这里抛出异常。

      2. 如果不是丢弃模式

         <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240619002457639.png" alt="image-20240619002457639" style="zoom:67%;" />

         如果解析到的数据大于maxFrameLength，丢弃掉这一段数据，并且用fail来向下传递异常。

         如果能继续的话，则继续判断是否包含分隔符，和行分隔符一样。

   2. 如果没有找到分隔符

      1. 如果不是丢弃模式

         <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240619003236018.png" alt="image-20240619003236018" style="zoom:50%;" />

         如果readableBytes长度已经超过了最大可允许的长度，那就把要丢弃的长度记录下来，并且跳过这一段，并且标记进入丢弃模式，并且如果是快速丢弃，就从fail里面抛出异常；

         如果readableBytes长度没有超过了最大可允许的长度，不做任何操作，继续等以后的数据。

      2. 如果是丢弃模式

         <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240619003514482.png" alt="image-20240619003514482" style="zoom:50%;" />

         添加需要丢弃的字节到tooLongFrameLength里面，然后跳过这段数据。

# 基于长度域解码器分析

长度域[解码](https://so.csdn.net/so/search?q=解码&spm=1001.2101.3001.7020)器的几个参数：

* **lengthFieldOffset** ：长度域的偏移量，也就是长度域要从什么地方开始
* **lengthFieldLength**：长度域的长度，也就是长度域占多少个字节
* **lengthAdjustment**：长度域的值的调整，也就是我们去到长度域里面的值，然后还要做多少调整才是符合要求的，这个值可以是正值，可以是负值，正值表示还要加多少，负值表示还要减去多少
* **initialBytesToStrip**：原始需要跳过多少才返回给用户

下面看代码，直接看decode方法：

1. 计算需要抽取的数据包的长度

   第一段代码是丢弃模式的，我们直接跳过看第二段：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623001200193.png" alt="image-20240623001200193" style="zoom: 67%;" />

   lengthFiledEndOffset在LengthFieldBasedFrameDecoder的构造器里面初始化：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623001420598.png" alt="image-20240623001420598" style="zoom:67%;" />

   代表长度域偏移加上长度域长度，在下面这个例子就是08的后一位：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623001623098.png" alt="image-20240623001623098" style="zoom: 33%;" />

   如果`in.readableBytes()<lengthFieldEndOffset`的话，说明数据包不完整，连lengthFiledOffset的长度都达不到，数据包肯定不完整，所以返回空，然后累加器就会继续累加，知道数据包完整为止。

   接下来看这一段：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623002416880.png" alt="image-20240623002416880" style="zoom:50%;" />

   lengthFiledOffset是我们的相对偏移量，加上readerIndex就是在这个byteBuf里面的绝对偏移量了，通过这个绝对偏移量，我们后续的操作就可以变得更加简单。

   接着通过in这个byteBuf，actualLengthFieldOffset实际偏移量和lengthFieldLength就能找出我们信息的长度。点击去看怎么实现的：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimgimage-20240623003845464.png" alt="image-20240623003845464" style="zoom:50%;" />

   这个方法就是获得长度域表示的值，可以看到LengthFieldBasedFrameDecoder只支持1、2、3、4、8这几个长度。

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623004837432.png" alt="image-20240623004837432" style="zoom:50%;" />

   如果取出的frameLength<0，那就是不符合条件的，抛出异常。

   长度域里面内容不一定表示的就是实际内容的长度，它可能加上长度域的长度或者head或者两者都有，所以我们通过这个来调整。所以最终frameLength就是frameLength加上调整值，再加上lengthEndOffset。

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623001623098.png" alt="image-20240623001623098" style="zoom: 33%;" />

   由于长度域表示的值是8，这个8包括了长度域2和长度域前面的两个内容2，当然还有最终的信息4，所以是8，所以需要调整 -4，也就是减去长度域2和长度域之前的内容2。最终上面那个表达式的值就是8，也就是最终读取到的整段信息的内容。

   这个frameLength是经过调整之后的frameLength，经过调整之后的frameLength是整段信息内容也就是包括lengthFieldEndOffset的，但是现在小于lengthFieldEndOffset明显不对，跳过lengthFieldEndOffset这段内容，并且抛出异常：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623010956190.png" alt="image-20240623010956190" style="zoom: 50%;" />

   如果需要跳过的直接大于当前已有的字节，当然也是不对的，继续抛出异常。

2. 跳过字节逻辑处理

   如果需要跳过的直接大于当前已有的字节，当然也是不对的，继续抛出异常。

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623011858817.png" alt="image-20240623011858817" style="zoom: 50%;" />

   接下来就是跳过字节逻辑的处理：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623011823136.png" alt="image-20240623011823136" style="zoom:67%;" />

   取出readerIndex，并且求出真正的跳过之后信息的长度：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623011941440.png" alt="image-20240623011941440" style="zoom: 50%;" />

   最后把需要的信息提取出来，并且移动指针，然后把取到的信息返回：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623012034960.png" alt="image-20240623012034960" style="zoom:50%;" />

3. 丢弃模式下的处理

   我们先分析代码是如何进入丢弃模式的，也就是这一段：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623012411405.png" alt="image-20240623012411405" style="zoom: 50%;" />

   当解析到的frameLength大于maxFrameLength了，那就是可能要进入丢弃模式了。

   `discard=frameLength-in.readableBytes`，通过这段代码计算除了当前的可读字节，还需要丢弃的字节数量。

   frameLength本来这一段都是要丢弃的，但是有可能当前可读的数据还小于frameLength的长度，那就还有discard这么多的字节，后续需要丢弃。

   如果discard<0，说明当前可读的大于frameLength长度了，frameLength这么长的数据就直接丢弃：

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623014244548.png" alt="image-20240623014244548" style="zoom: 50%;" />

   否则，就要进入丢弃模式：

   进入丢弃模式，并且需要记录剩下的还没有丢弃的数据，然后跳过当前可读的数据。

   <img src="https://cdn.jsdelivr.net/gh/candyboyou/imgs/imgimage-20240623014516915.png" alt="image-20240623014516915" style="zoom: 50%;" />

   

