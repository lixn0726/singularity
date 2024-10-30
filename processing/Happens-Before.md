# Happens-Before

Guarantee **a set of rules** that govern how the **JVM and CPU** is allowed to **reorder instructions** for performance gains.

Happens-Before is centered around access to **`volatile`** variables and variables accessed from **`within synchronized blocks`**.



Happens-Before is ==**provided by**== the **`volatile`** and **`synchronized`** declarations.



## Instruction Reordering

Modern CPUs have ability to execute instructions in parallel if the instructions do not depend on each other. 

- Can parallel

  ```java
  a = b + c;
  d = e + f;
  ```

- Cannot 

  ```java
  a = b + c;
  d = a + e;
  ```

Buf if 

```java
a = b + c;
d = a + e;

l = m + n;
y = x + z;
```

The instructions could be reorder like below, then the CPU can execute at least the first instructions in parallel, and as soon as the first instructions is finished, it can start executing the 4th instructions.

```
// After reorder
a = b + c;

l = m + n;
y = x + z;

d = a + e;
```

所以重排序是为了得到parallel execution吗？



As you can see, reordering instructions can increase parallel execution of instructions in CPU. Increased parallelization means increased performance.

Instruction reordering is allowed for JVM and CPU as long as the ==**semantics of the program**== do not change. The end result must be the same as if the instructions were executed in the exact order they are listed in the source code.

所以注意到这里，用到的描述也是semantics of the program，也印证了，JMM是抽象的。



## Reordering in Multi CPU Computers

Talk about a code example. But remember that the code example is not a recommendation in any way.

Image 2 threads to collaborate to draw frames on the screen as fast as they can. 

- Thread A: Generate the frame
- Thread B: Draw the frame on the screen

The 2 threads need to exchange the frames via some communication mechanism. Like below, by a class called `FrameExchanger`.

```java
public class FrameExchanger {
  private long framesStoredCount = 0;
  private long framesTakenCount = 0;
  private boolean hasNewFrame = false;
  private Frame frame = null;
  // called by Frame producing thread
  public void storeFrame(Frame frame) {
    this.frame = frame;
    this.framesStoredCount++;
    this.hasNewFrame = true;
  }

  // called by Frame drawing thread
  public Frame takeFrame() {
    while (!hasNewFrame) {
      // busy wait until new frame arrives.
    }
    this.framesTakenCount++;
    this.hasNewFrame = false;
    return this.frame;
  }
}
```

Notice how the 3 instructions inside the storeFrame() method seem like they do not depend on each other, which means, to JVM and CPU, it looks like it would be okay to reorder the instructions, in case the JVM or CPU determines that would advantageous.

However, imagine what would happen if the instructions were reordered like:

```java
// origin
public void storeFrame(Frame frame) {
  this.frame = frame;
  this.framesStoredCount++;
  this.hasNewFrame = true;
}
// after reordering ... 
public void storeFrame(Frame frame) {
  this.hasNewFrame = true;
  this.framesStoredCount++;
  this.frame = frame;
}
```

Now, the field `hasNewFrame` is now set to true before the `frame` excatly assigned to reference the incoming `Frame` object.

Which means, the **frame drawing thread** will jump out the `while` loop, and take the old `Frame` object. That will result in a redrawing of an old Frame, leading to waste of resources.



## The Java volatile: Visibility Guarantee

Volatile guarantees for when writes to, and reads of, volatile variables result in synchronization of the variables' value and from main memory.

This synchronization to and from the main memory is what exactly makes the value *visible* to other threads. Hence the term ***visibility guarantee***.



### The Java volatile Write Visibility Guarantee

When writing to a Java `volatile` variable the value is guaranteed to be written directly to main memory. All variables visible to the thread writing to the volatile variable will also get synchrnized to main memory.

There's an example

```java
this.nonVoltileVarA = 34;
this.nonVolatileVarB = new String("Text");
this.volatileVarC = 300;
```

There's 2 writes to `non-volatile`, and 1 write to `volatile`. Because `volatile` restricts the **instruction reordering**, the 2 writes to non-volatile cannot be reordered after the write to volatile. And since the `volatile` will force written directly to the main memory, the write above will also be synchronized to the main memory, which brings up the **safe publication**.



