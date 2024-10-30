# AdvanceBits

主要是：https://www.youtube.com/watch?v=iTZNhknTGrg&t=2026s

这个视频，从34：00开始的例子解读。



## 1

```java
int x, y;

@Actor
void actor() {
  synchronized (this) {
    x = 1;
  }
  synchronized (this) {
    y = 1;
  }
}

@Actor
void observer(II_Result r) {
  r.r1 = *(int) VH_Y.getVolatile(this);
  r.r2 = *(int) VH_X.getVolatile(this);
}

```

- 自己的猜测

  由于是先读y，VH我不太熟但是他这是volatile的话，不允许reorder

  那么

  ```java
  (r1, r2) = (1, 1), (0, 0), (0, 1)
  ```

  其中，(1, 1)和(0, 0)是很好理解的。

  (0, 1)是因为，synchronized不会导致重排序，第一个synchronized必定会先执行，那么假如是下面的序列

  ```
  w(x, 1) -- SO --> w(y, 1)
  r(x):1 -- SO --> r(y):?
  
  那么根据happens before有
  w(x, 1) -- HB --> r(x):1 -- SO --> r(y):?
  ```

  那么，y可以是任意的，所以会出现(r1 = y = 0, r2 = x = 1)

- 结果是都有可能，也就是

  ```java
    RESULT     SAMPLES     FREQ       EXPECT  DESCRIPTION
      0, 0  19,012,494   85.26%   Acceptable  Boring
      0, 1      26,142    0.12%   Acceptable  In order
      1, 0       8,807    0.04%  Interesting  Whoa
      1, 1   3,253,248   14.59%   Acceptable  Boring
  ```

  asfasfasfasfas

  在解释这个现象之前，还给了一个例子

  ```java
  int x, y;
  @Actor
  public void actor1() {
    x = 1;
    synchronized (new Object()) {}
    y = 1;
  }
  
  @Actor
  public void actor2() {
    r.r1 = y;
    synchronized (new Object()) {}
    r.r2 = x;
  }
  ```

  如果使用jcstress运行这个测试，同样会得出(r1 = y = 1, r2 = x = 0)这样的结果，因为根据之前说的，synchronized会保证一些synchronization，但是在这里，它并不算是一个offense，如果把这个`synchronized (new Object())`替换成显式声明的屏障，比如说`UNSAFE.fullFense()`，那么(1, 0)就不会出现。

  再看一个例子，叫做==**IRIW**==的：

  ```java
  int x, y;
  @Actor
  public void actor1() {
    VH_X.setOpaque(this, 1);
  }
  
  @Actor
  public void actor2() {
    VH_Y.setOpaque(this, 1);
  }
  
  @Actor
  public void actor3(IIII_Result r) {
    r.r1 = (int) VH_X.getOpaque(this);
    r.r2 = (int) VH_Y.getOpaque(this);
  }
  
  @Actor
  public void actor4(IIII_Result r) {
    r.r3 = (int) VH_Y.getOpaque(this);
    r.r4 = (int) VH_X.getOpaque(this);
  }
  ```

  在这里，对于(r1, r2, r3, r4)的结果，是有可能出现(1, 0, 1, 0)这样的组合的，这是因为多CPU中各自有各自的缓存，也比较好推断出来。但是，还有一点需要声明的是，字啊x86_64架构的机器上，是不可能出现(1, 0, 1, 0)这样的结果的，因为x86_64架构本身保证了TSO (Total Store Order)，有点类似于对共享变量的写都会保证synchronization的特点，所以不会出现(1, 0, 1, 0)这种意外结果。



还有一个特别有意思的例子

```java
volatile int x = 0;

@Actor
void actor1() {
  for (int i = 0; i < 5; i++) {
    x++;
  }
}

@Actor
void actor2() {
  for (int i = 0; i < 5; i++) {
    x++;
  }
}

@UntilOtherThreadFinish
void read(IResult r) {
  r.r1 = x;
}
```

可以猜一猜最后的x可能的范围是什么呢？

精确的答案是：[2, 10]

给出的解释如下

```
Thread 1: (read:0 ---------- stalled ----------> incre:1) (1->2)(2->3)(3->4)(4->5)
Thread 2: (0->1)(1->2)(2->3)(3->4)            [thread1 store](1 ---------- stalled ----------> 2)
```

也就是说，Thread 1，Thread 2分别占用一个CPU，Thread 1首先先停止运行，然后Thread 2会一直执行代码，但是此时 TODO

When the first thread reading zero then get installed, then other thread does these updates (illustrated above: (0->1)(1->2)(2->3)(3->4)), then, the first thread unparks, unstalled, and stores one into the main memory.

And after that, the other thread does the final operation reads one from the main memory, which will cover the executed updates, makes the previous result 4 disappear. And then the thread also gets installed, repeat what happened before.

这是由于race condition，而不是可见性的问题。

