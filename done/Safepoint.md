# 安全点

大家对 JVM 的了解大部分都是从八股文中获得的只言片语，八股文的内容暂且先不论对错，光是它的内容本身也是稍显片面。因此，考虑新开一个关于 JVM 的系列。首先写文章是为了落实费曼学习法；再者就是记录自己所学，毕竟闭目塞听毫无意义，将自己的见闻公诸于众，集众人才智才更能完善自己的理解。

至于为什么第一篇是安全点，因为安全点算是JVM中一个比较核心的机制，对于JVM中的各个组件之间的协作提供帮助；再者就是前段时间看到有群友问过。就花些时日，将我所知道的安全点写出来。

本文所述都是基于Hotspot VM，其他的虚拟机对于安全点的实现和Hotspot VM有所不同，不可混为一谈。并且对应的JDK版本为17，和JDK8下的Hotspot VM也会有些不一样。并且，所有的内容都出自个人理解和一些资料以及源码的查阅，所以仅供参考即可，主要是让大家留下一点关于安全点的印象。



## 什么是安全点

安全点可以分为两种：

- 全局 Safepoint : 针对整个 JVM 的安全点，也是本文的内容，表示需要所有的应用线程都进入安全点。
- 线程 Safepoint : 表示只需要让单个线程进入安全点，涉及到一个 JEP - Thread-Local Handshakes : https://openjdk.org/jeps/312

由于单个线程的Safepoint不常被提及，所以本文后续的内容都是针对全局Safepoint来展开的，而对Thread-Local Handshakes有兴趣的可以自行去看JEP，基本涵盖了所有需要了解的东西。

安全点实际上可以理解成：此时JVM可以对需要进行操作的Java线程进行一些数据收集或者修改的操作，因为这些Java线程在安全点上是无法继续运行Java代码的，最简单且直观的解释为：只有处于安全点的时候，Thread.interrupted才能准确的知道自己是否被中断，因为处于安全点的时候，线程的信息才是确定的。

而从定义上，我们可以将安全点的概念拆分如下：

- VM 中没有正在运行Java代码的Java线程
- 进入安全点的每个Java线程此时各自的GC root对VM来说是可见的



## 什么时候需要安全点

安全点是一个对VM来说全局安全的一个状态，实际上需要安全点的地方会比较多，下面只列举一些比较常见的情况：

- GC
- JIT编译后的代码的deoptimization
- 刷新CodeCache
- 类的重定义
- 取消偏向锁
- jstack、jmap和jstat等基于信号的命令

还有一些其他的场景，不过就不一一列举了，我们只需要了解一些日常比较常见到的场景即可。还有两个值得一提的地方：

1. 实际上，JVM还会定时进入Safepoint，我们可以在Hotspot中找到一个变量：`GuaranteedSafepointInternal`。而大部分情况下并不需要这个定时进入的机制，可以通过VM Option：`-XX:GuaranteedSafepointInternal=0`将它关闭掉。
2. 在高版本的JDK中，偏向锁的取消实际上不再需要全局Safepoint，也是在JEP-312中提到的东西



## 一些概念解读

在 JVM 源码中，JavaThread是一个单独的类，代表一种独特的线程，用来执行Java代码。它包含一个和安全点息息相关的字段：`_thread_state`，这里的状态理解起来和我们普遍了解的线程状态不太一致。这一个状态代表：该JavaThread当前正在执行的代码是什么代码，这么听起来似乎有点奇怪，不过有一个枚举如下：

![image-20240901162557288](/Users/mac/Desktop/assets/articles/assets/image-20240901162557288.png)

可以看到，对于正在运行的状态进行了比较细致的拆分，而不是像OS线程状态一样。这些状态可以划分为：

1. Mutable
   1. `in_vm`
   2. `in_java`
2. Immutable
   1. `new`
   2. `in_native`
   3. `blocked`
3. Transition
   1. 各种`_trans`

忽略掉那些标注了没有在使用的状态，有一个如下图的状态转移图示：

![image-20240903202620052](/Users/mac/Desktop/assets/articles/assets/image-20240903202620052.png)

