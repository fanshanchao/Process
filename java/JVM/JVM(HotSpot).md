**本文内容主要都是基于HotSpot虚拟机。**

![image-20201212232121155](E:\拎着自己飞呀\笔记本\images\Java体系图)

**注**：上图来自于《深入理解Java虚拟机》

# 自动内存管理

## Java内存区域与内存溢出异常

![image-20201213164347666](E:\拎着自己飞呀\笔记本\images\image-20201213164347666.png)

* 程序计数器：线程字节码指示器，也就是每个线程运行到的位置。
* Java虚拟机栈：每个方法被执行的时候都会创建一个栈帧用户存储局部变量表，操作数栈，动态连接，方法出口等信息。**局部变量表的内存在编译时就会分配好。**
* 本地方法栈：也就是调用本地方法的用到的栈。
* Java堆：**几乎**所有的对象实例和数组都是在这里分配内存。
* 方法区：用于存储已被虚拟机加载的类型信息，常量，静态变量等数据。HotSpot选择把垃圾收集器的分代设计扩展至方法区，这不是一个很好的选择，因为这样虽然方法区可以像堆一样被回收，但同时也有了和堆一样的限制，例如内存的上限，会引起内存溢出。到了JDK8以后才完全移除了永久代这个概念。转而使用本地内存中实现方法区。

上面这个图是Java虚拟机规范所规定的运行时数据区，但实际上每个**虚拟机/垃圾收集器**的具体实现可能是不同的，例如方法区的实现，在HotSpot/CMS收集器的实现中，JDK7之前方法区是用**永久代**实现的，所以方法区会有和堆一样的限制。

### HotSpot对象

对象的创建过程：

1. 首先去方法区中的常量池看是否能定位到一个类的符合引用
2. 如果能够定位到，检查这个类是否已经被加载，解析，初始化，如果没有则进行类加载
3. 给对象分配内存空间。分配内存空间这里其实会有一个并发问题（多个线程对内存的操作），一般来说有下面两种方案：
   * 使用CAS操作实现，失败重试
   * 给每个线程划分一个内存空间，可通过虚拟机参数开启
4. 对对象进行必要的设置，如对象的哈希码，GC分代年龄等信息。这些信息都是存在对象头里面。

![image-20201213164403933](E:\拎着自己飞呀\笔记本\images\image-20201213164403933.png)

上图是一个对象的结构，Mark Word其实就是存了一些对象运行时数据，哈希码，锁状态之类的，根据虚拟机的配置，一般容量为32位/64位。

#### 对象的访问定位

一般来说对象的访问有两种方式：通过句柄访问和直接方法。



![image-20201213164423588](E:\拎着自己飞呀\笔记本\images\image-20201213164423588.png)

![image-20201213164432981](E:\拎着自己飞呀\笔记本\images\image-20201213164432981.png)

上图来自《深入理解Java虚拟机》。两种方式各有好坏，直接访问可以减少一次访问，但是如果对象地址发生变更，reference本身也需要发生改变。HotSpot虚拟机主要使用直接访问的方式定位对象。

### OutOfMemoryError的解决方案

#### Java堆内存OutOfMemoryError解决方案

1. 首先通过内存映像分析工具对Dump出来的堆转储快照进行分析。

   * 可在JVM启动时配置如下参数即可内存溢出时生成Dump文件：

     ```
     -XX:+HeapDumpOnOutOfMemoryError
     -XX:HeapDumpPath=E:\idea-workspace\JVMTest(HotSpot)\MemoryFiles
     ```

   * 在发生程序异常时还可以通过执行指令，直接生成当前JVM的dump文件，6214是JVM进程号

     ```
     jmap -dump:format=b,file=/home/admin/logs/heap.hprof 6214
     ```

     

2. 确认是内存溢出还是内存泄漏导致的。

   * **注**：内存泄漏就是代码不合理导致的，持有了无用对象却没有释放，这个时候积少成多会导致OutOfMemoryError。而内存溢出就是我们设置的堆太小了，这个时候应该考虑增加堆的内存。

