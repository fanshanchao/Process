# JUC核心类源码分析

## AbstractQueuedSynchronizer

​		JUC包中核心类，像ReentrantLock，CountDownLatch、Semaphore、FutureTask等等都是基于AbstractQueuedSynchronizer实现的。我们如果需要定制化锁也可以在此类的基础上进行开发。AbstractQueuedSynchronizer也被翻译为同步器，多个线程竞争锁时，竞争失败的线程会阻塞在AbstractQueuedSynchronizer的阻塞队列上。后面AbstractQueuedSynchronizer简称为AQS。

> 注意一点，AbstractQueuedSynchronizer是抽象类

### 关键属性

```java
//头结点，获取到这个头结点的线程相当于获取到了锁
//特别注意，线程拥有这个主节点不代表在阻塞队列中，而是获取到了锁才会拥有这个主节点
private transient volatile Node head;
//阻塞队列的尾节点，竞争失败的线程会阻塞队列中，每次进来一个都放入阻塞队列的尾部
private transient volatile Node tail;
//代表当前锁的状态 0代表没有占用 大于0代表有线程占用
//如果需要实现定制化锁，也是通过这个变量值来控制可占用锁的线程
//注意这里也是用volatile来保证了可见性的
private volatile int state;

//父类的属性，用来标记占有锁的线程
private transient Thread exclusiveOwnerThread;
```

#### 内部类Node

​		这个类对象就是阻塞队列中的结点。看下这个内部类的关键属性：

```java
//用于标记结点在共享模式下
static final Node SHARED = new Node();
//用于标记结点正在以独占锁模式进行等待
static final Node EXCLUSIVE = null;

//结点的状态就用下面这些属性来判断的
//表示当前结点取消争抢这个锁
static final int CANCELLED =  1;
//表示当前结点的下一个结点中的线程需要被唤醒
static final int SIGNAL    = -1;

//下面这两个属性用于Condition类
static final int CONDITION = -2;
static final int PROPAGATE = -3;

//结点的状态 取值范围就是前面这几个以及默认值0 默认值0可以理解为当前结点没有后续结点需要唤醒
volatile int waitStatus;

//结点的前一个结点
volatile Node prev;
//结点的后一个结点
volatile Node next;
//结点的线程
volatile Thread thread;
```

​		在多线程情况下，AQS同步器的阻塞队列会如下图所示，head结点是占有当前锁的线程，要注意head头结点不包括在阻塞队列中

![image-20210409225522063](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210409225522063.png)

### 关键方法

#### acquire方法

​		这个方法用于获取锁，但是这个锁怎么控制进入进程数，以及如何去竞争这个锁是AQS的子类锁要关注的。AQS的acquire方法关注的是，只要竞争失败，那么就将当前线程加入到阻塞队列中。