Mutable和Immutable的核心区别在于：==**JavaThread是否可以将自己的 GC roots 暴露给 VM，以及是否可以修改自身的引用关系**==。Transition代表过渡的状态，属于一种中间状态，可以看到基本所有的中间状态都会触发一次Safepoint check。

对于JavaThread这个类，还有几个和安全点有关的字段：

- `JNIHandleBlock _active_handles` : 代表一个可以间接访问当前正在处于 `in_native` 状态的JavaThread正在运行的 `jni` 代码的一个句柄，由于必须要在 `in_vm` 的状态下才能进行这一句柄的分配或者解除，所以，当需要执行 jni 方法的时候，就会导致当前线程进行一次 `in_native` 到 `in_vm` 的状态转换，而这会进入过渡状态而导致一次Safepoint check。所以会看到有文章会说，==在 Java 线程执行完本地方法后会进行安全点检查==，这就是原因。并且，`JNIHandleBlock` 是由 JVM 自动管理的分配和接触，无法通过在Java应用中编写代码来主动触发。

- `JavaFrameAnchor _anchor` : 实际上是在 JavaThread 的父类 Thread 里的一个字段。可以理解成 `last java frame`，也就是上一个 Java 栈帧，这是用来提供 *Stack Trace* 功能的，但是，只有在当前 JavaThread 不处于 `in_java` 这个状态并且至少有一个 activated java record，这个 activated java record 其实是：

  ![image-20240902191604799](/Users/mac/Desktop/assets/articles/assets/image-20240902191604799.png)

  可以看到，其实相当于一个方法的调用信息。也就是说，当这个线程在执行 Java 方法的时候被外界因素打断了运行，也就是离开了 `in_java` 这个状态的时候，在 JVM 中，就会记录这个信息，了解一下即可。

  而在 `JavaFrameAnchor` 中，包含有两个变量

  1. `_last_java_sp` : 上一个 JavaThread 栈的地址，这是一个比较重要的字段。
  2. `_last_java_pc` :  CPU 的 PC。当线程从 `in_java` 转换为 `in_native` 的时候，会在这里保存一次。

- `SafepointMechanism::ThreadData _poll_data` : 这里存放了每个JavaThread各自的poll page，如果对安全点或者JIT有所了解的话，可以知道这个东西的作用，后文也会提到。



而通过这些字段我们也可以得到一个结论：当JVM需要进入安全点的时候，实际的语义并不是将所有的JavaThread给阻塞住，而是需要所有的JavaThread进入Immutable状态，因为`in_native`对于VM来说也是Immutable的。所以，即使有Java线程在调用JNI方法，这样的线程也不会阻止VM进入安全点。



## 安全点的进入和退出

实际上我只会比较详细的指出如何进入安全点，因为进入和退出可以看作一对相反的操作，了解其中的一个就可以大致理解另一个操作的实际语义，有兴趣的可以自行查阅资料。

首先给出一个大致的流程图：

![image-20240903170910832](/Users/mac/Desktop/assets/articles/assets/image-20240903170910832.png)

可以看到，首先是从VMThread里的一个叫做`loop`的方法开始的，了解过Netty或者了解过一些事件驱动框架的同学应该很快就可以知道这是干什么的，简略的代码如下：

![image-20240904135243085](/Users/mac/Desktop/assets/articles/assets/image-20240904135243085.png)

不难看出，主要就是等待一个需要执行的VMOperation。提一嘴这里的VMOperation，我看到很多文章将不同的VMOperation看作是多种Safepoint，个人认为这是不对的，因为从JVM的角度来说，安全点只是一个状态，前面说的线程Safepoint和全局Safepoint只是范围的不同。而VMOperation代表的是线程需要执行的操作，并不能混为一谈，只能说VMOperation需要在Safepoint中执行。

代码层面上来看，理解这段代码并不难，这里还有一点需要提的就是，在JDK17之前的版本中，这里并不是采用的 `_next_vm_operation` 这个变量，而是有一个VMOperationQueue，也就是在异步框架中广泛应用的==无限循环+队列==的设计。