## The Java volatile Read Visibility Guarantee

Also, there's an example:

```java
c = other.volatileVarC;
b = other.nonVolatileB;
a = other.nonVolatileA;
```

For the reason above, if the c can read the latest value written by writer, the thread here can also see the latest value of those non-volatile variables.



## The Java Volatile HappensBefore Guarantee

So, we can know that, the Java `volatile` Happens Before guarantee sets some restrictions on instructions reordering around `volatile` variables. With the modified `FrameExchanger` below as an example:

```java
public class FrameExchanger {
  private long framesStoredCount = 0;
  private long framesTakenCount = 0;
  // change here .
  private volatile boolean hasNewFrame = false;
  private Frame frame = null;
  // called by Frame producing thread
  public void storeFrame(Frame frame) {
    this.frame = frame;
    this.framesStoredCount++;
    this.hasNewFrame = true;
  }

  // called by Frame drawing thread.
  public Frame takeFrame() {
    while (!hasNewFrame) {
      // busy wait until new frame arrives.
    }
    this.framesTakenCount++;
    this.hasNewFrame = false;
    return this.frame;
  }
}
```

Now, when the variable `hasNewFrame` is set to true, the other variables are synchronized to main memory. Additionally, every time the drawing thread reads the `hasNewFrame` in the while-loop, the `frame` and `framesStoredCount` will also be refrehed from main memory.



Imagine if the JVM reordered the instructions inside the `storeFrame` method, like:

```java
public void storeFrame(Frame frame) {
  this.hasNewFrame = true;
  this.framesStoredCount++;
  this.frame = frame;
}
```

Now, the `framesStoredCount` and `frame` fields will get synchronized to main memory when the first instruction is executed ( why ?), which is ***before*** they have their new values assigned to them.

This means, the drawing thread executing the `takeFrame` may exit the `while`-loop before the new value is assigned to the `frame` variable. Even if a new value had been assigned to the `frame` variable by the producing thread, there would **not be any guarantee** that this value would have been synchronized to main memory so ti's visible for the drawing thread.

所以这种写法不符合safe publication，不符合sequential consistency，所以这种reorder是不被允许的。



This type of instruction reordering will make this application malfunction. So it's illegal.



### Happens Before Guarantee for Writes to volatile variable

The instruction reordering above is where the volatile write Happens Before comes in - to put restrictions on that kind of instruction reordering is allowed around writes to volatile variables.



A write to a non-volatile or volatile variable that 
happens before
a write to a volatile variable is guaranteed to happen before 
the write to that volatile variable.

这么分比较容易看得懂。

In the case above, means that the two first write cannot be reordered to happen after the last write to `hasNewFrame`, since `hasNewFrame` is a volatile variable.

```java
public void storeFrame(Frame frame) {
  this.framesStoredCount++;
  this.frame = frame; 
  this.hasNewFrame = true; // volatile here
}
```

But, the first 2 writes before the `hasNewFrame` can be reordered freely, thus, this reordering is allowed:

```java
public void storeFrame(Frame frame) {
  this.frame = frame; 
  this.framesStoredCount++;
  this.hasNewFrame = true; // volatile here
}
```

This reordering does not break the code in the `takeFrame`, as the `frame` variable is still written to before the `hasNewFrame` is written to. The total program still works as intended.



### Happens Before Guarantee for Reads of volatile variables

There's a similar Happens Before guarantee for reads of voltile variables:

A read of volatile variable will 
happen before 
any subsequent reads of volatile and non-volatile variables.



So, **all instructions** before the volatile write, remains before.
And, **all instructions** after the volatile read, remains after.

There are **all instructions**, no matter what instruction is, it cannot betray this volatile Happens Before guarantee.

Back to the volatile read. Example below:

```java
int a = this.volatileVarA;
int b = this.nonVolatileVarB;
int c = this.nonVolatileVarC;
```

Both b, c must remain after the first instruction, because it's a volatile read. Besides, they are free to reorder like below:

