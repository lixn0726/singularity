# Netty的现在和未来：4.1 vs 4.2

这篇文章的主要内容是围绕norman提出的PR：https://github.com/netty/netty/pull/13991。
这是在这几天看到的一个挺有意思的改进，就顺便去了解了一下，也当做是自己的一个学习笔记。而之前说过会更新的JIT，我打算插队一篇JMM，写了大半部分，一直没发出去，因为感觉读起来差点意思，不过应该也很快就可以搞定了。
那么回到正题，首先先来看看这个PR的内容大致是什么，再稍微回顾一下Netty 4.1的设计，最后再来详细的介绍一下这个PR介绍的东西带来的变动。

## PR13991

作者主要通过这个PR提出了一些他认为的4.2之前的Netty在设计上缺陷：
1. 难以扩展`EventLoop`的实现，因为它本身就是not extensible的，并且需要将相同的逻辑给添加到对应的多线程的EventLoop。
2. registration/deregistration限制在了Channel中，导致失去了伸缩性。比如有一些操作系统允许`EventLoop`处理一些其他的IO。
3. 不同的`EventLoop`实现中存在很多的重复代码，而大部分都是用来运行非IO任务的逻辑
而作者由此抛弃了原本的一些实现，将多个不同的实现进行了统一，再将其中的不同的逻辑单独抽象成接口，使得`EventLoop`和`EventLoopGroup`更加的有扩展性。

那么在知道这个PR主要解决的问题之后，我们先回顾一下Netty4.2之前的设计，来看看到底是什么引起了作者的不满。

## Netty 4.1的设计
我们知道Netty中的核心组件包括：
- EventLoop
- EventLoopGroup
- Channel

其实还有很多，比如说`ByteBuf`、`ChannelHandler`等等，而目前我们只关注这几个就好，因为这篇文章的很多内容基本是围绕这些东西的。
我们知道，`Channel`可以挂载在`EventLoop`下，`EventLoop+Channel`就是Netty的io核心，通过`EventLoop`去获取到io事件，再去调用`Channel.Unsafe`去进行业务逻辑处理。

我们直接从代码层面来了解，具体的不会说太多，大概捋一下各个api的调用就是了。

`ServerBootstrap.bind`
- ServerBootstrap.bind
	- AbstractBootstrap.doBind
		- AbstractBootstrap.***initAndRegister***
			- AbstractBootstrap.doBind0 
				- Channel.bind

`Bootstrap.connect`
- Bootstrap.connect
	- AbstractBootstrap.doResolveAndConnect
		- AbstractBootstrap.***initAndRegister*** 
			- Bootstrap.doResolveAndConnect0
				- Bootstrap.doConnect
					- Channel.connect
					顺带一提，`ServerBootstrap`的启动有一个重要的东西：
					`ServerBootstrapAcceptor`，这东西只覆盖了channelRead和exceptionCaught方法：
- channelRead：因为这是`ServerBootstrap`的channel里的handler，它的channel也就是类似于`ServerSocketChanne`l这样的东西，是用来监听链接的，所以这里的channelRead，read到的msg都是`Channel`。
	所以，这里channelRead的作用也就是将新的`Channel`给注册到childGroup去
- exceptionCaught：直接fire这个事件，如果`ServerBootstrap`配置了autoRead，那么会首先 stop accept new connections for 1 second to allow the channel to recover，并且在这之后会将autoRead设置为false

根据上面的链路可以看到其实大致的逻辑差别不是很大，最终都会落到一个Channel.Unsafe上去执行真正的操作。不过，在这里可以关注一下`initAndRegister`这个方法，这个方法的大致伪代码如下：
```java
final ChannelFuture initAndRegsiter() {
	Channel channel = newChannel();
	init(channel);
	// 在这里就会start eventLoop
	getGroup().register(channel);
}
```
可以看到是创建一个channel，然后将这个`Channel`给注册到对应的`EventLoopGroup`上。看到我有标注一个start eventLoop，那么这里的`register`方法也可以看看代码：
```java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
	// ...
	if (eventLoop.inEventLoop()) {
		register0(promise);
	} else {
		eventLoop.execute(() -> register0(promise));
	}
	// ...
}
```