而在JDK17之后，Hotspot将这个队列移除了，取而代之的就是图中的`_next_vm_operation`。主要的原因有一部分VMOperation被移除或者说优化了，导致目前需要进入全局Safepoint的基本上是和GC有关的VMOperation，所以这里的queue也不再被需要了，详细的可以看：https://bugs.openjdk.org/browse/JDK-8212107 这个链接下的comments，大佬们的讨论更加精准。

那么我们就往下看`inner_execute`这个方法，同样先给出简略的代码：

![image-20240904135050069](/Users/mac/Desktop/assets/articles/assets/image-20240904135050069.png)

从方法名可以看出第一个if语句就是判断是否需要开启安全点的，其中涉及两个条件：

1. 当前VMOperation是否需要在安全点执行
2. 当前VM并不处于安全点

可以看出，全局Safepoint之间是互相冲突的。而如果上述条件都满足，那么就会开启一个Safepointing process，也就是将整个JVM带入到Safepoint的处理。核心方法则在于`SafepointSynchronize::begin`。

### SafepointSynchronize

从上图的代码片段可以看到，安全点的核心方法都在这个类里面，分别是`begin`和`end`方法。我们先从`begin`开始。

#### SafepointSynchronize::begin

- `heap::safepoint_synchronize_begin`

  这个方法主要是去打断并发的GC线程，因为GC线程如果在并行运行的话，可能会导致对象的引用被改变，也就是GC线程的运行会导致VM处于mutable的状态，所以此时是需要打断GC的。而具体的实现就落到了不同的堆上，而堆的不同实现则是由不同的 GC 来决定的，所以不展开，知道大致的目的即可。

- `safepointSynchronize::arm_safepoint`

  这个方法的核心目的是推进整个JVM进入Safepoint。前面提到，JVM进入全局Safepoint需要所有的Java线程都处于immutable的状态，所以这里也分不同的情况来处理：

  - JavaThread运行解释代码：解释器会在方法退出之后或者进入分支的时候检查是否需要进入安全点，这里会涉及到一个`WaitBarrier`。

  - JavaThread运行本地方法：此时JavaThread需要去检查是否需要阻塞，但是VMThread是不需要等待这种JavaThread的，此时VMThread会发起一条内存屏障指令。

  - JavaThread运行编译代码：JIT会在方法退出之后或者进入下一次循环之前插入一条`polling check`的指令，具体就是去访问一个给定的内存地址。这个内存地址有两种不同的状态：

    - 需要进入安全点的话，这个内存地址会被设置为无法访问

    - 不需要进入安全点的话，这个内存地址会被设置为只读

    需要进入安全点的时候，访问这个内存地址会导致`segmentation fault`，这个fault会被JVM中特定的`Signal Handler`处理来让线程暂停运行。

  - JavaThread已经处于`in_blocked`

  - JavaThread处于`in_vm`或者处于过渡状态

    上面说到了一个`WaitBarrier`，不深入展开，但是这个东西主要是针对运行解释代码的JavaThread的，因为编译器是借助访问`poll_page`这样的机制来判断安全点的，但是解释代码并不是通过这种机制来判断的。这是一个很容易被忽视的点，我看到很多人都说Java线程是通过访问一个页面来判断是否进入安全点，这是片面的。`WaitBarrier`的核心机制和Java的JUC中的`Semaphore`很相似，也就是运行解释代码的JavaThread会在WaitBarrier上等待，这一点有了解即可

    实际上涉及到`WaitBarrier`的还有一个关键的类`SafepointMechanism`，不过不太需要了解，所以不赘述。执行完这个方法后，对于JVM来说，可以看作是它已经告知所有JavaThread此时需要进入安全点。

- `safepointSynchronize::synchronize_threads`

  这个方法的核心目的是阻塞当前的VMThread，等到所有的JavaThread都达到了immutable状态之后就退出。它的逻辑只是不停的自旋，并判断所有的JavaThread的状态是否满足条件。

结合上面的流程图，从begin方法跳出来之后，说明此时JVM已经处于Safepoint，那么后续就是安心的执行VMOperation，在执行结束后，就会通过`end`方法退出Safepoint。