```java
//这里参数arg的值是对state变量要操作的值 
public final void acquire(int arg) {
    //tryAcquire方法需要子类去实现，也就是在这个方法里面控制进入锁的线程数，如何去竞争这个锁
    //tryAcquire方法返回false代表线程竞争锁失败，那么通过addWaiter方法将线程包装成一个Node对象加入阻塞队列中
    //而acquireQueued方法用于竞争失败的线程加入阻塞队列后再尝试去竞争一次锁，这是因为你有可能是阻塞队列的第一个，而你的前面一个结点的就是head结点。竞争失败线程会真正的阻塞
    //正常应该返回false
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        //acquireQueued返回true，说明当前线程需要被中断
        selfInterrupt();
}
private Node addWaiter(Node mode) {
    //利用当前线程创建一个Node对象
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    //在阻塞队列不为空的情况下，加入阻塞队列的尾部
    if (pred != null) {
        node.prev = pred;
        //进行CAS操作添加至尾部，注意这里有可能CAS操作失败，因为可能有个线程在竞争，对于CAS操作失败的，会在enq方法中进行处理
        if (compareAndSetTail(pred, node)) {
            //成功就真正成为阻塞队列的tail结点了
            pred.next = node;
            return node;
        }
    }
    //进入这个方法说明 阻塞队列为空/CAS添加至阻塞队列尾部失败
    enq(node);
    return node;
}
//循环进行CAS操作 一定要把node结点加入到阻塞队列中
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        //处理阻塞队列为空的情况，初始化一个head结点
        //其实这里有一个和前面说的矛盾点，不是说拥有head的结点的是持有锁的线程吗？为什么这里面会是马上加入阻塞队列的线程来初始化一个头结点？
        //答：其实对state变量进行CAS操作成功，直接就抢锁成功了，是不会生成一个head结点的，而是第一个加入阻塞队列的结点中的线程会创建一个初始化的head结点，这个head结点的waitstatus是0，然后可以理解为这个head结点就是原来抢锁成功的那个线程的
        if (t == null) { 
            //可能有多个线程进行竞争，所以CAS操作
            if (compareAndSetHead(new Node()))
                //设置tail指针也指向这个初始化的head结点，注意这个初始化head结点的waitstatus状态是0
                //在后面的循环，这个tail指针就会指向真正的尾结点
                tail = head;
        } else {
            //阻塞队列有结点，加入阻塞队列末尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
//再回过来看acquireQueued方法，正常情况这个方法都应该返回false。加入阻塞队列的线程也是阻塞在这个方法的代码中。
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        //是否被中断标记
        boolean interrupted = false;
        //注意这里是一个死循环
        for (;;) {
            final Node p = node.predecessor();
            //这里如果发现添加进阻塞队列的前驱结点是头结点，那么会进行一次抢锁操作，首先因为你是阻塞队列的头，肯定要试一试的。其次，head结点可能是在enq方法中刚刚初始化的head结点
            //后面如果线程被唤醒了，也会进入这个分支，将自己设置为头结点
            if (p == head && tryAcquire(arg)) {
                //抢锁成功以后就将自己设置为头结点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //进入这个分支说明当前结点不是阻塞队列的第一个或者抢锁又失败了
            //shouldParkAfterFailedAcquire方法返回true代表当前结点的线程确实该阻塞
            //parkAndCheckInterrupt()方法会阻塞当前结点的线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //前面说过了waitstatus的取值作用
    int ws = pred.waitStatus;
    //进入这个分支，说明当前结点的前置结点状态正常，也就是waitstatus=-1，后面前驱结点释放锁后会唤醒当前结点
    if (ws == Node.SIGNAL)
        /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
        //返回true就是当前结点需要阻塞咯
        return true;
    //前驱结点的waitstatus>0 说明前驱结点已经取消了这个锁，那么将当前结点往前移
    if (ws > 0) {
        /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        //进入这个分支有以下情况:
        //1. 前驱结点的waitstatus == 0，也就是前驱结点是刚刚创建的，那么将前驱结点的waitstatus置为-1。如果前驱结点时刚刚初始化的head结点，也属于这种情况。
        //2. 前驱结点的waitstatus < 0 && waitstatus != -1 ，这个是Condition用到的，后面再说
        //到这里也说明了一点，阻塞队列中，一个结点的waitstatus是靠后置结点来设置的，并且设置完成后，下一次循环会进入这个方法的第一个if分支，然后return true，然后阻塞当前结点的线程。
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
//阻塞当前线程，线程会阻塞在这里
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

​		总结下acquire方法，也就是获取锁的流程：

1. 首先根据子类实现的方式去获取锁，如果获取失败了就利用当前线程构建一个Node结点，此时waitstatus为0。
2. 如果发现当前阻塞队列中没有结点，那么初始化一个waitstatus为0的head结点。否则说明当前阻塞队列中已经有结点，那么进行CAS操作，加入到阻塞队列末尾。
3. 加入阻塞队列后，如果前置结点是head结点，那么再尝试一次去抢锁，抢锁成功则将自己设置为头结点。否则说明当前结点所在的线程还是应该要阻塞的，继续下一步阻塞操作。
4. 阻塞线程之前找到一个后续可以唤醒自己的前驱结点，这里找到是指前置结点的waitstatus为-1。如果前置结点的waitstatus为0，那么会将前置结点的waitstatus置为0，如果前驱结点的waitstatus>0，那么会一直往前面找，直到找到一个不大于0的，必须保证前置结点waitstatus为-1后续才能唤醒你。
5. 线程会阻塞在acquireQueued方法中。

#### release方法

​		release用于释放锁，并且唤醒阻塞队列中的下一个线程（如果有）。

```java
public final boolean release(int arg) {
    //tryRelease需要子类实现，可以理解为释放锁的方式
    if (tryRelease(arg)) {
        //释放成功以后获取head结点
        Node h = head;
        //如果head结点不为空且waitStatus不为0 那么说明阻塞队列中有线程需要被唤醒，那么去唤醒它。
        if (h != null && h.waitStatus != 0)
            //唤醒阻塞队列中的一个线程 默认是唤醒head的后继结点的线程，如果后继结点已经被取消，那么再往后面遍历
            unparkSuccessor(h);
        return true;
    }
    return false;
}
private void unparkSuccessor(Node node) {
    /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
    int ws = node.waitStatus;
    if (ws < 0)
        //将head结点的状态设置为
        compareAndSetWaitStatus(node, ws, 0);

    /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
    //遍历阻塞队列，找到一个可被唤醒的线程
    Node s = node.next;
    //因为head的后置结点可能被删除/取消了，所以需要从阻塞队列中找到一个可被唤醒的，也就是waitstatus <= 0的
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        //唤醒结点所在的线程
        LockSupport.unpark(s.thread);
}
```

​		总结下释放锁的流程：

1. 线程释放锁成功后，获取到head结点。
2. 判断head的后继结点是否需要被唤醒，如果不需要再进行往后找，直到找到或者阻塞队列为空。
3. 被唤醒的结点会将自己设置为头结点，在acquireQueued方法中。

### 总结

​		AQS其实就是使用CAS操作其内部的state变量，如果操作成功（满足自己的期望才叫成功，这个期望需要子类去实现）则理解为获取到了锁，不用做任何处理，但在解锁时需要判断是否需要阻塞在阻塞队列中的线程。对内部的state变量进行CAS操作失败的线程，会包装成一个Node对象，加入到一个用双向链表实现的阻塞队列的末尾。

​		正常情况下，阻塞队列中的除了尾结点之前的结点的waitstatus都应该为-1。假设现在有4个线程，只有线程1获取到了锁，其它三个线程都阻塞到在了阻塞队列上，那么它们在AQS阻塞队列上的结构图如下：

![image-20210410015724369](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210410015724369.png)

> 补充说明下：线程1直接就CAS抢锁成功了，head结点是不会有线程1的相关信息的，在线程1释放锁之后，线程2成功head结点，这个时候head结点就会有线程2的相关信息了。

​		阻塞一个线程和唤醒一个线程用的是LockSupport.park()和LockSupport.unpark()方法。

## ReentrantLock

​		ReentrantLock是我们用的最多的锁，它其中有几个内部类实现了AQS同步器，这个几个内部类分别有着不同的加锁/解锁方式。但是本质上都是实现AQS同步器的tryAcquire方法和tryRelease方法。这两个方法就是在对AQS的state变量进行CAS操作，从而判断是否加锁/解锁成功。加锁其实就是对state变量做加法操作，释放锁就是对state变量做减法操作。

### 关键属性

```java
//当前选择使用的AQS同步器 这个类是ReentrantLock的内部类，它继承了AQS同步器
private final Sync sync;
```

#### 内部类Sync

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    //....其它内容省略
    
    abstract void lock();//加锁方法
    
    //实现了AQS的tryRelease方法，从而去控制如何释放锁
    //参数releases是对state变量要减的值
    protected final boolean tryRelease(int releases) {
        //c是对state变量进行减法后的值
        int c = getState() - releases;
        //这里会做个判断，判断当前需要释放锁的线程是不是拥有锁的线程
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        //如果进行减法操作后，state等于0，那么说明锁将会被释放成功，那么需要做释放锁的相关操作。
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        //对state变量进行CAS操作，也就是将state设置为c
        setState(c);
        return free;
    }
    
    //.....其它内容省略
}    
```