可以看到一个很经典的判断：`eventLoop.inEventLoop()`，这个就是判断当前运行到这一个语句的线程是否是这个`EventLoop`绑定的线程，在第一次执行到这里的时候，这个`EventLoo`p只是在它的group中被初始化，并没有绑定线程，所以在每个`EventLoop`的第一次判断，这里都为false，所以会给这个`EventLoop`提交一个异步任务，在这个execute中就会启动一个线程来执行这个register方法，所以，这里可以得出一个结论：
- 所有的EventLoop是在EventLoopGroup初始化的时候同时初始化的
- 每个EventLoop在第一次提交异步任务的时候才会绑定线程
算是个小tips，了解一下即可。

在这里提到`EventLoop`的启动，是为了引出和io相关的操作。去看到`EventLoop`的类继承关系，可以看到`EventLoop`也是一个`Executor`，当前面的`initAndRegister`将`EventLoop`启动之后，就会进入它的run方法，各自的io实现中，这里的run都是无限循环的，所以从此之后，EventLoop就开始了它操劳的一生。

而到了`EventLoop`这里，我们就可以看到在文章开头的pr中，作者提到的问题了。

## EventLoop与IO
我们知道，在Netty中，`EventLoop`是一个核心组件。它既负责io任务，也负责处理普通任务以及定时任务，而这种任务之间的调度机制可以从各个`EventLoop`的源码实现中得出大致逻辑如下：
1. 判断是否需要进行io操作
2. 进行io(或者跳过)
3. 处理io事件以及普通任务

即使是不同的io方式，它们的run方法的大致框架是没什么差别的，但是，Netty还是在不同的`EventLoop`实现中还是写了不少的重复代码，具体可以看到Netty 4.1源码中的`NioEventLoop`以及`EpollEventLoop`中各自的run方法：
```java
// NioEventLoop.run
protect void run() {
	for(;;) {
		// 判断是否需要进行io，这里都假设需要
		io();
		processSelectedKeys(); // using Channel.Unsafe
		runAllTasks();
	}
}

// EpollEventLoop.run
protect void run() {
	for(;;) {
		// 判断是否需要进行io，这里都假设需要
		io();
		processReadys(events); // using Channel.Unsafe
		runAllTasks();
	}
}
```

可以看出大致的逻辑也是一致的，只是在处理io结果的时候，Nio是通过`SelectionKey`来获取`Channel.Unsafe`处理，而Epoll是通过events来获取`Channel.Unsafe`处理。本质上都需要`Channel.Unsafe`。也就是说，不论是Nio还是Epoll，最终都是获取到事件对应的`Channel`然后直接通过`Unsafe`进行处理；这也导致了`EventLoop`和`Channel`强耦合，导致如果需要对于io事件的处理进行一些监控或者自定义操作，全部都要堆积到`ChannelHandler`的逻辑中去，就可能会导致`ChannelHandler`的逻辑比较臃肿，也会导致`EventLoop`的任务处理效率受到影响。
在实际的源码实现中，`NioEventLoop`和`EpollEventLoop`的run分别进行了具体的实现，也就导致有一大部分的重复逻辑。那么从这里我们就可以看到PR提到的第1、3点。

## 那么Registration呢
那么还有最后一点的registration/deregistration是什么问题呢？我们倒回到前面说的`initAndRegister`方法上，希望这时候还能对这个方法的逻辑留有一点印象。
其实，我们并没有看到最后的`initAndRegister`到底做了什么，实际上这个也并没有什么看的意义，它就是将channel需要感知的io事件给注册到selector/epoll instance上。而这里的问题是什么呢？问题就在于这个register动作是定义在channel中的，也就是说，如果我们想要一些nio/epoll channel感知到一些比较特殊的io事件，处理起来会比较麻烦，比如像这样：
```java
public class CustomNioInterestOpsChannelHandler extends SimpleChannelInboundHandler<Object> {  
    private final int interestOps;  
  
    public CustomNioInterestOpsChannelHandler(int additionInterestOp) {  
        interestOps = additionInterestOp | SelectionKey.OP_READ;  
    }  
  
    @Override  
    public void channelActive(ChannelHandlerContext ctx) throws Exception {  
        Channel channel = ctx.channel();  
        if (channel instanceof AbstractNioChannel) {  
            AbstractNioChannel nioChannel = (AbstractNioChannel) channel;  
            Class<AbstractNioChannel> nioChannelClass = AbstractNioChannel.class;  
            Method selectionKey = nioChannelClass.getDeclaredMethod("selectionKey");  
            selectionKey.setAccessible(true);  
  
            SelectionKey invoke = (SelectionKey) selectionKey.invoke(nioChannel);  
            invoke.interestOps(interestOps);  
        }  
    }  
  
    @Override  
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, Object o) throws Exception {  
        // no ops  
    }  
}
```
可以看到，处理起来比较麻烦，并且由于register的对象是`Channel`、`EventLoop`，所以如果说我们想要将一些可以特殊处理的io，比如PR中提到的io_uring，挂载到一个`EventLoop`上，这样的操作也是很难的，我们必须重写一个这样的`Channel`才可以，这样就会导致工作量非常的大，并且适配性也不一定好。
这就是PR提到的问题的第2点。

