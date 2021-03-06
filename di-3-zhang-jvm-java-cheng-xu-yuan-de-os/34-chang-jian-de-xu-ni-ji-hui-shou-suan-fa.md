![][1]  
上半部分为 `Young` 空间可以使用的 `GC` 方式，下半部分为 `Old` 空间可以使用的 `GC` 方式，相互之间的连线代表它们的兼容性，如果能通过一条线直接打到，则说明它们在配置过程中是兼容的。  
  
例如：设置参数 `-XX:UseParallelGC` ，此时 `Young` 空间启用 `Parallel Scavenge` ， `Old` 空间启用 `MSC` 方式；如果设置参数 `-XX:UseParallelOldGC` ，此时 `Young` 空间保持不变， `Old` 空间采用 `Parallel Old` 方式。  
  
要介绍这些 `GC` 方式，就从最原始的 `GC` 开始，串行 `GC` 就是最原始的，并行、并发都是在实际场景中由驱动衍生出来的。  
  
## 4.1 串行 GC
串行 `GC` 就是在 `GC` 时由单个线程来完成，一般在“ `Client` 模式” 下这是默认的，或者增加： `-XX:UseSerialGC` ，也就会采用串行 `GC` 了。  
  
建议小内存程序，且 `CPU` 有限，用串行 `GC` 。它就是一个线程做 `GC` ，没有锁本身带来的逻辑结构。当然面对较大的内存，如果仅仅针对 `Young` 区的回收用串行 `GC` 就可以满足需求（因为此空间活着都是少数），但如果发生 `Full GC` ，那么串行回收的效率远远不够，而且 `CPU` 也没有好好的利用。这时候可以采用并行 `GC` 。  
  
## 4.2 `ParallelGC` 与 `ParallelOldGC`
如果系统启动默认设置了 `-server` 参数（读者发现 `JDK1.8 jre/bin` 下并没有 `client` 文件夹，即只支持 `server` ，而 `JDK1.7` 有次文件夹），那么默认使用 `-XX:UseParallelGC` 参数，还有一个 `-XX:UseParallelOldGC` 两者有什么区别呢？  
  
它与串行收集器其实是类似的，最重要的区别在于它是用多线程来处理的。通过增加参数 `-XX:+PrintGCDetails` 来看看它们对 `Young` 、 `Old` 、 `Perm` 空间分别采用了方法做 `GC` 。  
  
使用 `-XX:+PrintGCDetails -XX:+UseParallelGC` 后  
![][2]  
使用 `-XX:+PrintGCDetails -XX:+UseParallelOldGC` 后  
![][3]
  
读者使用 `JDK 1.8` 发现 `Old` 区都是使用的 `ParOldGen` 算法。  
`PSYoungGen` 是针对 `Young` 空间来讲，是“并行清除的意思”（ `Parallel Scavenge` ），它讲究“吞吐量”，尽量增加平均每段时间内对外提供服务的时间。但是并不会像并发 `GC`（ `CMS` ）那样考虑用户访问的尽量不暂停，也许 `PSYoungGen` 会暂停一会儿，但是它对于整体时间来讲，服务时间更多。当然可以设置 `-XX:MaxGCPauseMillis` 参数设置最多的暂停时间，也可以设置 `-XX:GCTimeRatio` 参数来设置 `GC` 时间的比例。  
  
### 4.2.1 
不过，启用 `-XX:+UseParallelGC` 仅仅针对 `Minor GC` ， `Major GC` 依然使用一个线程去做（虽然它显示为 `PSOldGen` ，但这可能是程序的 `Bug` ），它对 `Old` 区的回收采用了 `MSC` （ `Mark Sweep Compact` ）方式，它会对活着的对象标记完成后，对没有或者的对象做清除操作（包括 `Old` 区域、 `Perm` 区域、 `C Heap` 区中的内容），然后进行压缩。
  
