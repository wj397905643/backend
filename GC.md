### 新生代收集器

#### Serial收集器
Serial（串行）收集器是最基本、发展历史最悠久的收集器，它是采用复制算法的新生代收集器，曾经（JDK 1.3.1之前）是虚拟机新生代收集的唯一选择。它是一个单线程收集器，只会使用一个CPU或一条收集线程去完成垃圾收集工作，更重要的是它在进行垃圾收集时，必须暂停其他所有的工作线程，直至Serial收集器收集结束为止（“Stop The World”）。

#### ParNew 收集器
一款多线程的收集器，采用复制算法，主要工作在 Young 区，可以通过 -XX:ParallelGCThreads 参数来控制收集的线程数，整个过程都是 STW 的，常与 CMS 组合使用。

#### Parallel Scavenge 收集器
Parallel Scavenge收集器也是一个并行的多线程新生代收集器，它也使用复制算法。Parallel Scavenge收集器的特点是它的关注点与其他收集器不同，CMS等收集器的关注点是尽可能缩短垃圾收集时用户线程的停顿时间，而Parallel Scavenge收集器的目标是达到一个可控制的吞吐量

### 老年代收集器
#### Serial Old
Serial Old 是 Serial收集器的老年代版本，它同样是一个单线程收集器，使用“标记-整理”（Mark-Compact）算法。

#### Parallel Old收集器
Parallel Old收集器是Parallel Scavenge收集器的老年代版本，使用多线程和“标记-整理”算法。前面已经提到过，这个收集器是在JDK 1.6中才开始提供的，在此之前，如果新生代选择了Parallel Scavenge收集器，老年代除了Serial Old以外别无选择，所以在Parallel Old诞生以后，“吞吐量优先”收集器终于有了比较名副其实的应用组合，在注重吞吐量以及CPU资源敏感的场合，都可以优先考虑Parallel Scavenge加Parallel Old收集器。

#### CMS收集器
CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器，它非常符合那些集中在互联网站或者B/S系统的服务端上的Java应用，这些应用都非常重视服务的响应速度。从名字上（“Mark Sweep”）就可以看出它是基于“标记-清除”算法实现的。
+ 对CPU资源非常敏感 其实，面向并发设计的程序都对CPU资源比较敏感。在并发阶段，它虽然不会导致用户线程停顿，但会因为占用了一部分线程（或者说CPU资源）而导致应用程序变慢，总吞吐量会降低。CMS默认启动的回收线程数是（CPU数量+3）/4，也就是当CPU在4个以上时，并发回收时垃圾收集线程不少于25%的CPU资源，并且随着CPU数量的增加而下降。但是当CPU不足4个时（比如2个），CMS对用户程序的影响就可能变得很大，如果本来CPU负载就比较大，还要分出一半的运算能力去执行收集器线程，就可能导致用户程序的执行速度忽然降低了50%，其实也让人无法接受。
+ 无法处理浮动垃圾（Floating Garbage） 可能出现“Concurrent Mode Failure”失败而导致另一次Full GC的产生。由于CMS并发清理阶段用户线程还在运行着，伴随程序运行自然就还会有新的垃圾不断产生。这一部分垃圾出现在标记过程之后，CMS无法再当次收集中处理掉它们，只好留待下一次GC时再清理掉。这一部分垃圾就被称为“浮动垃圾”。也是由于在垃圾收集阶段用户线程还需要运行，那也就还需要预留有足够的内存空间给用户线程使用，因此CMS收集器不能像其他收集器那样等到老年代几乎完全被填满了再进行收集，需要预留一部分空间提供并发收集时的程序运作使用。
+ 标记-清除算法导致的空间碎片 CMS是一款基于“标记-清除”算法实现的收集器，这意味着收集结束时会有大量空间碎片产生。空间碎片过多时，将会给大对象分配带来很大麻烦，往往出现老年代空间剩余，但无法找到足够大连续空间来分配当前对象。

### GC 场景
#### 空间震荡
定位：观察 CMS GC 触发时间点 Old/MetaSpace 区的 committed 占比是不是一个固定的值，或者像上文提到的观察总的内存使用率也可以。

解决：尽量将成对出现的空间大小配置参数设置成固定的，如 -Xms 和 -Xmx，-XX:MaxNewSize 和 -XX:NewSize，-XX:MetaSpaceSize 和 -XX:MaxMetaSpaceSize 等。确保 -Xms 和 -Xmx 设置的是一个值（即初始值和最大值一致），获得一个稳定的堆，同理在 MetaSpace 区也有类似的问题。
#### MetaSpace 区 OOM
MetaSpace 主要由 class Metaspace 和 NoKlass Metaspace 两大部分组成。