```java
int a = this.volatileVarA;
int c = this.nonVolatileVarC;
int b = this.nonVolatileVarB;
```

Because of the volatile Happens Before, when this volatile variable is read from main memory, all other variables visible to the thread at that time. 

Thus, the 2 non-volatile variables will also be read from main memory (why ?). This means the thread that reads volatile can rely on non-volatile to be up-to-date with main memory too (?).

If any of these 2 reads were to be reordered above the first volatile read, the guarantee would not hold up.



## The Java Synchronized Visibility Guarantee

Java `synchronized` blocks provide visibility guarantees that are similar to those of Java volatile variables.

So we just talk about it briefly.



### Java Synchronized Entry Visibility Guarantee

When a thread enters a synchronized block, all variables visible to the thread are refreshed from main memory.



### Java Synchronized Exit Visibility Guarantee

When a thread exits from a synchronized block, **all variables visible to ==the thread==** are written back to main memory.



### Example

```java
public class ValueExchanger {
  private int valA;
  private int valB;
  private int valC;
  
  public void set(Values V) {
    this.valA = v.valA;
    this.valB = v.valB;
    synchronized (this) {
      this.valC = v.valC;
    }
  }
  
  public void get(Values v) {
    synchronized (this) {
      v.valC = this.valC;
    }
    v.valA = this.valA;
    v.valB = this.valB;
  }
}
```

Notice how the 2 blocks are placed last and first in the 2 methods. 

The `synchronized` blocks at the end of the method, will force all the variables to be synchronized to main memory after being updated. This flushing of the variable values to main memory happens **when the thread exits the `synchronized` block**.

That is the reason it has been places last in the method - to guarantee that all updated variable values are **flushed to main memory.**



The `synchronized` blocks at the beginning of the method makes **all variables are ==re-read in== from main memory**. That is why this synchronzied block is placed at the beginning of the method - to guarantee that all variables are refreshed from main memory before they are read.



### Java Synchronized Happens Before Guarantee

So, Java synchronized blocks provide 2 Happens Before guarantees, we'll briefly conclude them below.



### Java Synchronized Block Beginning Happens Before Guarantee

As mentioned before, entering a `synchronized` block will force **all visible variables to the thread (still a question that how to define what's visible to a thread ?)** refreshed from the main memory.

To be able to uphold that guarantee, a set of restrictions on instruction reordering are necessary. Example below:

```java
public void get(Values v) {
  synchronized (this) {
    v.valC = this.valC;
  }
  v.valB = this.valB;
  v.valA = this.valA;
}
```

The synchronized Happens Before guarantees will force:

- `this.valC`
- `this.valB`
- `this.valA`

are all refreshed from main memory (so how to detect this ?). And the following reads of these variables will then use the latest value.

If a read, reordered to appear before the beginning of the `synchronized` block, you would lose the guarantee of this variable value being refreshed from main memory.

So, if the reordering like this:

```java
public void get(Values v) {
  v.valB = this.valB;
  v.valA = this.valA;
  synchronized (this) {
    v.valC = this.valC;
  }
}
```

You will not get the guarantee of the latest value of `this.valA` and `this.valB`. So, this kind of reordering is ==**unpermitted**==, because it will generate the outcomes that not conform to the JMM.



### Java Synchronized Block End Happens Before Guarantee

As mentioned before, the end of a synchronized block provides the visibility guarantee that all changed variables will be written back to main memory when the thread exit the synchronized block.

To be able to uphold that guarantee, a set of restrictions on instruction reordering are necessary. Example below:

```java
public void set(Values v) {
  this.valA = v.valA;
  this.valB = v.valB;
  synchronized (this) {
    this.valC = v.valC;
  }
}
```

The synchronized block at the end, guarantee that all of the changed variables:

- `this.valC`
- `this.valB`
- `this.valA`

will be written back to main memory when the thread exit the synchronized block.

Also, there is a illegal reordering as below:

```java
public void set(Values v) {
  synchronized (this) {
    this.valC = v.valC;
  }
  this.valA = v.valA;
  this.valB = v.valB;
}
```

Some reasons are as above. We don't talk here anymore.





