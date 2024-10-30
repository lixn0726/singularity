# Java Memory Model

Conclusion of ***Jenkov's blog***. 

Java Memory Model specifies how the JVM works with computer RAM.

JVM is also a model, of a whole computer, so, this model naturally includes a memory model - that is our Java Memory Model, aka JMM.

JMM, which will help us to design correctly behaving concurrent programs. The JMM specifies, how and when, different thread can see values, which are written to shared variables when necessary.

The original JMM was insufficient, so it was revised since Java 1.5, and this version is still in use in Java today.



### Internal JMM

我们都知道，JVM的内存划分可以简单的分为两部分：

- Heap
- Stack

运行在JVM上的每个线程都拥有自己独立的一个stack，线程的stack则包含，当前线程执行到当前这条指令时，调用过的方法的信息，也就是我们熟知的方法栈，或者说调用栈。而这个栈则会随着线程调用的方法改变而改变。

线程的栈同样包括各个方法包含的local variable，每个线程只能访问自己的栈，所以，线程栈里的local variables是不能被其他线程访问的。即使两个线程在执行同样的方法，它们各自的变量也是独立的。

所有的primitive local variables都是完全在栈中存储的，也就是其他线程不可见的，线程可以将primitive变量传递一个值的复制给其他线程，但是无法和其他线程共享这一个primitive变量。

相比于stack，heap中存放所有在应用中创建的java对象（被jit栈上分配的除外），包括各种primitve的包装类型（题外话，在应用中，对象频繁的创建、销毁也会带来巨大的开销，即使有GC存在）。无论一个对象被分配给一个local variable，或者在另外一个对象中作为member存在，在被回收之前，它会一直在heap中存在。



在stack中，一个local variable可以是primitive，也可以是一个reference。但是我们需要知道一点，当一个local variable是一个reference时，在stack上，它是作为一个类似于memory_addr(obj)的一个ref，而对象本身，仍旧存在于heap中。

一个对象包含方法，而方法中则包含local variables，这些variables同样，无论如何都是存在于stack上。所以得出结论：

- 一个对象，它本身的member是存在于heap中的，无论它的member是primitive，亦或是另一个对象，这些members都存在于heap中。
- 而一个stack local variable，它永远都是在stack上，无法在堆中保存，可以理解为，stack variable只是一个门牌号



所以我们可以知道，primitive type永远不会独立存在于heap中，但是它们可以作为object's member在heap中存放。

在heap中的对象，只能被stack variable通过reference的方式使用，当一个线程可以访问一个对象的时候，它同样可以访问对象中的members，所以，当多个线程对同一个对象含有access的时候，这些线程可以同时访问这个对象的members，但是这些members都是在各自的线程栈中以local variable reference的形式存在的。

当前，相信各位作为有工作经验的Java开发，对这些都是了然于胸的。

下面也通过一个图，来展示大致的JVM Memory Layout：

![image-20240910142344808](/Users/mac/Desktop/assets/articles/assets/image-20240910142344808.png)

这两个线程的stack中的local variable都指向同一个对象，这两个**不同的reference**指向同一个对象。同时，可以看到object3的members包含object2，object4，而此时，这两个local variable可以同时访问这两个object。

用一段代码来描述上面这个图，如下：

```java
public class MyRunnable implements Runnable {
  public void run() {
    methodOne();
  }
  public void methodOne() {
    int localVariable = 45;
    MySharedObject localVariable2 = MySharedObject.sharedInstance;
    // other local variables
    methodTwo();
  }
  public void methodTwo() {
    // because Integer has cache from (-128 ~ 127)
    Integer localVariable1 = new Integer(128); 
    // other local variables
  }
}

public class MySharedObject {
  // sharedInstance is the object3
  public static final MySharedObject sharedInstance = new MySharedObject();
  private MySharedObject() {}
  public Integer object2 = new Integer(22);
  public Integer object4 = new Integer(44);
  // member
  public long number1 = 12345;
  // member
  public long number2 = 67890;
}
```

当有两个`MyRunnable`启动时，就会有这样的内存布局。

每个执行`methodOne`的线程都会在栈上创建对于`localVariable1`和`localVariable2`的copy，但是，实际上这两个copy都会指向同一个对象也就是object3。



## Hardware Memory Architecture

现代硬件的内存架构和JMM略有不同，我们只看最常见的架构。

我们知道CPU中包含各自的寄存器，CPU还有高速缓存，以及RAM，也就是主存。

现代计算机一般拥有多个CPU，而一个CPU可能包含多个核心，所以，当前的计算机是可以让多个线程同时运行的。每个CPU都可以在运行一个线程，所以，当前的多线程Java应用可以充分的利用多线程的优势。

每个CPU都包含一众的寄存器，CPU对于寄存器上的变量的操作，是比去操作主存里的变量要快的多的（众所周知，越靠近CPU，CPU执行的越快）。而CPU同时含有高速缓存架构，一般包含多层高速缓存，每层的缓存大小不同。CPU对于高速缓存中的变量操作也是比主存的变量要快的。按照access速度排序，从快到慢为：

寄存器 -> 高速缓存 -> 主存

不过，只需要知道CPU拥有一个缓存机制即可。

每台计算机都包含一个RAM，也就是主存，所有的CPU都可以访问主存，主存一般是比CPU缓存要大的非常非常多。

一般来说，当一个CPU需要访问主存里的东西的时候，CPU会将这一部分的主存都读取到它的高速缓存中，它甚至可能会读取一部分的CPU高速缓存里的内容到寄存器中，然后对里面的变量进行操作。