#### 内部类FairSync

​		看名字都能看出来，这是公平同步器，也就是如果用这个同步器去实现锁，那么这个锁是公平的。FairSync的tryAcquire方法控制了如何去实现这个公平锁。

```java
//返回false则代表加锁失败，true为加锁成功
static final class FairSync extends Sync {
	//加锁方法 会调用AQS的acquire方法，然后又返回来调用FairSync的tryAcquire方法
    //从这行代码也可以得出结论：ReentrantLock加锁是对state进行+1操作
    final void lock() {
        acquire(1);
    }
	//这个方法用于控制reentrantLock的加锁方式 无处不在的模板方法模式hhhhhh
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        //如果state的值为0 那么说明当前锁没有线程占有，那么会进行一次抢锁操作
        if (c == 0) {
            //注意，这里就是公平锁和非公平锁的区别
            //hasQueuedPredecessors方法判断阻塞队列中是否有其它线程在等待，只有返回false，也就是阻塞队列中没有其它线程在等到，才会调用compareAndSetState进行一次抢锁操作
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                //抢锁成功将当前线程设置为锁的拥有线程
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //进入这个分支说明当前线程是来重入锁的 这也说明ReentrantLock是可重入锁
        else if (current == getExclusiveOwnerThread()) {
            //重新入锁会将state再进行加法操作
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

#### 内部类NonfairSync

​		看名字都能看出来，这是非公平同步器，也就是如果用这个同步器去实现锁，那么这个锁不是公平的。NonfairSync的tryAcquire方法控制了如何去实现这个非公平锁。

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    //这个lock方法和公平锁差别有点大，它调用lock方法一定会进行一次抢锁操作，不管当前是什么情况，抢了再说
    final void lock() {
        //CAS操作抢锁
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            //抢锁失败再调用acquire看下能不能抢锁成功
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        //在父类ReentrantLock中实现
        return nonfairTryAcquire(acquires);
    }
}
//ReentrantLock.java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        //注意这里和公平锁的不同，发现state为0，那么不管阻塞队列有没有都会进行一次抢锁操作
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //重入锁 实现和公平锁是一样的
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

### 关键方法

#### 默认构造方法

```java
public ReentrantLock() {
    sync = new NonfairSync();
}
```

​		可以看到默认是使用公平锁的。

#### 带参构造方法

```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

