# G1 Details



## G1 logging

- -XX:+PrintGCDateStamps : prints date and uptime
- -XX:+PrintGCDetails : prints G1 phases
- -XX:+PrintAdaptiveSizePolicy : prints ergonomic decisions
- -XX:+PrintTenuringDistribution : prints aging information of survivor regions

也就是G1本身自带一些很详细的日志，通过JVM参数开启就行，并且开销比较低。

Always keep them enabled

## G1 Memory Layout

- G1 divides heap into small "regions"
- Targets 2048 regions : tuned via -XX:+G1HeapRegionSize
- Eden, Survivor, Old regions
- "Humongous" regions
  - when a single object occupies > 50% of a region
  - typically byte[] or char[]



## G1 Young GC

- jvm starts, prepare Eden regions
- application runs and allocate into Eden regions (TLAB)
- Eden regions fill up
- when **all** Eden regions are full -> Young GC 



- application modifies the pointers of existing objects

- an "old" object may point to an "eden" object

  an "old" Map has just been put() a new entry

- G1 must track these inter-generatino pointers

  (Old|Humongous) -> (Eden|Survivor) pointers



Young GC只看Eden，否则会too expensive。

由于上面的一点：an "old" object may point to an "eden" object。

所以需要一些额外的数据结构来track。这里的数据结构为：

![image-20241025132346033](/Users/mac/Desktop/assets/articles/assets/image-20241025132346033.png)

Remembered Set: data structure that remembers for a particular Eden region, what are the objects that point to the objects that live into this region. ???

Says that there's someone from outside, that's pointing to me, and the Remembered Set can tell where these guys from outside are.

The card table tells inside this particular region where are the external guys that points to this region.



所以，总结一下就是这些额外的数据结构：

- Card Table

  存放reference modification information

- Remembered Set

  存放outer objects that points to this region



### G1 Remembered Set

- A write barrier tracks pointer updates

  object.field = <reference>

- Triggers every time a pointer is written

  - records the write information in the card

  - cards are stored in a queue (dirty card queue)

  - the queue is divided into 4 zones:

    - white

      nothing happens, buffers are left unprocessed 

    - green (-XX:G1ConcRefinementGreenZone=<>)

      G1 starts background threads (called Refinement Threads)

      - refinement threads are activated
      - Buffers are processed and the queue drained

    - yellow (-XX:G1ConcRefinementYellowZone=<>)

      start all the refinement threads

      - All available refinement threads are active

    - red (-XX:G1ConcRefinementRedZone=<>)

      slow down the application, ensure the g1 can catch up the modification.

      - **Application threads** process the buffers

Every time a pointer is written, G1 stores the information in the card, it marks the card and say it's dirty. Then put this **information** into a queue (dirty card queue). 

This queue can grow up along with the application modification. 

这里为什么不immediatly更新remembered set：因为rs包含多个对象，有可能会造成赋值操作对同一个region进行竞争，反正也不需要时刻保证up-to-date，所以用一个queue存起来慢慢更新就可以了。





### G1 Young GC Phases

- G1 Stop The World

- G1 builds a **collection set**

  contains:

  - Eden regions
  - Survivor regions



#### Young GC Actual Phases

1. Root Scanning

   G1 has to start from **known places** and **known to be alive**. Which means the static fields in classes, and the local variables live in thread stacks are scanned. 

   G1 walk through all the thread stack, **frame by frame**.

2. Update Remembered Set

   Must drains the dirty card queue to update the Remembered Set to get a consistent remembered set.

3. Process Remembered Set (19:32~19:56 not that understand...)

   detect the Eden objects pointed by old objects.

   Go through the Remembered Set, follow the pointers **back to** all the old regions, **figure out what are the objects that points to me**.

4. Object copy

   the object graph is traversed.

   live objcets copies to Survivor/Old regions.

   Does many things at once. Start from the alive roots, copy them, traverse the children, copy them, and so on.

   并且还可以知道what kind of the object. Keep them aside, process them later.

   并且，还可以keep track of how long does it take to clean up one sigle region.

