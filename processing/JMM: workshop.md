# JMM Workshop



## JMM：under the veil

Java内存模型 (Java Memory Model，AKA JMM) 对于Java程序员来说是一个神秘而又想要接近的东西。因为很多人都在说JMM是很难的东西，也有说这是不存在的一个东西，众说纷纭，而谁也不知道JMM到底是什么。

而从去年开始，我也对这个东西开始感兴趣了起来，不论是JLS (Java Language Specification) 第17章，还是JSR-133 Cookbook，亦或是各式各样的Specification，我也略有了解。时至今日，自觉可以做个阶段性的总结，当然，本文的主要目的仍然是分享，而不是指导。只是借助这篇文章给诸位分享一下我的所见所得。那么让我们开始吧。

## Definition

首先根据Aleksey Shipilev大佬的定义 (https://shipilev.net/blog/2016/close-encounters-of-jmm-kind/)，JMM本身只是一个规范，有点类似于JLS，它只是给出一些定义，并提供一些指导，而对实际的实现并不做限制。就像Java语言，它给我们提供了一套约定俗成的语法，至于程序要怎么写，全是各位coder自行发挥，并且提供了"Code Once, Run Anywhere"这样的功能，而实现这个功能的介质就是JVM，JVM毫无疑问是Java的一个核心，而JVM当然也就和JMM紧密相关，简单来说，JVM实现了JMM的规范，当然也不是全部实现，而是部分。

让我们看回到JMM，它实际上是一个抽象的东西，目的只是规定一段Java代码可能产出什么结果。我们知道，Java程序的执行通常来说包含三个阶段：

1. interpreter
2. compiler
3. CPU

JMM实际上规定了，JVM如何和宿主机的内存，也就是RAM一起工作。我们知道，JVM对于Java应用来说就是一台完整的computer，因为JVM帮助我们屏蔽了底层的操作。

而真正得到执行，则是在CPU，它真正帮我们执行一条指令来帮助我们完成我们需要完成的目标。也就是说，到了真正执行的时候，我们写的代码就不复存在，取而代之的则是一条或者几条相似的CPU指令。那我们的代码去哪了呢？



## 关于Workshop本身...



Data Race: Definitions

- Conflict: At least 2 threads accessing same variables, and at least 1 thread is writer.

- Data Race: A conflict that is not ordered by synchronization

  - SC-DRF: Sequential Consistency for Data Race Freedom

    If you don't have any data race in your program, all the behaviors of the program is like they are SC.

- Race condition: System behavior is dependent on the timing of events



 

Memory-model-wise, there is a difference: 

```java
int m1() {						int m2() {
  int x1 = field;				int x1 = field;
  int x2 = field;				int x2 = x1;
  return x1 + x2;				return x1 + x2;
}											}
```

对于Java程序员来说这是很理所应当的，因为field在堆，而x1、x2是本地变量，对field的read操作有可能会出现data race，那么就是racy read。





Access Atomicity: Takeaway

这里的前提都是64位的操作系统

- Most built-in types are access atomic

  - Almost all are naturally alighed 

    比如说int占用4字节也就是32位，JVM自动填充到64位

  - Unless 32-bit JVMs are present

    32位的JVM会导致超过32位的primitive value操作变成非原子

- Doing unnatural accesses break atomicity again

  - ByteBuffer compound opearations
  - Unsafe compound operations

- Larger types would break access atomicity again



提到了一个例子：Project Valhalla，代码大概如下：

```java
public static class Values {
  static primitive_class Value {
    long x;
    long y;
    public Value(long x, long y) {
      //...
    }
  }
  
  Value v = Value.default;
  @Actor
  public void writer() {
    v = new Value(1, 1);
  }
  
  @Actor
  public void reader(JJ_Result r) {
    Value tv = v;
    r.r1 = tv.x;
    r.r2 = tv.y;
  }
}
```

Value class will be inlined here, 具体的我没有听太清楚，不过大致的意思我感觉是：Valhalla会将这样的class里的基础变量给inline到一起，也就是将它们当做一个整体去操作。但是如果这里是两个long，也就是一共128位，当writer和reader同时进行的时候，也会导致出现在32位JVM上long变量读写的问题，因为被拆成了两部分来操作。

但是我感觉这个可以理解，毕竟你怎么样说也是从对象里面拿了。

此时要解决这个问题也很简单，直接上volatile：

```java
public static class Values {
  static primitive_class Value {
    long x;
    long y;
    public Value(long x, long y) {
      //...
    }
  }
  
  volatile Value v = Value.default;
  @Actor
  public void writer() {
    v = new Value(1, 1);
  }
  
  @Actor
  public void reader(JJ_Result r) {
    Value tv = v;
    r.r1 = tv.x;
    r.r2 = tv.y;
  }
}
```

不过在现在的Valhalla项目中，像volatile Value这样的declaration，会导致Valhalla不再将它这个Value里面的变量给内联掉。因为它不能确保Value里面的变量被flatten之后，可以在64位JVM里进行原子操作。



## Coherence

连贯性。The write==s== to the single memory location appear to be in a total order consistent with program oreder.

---

其实大部分的情况下，可以将program order看做source code order，如果有特殊情况会另外声明，下面的内容都是以这个作为前提的。

---

这个Coherence是什么意思呢？





但是

- 大部分的硬件目前是本身就提供了这个特性的，因为目前的很多硬件都已经通过Currency Protocol保证Cache Coherence，也就不需要JVM去通过内存屏障之类的手段来保证。

  Notice that concurrency problems come when you have several locations updated at the same time, where the juxtaposition between those updates gives you interesting behavior. 

  If the updates are all at the same location, hardware always maintain the single order of these updates, ==**unless**== there is (you have) ==**optimizing compilers which give up on this property (means Coherence), because they don't check the order of reads.**== They can reorder reads unless there is something prevent them from doing this, like ==**memory barrier or synchronization operations**==.



```java
public static class SameRead {
  private final Holder h1 = new Holder();
  private final Holder h2 = h1;
  
  private static class Holder {
    int a;
    int trap;
  }
  
  @Actor
  public void actor1() {
    h1.a = 1;
  }
  
  @Actor
  public void actor2(II_Result r) {
    Holder h1 = this.h1;
    Holder h2 = this.h2;
    
    // Spam null-pointer check folding: try to step on NPEs early
    // Doing this early frees compiler from moving h1.a and h2.a loads around, because it would not have to maintain exception order anymore.
    h1.trap = 0;
    h2.trap = 0;
    
    // Spam alias analysis: the code effectively reads the same field twice, but the compiler does not know (h1 == h2) (i.e. does not check it, as this is not a profitable opt for real code), so it issues two independent loads
    r.r1 = h1.a;
    r.r2 = h2.a;
  }
}
```

这里的代码会产出的结果是：

```
			(r1, r2)
1.    (0 , 0)
2.    (1 , 0)
3.    (0 , 1)
4.    (1 , 1)
```

直观来看，最有迷惑性的可能是 (1, 0) 这个答案，但是通过JCStress是可以看到，这个答案虽然出现的概率很小，但他还是可能会出现的。

至于出现这个的原因，结合上面代码中的注释，唯一合理的结果就是，compiler将

```
r.r1 = h1.a;
r.r2 = h2.a;
```

这两个read操作进行了reorder，变成了

```
r.r2 = h2.a;
r.r1 = h1.a;
```

然后，由于并发的操作，出现(r1, r2) = (0, 1)这样的结果的合理的执行顺序为

```
1. r.r2 = h2.a;
2. h1.a = 1;
3. r.r1 = h1.a;
```



If we expose these reads to the machine in the hardware order it's fine, because hardware provides us with the concurrency.



## Causality

If **A** happened, then **B** happened too. ( what's the relation between A & B ? )

- By far the most basic guarantee made by most memory models, extremely hard to accept as the guiding principle 

  至今为止，大部分内存模型所保证的，最基础的保证。

  在学内存模型标准的时候比较难以理解。

- The cornerstone of most (all?) distributed consistency models

  因果性是大部分分布式一致性模型的基石

看到jcstress本身的例子，在BasicJMM_06_Causality.OpaqueReads这个例子里，由于前面说到，Opaque会避免这条指令被reorder，那么在它给出的例子中，OpaqueReads不应该出现 (x, y) = (1, 0) 这种情况，但是如果是在arm64架构的机器上，还是会有这种情况发生。

原因的话，arm64比x86_64的ordered特性更弱一点，也就是说arm64的指令顺序的顺序性更弱。



### Safe publication

简单来说就是 all or nothing.

- The major and simple rule
  - identify your **acquires** and **releases**
  - check that acquires/releases are on all paths
  - learn this rule again and again
- The whole thing does not require JMM reasoning
  - hardly anyone applies <<happens-before>> correctly
  - hardly anyone can do it reliably 
  - it's very easy to miss the racy access



对于 volatile 变量，由于 SO，volatile不会被reorder，那么当一个volatile writer进行write A，而另一个线程volatile reader读到了这个值A，那么在这个write A之前的所有的write操作都会被看见。比如：

```java
volatile x;
int y, z;
double a, b, c;
public void writerWrite() {
  y = 1;
  z = 2;
  a = 1.0;
  b = 2.0;
  c = 3.0;
  x = 999;
}
public void readReader() {
  // how to arrange your int val = x; ??
  // determines what other values you gonna see...
}
```

对于所有的variable read，如何安排他们之间的顺序对于并发时的行为是很关键的。将这些read分为几部分：

1. read int 
2. read double 
3. read volatile

那么有 todo

```
1 -> 2 -> 3 / 2 -> 1 -> 3:
同上


1 -> 3 -> 2:
- x = 0, int, double的值是不确定的
- x = 99, int的值是不确定的, double的都是新的值

2 -> 3 -> 1:
- x = 0, int, double的值是不确定的
- x = 99, double的值是不确定的, int的都是新的值

3 -> 2 -> 1 / 3 -> 1 -> 2:
- x = 0, int, double的值是不确定的
- x = 99, int, double的值都是确定的
```



所以，一种常见的concurrency code optimization为，将关键的变量定义为volatile，其他的都定义为plain variable，将volatile的write放到最后，将volatile的read放到最前，而volatile能保证一些Causality，即使不做一些额外的同步操作。



## Consensus

Momentary agreement among threads about program state

- THere are different power of consensus, but even the most basic Consensus-1 is useful
- Consensus is mostly about ==**mulitple variables at once**==. Otherwise, **==coherence==** is enough



Very strong one. 



```java
    public static class PlainDekker {
        int x;
        int y;

        @Actor
        public void actor1(II_Result r) {
            x = 1;
            r.r1 = y;
        }

        @Actor
        public void actor2(II_Result r) {
            y = 1;
            r.r2 = x;
        }
    }

    public static class VolatileDekker {
        volatile int x;
        volatile int y;

        @Actor
        public void actor1(II_Result r) {
            x = 1;
            r.r1 = y;
        }

        @Actor
        public void actor2(II_Result r) {
            y = 1;
            r.r2 = x;
        }
    }

    public static class AcqRelDekker {
        static final VarHandle VH_X = ...;
        static final VarHandle VH_Y = ...;

        int x;
        int y;

        public void actor1(II_Result r) {
            VH_X.setRelease(this, 1);
            r.r1 = (int) VH_Y.getAcquire(this);
        }

        public void actor2(II_Result r) {
            VH_Y.setRelease(this, 1);
            r.r2 = (int) VH_X.getAcquire(this);
        }
    }

```

这上面的三种不同情况下的access，只有volatile能保证不会出现(r1, r2) = (0, 0)的情况。

比较有疑惑的在于VarHandle这一块. TODO



- Consensus is good
  - extremely useful to think about correctness
  - avoid non-sequential-consistent data races by going *volatile*
  - sprinkle enough *volatile* around your program, and it eventually becoms data-race-free.
- Consensus is bad
  - extreme cost to get sequential-consistency in distributed system
  - most examples so far were fine with just release/acquire
  - relaxing SC is by far the most common optimization technique



也就是说其实对于硬件来说，SC并不是特别的重要，而常见的优化手段就是放弃SC。一般来说，ralease/acquire就够了。

简单得来说，只要在你的代码里面加入足够多的volatile，你的代码实际上运行起来的顺序就会变的和你的源代码顺序一致，因为volatile保证了足够多的安全性，避免了很多奇怪的data race。





## Final

Declared immutable fields, with additional semantices.

- final are very special: able to hide data race
- the defense-in-depth strategy for concurrent code: work even when external synchronization is broken





## Benign races: Canonical Form

```java
public V racyRacy() {
  V lv = v;
  if (lv == null) {
    lv = compute();
  }
  return lv;
}
```





## About JCStress

Hotspot VM has at least 3 ways to execute Java code:

1. interpreter (-Xint)
2. C1 (Baseline, Client, -XX:-TieredStopAtLevel=1)
3. C2 (Optimized, Server, -XX:-TieredCompilation)





## Advanced Bits