​		带参的构造方法可以由我们控制选择公平锁还是非公平锁。

#### lock方法

```java
public void lock() {
    //调用内部类实现的同步器的lock方法，前面分析过了lock方法
    sync.lock();
}
```

#### unlock方法

```java
public void unlock() {
    //调用内部类实现的同步器的unlock方法，前面也分析过了
    sync.release(1);
}
```

### 总结

​		从ReentrantLock的几个关键属性，内部类，关键方法可以得出结论：**ReentrantLock本质上就是在控制对state变量的值从而达到加锁/解锁的目的**，至于加锁成功/失败和解锁成功/失败后对线程的处理，那就是完全由AQS同步器去控制的，ReentrantLock只关注如何去加锁和解锁。

​		ReentrantLock有两个内部类实现了AQS同步器，分别代表了公平锁和非公平锁的实现。对于公平锁而言，只有state变量为0且阻塞队列中没有其它线程才会进行抢锁，而对于非公平锁而言，它的不公平体现在如下两个地方：

1. 调用lock方法后就进行一次抢锁。

2. 第一次抢锁失败后，如果发现state变量为0，不管阻塞队列有没有线程又会再进行一次抢锁。

   而对于释放锁，公平锁和非公平锁的实现都是一样的，两者都是可重入锁，且只有state为0以后才算释放掉了锁。