5. Reference Processing

   sofe, weak, phantom, final, jni weak references.

   always enable -XX:+ParallelRefProcEnabled, this can enable to have multi threads to process these references, default is single-threaded.

   more details with -XX:+PrintReferenceGC, but 貌似有开销而且不小？



- G1 tracks phase times to autotune
- phase timing used to change the # of regions
  - Eden region count
  - Survivor region count
- By updating the # of the regions
  - respect of max pause target
- Typically, the shorter the pause target, the smaller the # of Eden regions

If the tracked time is larger than the specified, means that maybe the Eden regions are too much: so reduce the number of Eden regions, which may cause more GC actions.

On the contray, if the tracked time is smaller than the specified, G1 can make more regions as Eden, which will cause less GC actions.



## G1 Old GC

- G1 schedules an Old GC based on heap usage
  - By default, when the entire heap is 45% full
    - Checked after a Young GC or a humongous allocation
  - Tunable via -XX:InitiatingHeapOccupancyPercent=<>
- The Old GC consists of old region marking
  - Finds all the live objects in the old regions
  - old regions marking is concurrent

We notice that the marking phase is concurrent, how does it works ? because we all know that the application will always changing the pointer when running.

- The answer is 3-color (Tri-color) marking



### Tri-color marking

1. Roots marked blacked - children marked gray, then put the children in a queue

2. Then, Take the objects out of queue in order, follow its pointers, and mark these objects gray, once all the children are marked gray, the object itself turns to black.

   And, these follower objects will enqueue into the queue mentioned before.

   也就是说，有一个队列，我就把它叫做gray queue，里面存放的都是black object的children。并且每次找到black的children，标记为gray之后，把black object本身挪出去，然后把这些children挪进去。

   只要object进到这个队列，那么不管它有没有children，在G1把它拿出来之后，它都会变成black，这些object即使没有children，它也被其他object point to了，那么也就是从GC Root出来的，所以都是alive的，所以最终的归宿，要么black，要么white，符合历史。

   - black: alive
   - white: garbage

   并且，永远不可能会有black -> white这样的pointer。



#### Concurrent marking: Lost Object Problem

假设一个原本的场景：

- Root(black) -> A(black) 
- Root(black) -> B(gray) -> C(white)

此时GC正在marking，假设此时GC thread CPU时间片用完了，暂停运行。此时应用线程还在跑，假设application thread执行

1. b.c = null
2. a.c = C

那么此时，我们期望中的链路应该是：

- Root(black) -> A(black) -> C(gray)
- Root(black) -> B(gray)

但是，由于此时GC thread暂停了，并且这些赋值information是放到card table的，所以GC thread感知不到，此时GC thread恢复，从第一个链路中取出gray object，也就是B，没有children pointer，那么将B标记为black之后，traverse就结束了。那么，此时C就会保持为white，就会被视为garbage，被GC掉。

那么，此时就和应用的状态违背了，因为最开始的a.c = C被GC thread忽略了，GC thread只看到了B没有了point to C的pointer，那么，假设这样运行完之后，访问a.c就会出现问题，有可能会是其他的object，也有可能是segment fault。

为了解决这个问题，G1：

- G1 uses a write barrier to detect: b.c = null
  - more precisely that a pointer to C has been deleted 
- G1 now knows about object C
  - speculates that object C will remain alive
- Snapshot-At-The-Begining (SATB)
  - preserves the object graph that was live at marking start
  - C is queued and processed during remark
  - may retain floating garbage, collected the next cycle

所以这里就知道了，G1有两种write barrier：

- o.x = b    : 引用被修改成一个真实的对象
- o.x = null : 引用被删除 (SATB)

在G1，这是两个不同的write barrier (吗)。