## 作者提出的改变
- 添加IoHandleEventLoopGroup/IoHandleEventLoop的概念，并添加对应的IoHandle，IoHandler以及IoHandlerFactory，这样是为了将IO和非IO任务给分离开，那么在不同的IO实现中，就只需要实现IO的部分就可以了，而其他的任务执行、任务调度规划等代码都可以共享。
- 添加`MultiThreadIoHandleEventLoopGroup`和`SingleThreadIoHandleEventLoop`这两个实现类，可以用来给大部分通信实现，而不同的io方式只需要实现它们各自的`IoHandler/IoHandlerFactory`即可。

## Netty 4.2的新玩意
所以，我们来看看Netty4.2的改动，整体来说是比较有意思的，首先引入了很多的新的概念：
- IoEventLoop
- IoEventLoopGroup
- IoHandle
- IoHandler
- IoRegistration
- IoEvent
这些是比较核心的东西，其中重点在于以下：
- IoEventLoop
- IoHandle
- IoHandler
- IoRegistration
我们可以这么去理解这些概念：
- IoEventLoop：用于io的EventLoop
- IoHandle：可以理解为processReadys/processSelectedKeys这样的方法，只不过这里是一个可以扩展的点
- IoHandler：用来处理io的handler，对应到io
- IoRegistration：用来解决前面说到的registration问题

首先看到`IoEventLoop`，前面提到，既然`EventLoop`的各个实现有很多重复的地方，并且大体的逻辑相差不多，那么第一个想到的自然就是模板方法模式，这也是norman的做法，他将所有的`EventLoop`重新分类，引出了`IoEventLoop`以及`IoEventLoopGroup`这两个接口，从名字来看我们就知道，这类`EventLoop`就是用来进行io的，同时将前面说到的不同`EventLoop`的大致逻辑进行了封装，统一到了一个`SingleThreadIoEventLoop`类中，新的逻辑如下：
```java
protected void run() {
	do {
		runIo();
		if (isShuttingDown()) {
			ioHandler.prepareToDestroy();
		}
		runAllTasks(maxTasksPerRun);
	} while (!confirmShutdown());
} 

private void runIo() {
	ioHandler.run(context);
}
```
可以看到，这个类将run方法的大体逻辑进行了封装，并将独立的io逻辑交付给了`IoHandler`这个类。所以，在4.2版本中，对于不同的io，不再是单独的`EventLoop`，而是不同的`IoHandler`实现，在它的run方法里实现具体的io逻辑。这也就解决了PR提到的第3点问题。
并且，由于`IoEventLoop`的实现全部统一在了一个类，那么我们可以简单的扩展这一个实现类，来获取一些监控数据指标，也就解决了PR提到的第1点问题。

而看到IoHandle，我们先看它的定义：
```java
public interface IoHandle extends AutoClosable {
	void handle(IoRegistration registration, 
				IoEvent ioEvent);
}
```
其实从这里的方法签名就可以看出来，`IoEvent`和`IoHandle`之间就是event-handler的关系，在4.2中，各种io的结果都会以`IoEvent`的形式出现，而`IoHandler`更多的是一个处理io的handler类。并且，`IoEvent`是针对单个通道的，所以`IoHandle`可以当做是`Channel`的一个补充，只不过IoHandle是针对io事件的一个东西，它并不一定要是网络操作，也可以是文件操作。

