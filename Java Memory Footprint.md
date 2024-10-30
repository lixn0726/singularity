原视频链接：https://www.youtube.com/watch?v=c755fFv1Rnk
在Linux上通过`top -o %MEM`可以查看机器上所有的进程，并按照内存占用从大到小排列。解释一下top的两个输出：
- VIRT   : 进程占用的虚拟内存大小
- RES    : 进程实际占用的内存大小
实际上可以通过`pmap -x <PID>`来看具体某个进程的内存占用。
一般来说JVM运行时的内存占用包括几部分：
- Java Heap       : -Xmx
- Class Loading   : no limitation
- JIT Compilation : code caches
- Threads         : -Xss
JDK本身带有NMT(Native Memory Tracking)来查看内存占用，通过`-XX:NativeMemoryTracking=[summary|detail]`来开启即可。
但是，NMT的开销比较大，一般在5-10%的性能开销；并且对于每个申请的内存块，NMT需要占用2个额外的字，也就是4bytes。
但是，NMT只能监控JVM申请的内存，在运行时可以通过
```
jcmd <pid> VM.native_memory [detail]
```
来查看正在运行的VM的内存信息。
这里要注意的是，只有开启了NMT才可以通过jcmd来实时获取内存信息，在summary模式下的输出大致如下图：
![[Pasted image 20240927155006.png]]
BTW，jcmd是很有用的工具。

## Java Heap部分
![[Pasted image 20241023150222.png]]
![[Pasted image 20241023150229.png]]
包含上面那张图的Java Heap和GC这两部分。
GC指的是不同GC的堆，需要额外的non-heap空间来实现它的GC算法，并且这里的空间消耗是不可忽视的，根据视频里的一个截图，在Java13下，这里的额外开销最多可达到11%，这是在使用ZGC的情况下，而G1次之，所以，虽然JDK版本升级带来的新的GC会带来更高的吞吐量以及更短的STW，但是显而易见的是这是有成本的，并且这个成本不算低。

对于Java Heap，堆的实际占用大小（物理内存）并不是只指定`-Xmx -Xms`就可以的，实际上这个只会去帮你去申请虚拟内存，比如我们运行一个空的Java应用，运行之后通过top命令来观察实际占用的内存大小（在Linux中，观察top中的RES选项）：
1. `-Xmx4G -Xms4G`:远远小于4G
2. `-Xmx4G -Xms4G -XX:+AlwaysPreTouch`:刚好就是4G
这里的AlwaysPreTouch就是让JVM在申请虚拟内存的时候，直接将对应的物理内存给直接占用了。

### AdaptiveSizePolicy
动态堆大小，允许JVM将堆的物理内存返还给OS，这个东西对于桌面端应用来说是比较有用的，比如说IDE，因为大部分时候它并不需要占用太多内存，所以可以还给OS来让OS调度更多的其他进程。


## Class Loading部分
![[Pasted image 20241023150244.png]]
包含上面那张图的Class这一部分。如果是JDK10以上的版本，那么还可以看到分为Metadata和Class Space两块。其实合起来就是常见的Metaspace，只是说Metaspace里面又可以详细分成两块。
通过参数`-XX:MaxMetaspaceSize=[unlimited]`来控制大小，默认情况下，由于Metadata空间是直接占用堆外的内存，所以是无限的。
### Metadata
- methods
- constant pools
- symbols
- annotations
了解一下就好。
### Class Space
- Compress Classes
对应的JVM参数为`-XX:CompressedClassSpaceSize=1G`，不同于Metadata空间，默认情况下Class Space占用1G的内存大小，所以这里是有可能发生OOM的。

前面提到过，jcmd是一个很有用的工具命令，这里，可以通过
```
jcmd <PID> VM.classloader_stats
```
看到应用中所有类加载器分别加载了多少类，占用了多少空间等等。或者，也可以通过
```
jcmd <PID> GC.class_stats
```
来查看更详细的各个类的信息。不过，首先需要在运行之前添加参数`-XX:+UnlockDiagnosticVMOptions`来启用这个jcmd的command，并且是严格区分大小写的。

需要知道的是，参数`-XX:MaxMetaspaceSize`里指定的大小指的是包含了Class Space的大小的。
其中，还有另外一个参数`-XX:MetaspaceSize`，它并不用来指定Metaspace的大小，实际上，它只起到一个高水位线的作用：当Metaspace的空间占用达到了这个阈值，那么就会触发一次Full GC。默认情况下，这个阈值是20MB。

## JIT Compilation部分
![[Pasted image 20241023150204.png]]

### Code Cache
- nmethods
- interpreter, stubs(runtime stubs)

- -XX:InitialCodeCacheSize  (min)
- -XX:ReservedCodeCacheSize (max)

除了code cache本身，还有一些额外的，compiler本身需要的一些内存，不过graal的这部分内存是放在heap里面的。

code cache存储的就是c1和c2编译的方法，有一个ReservedCodeCacheSize，如果开启了TierCompilation，那么实际的size就是5倍的这个参数，否则就等于这个参数。
code cache eviction，可能是cache size达到上限，那么earlier compiled就会被evict，或者说不符合JIT，也会deoptimization。

## Thread部分
![[Pasted image 20241023144058.png]]
这里图示信息表示：该应用包含23个线程，这些线程一共占用了45320kb的栈大小，那么可以得出默认的线程栈的大小为1MB。
这个大小由-Xss控制，默认-Xss1M，最小到200k。