+ class MetaSpace：就是用来存 class 的，就是 Class 文件在 JVM 里的运行时数据结构，这部分默认放在 Compressed Class Pointer Space 中，是一块连续的内存区域，紧接着 Heap。Compressed Class Pointer Space 不是必须有的，如果设置了 -XX:-UseCompressedClassPointers，或者 -Xmx 设置大于 32 G，就不会有这块内存，这种情况下 Klass 都会存在 Noclass Metaspace 里。
+ Noclass MetaSpace：专门来存 class 相关的其他的内容，比如 Method，ConstantPool 等，可以由多块不连续的内存组成。虽然叫做 NoKlass Metaspace，但是也其实可以存 class 的内容，上面已经提到了对应场景。

解决：原理理解比较复杂，但定位和解决问题会比较简单，经常会出问题的几个点有 Orika 的 classMap、JSON 的 ASMSerializer、Groovy 动态加载类等，基本都集中在反射、Javasisit 字节码增强、CGLIB 动态代理、OSGi 自定义类加载器等的技术点上。

#### 过早晋升

这种场景主要发生在分代的收集器上面，专业的术语称为“Premature Promotion”。90% 的对象朝生夕死，只有在 Young 区经历过几次 GC 的洗礼后才会晋升到 Old 区，每经历一次 GC 对象的 GC Age 就会增长 1，最大通过 -XX:MaxTenuringThreshold 来控制。
原因： 
+ Young/Eden 区过小：过小的直接后果就是 Eden 被装满的时间变短，本应该回收的对象参与了 GC 并晋升，Young GC 采用的是复制算法，由基础篇我们知道 copying 耗时远大于 mark，也就是 Young GC 耗时本质上就是 copy 的时间（CMS 扫描 Card Table 或 G1 扫描 Remember Set 出问题的情况另说），没来及回收的对象增大了回收的代价，所以 Young GC  时间增加，同时又无法快速释放空间，Young GC 次数也跟着增加。
+ 分配速率过大：可以观察出问题前后 Mutator 的分配速率，如果有明显波动可以尝试观察网卡流量、存储类中间件慢查询日志等信息，看是否有大量数据被加载到内存中。
策略： 
适当增大 Young 区，具体怎么增加？一般情况下 Old 的大小应当为活跃对象的 2~3 倍左右，考虑到浮动垃圾问题最好在 3 倍左右，剩下的都可以分给 Young 区。



### 使用jstack分析cpu问题
+ 先用ps命令找到对应进程的pid
+ 接着用top -H -p pid来找到cpu使用率比较高的一些线程
+ 将占用最高的pid转换为16进制printf '%x\n' pid得到nid
+ 在jstack中找到相应的堆栈信息jstack pid |grep 'nid' -C5 –color

通常我们会比较关注WAITING和TIMED_WAITING的部分，BLOCKED就不用说了。我们可以使用命令cat jstack.log | grep "java.lang.Thread.State" | sort -nr | uniq -c来对jstack的状态有一个整体的把握，如果WAITING之类的特别多，那么多半是有问题啦。

### 频繁gc
使用jstat -gc pid 1000命令来对gc分代变化情况进行观察，1000表示采样间隔(ms)，S0C/S1C、S0U/S1U、EC/EU、OC/OU、MC/MU分别代表两个Survivor区、Eden区、老年代、元数据区的容量和使用量。YGC/YGT、FGC/FGCT、GCT则代表YoungGc、FullGc的耗时和次数以及总耗时

### 上下文切换
vmstat命令来进行查看，cs(context switch)一列则代表了上下文切换的次数。
我们希望对特定的pid进行监控那么可以使用 pidstat -w pid命令，cswch和nvcswch表示自愿及非自愿切换。

### 磁盘
直接使用df -hl来查看文件系统状态
磁盘问题还是性能上的问题。我们可以通过iostatiostat -d -k -x来进行分析

### 内存
+ Exception in thread "main" java.lang.OutOfMemoryError: unable to create new native thread
这个意思是没有足够的内存空间给线程分配java栈，基本上还是线程池代码写的有问题，比如说忘记shutdown，所以说应该首先从代码层面来寻找问题，使用jstack或者jmap。如果一切都正常，JVM方面可以通过指定Xss来减少单个thread stack的大小。另外也可以在系统层面，可以通过修改/etc/security/limits.confnofile和nproc来增大os对线程的限制

+ Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
这个意思是堆的内存占用已经达到-Xmx设置的最大值，应该是最常见的OOM错误了。解决思路仍然是先应该在代码中找，怀疑存在内存泄漏，通过jstack和jmap去定位问题。如果说一切都正常，才需要通过调整Xmx的值来扩大内存。

+ Exception in thread "main" java.lang.StackOverflowError
表示线程栈需要的内存大于Xss值，同样也是先进行排查，参数方面通过Xss来调整，但调整的太大可能又会引起OOM。

排查： OOM和StackOverflow的代码排查方面，我们一般使用JMAPjmap -dump:format=b,file=filename pid来导出dump文件，另一方面，我们可以在启动参数中指定-XX:+HeapDumpOnOutOfMemoryError来保存OOM时的dump文件。