3. 如果是内存泄漏，可进一步通过工具查看泄漏对象到GC Roots的引用链，具体分析垃圾回收器无法回收它们的原因。

   * 工具可以选用jvisualvm，在jdk的bin目录下，具体使用教程：[https://blog.csdn.net/lkforce/article/details/60878295](https://blog.csdn.net/lkforce/article/details/60878295)

#### 虚拟机栈和本地方法栈溢出

关于虚拟机栈和本地方法栈溢出的两种异常：

1. 栈深度大于虚拟机抛出StackOverflowError异常
2. 栈内存允许扩展时，如果无法申请到足够的内存时，则抛出OutOfMemoryError异常

**注1：**在HotSpot虚拟机中，栈的动态扩展是不支持的，所以只有StackOverflowError异常

**注2：**栈容量可通过-Xss参数来配置

#### 方法区和运行时常量池溢出

下面这段代码，在JDK6是会发生内存溢出，因为JDK6方法区的实现是通过永久代实现的，可通过JVM参数配置永久代大小。但是在JDK7以上，由于将常量池移到了堆内存中，所以得配置堆内存大小才会发现溢出情况。

```java
/**
 * @author fanshanchao
 * @date 2020/12/13 18:20
 * -Xmx6M
 */
public class PermGenOOM {
    public static void main(String[] args) {
        Set<String> set = new HashSet<String>();
        short i = 0;
        while(true){
            set.add(String.valueOf(i).intern());
            i++;
        }
    }
}
//Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
//从异常信息也可以看出，发生的是堆内存溢出
```

关于方法区溢出，在JDK8以后，由于永久代完全退出了，元空间作为其替代者登场。在默认设置下，动态创建新类型这种测试用例已经很难使虚拟机发生方法区异常溢出异常了。但是为了防止使用者出现一些破坏性操作，HotSpot还是提供了一些参数作为元空间的防御措施。具体参数可看文档。

#### 本机直接内存溢出

直接内存大小可通过-XX:MaxDirectMemorySize参数配置，默认与Java堆最大值一样。

一般来说直接内存的内存溢出有一个明显的特征，就是Heap Dump文件中不会看见有什么明显的异常情况，如果发现内存溢出后的dump文件很小，而程序中又直接或间接使用了直接内存，那么就可以考虑检查下这方面的原因了。

## 垃圾收集与内存分配策略

### 垃圾收集

垃圾收集器要解决的三个问题：

1. 哪些内存需要需要收集的？
   * Java堆和方法区，因为这两个区域的内存存在不确认性
2. 什么时候进行回收？
3. 如何回收？

#### 如何判断对象已死

##### 引用计数算法

这种算法简单，但是无法解决循环引用的问题。

##### 可达性分析算法

这个算法的关键在于找到一些Gc Root根对象作为器实节点集，然后沿着这些Gc Root的引用关系向下搜索，没有在Gc Root引用链上的对象就认为是垃圾对象，可被回收。

![image-20201213190343773](E:\拎着自己飞呀\笔记本\images\image-20201213190343773.png)

如何找到Gc Root对象？固定可作为GC Roots的对象包括以下几种：

1. **在虚拟机栈（栈帧中的本地变量表）中引用的对象，譬如各个线程被调用的方法堆栈中使用到的参数，局部变量，临时变量等。**
2. **在方法区中类静态变量属性引用的对象，例如Java类的引用类型静态变量**
3. **在方法区中常量引用的对象，例如字符串常量池**
4. 本地方法引用的对象
5. JVM内部的引用，如基本数据类型对于的Class对象，异常对象，类加载器
6. 所有被同步器锁持有的对象
7. 反映Java虚拟机内部情况的JMXBean，JVMTI中注册的回调、本地代码缓存等。

##### 谈谈四种引用

1. 强引用：强引用的对象不会被回收掉。我们常见的new出来的对象就是强引用
2. 软引用：软引用的对象在内存不足时会被垃圾回收掉
3. 弱引用：弱引用的对象只能活到下一次垃圾回收之前
4. 虚引用：这种引用的目的只是为了让垃圾回收这个对象时能有一个通知。

##### 死亡对象

即使可达性标记算法将对象标记为不可达，也不是说对象就一定非死不可。要经历两个标记过程才会将真正宣告一个对象死亡。

1. 标记出GC Roots链中不可达对象
2. 筛选出没有必要执行finalize方法的对象。正是有这个方法我们可以在finalize方法中对会垃圾回收的对象进行自救。但是不保证完全能自救成功，因为执行finalize方法的线程优先级很低。下面代码可以对一个对象执行自救。

```java
public class SaveObject {
    public static SaveObject saveObject = null;
	
    //注意这个方法只会执行一次，所以只能自救一次
    //我们开发过程中应该忘记这个方法的存在，这个方法并不是一个好的方法
    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        if(saveObject == null){
            saveObject = this;

        }
    }

    public static void main(String[] args) throws InterruptedException {
        saveObject = new SaveObject();
        //自救
        saveObject = null;
        System.gc();
        //执行finalize方法的线程优先级低，所以我们的主线程Sleep500ms
        Thread.sleep(500);
        if(saveObject != null){
            System.out.println("自救成功");
        }
    }
}
```

##### 方法区的回收

《Java虚拟机规范》不要求虚拟机在方法区中实现垃圾回收。是否会进行回收得看垃圾收集器的实现。

方法区主要回收两部分内存：废弃的常量和不再使用的类型。

废弃的常量很容易找出来，但不再使用的类型就必须同时满足下面三个条件：

1. 该类所有实例都已被回收
2. 加载该类的类加载器已经被回收，我理解这个条件是很难达成的
3. 该类对于的Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

### 垃圾收集算法

#### 分代收集

当前商业虚拟机大多数都遵循**分代收集**的理论。也就是下面这张图。

![image-20201213203757771](E:\拎着自己飞呀\笔记本\images\image-20201213203757771.png)



现在很多垃圾收集器都是遵循上面的分代理论对对象实际回收，但是分代存在一个问题，那就是**跨代引用**。目前HotSpot通过在某一分代建立一个全局的数据结构来保存哪些对象存在跨代引用解决这个问题。

针对上面这个分代理论，把垃圾收集行为分为以下几种：

1. Young GC：针对新生代的收集
2. Major GC：针对老年代的收集
3. Mixed GC：收集整个新生代以及部分老年代。目前只有G1收集器才有这种行为。
4. Full GC：收集整个Java堆和方法区的垃圾收集

#### 标记清除和标记复制以及标记整理算法

针对不同的分代，使用不同的垃圾收集算法能提高不同的效率。

* **标记清除算法：**这种算法优点是比较简单，缺点是效率不稳定（回收对象多的时候）且容易产生内存碎片。

![image-20201213205325384](E:\拎着自己飞呀\笔记本\images\image-20201213205325384.png)

* **标记复制算法**：这种算法将内存区域分为几个区域，将需要保留的对象从一个区域移动到另一个区域，然后直接回收原有区域就行。由于新生代的对象基本上都是朝生夕灭，所以这种算法很适合用于新生代。目前HotSpot的Serial，ParNew等新生代收集器都对这个算法进行了优化，将区域分成8:1:1，也就是上面的新生代图那样进行划分。从Eden加一个Survivor中找到存活的对象到另一个Survivor中，这样就只会浪费10%的内存。对于一次垃圾回收存活的对象超过10%的情况，会使用担保策略来解决这个问题，也就是从老年代中去分配担保。

![image-20201213205657564](E:\拎着自己飞呀\笔记本\images\image-20201213205657564.png)

* **标记整理算法：**对标记清除算法的优化。标记过程和标记清除算法是一样的，但是后续步骤不是对可回收对象进行清理，而是让所有存活的对象都向内存空间一端移动，然后直接清理掉边界以外的内存。对于垃圾清理频率比较低的分代较为适合，很明显这就是适合**老年代**的算法。但是这个移动对象这个步骤有一个问题，**移动存活对象并更新所有引用这些对象的地方将会是一种极为负重的操作，而且这种对象移动操作必须全程暂停用户应用程序才能进行。**这就是很多资料中出现的**Stop The World**概念。还有一种中和的方式，先采用标记-清除算法，当内存碎片影响到内存的分配时，再使用标记-整理方法，**CMS收集器**采用的就是这种方案。

![image-20201214222822828](E:\拎着自己飞呀\笔记本\images\image-20201214222822828.png)

### HotSpot算法细节实现

迄今为止，所有收集器在根节点枚举这一步骤时都是必须暂停用户线程的，因此毫无疑问根节点枚举与之前提及的整理内存碎片一样会面临相似的“Stop TheWorld”的困扰。由于方法区很大，查找GC Root肯定很耗时，为此Hot Spot使用了一种**OopMap**的数据结构来加快GC Root的枚举。

#### 安全点

HotSpot不可能为每条指令都生成OopMap，只有在特定的位置记录这些信息，这些位置就是安全点。安全点的设定，也就决定了用户程序执行时并非在代码指令流的任意位置都能够停顿下来开始垃圾收集，而是强制要求必须执行到达安全点后才能够暂停。

**如何让停顿用户线程？**

在垃圾收集发生时，所有的线程都跑到最近的安全点，然后停顿下来。HotSpot采用线程轮询的方式来让用户到线程判断是否到达安全点。

#### 记忆集与卡表

前面在分代理论的时候说过一个保存跨代引用的数据结构，这个数据结构就是记忆集。记忆集是一种用于记录从非收集区域指向收集区域的指针集合的抽象数据结构。而卡表就是记忆集的具体实现。详情可看书。

#### 总结

通过**OopMap**加速GC Root的枚举 ------>使用**安全点**优化OopMap的生成  ------> 使用**线程轮询**方式让用户到达安全点停顿（优化到汇编层级）------> 使用**记忆集**解决跨代引用问题，缩小GC Root扫描范围 ------> 通过**增加更新和原始快照**来保证回收正确。

### 垃圾收集器

下图是目前HotSpot可以使用到的垃圾收集器，用线连起来的收集器表示可以一起使用。

![image-20201214230909439](E:\拎着自己飞呀\笔记本\images\image-20201214230909439.png)

#### Serial收集器

Serial收集器一般是单线程收集器，这是一个新生代收集器。对于资源受限的环境/单核处理器/处理器核心数较少的环境，比较适用于这种收集器。例如桌面应用，部分比较小的微服务。

![image-20201214233351356](E:\拎着自己飞呀\笔记本\images\image-20201214233351356.png)

#### ParNew收集器

本质上是Serial收集器的多线程并行版本。涉及到的可用控制JVM参数如下：

-XX：SurvivorRatio、-XX：PretenureSizeThreshold、-XX：HandlePromotionFailure

![image-20201214233941671](E:\拎着自己飞呀\笔记本\images\image-20201214233941671.png)

ParNew收集器还有一个非常重要的点，只有它能与**CMS收集器**一起工作。使用**-XX：+UseConcMarkSweepGC**参数激活CMS后默认会激活ParNew收集器。默认情况下ParNew收集器的线程数和处理器线程数量相同，可以用**-XX：ParallelGCThreads**参数设置线程数。

#### Parallel Scavenge收集器

Parallel Scavenge也是一款新生代收集器，同样也是基于标记-复制算法实现的。从名字也能看出来还是一款能够并行收集的多线程收集器。Parallel Scavenge收集器的特点是它的关注点与其他收集器不同，CMS等收集器的关注点是尽可能地缩短垃圾收集时用户线程的停顿时间，而**Parallel Scavenge收集器的目标则是达到一个可控制的吞吐量**（Throughput）。所谓吞吐量就是处理器用于运行用户代码的时间与处理器总消耗时间的比值。

自适应调节策略是Parallel Scavenge收集器区别于ParNew收集器的一个重要特性。

##### Parallel Scavenge收集器JVM参数

**-XX：MaxGCPauseMillis** 参数允许的值是一个大于0的毫秒数，收集器将尽力保证内存回收花费的时间不超过用户设定值。

**-XX：GCTimeRatio** 参数的值则应当是一个大于0小于100的整数，也就是垃圾收集时间占总时间的比率，相当于吞吐量的倒数。

**-XX：+UseAdaptiveSizePolicy**  这是一个开关参数，当这个参数被激活之后，就不需要人工指定新生代的大小（-Xmn）、Eden与Survivor区的比例（-XX：SurvivorRatio）、晋升老年代对象大小（-XX：PretenureSizeThreshold）等细节参数了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量。

#### Serial Old收集器

Serial Old是Serial收集器的老年代版本，它同样是一个单线程收集器，使用标记-整理算法。这个收集器主要也是也是供客户端模式下的HotSpot虚拟机使用。

![image-20201216203819190](E:\拎着自己飞呀\笔记本\images\image-20201216203819190.png)

 #### Parallel Old收集器

Paralle Old收集器是Parallel Scavenge收集器的老年代版本，支持多线程并发收集，基于标记-整理算法实现。在注重吞吐量或者处理器资源较为稀缺的场合，都可以优先考虑Parallel Scavenge加Parallel Old收集器这个组合。

![image-20201216205839669](E:\拎着自己飞呀\笔记本\images\image-20201216205839669.png)

#### CMS收集器

CMS收集器是一种以获取最短回收停顿时间为目标的收集器。CMS收集器是基于**标记-清除**算法实现的，它的运作过程分为以下四步：

1. 初始标记：这个阶段仅仅只是标记一下GC Roots能直接关联到的对象，速度很快。需要**Stop The World**。
2. 并发标记：从GC Roots的直接关联对象开始遍历整个对象图的过程。这个过程耗时较长，但是可以和用户线程一起。
3. 重新标记：这一步修正上一步产生的标记发生变动的对象。这个阶段需要**Stop The World**
4. 并发清除：清理删除掉标记阶段判断死亡的对象，这一步可以和用户线程同时并发。

![image-20201216211635598](E:\拎着自己飞呀\笔记本\images\image-20201216211635598.png)

CMS收集器有几个缺点：

1. CMS收集器对处理器资源敏感。因为这是并发的收集器。
2. CMS无法处理浮动垃圾。这里的浮动垃圾是指并发标记和并发清理时产生的垃圾。
3. 标记-清除算法的缺点，这个不用多说了。有可能会引发Full GC

##### 相关JVM参数

-XX：CMSInitiatingOccu-pancyFraction：设置CMS触发回收时的百分比。

#### G1收集器

G1收集器开创了收集器面向局部收集的设计思路和基于Region的内存布局形式。G1收集器是一个面向全堆的垃圾收集器，G1的思路是哪块内存中的垃圾多，回收收益大，那就收集哪块内存。下图是Region示意图：

![image-20201216215128230](E:\拎着自己飞呀\笔记本\images\image-20201216215128230.png)

从上图可以看出，G1收集器还是保留了新生代和老年代的概念，但是新生代和老年代不再是固定的了，它们是一系列区域的动态集合。注意上图中还有标记为H的区域，这个区域叫Humongous区域，专门用来存储大对象，G1垃圾器认为只要大小超过了Region容易一半的对象就可判定为大对象。G1根据用户设置的停顿时间找出收益最大的Region进行收集，收集单位是Region。

G1的运作流程：

1. **初始标记**：和CMS差不多，但是可以和Minor GC的时候同步完成，所以这个阶段实际停顿很少
2. **并发标记：**和CMS差不多。
3. **最终标记：**和CMS差不多。
4. **筛选回收：**这个阶段根据各个Region区域的回收价值和成本进行排序，根据用户期望的停顿时间来执行回收计划。把决定回收的Region复制到空的Region中，再清理掉旧的全部空间。这个阶段也存在**Stop The World**

![image-20201216221650509](E:\拎着自己飞呀\笔记本\images\image-20201216221650509.png)

##### 相关JVM参数

-XX：G1HeapRegionSize 设置Region内存大小。大小为1-32MB

-XX：G1HeapRegionSize 允许的收集停顿时间

#### Shenandoah收集器和ZGC收集器

这个需要时再看吧，目前应用的范围小

#### 如何选择最合适的垃圾收集器

衡量垃圾收集器的三项最重要的指标是：内存占用（Footprint）、吞吐量（Throughput）和延迟（Latency），三者共同构成了一个“不可能三角”。下图是各款收集器的并发情况，深色部分为并发部分。

![image-20201216222233265](E:\拎着自己飞呀\笔记本\images\image-20201216222233265.png)

三个方面去选择垃圾收集器：

1. **应用的关注点是什么？**

如果时数据分析，科学计算类的任务，目标是为了尽快算出结果，那么**吞吐量**就是主要关注点。如果是SLA应用（企业级应用），停顿时间会影响服务质量，甚至事务超时等问题，那么**延迟**就是需要关注的点。而嵌入式应用/客户端应用这种就必须考虑内存占用。

2. **运行应用的基础设施如何？**
3. **使用JDK的发行商是什么？**

遗留系统，可以根据内存大小衡量下，4GB到6GB内存用CMS，更大内存考虑G1。

#### 虚拟机及垃圾收集器日志

> :sob:终于到了心心念的地方了。好好实验总结下。

JDK9以前，HotSpot没有统一的日志处理框架，虚拟机各个功能模块的日志开关分布在不同的参数上，日志级别、循环日志大小、输出格式、重定向等设置在不同功能上都要单独解决。直到**JDK9**才有了统一的**-Xlog**参数。

```
-Xlog[:[selector][:[output][:[decorators][:outout-options]]]]
```

这个参数由Select，Tag，Level组成。Tag可以理解为虚拟机中某个功能模快的名字，告诉日志框架希望得到哪些功能的日志输出。Level有Trace,Debug,Info,Warning,Error,Off六种级别，默认是Info。

##### 如何看日志？

![image-20201218225850664](E:\拎着自己飞呀\笔记本\images\image-20201218225850664.png)

##### JVM参数

-XX:+PrintGC	 JDK9之前打印GC日志

-Xlog:gc	 JDK9之后打印GC日志

-XX:+PrintGCDetails	 JDK9之前打印详细GC日志

-Xlog:gc* 	 JDK9之后打印详细GC日志

-XX:+PrintHeapAtGC	JDK9之前查看GC前后的堆、方法区可用容量变化

-Xlog：gc+heap=debug 	JDK9之后查看GC前后的堆、方法区可用容量变化

-XX:+PrintGCApplicationConcurrentTime，-XX：+PrintGCApplicationStoppedTime 	JDK9之前查看GC过程中用户线程并发时间及停顿的时间

Xlog：safepoint	JDK9之后查看GC过程中用户线程并发时间及停顿的时间

-XX：+PrintAdaptive-SizePolicy	JDK9之前查看收集器Ergonomics机制

-Xlog：gc+ergo*=trace	JDK9之后查看收集器Ergonomics机制	

-XX：+PrintTenuringDistribution	JDK9之后查看熬过收集后剩余对象的年龄分布信息

![image-20201220163512253](E:\拎着自己飞呀\笔记本\images\image-20201220163512253.png)

![image-20201220163519198](E:\拎着自己飞呀\笔记本\images\image-20201220163519198.png)

#### 对象的一些分配规则

1. 大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够的空间分配对象时就会引发一次**Minor GC**。看下面的例子代码：

```java
/**
 * @author fanshanchao
 * @date 2020/12/18 23:16
 * -XX:+PrintGCDetail -Xms20M -Xmx20M -Xmn10M 堆大小为20M 新生代为10M
 * 分配3个2M的对象，再分配一个4M的对象
 */
public class TestAllocation {
    private static int SIZE_MB = 1024*1024;
    public static void main(String[] args) {
        byte[] b1 = new byte[SIZE_MB*2];
        byte[] b2 = new byte[SIZE_MB*2];
        byte[] b3 = new byte[SIZE_MB*2];
        byte[] b4 = new byte[SIZE_MB*2];
    }
}
//下面是打印的日志
//可以看出年轻代内存下降了很多，但是总内存几乎没变小
[GC (Allocation Failure) [PSYoungGen: 7989K->744K(9216K)] 7989K->6896K(19456K), 0.0042968 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Ergonomics) [PSYoungGen: 744K->0K(9216K)] [ParOldGen: 6152K->6745K(10240K)] 6896K->6745K(19456K), [Metaspace: 3275K->3275K(1056768K)], 0.0056104 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
//堆的日志情况可以看出最后b4对象分配在了新生代，而其它三个对象则进入了老年代
Heap
 PSYoungGen      total 9216K, used 2214K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 27% used [0x00000000ff600000,0x00000000ff829860,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
  to   space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
 ParOldGen       total 10240K, used 6745K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 65% used [0x00000000fec00000,0x00000000ff2964c8,0x00000000ff600000)
 Metaspace       used 3296K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 359K, capacity 388K, committed 512K, reserved 1048576K
```

2. 大对象直接进入老年代，**-XX：PretenureSizeThreshold**参数可以配置多大的对象直接在老年代分配。但是得注意这个参数只对**Serial**和**ParNew**两款新生代收集器有用。这个原则可以从原则3的代码示例中验证出。

3. 长期存活的对象将进入老年代。可以通过**-XX：MaxTenuringThreshold**参数配置进入老年代的年龄，经历了一次Minor GC就增加一岁。看下面这个例子，非常的**重要**！！！！找了好久终于找出和书上结果不一致的原因，顺带验证了下原则2。

```java
/**
 * @author fanshanchao
 * @date 2020/12/18 23:40
 *
 * -verbose:gc
 * -Xms20M
 * -Xmx20M
 * -Xmn10M
 * -XX:+PrintGCDetails
 * -XX:SurvivorRatio=8
 * -XX:MaxTenuringThreshold=1
 * -XX:+PrintTenuringDistribution
 */
public class TestAllocation2 {
    private static int SIZE_MB = 1024*1024;
    public static void main(String[] args) {
        byte[] b1 = new byte[SIZE_MB/4];
        byte[] b2 = new byte[SIZE_MB*4];
        byte[] b3 = new byte[SIZE_MB*4];
//        //为了引发垃圾回收
        b3 = null;
        b3 = new byte[SIZE_MB*4];
    }
}
Heap
 PSYoungGen      total 9216K, used 6361K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 77% used [0x00000000ff600000,0x00000000ffc366d8,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
  to   space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
 ParOldGen       total 10240K, used 8192K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 80% used [0x00000000fec00000,0x00000000ff400020,0x00000000ff600000)
 Metaspace       used 3295K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 359K, capacity 388K, committed 512K, reserved 1048576K
```

从上面的日志可以看出，上面的代码并没有发生GC，没有和书上的代码一样发生GC，这是因为HotSpot还有一个规则：**大对象直接进入老年代，这里的大小默认应该是超过Eden区域的一半**（详细大小我也不确定）。除此之外上面的日志还有一个观察点，新生代的使用空间的**竟然是6361K**，不应该是1024*4+256=4325K吗？经过测试发现，执行一个空代码，就是什么都不做的代码，Eden区域默认会有2000K左右被占用了。这里为什么被占用目前还没找到具体原因，后续有时间可以再**研究**一下这里。

我们是为了测试age=1时自动进入老年代，上面的代码不发生GC那肯定测试不了，所以得加上两个参数。

```java
/**
 * @author fanshanchao
 * @date 2020/12/18 23:40
 *
 * -verbose:gc
 * -Xms20M
 * -Xmx20M
 * -Xmn10M
 * -XX:+PrintGCDetails
 * -XX:SurvivorRatio=8
 * -XX:MaxTenuringThreshold=1
 * -XX:+PrintTenuringDistribution
 * 下面两个参数是第二次执行带的
 * -XX:PretenureSizeThreshold=5145728 设置直接进入老年代对象的大小大于5M
 * -XX:+UseParNewGC 这里得给新生代换一个垃圾收集器，因为默认是Parallel Scavenge+Serial Old组合，不适用-XX:PretenureSizeThreshold参数
 */
public class TestAllocation2 {
    private static int SIZE_MB = 1024*1024;
    public static void main(String[] args) {
        byte[] b1 = new byte[SIZE_MB/4];
        byte[] b2 = new byte[SIZE_MB*4];
        byte[] b3 = new byte[SIZE_MB*4];
//        //为了引发第二次垃圾回收
        b3 = null;
        b3 = new byte[SIZE_MB*4];
    }
}
//现在这个日志就准了
//第一次发生GC将b1对象的age置为1
[GC (Allocation Failure) [ParNew
Desired survivor size 524288 bytes, new threshold 1 (max 1)
- age   1:     873640 bytes,     873640 total
: 6197K->879K(9216K), 0.0041630 secs] 6197K->4975K(19456K), 0.0042090 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
//第二次将b1对象移动到老年代
[GC (Allocation Failure) [ParNew
Desired survivor size 524288 bytes, new threshold 1 (max 1)
- age   1:        504 bytes,        504 total
: 5060K->357K(9216K), 0.0009843 secs] 9156K->5324K(19456K), 0.0010198 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 par new generation   total 9216K, used 4681K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  52% used [0x00000000fec00000, 0x00000000ff038e88, 0x00000000ff400000)
  from space 1024K,  34% used [0x00000000ff400000, 0x00000000ff4597d8, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
                               //老年代内存为1个4M对象+256K 这里还有一些多余的内存不确定是干嘛的！！！
 tenured generation   total 10240K, used 4966K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  48% used [0x00000000ff600000, 0x00000000ffad9af0, 0x00000000ffad9c00, 0x0000000100000000)
 Metaspace       used 3211K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 353K, capacity 388K, committed 512K, reserved 1048576K
```

4. HotSpot虚拟机并不是永远要求对象的年龄必须达到-XX：MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到-XX：MaxTenuringThreshold中要求的年龄。这个从上面的那个没有进行GC的例子就能看出来，对象在分配的时候直接进入了老年代。
5. 在发生Minor GC之前，虚拟机必须检查老年代的最大可用连续空间是否大于新生代所有对象总和，如果这个条件成立，那么这一次Minor GC就可以确保是安全的。如果不成立，虚拟机会看**-XX:HandlePromotionFailure**参数的设置值是否允许担保失败。如果允许，则会进行检查老年代可用连续空间是否大于新生代历次晋升到老年代的对象大小，如果大于，则会尝试一次Minor GC，这次Minor GC是有**风险**的。如果小于或者参数设置不允许**冒险**，那么会进行Full GC。这里的冒险失败可能会导致停顿时间过长。

**注：**-XX:HandlePromotionFailure这个参数在JDK6 Update24以后已不再被使用，规则变成只要老年代可用连续空间大于新生代对象总和或大于历次晋升的平均大小，那么就进行Minor GC，否则将进行Full GC。

------



## 虚拟机性能监控，故障处理工具



> 下面的工具都来自于JDK8

### 基础故障处理工具

#### jps:虚拟机进程状况工具

功能类似linux下的ps命令。可以列出正在运行的虚拟机进程，并显示虚拟机执行主类名称以及这些进程的**本地虚拟机唯一ID（LVMID）**。这个唯一ID很重要，其它很多工具都需要这个唯一ID来定位虚拟机进程。jps命令的格式如下：

```java
jps [options] [hostid]
```

相关参数如下图：

![image-20201219150854584](E:\拎着自己飞呀\笔记本\images\image-20201219150854584.png)

本地测试一下：

![image-20201219162900072](E:\拎着自己飞呀\笔记本\images\image-20201219162900072.png)

> 这里测试的时候遇到了一个小问题，因为我电脑中安装了JDK11，如果不进入到jdk1.8/bin 目录下来执行jps命令，默认会使用jdk11的jps命令，导致**查不到相关进程**，其实这里我也比较疑惑为什么会默认使用JDK11，我JavaHome配的是1.8。

另外虚拟机会自动将进程进行生成在C:\Users\{user}\AppData\Local\Temp\hsperfdata_user目录下，我们可以看下这个目录下是有个名为7832的文件，这个文件就对应上面的这个自己测试的进程：

![image-20201219163340946](E:\拎着自己飞呀\笔记本\images\image-20201219163340946.png)

#### jstat:虚拟机统计信息监控工具

该工具用于监控虚拟机的各种运行状态信息的命令行工具。可以显示本地/远程虚拟机进程中的类加载，内存，来及收集，即时编译等运行数据。

命令格式为：

```java
jstat [option vmid [interval[s|ms][count]]]
```

参数vmid是虚拟机进程ID，interval和count代表查询间隔和次数，默认是只查询一次。下面的例子查询虚拟机进程的GC情况，配置的是1000毫米查一次，一共查10次。

![image-20201219163631088](E:\拎着自己飞呀\笔记本\images\image-20201219163631088.png)

下图是jstat的重要参数：

![image-20201219163701983](E:\拎着自己飞呀\笔记本\images\image-20201219163701983.png)

#### jinfo:Java配置信息工具

jinfo的作用是实时查看和调整虚拟机的各项参数，注意在Windows下限制比较大。jinfo命令格式如下：

```
jinfo [option] pid
```

下面的例子查看堆的最大内存

![image-20201219170736184](E:\拎着自己飞呀\笔记本\images\image-20201219170736184.png)

#### jmap:Java内存映像工具

这个工具用于生成**dump文件**，查询finalize执行队列，Java堆和方法区的详细信息，如空间使用率，当前用的是哪种收集器。在出现内存溢出的时候可以用这个命令生成dump文件，然后再去分析dump文件，从而分析出真正原因。

jmap命令的格式如下：

```
jmap [option] vmid
```

下面例子用于生成dump文件（文件名后缀可以随意填写）：

![image-20201219171741622](E:\拎着自己飞呀\笔记本\images\image-20201219171741622.png)

![image-20201219171814091](E:\拎着自己飞呀\笔记本\images\image-20201219171814091.png)

jmap主要配置如下：

![image-20201219171930512](E:\拎着自己飞呀\笔记本\images\image-20201219171930512.png)

#### jhat：虚拟机堆转储快照分析工具

和jmap搭配使用。一般使用比较少，因为分析代价较大，不如其它可视化工具。

#### jstack：Java堆栈跟踪分析工具

jstack命令用于生成虚拟机当前时刻的线程快照。命令参数如下：

```
jstack [option] vmid
```

下面例子查看线程堆栈结果：

![image-20201219172437095](E:\拎着自己飞呀\笔记本\images\image-20201219172437095.png)

主要相关参数如下：

![image-20201219172457803](E:\拎着自己飞呀\笔记本\images\image-20201219172457803.png)

#### 其它常见工具

**基础：**

![image-20201219172657636](E:\拎着自己飞呀\笔记本\images\image-20201219172657636.png)

**安全：**

![image-20201219172712120](E:\拎着自己飞呀\笔记本\images\image-20201219172712120.png)

**性能监控和故障处理：**

![image-20201219172853489](E:\拎着自己飞呀\笔记本\images\image-20201219172853489.png)

![image-20201219172901430](E:\拎着自己飞呀\笔记本\images\image-20201219172901430.png)

### 可视化故障处理工具

可视化工具主要包括**JConsole**、**JHSDB**、**VisualVM**和**JMC**四个。注意JMC在生产环境上需要付费才能使用。

#### JHSDB：基于服务性代理的调试工具

**该工具在JDK9以后才有**，JHSDB和其它基础工具的对比：

![image-20201219173312497](E:\拎着自己飞呀\笔记本\images\image-20201219173312497.png)

下面的例子用JHSDB找到下面代码中的staticObj，InstanceObj，localObj这三个变量（不是指所指向的对象）本身存放在哪里？

1. 首先运行代码：

   ```java
   /**
    * @author fanshanchao
    * @date 2020/12/19 17:38
    * 查看staticObj  instanceObj  localObj的分布情况
    * JVM参数-XX:-UseCompressedOops禁用压缩指针
    * -Xmx10M -XX:+UseSerialGC -XX:-UseCompressedOops
    */
   public class JHSDBTest {
       static class Test{
           static ObjectHolder staticObj = new ObjectHolder();
           ObjectHolder instanceObj = new ObjectHolder();
           void foo(){
               ObjectHolder localObj = new ObjectHolder();
               System.out.println("完成");//断点打在这里
           }
       }
       private static class ObjectHolder{}
   
       public static void main(String[] args) {
           Test test = new JHSDBTest.Test();
           test.foo();
       }
   }
   ```

   2. 找到进程ID 

   ![image-20201219175325951](E:\拎着自己飞呀\笔记本\images\image-20201219175325951.png)

   3. 进入JHSDB的图形化模式

   ![image-20201219175403515](E:\拎着自己飞呀\笔记本\images\image-20201219175403515.png)

   4. 找到三个对象的起始地址

   ![image-20201219175555734](E:\拎着自己飞呀\笔记本\images\image-20201219175555734.png)

   5. 使用Console窗口，用Scanoops命令在堆的新生代（从Eden->to Survivor）范围内查找ObjectHolder实例。
   6. **终止测试，找不到对象，和书上不一样，先停止。**

   ![image-20201219181650250](E:\拎着自己飞呀\笔记本\images\image-20201219181650250.png)

   #### JConsole：Java监视与管理控制台

   JDK/bin目录下，直接点开用就行了。

   ![image-20201219182144790](E:\拎着自己飞呀\笔记本\images\image-20201219182144790.png)

   #### VisualVM：多合-故障处理工具

   功能很强大，优点有以下

   1. 可以安装插件
   2. 

   

