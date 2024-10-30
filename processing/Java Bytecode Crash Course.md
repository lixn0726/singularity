# Java Bytecode Crash Course

Link: https://www.youtube.com/watch?v=e2zmmkc5xI0

前段时间在看安全点的speech的时候，ppt里有一大部分的bytecode，当时看着有点小混乱，虽然之前自己有了解过字节码，也不太记得，没有深入的了解过。查到的一大部分文章或者教程之类的，要么篇幅过长，要么内容不多，照着jvm spec一顿cv。恰巧查到了一个很不错的视频，看完之后觉得收获颇丰，这篇文章基本出自这个视频，再加上一些额外的内容，而额外的内容也基本上是来自jvm spec和一些java champion的blog进行汇总。

## Why study bytecode

- lingua franca of Java Platform
- Javap output
- Foundation for future study
  - bridge methods
  - type erase
  - debugging



### javap

-v option to "disassemble" bytecode

-p option to see private methods

use latest version

 

## What is bytecode

byte-sized opcodes, only 256 possible opcodes

因为opcode是1byte-sized，也就是8位，所以最多只有256种指令。

operands can be 8 or 16 bits

字节码指令本身只有1byte，但是operand，可以理解为参数，可以是1byte或者2byte。



## Stack Machine

核心思想就是一个stack，把两个操作数压到栈里面，然后执行对应的指令之后，将结果留在栈中，大致如下图：

![image-20241029101955966](/Users/mac/Desktop/assets/articles/assets/image-20241029101955966.png)

大部分的bytecode都是这样的执行机制：take the input from the stack, put the result to the stack.

- Convenience

  - abstract away CPU register details

    根据CPU寄存器的机制来进行抽象，而并不依赖任意一个CPU，比如说它并不依赖于某种特定的指令集，所以这也是bytecode可以run on every platform的原因。

  - free liveness analysis



### Local variables

basically: memory locations

store/load commands

used to access method arguments

 给了个java代码：

```java
    void spin() {
        int i;
        for (i = 0; i < 100; i++) {
            ; // empty
        }
    }
```

先javac编译之后，通过javap -v可以看到对应的class文件的bytecode，如下图：

![image-20241029111546638](/Users/mac/Desktop/assets/articles/assets/image-20241029111546638.png)

可以看到在具体的bytecode opcodes之前，有几个参数：

- stack：当前方法需要的最大的栈的深度，这个参数可以让JVM去申请尽可能少的stack memory，而不是每次都是一个固定的stack memory size。

- locals：number of local variables，这里的java code看起来只有一个int变量，但是这里为2，后面会解释。

- args_size：可以看到这里是1，但是在java code里面，这个方法并不需要参数。那么其实，这个1指的是object本身，也就是一个引用。因为这个spin方法是一个virtual method，并非static method，所以，这里不管这个方法有没有被使用，这里都至少会有一个this引用，所以args_size为1。

  如果这里把spin改为static，那么重新编译之后可以看到字节码里的locals=1，args_size=0，也就印证了上面的说法。

实际上，一个方法，包含有：

1. stack
2. local variable slot

有一个网站总结了所有的bytecode opcode：https://javaalmanac.io/bytecode/

除此之外，https://javaalmanac.io 这个网站还包含了java version feature comparing等，算是一个很不错的工具网站，建议都收藏一下。

结合上面的内容和jvm spec，对上面的bytecode解析一下：

- iconst_0：push int constant 0，也就是往栈里面放一个常数0
- istore_1：store int into local variable 1，也就是将栈顶的值放给第1个slot的local variable。结合上面说到的，第0个slot的local variable是this pointer，所以第1个slot的local variable就是我们定义的i。
- iload_1：load int from local variable，也就是将第1个slot的int值压到栈中
- bipush：push byte，将100以byte的形式压到栈里
- if_icmpge：取出栈顶的两个值，此时，从栈顶到栈底的值value2=100,value1=1，如果value1和value2对比：value1 >= value2，那么就视为满足条件，跳转到对应的字节码上，显然这里不满足，所以往下走。
- iinc 1, 1：这里为了区分开，看作是iinc a, b。实际就是给第a个slot的local variable增加b，这里就是给i+1。
- goto：unconditional jump，显而易见，跳到对应的字节码上去。
- return：return void from method，显而易见，方法结束。

那么解析完这个版本，如果把i改成其他类型的，比如说double，是否会有点不一样呢？那么改造之后，javac、javap -v [classFile]，可以看到字节码：

![image-20241029142902549](/Users/mac/Desktop/assets/articles/assets/image-20241029142902549.png)