## ThreadPoolExecutor

​		ThreadPoolExecutor是线程池的实现类。一般我们使用线程池都是基于这个类的基础上去使用，线程池内部可以分为两个部分：**缓冲队列**+**线程池**。先来看下ThreadPoolExecutor的继承关系图，再继续分析源码。

![image-20210410163614467](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210410163614467.png)

​		Executor接口的作用是执行一个任务，它完成任务的执行和线程的调度。ExecutorService接口在Executor接口上扩展了一些功能，例如可以提交返回Future的任务，提供了控制线程池的方法。AbstractExecutorService抽象类则将执行任务的流程串了起来，。最下面的ThreadPoolExecutor则实现了复杂的运行流程。

### 关键属性

```java
//这个变量同时用来记录线程池状态和线程池中有效线程数量 高3位存线程池状态 低29位保存有效线程数量
//为什么要用一个变量记录？用一个变量记录两个值可以在需要保持两个值一致时不需要用到锁
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
//缓冲队列 如果线程池中没有足够的线程去执行任务，就将任务放入到缓存池中
//注意：这个队列是阻塞队列 也就是如果有线程没获取到任务，会阻塞在这个队列里面
private final BlockingQueue<Runnable> workQueue;
//缓冲池中的线程全部放在这个set里面
private final HashSet<Worker> workers = new HashSet<Worker>();
//线程池中的最大线程数量
private int maximumPoolSize;
//线程池中核心线程数量 线程池中的线程分为核心线程+普通线程 普通线程会被回收掉，核心线程不会被回收掉
private volatile int corePoolSize;
//拒绝策略处理器 当缓冲队列满了且线程池中的线程已经达到了最大线程数 那么要拒绝提交任务
private volatile RejectedExecutionHandler handler;
//空闲线程的存活时间 也就是线程空闲了多少秒以后就会被回收
private volatile long keepAliveTime;


//线程池的状态标志

//正常运行状态
private static final int RUNNING    = -1 << COUNT_BITS;
//预关闭状态 线程池停止接受新任务，但还会执行缓冲队列中的原有任务
private static final int SHUTDOWN   =  0 << COUNT_BITS;
//关闭状态 线程池停止接受新任务，也不会执行缓冲队列中的原有任务，中断正在执行任务的线程
private static final int STOP       =  1 << COUNT_BITS;
//所有任务都被回收了 此时线程池中工作线程数量为0。线程池进入这个状态时会调用terminated()方法
private static final int TIDYING    =  2 << COUNT_BITS;
//terminated()方法结束后进入这个状态
private static final int TERMINATED =  3 << COUNT_BITS;

```

#### 内部类

​		Worker类的作用就像它的名字一样，工人。线程池将其中的线程封装成一个个工人，然后不断的从缓冲池中获取任务执行。