也就是CPU是按照一条条指令来的，比如说一个简单的x++，对CPU来说就是一系列的语句，比如：

1. load var
2. store var，reg
3. incre reg
4. store reg，var

类似这样，比如thread1，读到0，说明至少已经将2给执行了，但是此时CPU没有时间片给它，它必须等待下一个时间片，那么，此时thread2可以执行n次的+1，并且每次都会将+1的结果写回到主存，即使此时这个volatile write，可以让其他CPU的cache失效，但是此时0这个旧值已经到了寄存器里，所以没有办法修改，即使此时CPU2的缓存可能已经更新了x=4/3/2/1，但是寄存器里的x仍然是0，那么CPU直接对0+1得到1，并直接保存到自己的缓存中，又将新的值给覆盖掉，然后直接写回到主存，导致thread做了无用功，结果被覆盖。

那么对于thread2后续的stall，也是这么个解释。













所以，让我们回顾一下，从被写下的一行Java代码到执行，中间它们会经历

1. javac：字节码，not a optimizing compiler, just to translate java code into something more machine-readable
2. 解释器：
3. 编译器C1：
4. 编译器C2：
5. 硬件指令集

每一层都可能有不同的实现，那么我们想要在如此众多的平台中编写出正确运行的Java代码，确实不是一件容易的事情。

但是，我从这一路学习JMM，到现在为止最大的感触就是，如果不确定是否安全，synchronized it。

如果能够明确的找出release/acquire，volatile就是我们能使用的最轻量级的同步机制。

还有一点就是，设计一个完全thread-safe的类到底有多困难，以及其中的要点。重点就在于引用暴露时机。



So why we need JMM ? Because we want Java Program to run faster and faster as much as it can, and that, introduces so much optimization for running a Java Program.

So it's always not that way you written in source code that the program running on the hardware.

So, that's why we need JMM, to make sure that the program will always get correctly run.





## Other Talk

https://www.youtube.com/watch?v=qADk_tj4wY8



Java does't guarantee **sequential consistency** necessary.

What does JVM and processors do to Java program fast is to optimize the Cache Read and Cache Write, which is a very crucial aspect of writing an efficient java program. 



How can we realise that there really is some optimization happens ??? ---> When we can see there are some suprising outcome from our program.

![image-20240912103107057](/Users/mac/Desktop/assets/articles/github-repo/processing/assets/image-20240912103107057.png)

Why JVM and processors want to do optimization:

![image-20240912103320031](/Users/mac/Desktop/assets/articles/github-repo/processing/assets/image-20240912103320031.png)





If code running in a single thread, JVM guarantees that things are consistent with program order.

And, Program Order is, the action in the same thread, will execute， as in the source code order.



Actions, can freely reorder as JVM or compiler want, but, if we coder want to make sure that, some pieces of code will only execute as we wrote, then we need to add some synchronization behavior, to make sure no reordering is allowed.



## Memory Ordering in the wild: Spring Beans

```java
class SomeBean {
  private Foo foo;
  private Bar bar;
  void setFoo(Foo foo) {
    this.foo = foo;
  }
  
  @PostConstruct void afterConstruction() {
    this.bar = new Bar();
  }
  
  void method() {
    assert foo != null && bar != null;
  }
}
```

An ApplicationContext stores beans in a volatile field after their ful construction, then guarantees that beans are only exposed via reading from this field to induce a restriction.



## Memory Ordering in the wild: Akka Actors

```java
class SomeActor extends UntypedActor {
  int foo = 0;
  @Override 
  public void onReceived(Object message) {
    if (message instanceof Foo) {
      foo = 42;
      getSelf().tell(new Bar());
    } else {
      assert foo == 42;
    }
  }
}
```

Akka does not guarantee that an actor receives its messages by the same thread. Instead, Akka stores and receives its actor references by a volatile field on before and ater every message to induce an ordering restriction.



## Memory Model Implementation

A JVM typically implements a stricter form of the JMM for pragmatic reasions.

For example, the HotSpot VM issues memory barriers after synchronization points. These barriers forbid certain (every) types of memory reordering.

Relying on such implementation details jeopardizes cross-platform compatibility.

依赖此类实现细节会危及跨平台兼容性，也就是说不同平台的实现细节可能是不一样的，我们需要的是将data flush back to main memory，但是这样的empty synchronized只是一种实现手段。

```java
synchronized (new Object()) { /* empty */}
```

When we see a piece of code like above, what does the author want to do is to issue a memory flushing, and it always tend to work.

他主要是想要表达，我们需要正确的搭建safe-concurrent program，而不是通过一些JVM trick来保证我们的并发安全，比如上面这个，在JVM中这个可能意味着



You should code against the specification not the implementation. So this might just stop working in a future java version. But in many frameworks, people writing code like this to force the JVM to flush memory to synchronize code.



**Always code against the specification, not the implementation.**



The transitive closure of all orders determines the set of legal outcomes.