### gc问题和线程
gc问题除了影响cpu也会影响内存，排查思路也是一致的。一般先使用jstat来查看分代变化情况，比如youngGC或者fullGC次数是不是太多呀；EU、OU等指标增长是不是异常呀等。
线程的话太多而且不被及时gc也会引发oom，大部分就是之前说的unable to create new native thread。除了jstack细细分析dump文件外，我们一般先会看下总体线程，通过pstreee -p pid |wc -l。
#### youngGC过频繁
 youngGC频繁一般是短周期小对象较多，先考虑是不是Eden区/新生代设置的太小了，看能否通过调整-Xmn、-XX:SurvivorRatio等参数设置来解决问题。如果参数正常，但是young gc频率还是太高，就需要使用Jmap和MAT对dump文件进行进一步排查了。
#### youngGC耗时过长
耗时过长问题就要看GC日志里耗时耗在哪一块了。以G1日志为例，可以关注Root Scanning、Object Copy、Ref Proc等阶段。Ref Proc耗时长，就要注意引用相关的对象。Root Scanning耗时长，就要注意线程数、跨代引用。Object Copy则需要关注对象生存周期。而且耗时分析它需要横向比较，就是和其他项目或者正常时间段的耗时比较。比如说图中的Root Scanning和正常时间段比增长较多，那就是起的线程太多了。
#### 触发fullGC
+ 并发阶段失败：在并发标记阶段，MixGC之前老年代就被填满了，那么这时候G1就会放弃标记周期。这种情况，可能就需要增加堆大小，或者调整并发标记线程数-XX:ConcGCThreads。
+ 晋升失败：在GC的时候没有足够的内存供存活/晋升对象使用，所以触发了Full GC。这时候可以通过-XX:G1ReservePercent来增加预留内存百分比，减少-XX:InitiatingHeapOccupancyPercent来提前启动标记，-XX:ConcGCThreads来增加标记线程数也是可以的。
+ 大对象分配失败：大对象找不到合适的region空间进行分配，就会进行fullGC，这种情况下可以增大内存或者增大-XX:G1HeapRegionSize。
+ 程序主动执行System.gc()：不要随便写就对了。

### 推外内存
首先堆外内存溢出表现就是物理常驻内存增长快，报错的话视使用方式都不确定，如果由于使用Netty导致的，那错误日志里可能会出现OutOfDirectMemoryError错误，如果直接是DirectByteBuffer，那会报OutOfMemoryError: Direct buffer memory。

堆外内存溢出往往是和NIO的使用相关，一般我们先通过pmap来查看下进程占用的内存情况pmap -x pid | sort -rn -k3 | head -30，这段意思是查看对应pid倒序前30大的内存段。这边可以再一段时间后再跑一次命令看看内存增长情况，或者和正常机器比较可疑的内存段在哪里。

### TCP/IP
那么在实际开发中，我们怎么能快速定位到tcp队列溢出呢？
netstat命令，执行netstat -s | egrep "listen|LISTEN"

ss命令，执行ss -lnt   .Send-Q 表示第三列的listen端口上的全连接队列最大为5，第一列Recv-Q为全连接队列当前使用了多少。

### RST异常
RST包表示连接重置，用于关闭一些无用的连接，通常表示异常关闭，区别于四次挥手。在实际开发中，我们往往会看到connection reset / connection reset by peer错误，这种情况就是RST包导致的。
我们在排查故障时候怎么确定有RST包的存在呢？当然是使用tcpdump命令进行抓包，并使用wireshark进行简单分析了。tcpdump -i en0 tcp -w xxx.cap，en0表示监听的网卡。
接下来我们通过wireshark打开抓到的包，可能就能看到如下图所示，红色的就表示RST包了。

### TIME_WAIT和CLOSE_WAIT
用ss命令会更快ss -ant | awk '{++S[$1]} END {for(a in S) print a, S[a]}'

#### TIME_WAIT
time_wait的存在一是为了丢失的数据包被后面连接复用，二是为了在2MSL的时间范围内正常关闭连接。它的存在其实会大大减少RST包的出现。过多的time_wait在短连接频繁的场景比较容易出现。这种情况可以在服务端做一些内核参数调优:

(''')
#表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭
net.ipv4.tcp_tw_reuse = 1
#表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭
net.ipv4.tcp_tw_recycle = 1
(''')

#### CLOSE_WAIT
close_wait往往都是因为应用程序写的有问题，没有在ACK后再次发起FIN报文。close_wait出现的概率甚至比time_wait要更高，后果也更严重。往往是由于某个地方阻塞住了，没有正常关闭连接，从而渐渐地消耗完所有的线程。
想要定位这类问题，最好是通过jstack来分析线程堆栈来排查问题。这里仅举一个例子。