由于：committed != resident
![[Pasted image 20241023145208.png]]
如上图，貌似意思是，即使commit了这么多，实际上的方法调用一般不会有这么深的栈，所以一般来说，一个线程占用的方法栈大概为50-200k。

## Symbol部分
![[Pasted image 20241023145310.png]]
包含：
- SymbolTable: names, signatures, etc.
- StringTable: interned strings.
可以通过：`jcmd <pid> VM.stringtable | VM.symboltable`
查看详细的信息，可以看到具体的number和占用的内存大小，这些都是non-heap的部分。

- how to print symbol table contents
	https://stackoverflow.com/q/35238902/344841
	9

## Internal部分
![[Pasted image 20241023150311.png]]
这一块是可以grow innormally large的，在视频里他给出的截图里面，internal这部分占用了大约4GB，是一个非常恐怖的数字。
而这里面基本是：
- Direct ByteBuffer of application
- Unsafe.allocateMemory 
JDK11: Internal -> Other，也就是不叫做Internal了，叫做Other。

### Direct ByteBuffer
在Java中，这样的ByteBuffer可以通过两种方法来申请：
1. ByteBuffer.allocateDirect
	通过-XX:MaxDirectMemorySize可以限制大小
	默认大小=Xmx，也就是和heap的大小一致。 
	
1. FileChannel.map
	没有限制，也不会被NMT计入，但是可以通过`pmap -x <pid>`来查看大小。

### Reclaimation of ByteBuffers
- Automatically after GC
direct内存的申请和使用和heap内存很相似：如果direct内存的使用达到了上限，那么此时jdk会尝试调用一次System.gc，然后等待一段时间，再看内存是否被回收。

值得一提的是这里是调用的System.gc，而不是像正常的gc一样是由safepoint发起的，所以如果Java应用禁用了这个调用，也就是参数-XX:+DisableExplicitGC，那么在direct内存使用到上限，并且此时还没有发生过gc的时候，就可能会抛出OOM。
所以，如果应用对direct内存的消耗比较快，建议使用-XX:+ExplicitGCInvokesConcurrent，保证System.gc可用。

那么，使用heap buffer是否会比较好呢？

### Is Heap ByteBuffer better ?
```
byte[] array = new byte[1024];
// fill array
socketChannel.writ(ByteBuffer.wrap(array));
```
并不是，因为Java不能使用heap byteBuffer来进行io。
上面的代码会导致：
1. allocate temporary direct buffer
2. writet to direct buffer
3. release temp direct buffer
所以，jdk在internal会有一个direct byteBuffer cache，他给了个截图：
![[Pasted image 20241023152617.png]]

这里的sun.nio.ch.Util.BufferCache是一个never shrink的东西，并且它是类似于threadLocal的，所以如果有多个线程同时使用heap buffer，这里的cache会变得很大。
自从jdk 8u102之后，出现了个参数-Djdk.nio.maxCachedBufferSize，可以控制这个cache的大小。

而direct buffer还有另一个问题：
```
while (true) {
	ByteBuffer.allocateDirect(random.nextInt(10000));
}
```
这里在不断的申请direct buffer，但是没有将它赋值给其他的对象，所以它们应该可以被马上释放掉。他给出了一个例子，在`-Xmx1G -XX:MaxDirectMemorySize=2G -XX:+UseG1GC`的jvm下运行这段代码，通过top来查看内存占用，可以看到这个java进程的rss会不断的增长，并且会远远超过Xmx+MaxDirectMemorySize = 3G这个大小。但是如果通过nmt来查看的话，会发现其实实际占用的内存并没有超过3G，他给出的截图如下：
![[Pasted image 20241023153453.png]]
那么就有一个肯定是错的，此时通过pmap来查看具体的内存使用，如下图：
![[Pasted image 20241023153537.png]]

这里的问题就在于，jdk里的内存申请是通过malloc申请的，最后走的是mmap。

malloc reserves the memory from the OS in 64M per chunk.
malloc commits memory with `mprotect` with the smaller portions.

简而言之就是malloc默认情况下会导致内存碎片的问题，然后就会不停的去申请chunks，每个chunk默认64MB，即使chunk里的使用率很低，碎片率很高，他还是不停的申请，导致在os层面看到他的rss会很高，但是在jvm看来，他使用的内存并不会有这么多。
所以一般会替换掉默认malloc，比如
1. jemalloc: http://jemalloc.net
2. tcmalloc: https://giithub.com/gperftools/gperftools
3. mimalloc: https://github.com/microsoft/mimalloc
他用的是jemalloc，直接设置一个系统参数：`LD_PRELOAD=/usr/lib64/libjemalloc.so`就可以了。
替换掉之后，就可以在top里的rss看到和nmt一样的大小了。

他还给了一个jemalloc的观测方法：
![[Pasted image 20241023154439.png]]
基本也就是：
![[Pasted image 20241023154616.png]]
在终端里面设置一下系统参数就可以了好像。
运行一段时间之后，在使用最后一行的那个命令，就可以得到一个svg文件，就可以观测到具体的malloc行为的调用路径，以及malloc的大小。
但是这个svg的问题在于，它只能追踪native invoke，所以并不能看到Java程序的调用(只能看到一串16进制)，除非是jvm的调用。