### 4.2.2 
当启用 `-XX:+UseParallelOldGC` 后，对 `Old` 区采用 `ParOldGen` 算法，即 `Parallel Compacting` （并行压缩）（过程开销是巨大的），而且是部分压缩。什么意思呢？  
`JVM` 发现一种规律，按照前面介绍的年轻代的概念，生存越久的对象才会进入 `Old` 区，但是 `Old` 区的对象也未必代表“真的够长命”。 `Old` 区每次压缩时通常是将相邻的对象合并在一起，在经过多次压缩后，真的长命的对象通常会被排布到 `Old` 区的“底部”，而其余部分在 `Full GC` 通常后是比较稀疏的，比较稀疏的地方不太影响空间的分配，因此就有了这样一个部分压缩的概念。  
  
作者认为， `JVM` 想要解决的问题是：某些对象是否真正的够长命？  
  
进入 `Old` 区有可能是 S0 装不下而已。此时，我们认为往 `Old` 区放对象就像是往一个筐子里放物品一样，较早进入 `Old` 区的对象，简单地认为在筐子的下面，而近期晋升进入的对象，在筐子的上面。部分压缩，就可以认为对筐子“底部”的空间很少去关心，因为在通常情况下，认为进入 `Old` 区后，“最近”放进去的对象也是最容易 `GC` 的。  
>`JVM` 并行 `GC` 的“悲观策略”：  
就拿我们提到的“筐子”来讲吧，每次放进筐子里面的东西“有大也有小”，但在并行 `GC` 中，如果有对象晋升到 `Old` 空间，则会记录每次晋升对象的大小，也会在晋升后计算 `Old` 区剩余的空间大小，如果这次 `Minor GC` 顺利完成后，即使 `Old` 区还可能会剩余空间，但是如果 `JVM` 发现 `Old` 区剩余的空间已经不能承载“每次晋升对象的平均大小”时，它会再做一次 `Full GC` 尝试释放内存，换句话说，就是这个时候根本还没有满，但是做了 `Full GC` 。  
把这个概念放大，如果每次晋升的内容都是几百 `MB` 的大小，也许 `Old` 区还剩几百 `MB` 就开始 `Full GC` 。

### 4.2.3 小总结
1. 在并行 `GC` 最少有两种情况会导致 `Full GC` ： `Old` 区满的时候（确切的说，是要晋升的对象大于 `Old` 区剩余的空间）、 `Old` 区剩余空间已经小于平均晋升空间的大小的时候。  
2. 还有可能 `Perm` 区满的时候也会导致 `Full GC` 。
3. 系统使用 `System.gc()` 的时候也会默认导致 `Full GC` ，可以设置 `-XX:+DisableExplicitGC` 使代码中的 `System.gc()` 不生效（包括第三方包，这可能引发问题）。

## 4.3 `CMS GC` 与未来的 `G1` 
`CMS GC` 又称为并发 `GC` ， `CMS` 即 `Concurrent Mark Sweep` ，是并发标记清除的意思。它希望在用户使用过程中暂停尽量较小，但是绝非没有暂停。  
  
对于 `10GB` 内的 `JVM` 内存，如果代码没有问题，那么 `GC` 的开销时间并不多；如果采用了并行搜集，每次 `GC` 都是在毫秒级，除非遇到存在大量活着的而不改活着的对象情况，这种情况不论用哪种 `GC` 都很慢。  
  
当 `JVM` 内存达到 `100GB` 时， `GC` 的暂停时间将会达到 `10s` 以上，这个时候你可以将一个大内存的机器划分为多个虚拟机活着启动多个 `JVM` 实例来实现大内存的使用。
>多实例方法的效率通常高于虚拟机，但是多实例部署存在大量端口的修改，比较麻烦，而资源隔离性，虚拟机会更好，所以在应用部署上通常选择虚拟机划分方式。  
  