也许这么说还是不太好理解，不过可以直接看代码里的使用，应该会更加的明确：
```java
// 以NioIoHandler为例
private void processSelectedKey(SelectionKey k) {
	final DefaultNioRegistration registration = (DefaultNioRegistration)k.attachment();
	// handle when registration is invalid
	// ...
	registration.handle(k.readyOps());
}

// class DefaultNioIoRegistration
void handle(int readyOps) {
	handle.handle(this, NioIoOps.eventOf(readyOps));
}
```

而这样的新设计带来的好处是什么呢？解决的就是PR提出的第3点的问题。很显然的就是我们可以简单的将支持特定io的`IoHandle`挂载到已有的`EventLoop`上，并且不改动任何的逻辑，比如：
```java
public void addCustomIo() {
	MultiThreadIoEventLoopGroup boss = new MultiThreadIoEventLoopGroup(1, EpollIoHandler.newFactory());
	MultiThreadIoEventLoopGroup boss = new MultiThreadIoEventLoopGroup(Runtime.getRuntime().availableProcessors() << 1, EpollIoHandler.newFactory());
	
	IoRegistration customIoRegistration = boss.next() // 获取一个eventLoop
		.register(new IoUringFileIoHandle()) // 将自定义的io逻辑挂载到原生eventLoop上
		.await()
		.get();
	customIoRegistration.submit(EpollIoOps.EPOLLIN); // 注册感兴趣的io事件
}

public class IoUringFileIoHandle implements EpollIoHandle { /* ... */ }
```

那么在挂载之后，我们自己的特殊的`IoHandle`也可以享受Epoll带来的性能优势，并进行一些其他的io操作，比如我们自己实现io_uring。那么在这之后，这个`EventLoop`就会自动帮我们关注这个`IoHandle`代表的io句柄的EPOLLIN事件，然后在io事件触发之后，就可以通过`EpollIoHandler`分发对应的`IoEvent`，自定义的`IoHandle`就可以感知到，然后运行我们自己想要的io逻辑。

从这里可以看到，指定io方式不再是通过特定的`EventLoopGroup`，而是通过传递特定的`IoHandler`给到`MultiThreadIoEventLoopGroup`去，所以，如果需要自定义io的话，通过实现`IoHandler`，然后再作为参数传递一个Factory进去即可；而在4.1版本中，自定义io则涉及到很多的实现类，包括`EventLoopGroup`、`EventLoop`、`Channel`等等，就会比较麻烦。新版本将`EventLoop`的逻辑进行了统一，所以不再需要编写那些繁琐的逻辑。

而在4.1版本中，这个处理起来是比较麻烦的，因为4.1中，`EventLoop`和`Channel`是绑定在一起的，因为对应的io事件需要通过`Channel`的`Unsafe`来给到channel的pipeline进行处理。在新版本中，则通过`IoHandle`来进行挂载，大致的调用链路为：
- SingleThreadIoEventLoop.runIo
	- IoHandler.run
		- IoRegistration.handle
			- IoHandle.handle
				- Unsafe
					- ChannelPipeline
					而在4.1版本中，大致的调用链路为：
- XXEventLoop.run
	- Channel
		- Unsafe
			- ChannelPipeline
			可以看到，4.2版本的架构虽然链路比较长，但是将`EventLoop`和`Channel`解开了耦合，并抽象出了`IoHandler`、`IoHandle`这两个关键接口来便于扩展，从整体设计上来说，个人感觉是比4.1的设计要好很多的，也可以看出，在编写代码的时候，组合使用多个简单的接口是更加优雅的架构设计选择。

## 后记
总的来说，Netty4.2是一个挺有意思的版本，因为作者也在PR提到，4.2.0的release版本会包括这个PR，并将支持的最低的Java版本提升到Java 8，还有一些未来的低风险的change，个人感觉这也许会成为未来风靡的版本，并且也可以从这个PR中看出一些在设计上遵循的SOLID思想，在日常的编码中也许并不能接触到这些代码，但是作为一个编码的人，心里有些追求总是好的吧。
最后，也贴一个有意思的原则：KISS原则，这是在我翻阅《UNIX编程艺术》的时候看到的，感觉是一个终身受用的准则，也分享给大家：

K.I.S.S.：Keep It Simple, Stupid!
								