```java
//注意看这个类，继承了AQS，说明实现了Worker自己想要的锁
//实现了Runnable接口，说明可以当作任务被线程类Thread执行
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    private static final long serialVersionUID = 6138294804551838833L;

    //Worker对象中包装的线程
    final Thread thread;
    //Worker对象最先执行的任务 可以是null，如果是null就从阻塞队列中去获取任务执行
    Runnable firstTask;
    //记录完成的任务数量
    volatile long completedTasks;

	//构造方法只有一个带参的构造方法 说明创建Worker对象必须指定一个任务，这个任务可以是null
    Worker(Runnable firstTask) {
        //这一步是为了禁止Worker对象被中断
        setState(-1); 
        this.firstTask = firstTask;
        //创建一个线程，线程执行的任务就是Worker对象本身
        this.thread = getThreadFactory().newThread(this);
    }

    //线程创建后将执行这个方法
    public void run() {
        runWorker(this);
    }

	//用于判断当前Worker是否已经加上锁 0是没加锁 1是加锁了
    protected boolean isHeldExclusively() {
        return getState() != 0;
    }
    
	//Worker的加锁方法 可以看到Worker使用的是不可重入锁
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }
	//释放锁
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }
	//中断线程
    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

​		Worker类还是比较简单的，创建一个Worker对象需要指定任务，然后Worker对象会初始化一个线程去执行任务。执行任务这个方法的主体在ThreadPoolExecutor.runWorker方法中，接着再看下这个方法：

```java
//参数是Worker对象 注意
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    //获取Worker对象的第一个任务
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //Worker对象循环执行任务
        //如果Worker对象的第一个子任务为空 那么需要从缓冲队列中去获取一个任务 获取到不到任务会阻塞在缓冲队列中。
        while (task != null || (task = getTask()) != null) {
            //使用Worker对象实现的不可重入锁 保证一个线程必须执行完一个任务才能执行其它任务
            //获取独占锁成功也可以说明当前线程正在执行任务
            w.lock();
            //如果线程池状态为关闭状态，需要中断当前执行任务的线程
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                //留给子类实现 使用了模板方法模式
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //Wroker对象（线程）执行获取到的任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //留给子类实现 使用了模板方法模式
                    afterExecute(task, thrown);
                }
            } finally {
                //执行完成后将task置为null  继续获取下一个任务
                task = null;
				//计算当前线程已执行的任务数量
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        //getTask方法返回null则会到这里 对Wroker对象进行回收
        processWorkerExit(w, completedAbruptly);
    }
}
```

​		可以看到Wroker对象创建后首先会执行自己的第一个任务，如果第一个任务为空，那么会从缓冲队列中获取任务，继续看下获取任务的getTask方法。

```java
//这个方法的执行情况有以下几种：
//1. 核心线程阻塞直到获取到任务返回。
//2. 普通线程超时退出或在超时时间前获取到任务返回
//3. 下面几种情况也不允许获取任务了，返回null：
//		线程池处于待关闭状态且缓冲队列中没有任务，也就是不允许接受新的任务了
//		线程池中有大于最大线程数个线程存在
//		线程池处于Stop状态，缓冲队列中的任务也不执行了
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 线程池处于待关闭状态且缓冲队列中没有任务，那么直接返回null
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // 这个变量用于标记在缓冲队列上阻塞是否需要超时返回 可以看到allowCoreThreadTimeOut标记为true或线程是普通线程会超时返回
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
		
        //线程池中有大于最大线程数个线程存在或
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            //普通线程或打开了allowCoreThreadTimeOut标记的时候，调用的是poll方法，从缓冲队列中获取任务有个超时时间，超时时间就是创建线程池时指定的keepAliveTime参数
            //核心线程获取任务使用的是task方法，获取不到任务会一直阻塞在缓冲队列中
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

### 关键方法

#### 构造方法

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

​		这个构造方法是参数最多的构造方法，通过这个构造方法可以指定线程池的一些参数，例如核心线程数，最大线程数等等。一般我们都会选择参数比较少的那几个构造方法来进行创建线程池。

#### execute方法