当CPU需要将处理结果写回到主存中时，它会先将处理结果写回到高速缓存中，然后在某个时间点，再将数据从缓存写回到主存中。

在缓存里的数据会在CPU需要将数据写回主存的时候一起写回，也就是是批量的操作。CPU可以将数据部分的写入它的缓存中，也可以将数据的一部分写回到主存。也就是==**不需要完整的写入或者读取整个缓存大小的数据**==。

It does not have to read/write the full cache each time it is updated.

一般来说，缓存的更新是以"cache lines"这样的内存块为单位来进行的。一次read/write都会操作N个cache line。



## Bridging the Gap between the Java Memory Model And the Hardware Memory Architecture

说到底，最终交互的还是底层硬件，所以我们来看JMM和硬件架构之间的交互。

上面说到，JMM和硬件的内存架构是不同的，硬件内存架构不会区分JVM里的stack和heap，对它来说，这些都是主存里的东西，所以，heap和stack里的内容会出现在CPU缓存以及CPU寄存器中。

当对象和变量被存储在主存中时，就会有两种问题出现：

- 共享变量的更新操作的可见性
- 竞态条件，也就是race condition



### Visibility of Shared Objects

如果多个线程共享同一个对象，如果不进行恰当的volatile修饰或者同步机制，那么就无法保证某个线程的更新操作对其他线程可见。

我们知道，对象是在堆中存储，那么假设一个对象x在堆中被初始化。此时线程在使用CPU 1，然后将这个x读取到CPU 1的缓存中，然后它对这个x进行一些修改，比如说内部的members赋值，只要这个CPU缓存没有被写回到主存，那么这个被修改的member对运行在其他CPU上的线程来说就是不可见的。那么，就有可能出现一种情况就是，所有的线程都分布在不同的CPU上，而不同的线程对这个共享变量x的修改都仍然保存在各自的CPU缓存中，并没有保存到主存。那么就会出现在同一个计算机上，不同的CPU对同一个x有多种不同版本的观测。

下面有一个图可以用来演示这种情况，可以看到，在不同的CPU缓存中，同一个对象里的member的值对不同的CPU是不同的。

TODO --- 图片。



而如果我们需要保证，主存中永远都是最新版本的值，并且可以被其他线程感知到。那么对于Java应用来说，就需要使用上面说到的volatile happens before，或者synchronization happens before，因为这两个happens before规则，通过JMM，保证我们的程序会按照我们想要的方式运行，并得到正确的并发结果。

而对于这种单一变量的情况，最好的解决方法就是使用volatile，因为JVM对于volatile关键字的实现，可以保证所有volatile read都是从主存中读取，并且在更新的时候直接更新到主存中。



### Race Condition

如果多个线程共享一个变量，并且至少有两个线程是在对这个变量进行更新，那么这就是一种**==race condition==**。

假设线程A，读取了共享变量中的成员变量`count`，并且，将它读入到了它的CPU缓存中。而假设此时另一个线程B，也在做同样的事情，只不过这个count被读取到了另一个CPU缓存中。而假设此时A将count+1，并且B也这么做，此时count被执行了两次+1，在每个CPU中各一次。

如果这两个+1操作可以被顺序的执行，那么count就会被+1两次，并且会将原始值+2，然后协会主存。

然而，在没有得到正确的同步机制时，这两个+1操作是并发执行的，不论是A、B的哪个+1被直接写回到主存，都会导致最终的结果为原始值+1，而不是+2。

而对于这种场景，只有volatile语义是不够的，为什么呢？

简略为一句话，就是volatile只保证可见，但是不保证写回操作是顺序的。如果结合这部分的CPU缓存来说就是：

假设共享变量volatile int x = 0，这是经典的场景。那么多线程并发执行x++这个操作，我们都知道是会有并发问题导致某几次x++被吞掉的。从硬件角度来看，多个CPU都将x = 0读取到了自己的缓存，此时有多个CPU同时执行x++，但是结果仍然保存在它们各自的CPU缓存中，那么总会有这种极端条件也就是，这几个CPU同时将x = 1的结果写入到主存中，由于volatile只保证将当前CPU处理结果对其他线程可见，但是并不能保证在多CPU同时写回的情况下保证顺序性，所以，在写入主存之后，其他还未写回主存的CPU会感知到这个新的值为1，那么会更新自己的CPU缓存。然而，此时已经发生了数据错乱，但是对于单个CPU的volatile语义来说，这是允许的，因为对于每个成功写回的CPU来说，它的更新操作只是一次x++，那么x当然为1，所以，这是符合volatile语义，但是不能覆盖我们需要的使用场景。所以，只用volatile无法保证并发更新的安全。

那么此时，只能使用synchronized，因为在Java8中，我们除了显式的使用锁以外，JDK本身给我们提供的并发元语只有volatile和synchronize，那么volatile不行，只能用synchronized。

由于synchronized对某个对象显式加锁，那么在同一时刻，只有一个线程也就是一个CPU可以进入这块synchronized block，而同时，synchronized保证在线程退出synchronized block之后，所有对这个线程可见的变量都会被写回到主存，并且对其他的线程可见。那么就意味着在其他线程允许进入synchronized block的时候，能首先看到其他线程退出synchronized block时候的最新结果，那么这样就可以保证通过synchronized取消了并行的执行，而是变成了串行，从而保证了race condition的数据安全。