可以看到很神奇的是，stack变成了4，locals变成了3，并且方法的整体字节码指令变多了，从14行变成了17行。更加明显的是这里的很多opcode的前缀变成了d，很明显可以知道不同的数值对应不同的opcode。

首先，看到这三个值，为什么stack=4，locals=3。

众所周知，double/long比int要占用更多的位，它们都是64位的，而除此之外，stack machine中的其他所有operand都是32位 (或者更少)，比如：

- boolean
- byte
- char
- short
- int
- float
- reference
- returnAddress

这些都是32位就可以表示的。所以，stack变成了原来的两倍，而local variables包含一个this pointer和一个double，所以是1+2=3。

那么同样来解析一下这一段bytecode：

- dconst_0，dstore_1，dload_1：不解释了，和上面的除了double和int，没区别，此时stack里有一个1。

- ldc2_w：说白了就是从constant pool里面获取一个值，然后压到operand stack去，这里就从constant pool里找了个100，压到栈中。

- dcmpg：从operand stack中pop出两个double，从栈顶到栈底为value2=100，value1=1，这里还不能直接去对比，会经历一次Value Set Conversion：

  JVM里定义了两种浮点值：

  1. standard value set：IEEE 754标准里的单精度和双精度格式
  2. extended-exponent value set：其他的浮点数

  这个conversion主要发生在：

  - float/double值被存储到局部变量
  - float/double值被传递给方法作为参数
  - float/double值作为返回值

  当操作数栈上的浮点值被转换为局部变量或者字段的时候，可能会发生value set conversion，然后会进行一次floating-point comparision：

  - value1 > value2 --- 1
  - value1 = value2 --- 0
  - value1 < value2 --- -1
  - 如果value1、value2其中一个是NaN，

  所以，这里应该是因为将两个浮点值拿来用作方法参数，所以会进行一次conversion。

  这里的dcmpg，返回的只会是1、-1、0。

- ifge：jump if int comparision with zero succeeds，也就是一个条件跳转，根据上面dcmpg的结果判断，当然，前100次都是-1，所以不会跳转。

- dload_1，dconst_1：不解释了，就是加载，然后压一个常数1进到栈里。

- dadd：同样，首先会尝试conversion，然后将栈顶的两个数相加。

- dstore_1，goto，return：上面解释过了。

那么，我们就将doubleSpin的字节码也过了一遍，可以看到，一个简单的int、double差别，在字节码上相差是很大的，double的操作会更多，并且栈一般也会比较深。实际上，intSpin和doubleSpin的区别可以推到32bitSpin和64bitSpin的区别，以小见大。

其中还有一个区别就是，对于浮点数，有一个ldc，而对于整数，用的是bipush，这是因为byte-based无法表示浮点数，只可以表示有限范围内的整形，所以浮点数需要ldc，在编译期间将复杂的constant存储到constant pool，constant pool只是class file的一块，一般来说，字符串和浮点数都会在编译期间被解析存储到constant pool中，需要使用的时候再ldc拿出来。

去翻阅了一下specification里的ldc这一类opcode，包含多种 (上文的也有多种，不过这个我觉得值得拎出来说一下)：

- ldc：push item from constant pool
- ldc_w：push item from constant pool (wide index)
- ldc2_w：push long or double from constant pool (wide index)

可以看到这里就有了一个constant pool的概念，这个constant pool和JVM heap里的constant pool不是同一个东西，这里的constant pool可以理解为一个表，包含多个entry，那么，每个constant value就有一个index，带有w后缀的ldc就通过index来获取对应的constant value。实际上，可以将它们分为两类，同样，分类的依据也是operand size：

- 1 size：获取的constant为32bit
  - ldc
  - ldc_w
- 2 size：获取的constant为64bit
  - ldc2_w

至于w后缀，只是传入的index的值范围不同，带有w后缀的接收2byte范围内的值，否则只接收1byte范围内的值。

int、double字节码的另一个明显的区别就是，对于int和double的value compare：

- int comparing：一条指令
- double comparing：两条指令

对于其他很多类型的compare，大多数情况下，JVM将它们统一视作int来进行compare。如果将i的类型改为short，编译之后看bytecode，也会发现它和int一样，一条指令就可以用来完成compare了。

不过在编译成short版本之后，如图，可以看到一条i2s指令：

![image-20241029184833145](/Users/mac/Desktop/assets/articles/assets/image-20241029184833145.png)

实际上，在自增的这一步，也没有对应的iinc字节码，而是一个operand operation，这也是一个区别。看到i2s这一条指令：

- i2s：convert int to short，某种程度上，这里也算是会自动避免了overflow的问题，因为是直接截断的。

不过我比较好奇的是，既然前面一直都是considered int，为什么这里的自增还要这么操作而不是直接iinc呢 ?