但是有一些单个实例就需要大内存计算的应用背景不适合，而 `CMS` 的存在就是为大内存做了一个伟大的铺垫， `CMS` 是一种并发 `GC` ，适合管理更大的内存。它是如何做到的呢？
1. `CMS-initail-mark` ，单线程处理环节，并且会完全暂停 `JVM` ，找到所有的 `root` 根，将 `root` 根所引用的对象标记在一个 `BitMap` 中。此时业务恢复正常运行， `GC` 进入下一个步骤。  
2. `CMS-concurrent-mark` ，根据第一个步骤对记录在 `BitMap` 中的信息进行并发标记对象。此时 `JVM` 内锁运行的程序不会暂停。
3. `CMS-concurrent-preclean` ，处理 `Young` 与 `Old` 之间的引用，尽量将可能的引用梳理出来，节约 `remark` 时间。
4. `CMS-concurrent-abortable-preclean` 。
5. `CMS-remark` ，`JVM` 全暂停的第二个阶段，由于前面几个步骤中存在业务的并发，所以导致一些“脏数据”。 `remark` 是来解决这个问题的，因此这个阶段也会全部暂停，但是它不是再用 `root` 开始遍历，而是根据一个“卡表”，换句话说，卡表中有我们想要的“脏”块信息。
6. `CMS-concurrent-sweep` ，不停机，单线程操作，按照内存的地址顺序遍历 `Old` 、 `Perm` 区，根据 `BitMap` 中的记录，对垃圾内存进行清除操作。  
7. `CMS-concureent-reset` ，既然中间是并发的，那么在标记过程中用户也是可以操作，因此内存结构很可能会发生改变，如果这些 `GC Root` 引用断开了怎么办？有了新的引用怎么办？因为在程序运行过程中很多局部变量脱离了作用域或被重新赋值，此时 `CMS-remark` 的作用就是进行这部分数据的回补，将中间产生变化的内容重新进行一次标记，然后开始进一步做并发的清除和 `reset` 工作。  
  
使用 `CMS GC` 一般是使用 `-XX:+UseConcMarkSweepGC` 来启用 `CMS` 。启用该参数后，它将默认对 `Young` 空间启用算法： `ParNew` ，相当于设置了 `-XX:+UseParNewGC` 参数，它们两个是一套组合拳。  
  
关于更多 `CMS GC` 的参数设置，原书讲解更加透彻，这里不做深入研究了。

### 4.3.1 G1
`G1（Garbage-first）` 可能是目前统治大内存的神。  
  
`Java` 堆将被划分为多个相对较小且大小相同的内存板块（有点类似于我们手工划分为多个虚拟机或多个进程，但是在同一个进程中，板块之间是存在许多关系的，所以要复杂许多），这些板块我们叫作 `region` 。它也有分代，在 `region` 里面分代，在运行过程中会维护一个优先级列表，这个优先级列表其实是按照垃圾多少来排序的，这样垃圾最多的 `region` 就是最前面的一个板块，每次就回收这个垃圾最多的板块。有了这个基本思想，下面从理论上来分析几个优势（不一定是 `G1` 要实现的目标哦）。  
  
1. 内存被划分为多个 `region` 后，每个 `region` 的管理就变成了一个小内存，至少是相对较小的内存，不论是查找还是遍历，小内存都会更快一些。  
2. 垃圾多的始终会被优先处理，如果垃圾多，那么活着的就少，由于活着的少，所以找活着的就快，这个暂停时间也将是最短的。
3. 理论上还可以做到：只有访问这个板块的线程将被暂停。（读者注：难道这就是表锁变行锁？）
4. 有些初始化加载的额对象，可能在某个或某些几种的 `region` 中，这些对象几乎不会成为垃圾多的 `region` ，那么也几乎不会排序在前， `GC` 也永远不会区扫描这些可能不需要回收的“老不死”的 `region` 。
5. 理论上，如果这个 `region` 一直不死，那么有些其他区域中很难死掉的对象也可以放进来，就像有许多代 `GC` 一样，这样很多“老不死”的就在一起了。  
  
另外，再强的 `GC` 算法，也抵不住大量内存的服务器。

## 4.4 简单总结
1. 串行 `GC` 用在 `CPU` 小，内存小的引用。例如本地开发、手机开发。
2. 使用 1~2GB 的 `JVM` 推荐启用 `-XX:+UseParallelOldGC` 。  
3. 业务代码尽量多使用小而美的内存，小在对象的带下，美在对象的生命周期短暂。

## 4.5 补充
`GC` 如何暂停所有的线程的执行？  
业务代码可以使用 `sleep` 、 `wait` ，而底层则使用“安全点”的概念，即指令运行到某个特定的引用变化位置，就可以被暂停。以及“安全区域”的概念，即 `GC` 暂停时让线程到某个区域，等 `GC` 完后，再回去执行业务。






[1]: /assets/3-1.png
[2]: /assets/3-2.png
[3]: /assets/3-3.png