其实说到底，核心就是keep as more objects alive as possible，只要前面扫到过这个obj，那么后续的marking阶段，这个obj都会被视作alive，而不会因为application thread的modification而导致这个obj被gc。

这个很明显会出现floating garbage也就是浮动垃圾，但是一点点的内存浪费，总好过更长的gc process time。下一次gc cycle，这些垃圾就会被回收掉。

SATB happen to be efficient，CMS就没有采用SATB这个技术。



A little bit more space, G1 works better.



## G1 Old GC Phases

1. G1 STW

2. performs a Young GC 

   piggyback old region roots detection (initial-mark)

   也就是说，Old GC的时候，也会触发一次Young GC，这次的Young GC就会找到Old Region Roots，G1就可以知道从哪里开始scan

3. resumes application threads

4. concurrent old region marking proceed

   - keeps track of references

   - computes per-region liveness information

     这里是计算每个Old region的存活率，比如说一个region里有多少垃圾，有多少存活对象

5. G1 STW

6. remark phase

   1. SATB queue processing (?)
   2. reference processing

7. cleanup phase

   empty old regions are immediately recycled

   full of garbage的region会立刻被回收

8. resume application threads 



可以看到少了一点东西：如果一个old region，即有garbage，又有live objects，这里并没有做处理。这就是G1和其他垃圾回收器的区别。

给出来了一个GC Log，如下：

```
[GC pause (G1 Evacuation Pause) (young) (initial-mark)]
[GC concurrent-root-region-scan-start]
[GC concurrent-root-region-scan-end, 0.0124142 secs]
[GC concurrent-mark-start]
[GC concurrent-mark-end, 0.0124142 secs]
[GC remark, 0.0124142 secs]
[GC cleanup 16G->14G(2G), 0.0124142 secs]
```

看到最后的16G->14G，回收了2G的空间，但是这个2G空间指的是，所有full of garbage的region加起来的空间。所以其实，old regions可能还包含有更多的garbage没有立刻被回收。



So, what about the non-completely full of garbage regions ?

- cleanup phase -> recycles empty old regions

- what about non-empty old regions

  how is fragmentation resolved ?

- non-empty old regions processing

  - happens during the next Young GC cycle
  - no rush to clean the garbage in old regions



## G1 Mixed GC

- "Mixed" GC - piggybacked on Young GCs

  - By default G1 performs 8 mixed GC
  - -XX:G1MixedGCCountTarget

- the collection set includes

  - part of the remaining old regions to collect (1/G1MixedGCCountTarget)
  - eden regions
  - survivor regions

- algorithm is identical to Young GC

  - STW, parallel, copying

  也就是使用和Young GC一模一样的算法

  know where are the objects alive --- navigate the object tree --- copy them to other region

  经过这样的GC之后，old region就可以被compact。并且，一部分old region可以被recycle。这是G1和CMS最大的区别。

所以，需要有Old GC发生之后，才会有Mixed GC，或者说，在Old GC发生之后，剩下的G1MixedGCCountTarget次Young GC都叫做Mixed GC，在这么多次之后，才是纯粹的Young GC。

Young GC有一个Mixed Flag，就是和这个有关的。



- old regions with most garbage are chosen first

  - -XX:G1MixedGCLiveThresholdPercent
  - defaults to 85%

  很简单，最多垃圾的region最先被处理，这和第二、三点的目的是契合的。

- G1 wastes some heap space

  - -XX:G1HeapWastePercent
  - Defaults to 5%

  因为如果一个region里的most part都是live object，那么为了清除一点点的garbage，去进行很多object的copy是很得不偿失的，所以，G1允许一些垃圾的存在，即使这会导致堆内存的浪费，这个指标针对的也是整个heap，指的是old regions waste的占比。