---

通过上面的解析我们可以知道，JVM进入安全点的耗时其实就可以分析出来了。而全局Safepoint导致的STW（Stop The World）时间，我们也可以分析出来：

- STW时间可以分为几大部分
  - 所有JavaThread中，其中一个JavaThread需要运行到下一个Safepoint check的最长时间。可以理解为==进入安全点的时间==。
  - VMOperation本身的执行时间。可以理解为==在安全点停留的时间==。

第一部分是可以通过优化编码来减少的，而第二部分则和Java代码无关。我们来看如何优化第一部分的时间来减少STW的时长。

不论对于编译代码还是解释代码，为了避免过多的Safepoint check导致性能损耗，Safepoint check基本会被安插在几个位置：

1. 在一个`non-counted`的循环中，进入下一次循环之前
2. 在方法结束之后，退出方法之前

需要知道的是，实际上在进入方法之前也可以放置一个Safepoint check，不过Hotspot并没有这么做，其他的VM有这么做。还有一点则是，我们知道JIT的优化手段包含了方法的内联，如果一个方法被内联了，那么它的Safepoint check就会被拿掉。

可以看到，基本上和程序员有关的就是循环这一部分。有一些八股文会解释，如果一个以int类型的值为边界的for循环可能会导致影响GC。实际上这也和JIT有关。而这涉及到的是JIT编译代码导致的OSR（栈上替换），由于一个for循环里的代码被频繁以同一个方式运行，当运行次数达到阈值时，JIT会将这个循环体里的方法编译掉并OSR，这就会导致原本解释代码里的Safepoint check被拿掉，所以，JIT编译导致Safepoint check被推迟，那么就导致==进入安全点的时间==变多，STW的时间也就随之变长。

而处理这一个问题的方法，也有提到就是将大的int值的循环改造为long。而另一种我认为比较好的做法是添加一个VM Option：`-XX:+UseCountedLoopSafepoints`，也就是让JIT不拿掉编译后的循环里的Safepoint check。

---

#### SafepointSynchronize::end

其实解析完`begin`方法，`end`方法就没有什么好说的了，核心的目的就是恢复线程的运行，基本上照着`begin`反着来就知道了，所以不再赘述。



## 总结

首先，总结一下在日常开发中涉及到安全点的优化手段：

- 取消定时Safepoint check：启动时添加VM参数：`-XX:GuaranteedSafepointInternal=0`
- 不让JIT拿掉循环中的Safepoint check：启动时添加VM参数：`-XX:+UseCountedLoopSafepoints`
- 在高并发环境的Java应用，关闭偏向锁，特别是低版本的JDK，避免频繁的进入全局Safepoint：启动时添加VM参数：`-XX:-UseBiasedLocking`

而如果想调试安全点的话，最好结合源码来看，在此之前可以通过VM参数：`-Xlog:safepoint=trace:stdout:utctime,level,tags`观察到JVM中涉及安全点的一些行为，这个参数只在JDK9之后的版本生效，JDK8无法使用。

至此，关于安全点，个人认为基本上将它的一些实现原理和处理流程都大致捋了一遍，相信大部分人看完会有些许收获。当然，也许会有一些地方我是说错了的，所以大家最好还是把这篇文章当做一份参考，如果发现有错误的地方，也可以指正出来。实际上的Safepoint还有更多的细节，不过考虑到我自己的功力，还是留到以后有缘再说吧。



## 胡言乱语

在经历了我认为人生中最灰暗的一段时间，或者应该说现在还在经历，之后，我还是选择继续发布一些文章，即使只为了满足自己，我也会持续的更新下去，因为我发现在知道这些东西的时候，我还是很开心的，这也算是我无聊生活中的一些慰籍吧。

后续的内容，可能大部分会是JVM相关的东西，下一篇也许会是JIT，不过更新时间不固定了，看我什么时候觉得我能写出来吧。

最后，用*James Gosling*的一句话结束这篇文章吧。

**"Be really stubborn. A lot of these things are really easy to give up on. Whether it’s organizations that you give up, or APIs, or software, a lot of times, it’s too easy to give up too early."**