### Constant pool

![image-20241029185406833](/Users/mac/Desktop/assets/articles/assets/image-20241029185406833.png)

看到上图的字节码，对应的Java代码如下：

```java
    void useManyNumeric() {
        int i = 100;
        int j = 100_0000;
        long l1 = 1;
        long l2 = 0xffffffff;
        double d = 2.2;
    }
```

实际上就是定义一些primitive value：

- 100：小于256，所以可以直接bipush进去，然后从栈里面拿出来存到slot-1去，因为slot-0是this pointer。
- 100_0000：很明显不能用byte表示，所以只能在编译期间放到constant pool，index为10，javap也会告诉我们对应的entry的值是什么。然后把这个值存到slot-2。
- 1：这里对于long也有一个直接的opcode，lconst，直接push一个1，然后存到slot-3。
- 0xffffffff：这里虽然可以用byte表示，但是实际上还是从constant pool加载出来的，不是很懂为什么。然后就存到slot-5，这里的5是因为，前面的long占用两个slot，所以从下标来说，这里是slot-5。
- 2.2：很明显，浮点数byte表示不了，所以从constant pool取出来，store到slot-7，原因同上。



### argument passing (virtual)

一段简单的代码来解释：

```java
    int addTwo(int a, int b) {
        return a + b;
    }
```

对应的bytecode如下图：

![image-20241029190359144](/Users/mac/Desktop/assets/articles/assets/image-20241029190359144.png)

现在都很容易理解这里的各个值了，也很容易能看懂这里的opcode，所以不解释了。

而如果将这个方法定义成static呢？



### argument passing (static)

```java
    static int addTwo(int a, int b) {
        return a + b;
    }
```

其实字节码也很容易知道，不过还是截个图：

![image-20241029190629979](/Users/mac/Desktop/assets/articles/assets/image-20241029190629979.png)

基本没差别，就是locals和arg_size比virtual method少了1，然后多了个flag，标注为静态方法，具体原因前面也说过了。



### method call (virtual)

那么，来看到方法调用，结合上面的addTwo方法，定义一个调用它的方法：

```java
int add12and13() {
  return addTwo(12, 13);
}
```

编译出来后，这个方法的bytecode为：

![image-20241029190846104](/Users/mac/Desktop/assets/articles/assets/image-20241029190846104.png)

注意到这里，第一行字节码为aload_0，也就是将local variable slot-0压到栈中，那么也就是this pointer。后面的bipush也可以理解，然后就看到这个invokevirtual，就是来调用addTwo这个virtual method，在这条opcode之前，栈里的内容从底到顶分别为：

0. this
1. 12
2. 13

然后，invokevirtual就根据这三项去获取constant pool的index=10的entry，这个entry装的就是addTwo这个方法的reference。

而如果addTwo改成静态的，那么其他的基本不变，不再需要第一步的aload_0，然后将invokevirtual调整为invokestatic即可，需要记住的是，方法调用在bytecode层面来说基本就是load constant pool entry (说基本，是因为有个特殊情况invokedynamic)，它只告诉JVM对应的方法引用，而在运行期间，则是由JVM来决定是调用哪个object的这个方法。

TODO：不过我有个好奇的地方是，为什么这里没有定义local variable，即使是static，locals也是1呢？



### method call (private)

看到一段代码：

```java
class Near {
  int it;
  public int getItNear() {
    return getIt();
  }
  private int getIt() {
    return this.it;
  }
}
```

主要关注getItNear，它对应的bytecode：

![image-20241029191937745](/Users/mac/Desktop/assets/articles/assets/image-20241029191937745.png)

看到这里，如果是private method call，对应的opcode为invokespecial。同样，如果是super method call，如下：

```java
class Far extends Near {
  public int getItNear() {
    return super.getItNear();
  }
}
```

编译出来的bytecode如下：

![image-20241029192705567](/Users/mac/Desktop/assets/articles/assets/image-20241029192705567.png)

可以看到也是个invokespecial，只是后面的签名有点长。



## types of calls

实际上，method call有5种opcode：

1. invokevirtual

   **instance method**

   this pointer

   virtual lookup

   default calling，在运行时通过this pointer来推断真正的调用方的类型，然后来进行方法调用。

2. invokestatic

   **class method**

   no this pointer

   static linking

   区别于invokevirtual，invokestatic在编译期其实就知道具体调用的是哪个方法。

3. invokeinterface

   **interface method**

   this pointer

   virtual lookup

   和invokevirtual差不多，但是会复杂一点，他没深入讲。