- Mixed GCs are stopped

  - when old region garbage <= waste threshold

  - therefore, mixed GC count may be less than G1MixedGCCountTarget

    结合上面的两个点，可以知道，Mixed GC，越往后执行，针对old region的收集效率就越低，因为garbage percent是一直在降低的，所以这里搞个策略来打断Mixed GC是蛮不错的。



## G1 General Advices

- avoid at all costs Full GCs
  - the Full GC is single threaded and really slow
  - also because **G1 like BIG heaps**.
- grep the GC logs for "Full GC"
  - use -XX:+PrintAdaptiveSizePolicy to know what caused it



- avoid "to-space exhausted"

  - not enough sapce to move objects to
  - increase max heap size
  - G1 works better with more room to maneuver 

  这里说的是，比如一个堆，众所周知Survivor区是比较小的，假设此时Eden存活的对象很多，并且Old也满了，那么此时要将live objects挪到Survivor，Survivor的空间不够，就会出问题 (他说会触发Full GC，后续需要去查一下)。

  不过，遇到这个问题，基本上意味着你的堆太小了，首先需要增大堆空间。



- avoid too many "humongous" allocation

  - -XX:+PrintAdaptiveSizePolicy prints the GC reason
  - increase max heap size
  - increase regions size: -XX:G1HeapRegionSize
  - Example
    - Max heap size 32G -> region size = 16M
    - humongous limit = 8M
    - allocations of 12M arrays
    - set region size to 32M
    - humongous limit is now 16M
      - 12M array are not humongous anymore

  基本上应该是对于humongous allocation，G1会执行更多的代码，那么假如说有大量的humongous allocation，会导致性能损耗，所以要扩大堆内存，或者增大region size。



- avoid lengthy reference processing
  - always enable: -XX:+ParallelRefProcEnabled
  - more details with: -XX:+PrintReferenceGC
- find the cause for WeakReferences
  - ThreadLocals
  - RMI
  - Third party libraries

 



### Real World Example

- Online Chess Game Application
- 20k requests/s
- 1 server, 64G RAM, 2x16 cores
- allocation rate: 0.5-1.2G/s
- 24 hours running
- CMS to G1 migration



#### Issues

1. Max target pause

2. Mixed GC

   这个东西我倒觉得值得留意一下，和max target pause息息相关

   ![image-20241028192914278](/Users/mac/Desktop/assets/articles/assets/image-20241028192914278.png)

   因为g1 track timing consume，所以一次Full GC之后，假如说1/G1MixedGCCountTarget的regions太多，那么后续的一次Mixed GC耗时就会非常的长，那么，g1就会shrink Eden regions，像他图片上的，Eden直接从12G被砍到了0.6G，也就是600M，考虑到它本身这个应用的allocation就在500M-1.2G/s，那么Young GC就会频繁的触发。

   - young generation shrunk by a factor 20x

     but the allocation rate does not change

   - Young GC becomes more frequent

     application throughput suffers

   - minimum mutator utilization (MMU) drops

     MMU: how long in a period of time is your application actually running

   不过被砍掉的Eden，后续是会回来的，也就是Eden后续是会被动态调整回来的。

   ![image-20241028193528469](/Users/mac/Desktop/assets/articles/assets/image-20241028193528469.png)

   看到这两个图，上面是GC发生的时间，蓝色柱子表示发生GC，下面的是当时的Eden size，可以看到在18420的时候，Mixed GC开始，导致Eden，boom，直接断崖式下降。

   ![image-20241028193520090](/Users/mac/Desktop/assets/articles/assets/image-20241028193520090.png)

   下面这个图是MMU，可以看到，Eden降低，导致Young/Mixed GC频繁，应用实际运行的时间也是会下降的，Y轴代表的是百分比，这就导致throughput也就是吞吐量受到很大的影响。

   ![image-20241028193738483](/Users/mac/Desktop/assets/articles/assets/image-20241028193738483.png)



## Conclusion

- easier to tune than CMS
- not yet that respectful of GC pause target
- still based on STW
- always use the most recent JDK