​		execute方法用于执行一个任务，来看看执行一个任务线程池会发生什么事。

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    //如果线程池当前线程数小于核心线程数 那么就创建一个Worker对象（线程）
    if (workerCountOf(c) < corePoolSize) {
        //创建的Worker对象第一个执行任务是当前提交的任务
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    //进入这个分支说明线程池是正常状态且当前线程数大于等于核心线程数 那么将需要执行的任务加入到缓冲队列中 这里有可能会加入失败
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        //进行一个复查 判断线程池是否是运行状态
        if (! isRunning(recheck) && remove(command))
            reject(command);
        //这一步是为了防止任务提交到了缓冲队列中，但是线程池的全部线程却关闭了的情况
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    //进入这个分支说明线程池是正常状态，但是任务加入缓冲队列失败，可能是缓冲队列满了。这个时候就需要去判断当前线程池的线程数是否小于最大线程数，如果小于则创建一个Woker对象去执行提交的任务，如果等于那么直接执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}
```

​		从execute方法的逻辑可以得出线程池在**正常运行**的情况下的对任务的调度策略如下：

1. 如果线程池线程数量小于核心线程数，那么会创建一个线程去执行提交的任务

2. 如果线程池线程数量大于等于核心线程数，那么首先会尝试将任务加入到缓冲队列中。

3. 如果加入缓冲队列失败，会判断线程池线程数量是否小于最大线程数，小于则创建一个线程去执行提交的任务，大于则执行拒绝策略。

   继续看给线程池创建一个线程的addWorker方法：

```java
//说下第二个参数 这个参数如果是true 线程池最大线程数量为核心线程数 反之为最大线程数。也可以理解为true代表创建的是核心线程，反之是普通线程
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 如果线程池处于非正常运行状态，且满足以下条件之一，会拒绝创建线程：
        //1. 线程池状态大于待关闭状态
        //2. 线程执行的任务不为空
        //3. 缓冲队列不为空
        //我理解这个判断的作用就是线程池处理待关闭状态就不允许创建线程了
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            //这里就体现了参数core的作用
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //将线程池中的线程数量加1 然后跳出循环去创建Worker对象
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        //创建一个Worker对象
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            //这里用了一个可重入锁 保证Worker对象能够正确被添加至线程池中
            mainLock.lock();
            try {
                int rs = runStateOf(ctl.get());
				
                //只有符合下面两个情况才会去创建Worker对象：
                //1. 线程池处于正常运行状态
                //2. 线程池处于待关闭状态，但是缓冲队列中还有任务需要执行，这种情况也会去创建Worker对象
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    //将Woker对象加入到缓存池的HashSet中，这个set保持了所有的Worker对象
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        //largestPoolSize用于记录线程池曾经到达过的最大线程数量
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                //Worker对象创建成功后，启动线程，会调用Worker对象的run方法，再接着调用外部类的runWorker方法
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

​		使用线程池的execute方法执行一个任务的整体流程其实就是根据线程池的调度策略决定做如下的操作之一：

1. 创建一个线程来执行任务
2. 将任务加入到缓冲队列
3. 执行调度策略

#### submit的三个方法

​		线程池执行任务还有一个方法，submit方法，这个方法在父类AbstractExecutorService中实现。

```java
//提交普通的Runnable任务
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    //将其封装成RunnableFuture后再交给execute方法执行
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}
//提交普通的Runnable任务 第二个参数会放入到Future中作为返回值 
public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    //将其封装成RunnableFuture后再交给execute方法执行
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}
//提交可获取返回值的Callable任务
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    //将其封装成RunnableFuture后再交给execute方法执行
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

​		可以看到submit方法和execute方法有以下区别：

1. submit方法可以提交Callable任务，而execute方法只可以提交Runnable任务
2. submit方法会将提交的任务封装成RunnableFuture对象，最后再将RunnableFuture对象作为参数调用execute方法。
3. submit方法本质上就是将提交的任务封装成RunnableFuture对象再调用execute方法。
4. RunnableFuture接口实现了Runnable接口和Future接口，所以使用submit方法提交的任务还可以得到一个Future对象，从而去获取任务的执行情况。

### 总结

​		下图是线程池运行流程图，看明白这张图就能搞清楚线程池的运行流程：

![image-20210411134601856](https://fanshanchao.oss-cn-shenzhen.aliyuncs.com/img/image-20210411134601856.png)				