4. invokespecial

   **everything else**

   - constructors：new的时候不可能new歪来。
   - super class：手动指定了extends，不可能指错地方。
   - private：当然就在当前类的本身，直接this，不可能指错地方。

   this pointer

   static linking

   special的点就在于，它即有this pointer，又有static linking，那么也就是，虽然它有this pointer，但是它不需要做virtual lookup，具体的调用方是在编译期间就得知的。

5. invokedynamic





### Constructor

```java
Object create() {
  return new Object();
}
```

直接看bytecode：

![image-20241029200411431](/Users/mac/Desktop/assets/articles/assets/image-20241029200411431.png)

注意到这里，invokespecial之前有一个new，new实际上只是去申请内存，因为如果自己动手试一下，可以看到这个类的constant pool，这里的index=4的地方实际上是一个Class信息：

![image-20241029200516155](/Users/mac/Desktop/assets/articles/assets/image-20241029200516155.png)

所以，这里是根据Object这个类的信息去申请分配内存，此时还没有执行构造函数。再往下看到一个dup。

- new：create an object，如果去到jvms，可以看到在分配完内存之后，对应的instance fields都会被initialized to default value，然后，会将这个新申请的object的reference (the reference that points into Java heap) 放在栈顶。
- dup：duplicate the top operand stack value，也就是在栈顶上搞一个相同值的东西。
- areturn：return reference from method，也就是返回一个pointer。以a开头的opcode都是和address相关的，也就是pointer/reference。

invokespecial的index=1可以看到就是个init方法，也就是构造函数，在这里就直接调用了。这里的dup的用处是，在执行完dup之后，stack里就有两个相同的指针，然后，将top of stack的那个指针给到构造函数去consume，对这个pointer指向的object进行初始化，然后就可以直接将剩下的那个pointer返回，因为它们指向的是同一个object，而不需要再去获取一次pointer。



### field access

最后再来看一下成员变量的访问，以get/set为例：

```java
int it;
public void setIt(int it) {
  this.it = it;
}
public int getIt() {
  return this.it;
}
```

对应的bytecode：

![image-20241029201706907](/Users/mac/Desktop/assets/articles/assets/image-20241029201706907.png)

很简单，也没啥说的。



## Others

还有一些其他的bytecode opcode，比如array相关的，还有switch，switch可以分为两类：

- switch - table

  如果switch里的各个case的值是连续的，那么就是table。

- switch - sparse

  不连续，那么就是sparse的，不过最后都是通过jump去进行字节码跳转。

还有一些和stack相关的opcode，比如pop、dup、swap这种，了解一下就好了。

还有一点就是和异常相关的，可以提一下，直接看代码：

```java
    void catchOne() {
        try {
            tryItOut();
        } catch (TestExc e) {
            handleExc(e);
        }
    }

    void tryItOut() throws TestExc {
    }

    void handleExc(RuntimeException e) {

    }
```

编译结果为：

![image-20241029202446018](/Users/mac/Desktop/assets/articles/assets/image-20241029202446018.png)

可以看到，这里会有一个exception table。而在invokevirtual之后紧跟着就是一个goto，是正常执行然后结束方法的逻辑。而根据这个exception table，如果在0-4这个区间内的opcode出现了TestExc这个异常，那么程序就会走到第7行opcode，也就是处理异常的逻辑，所以exception table：

if there is *typed exception* occur between [*from*, *to*], jump into *target*。

这里还可以提一嘴的就是，这里的locals=2，一个是this pointer，另一个就是exception e。



TODO：后续补全一下具体的逻辑。

最后，finally，假设在上面的Java代码里加一个finally，执行方法runWhateverHappen()，编译之后的bytecode：

![image-20241029203020867](/Users/mac/Desktop/assets/articles/assets/image-20241029203020867.png)

可以看到，多了一个finally，exception table里多了两个entry。并且在finally里的那个方法，在bytecode里面出现了三次，可以根据这个将bytecode分为三部分：

- 第一部分代表正常的执行，不出现异常

  ![image-20241030104229734](/Users/mac/Desktop/assets/articles/assets/image-20241030104229734.png)

- 第二部分代表tryItOut出现异常

  ![image-20241030104403046](/Users/mac/Desktop/assets/articles/assets/image-20241030104403046.png)

- 最后一部分看起来比较奇怪，但是实际上，这里处理的是uncaught exception的情况

  ![image-20241030104346443](/Users/mac/Desktop/assets/articles/assets/image-20241030104346443.png)

所以，再回头看到这段bytecode的locals=3，有点难以理解，我去查了一下对应的资料显式：

- 每个方法都有一个max_locals：

  the value of the max_locas item gives the number of local variables in the local variable array allocated upon invocation of this method, including the local variables used to pass parameters to the method on its invocation.

  

