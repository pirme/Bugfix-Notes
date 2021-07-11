## 前言

男生魅力最重要的部分，**是有勇气面对失败**；而生活的美好之处，**恰恰在于它的不确定性**

AbstractQueuedSynchronizer（AQS）是 Java 并发编程中绕不过去的一道坎，JUC 并发包下的 Lock、Semaphore、ReentrantLock 等都是基于 AQS 实现的。AQS 是一个抽象的同步框架，提供了原子性管理同步状态，基于阻塞队列模型实现阻塞和唤醒等待线程的功能

文章从 ReentrantLock 加锁、解锁应用 API 入手，逐步讲解 AQS 对应源码以及相关隐含流程

列出本篇文章大纲以及相关知识点，方便大家更好的理解

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201115144318438.png)



## 什么是 ReentrantLock

ReentrantLock 翻译为 **可重入锁**，指的是一个线程能够对 **临界区共享资源进行重复加锁**

确保线程安全最常见的做法是利用锁机制（Lock、sychronized）来对 **共享数据做互斥同步**，这样在同一个时刻，只有 **一个线程可以执行某个方法或者某个代码块**，那么操作必然是 **原子性的，线程安全的**

这里就有个疑问，因为 JDK 中关键字 **synchronized** 也能同时支持原子性以及线程安全



**有了 synchronized 关键字后为什么还需要 ReentrantLock?**



为了大家更好的掌握 ReentrantLock 源码，这里列出两种锁之间的区别

|                |   Synchronized   |         ReentrantLock          |
| :------------: | :--------------: | :----------------------------: |
|   锁实现机制   | 对象头监视器模式 |            依赖 AQS            |
|     灵活性     |      不灵活      | 支持响应中断、超时、尝试获取锁 |
|   释放锁形式   |    自动释放锁    |       显示调用 unlock()        |
|   支持锁类型   |     非公平锁     |       公平锁 & 非公平锁        |
|    条件队列    |    单条件队列    |          多个条件队列          |
| 是否支持可重入 |       支持       |              支持              |



<br/>

通过以上六个维度比对，可以看出 ReentrantLock 是要比 synchronized **灵活以及支持功能更丰富**



## 什么是 AQS

AQS（ AbstractQueuedSynchronizer ）是一个用来构建锁和同步器的抽象框架，只需要继承 AQS 就可以很方便的实现我们自定义的多线程同步器、锁



![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201114220450839.png)



如图所示，在 **java.util.concurrent** 包下相关锁、同步器（常用的有 ReentrantLock、 ReadWriteLock、CountDownLatch...）都是基于 AQS 来实现



**AQS 是典型的模板方法设计模式**，父类（AQS）定义好骨架和内部操作细节，具体规则由子类去实现







### AQS 核心原理

如果被请求的共享资源未被占用，将当前请求资源的线程设置为独占线程，并将共享资源设置为锁定状态

AQS 使用一个 Volatile 修饰的 int 类型的成员变量 State 来表示同步状态，修改同步状态成功即为获得锁

Volatile 保证了变量在多线程之间的可见性，修改 State 值时通过 CAS 机制来保证修改的原子性

如果共享资源被占用，需要一定的阻塞等待唤醒机制来保证锁的分配，AQS 中会将竞争共享资源失败的线程添加到一个变体的 CLH 队列中



![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201115014812244.png)



关于支撑 AQS 特性的重要方法及属性如下：

```java
public abstract class AbstractQueuedSynchronizer 
  extends AbstractOwnableSynchronizer implements java.io.Serializable {
  	// CLH 变体队列头、尾节点
    private transient volatile Node head;
  	private transient volatile Node tail;
  	// AQS 同步状态
   	private volatile int state;
  	// CAS 方式更新 state
  	protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
}
```



### CLH 队列

既然是 AQS 中使用的是 CLH 变体队列，我们先来了解下 CLH 队列是什么

CLH：Craig、Landin and Hagersten 队列，是 **单向链表实现的队列**。申请线程只在本地变量上自旋，**它不断轮询前驱的状态**，如果发现 **前驱节点释放了锁就结束自旋**

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201107234158422.png)

通过对 CLH 队列的说明，可以得出以下结论

1. CLH 队列是一个单向链表，保持 FIFO 先进先出的队列特性
2. 通过 tail 尾节点（原子引用）来构建队列，总是指向最后一个节点
3. 未获得锁节点会进行自旋，而不是切换线程状态
4. 并发高时性能较差，因为未获得锁节点不断轮训前驱节点的状态来查看是否获得锁



AQS 中的队列是 CLH 变体的虚拟双向队列，通过将每条请求共享资源的线程封装成一个节点来实现锁的分配

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201114161720040.png)

相比于 CLH 队列而言，AQS 中的 CLH 变体等待队列拥有以下特性

1. AQS 中队列是个双向链表，也是 FIFO 先进先出的特性
2. 通过 Head、Tail 头尾两个节点来组成队列结构，通过 volatile 修饰保证可见性
3. Head 指向节点为已获得锁的节点，是一个虚拟节点，节点本身不持有具体线程
4. 获取不到同步状态，会将节点进行自旋获取锁，自旋一定次数失败后会将线程阻塞，相对于 CLH 队列性能较好



### 认识 AOS

抽象类 AQS 同样继承自抽象类 AOS（AbstractOwnableSynchronizer）

AOS 内部只有一个 Thread 类型的变量，提供了获取和设置当前独占锁线程的方法

主要作用是 **记录当前占用独占锁（互斥锁）的线程实例**

```java
public abstract class AbstractOwnableSynchronizer implements java.io.Serializable {
    // 独占线程（不参与序列化）
    private transient Thread exclusiveOwnerThread;
    // 设置当前独占的线程
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }
    // 返回当前独占的线程
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```

### 为什么要掌握 AQS

如何能够体现程序员的水平，那就是掌握大多数人所不掌握的技术，这也是为什么面试时 AQS 高频出现的原因，因为它不简单



最初接触 ReentrantLock 以及 AQS 的时候，看到源码就是一头雾水，Debug 跟着跟着就 **迷失了自己**，相信这也是大多数人的反应

正是因为经历过，所以才能从小白的心理上出发，把其中的知识点能够尽数梳理



作者写的很用心，看过这篇文章的小伙伴，不敢保证百分百理解 AQS 和 ReentrantLock 的原理，但是一定会有所收获



## 独占加锁源码解析



### 什么是独占锁

独占锁也叫排它锁，是指该锁一次只能被一个线程所持有，如果别的线程想要获取锁，只有等到持有锁线程释放

获得排它锁的线程即能读数据又能修改数据，与之对立的就是共享锁

共享锁是指该锁可被多个线程所持有。如果线程T对数据A加上共享锁后，则其他线程只能对A再加共享锁，不能加排它锁

获得共享锁的线程只能读数据，不能修改数据



### 独占锁加锁

ReentrantLock 就是独占锁的一种实现方式，接下来看代码中如何使用 ReentrantLock 完成独占式加锁业务逻辑

```java
public static void main(String[] args) {
    // 创建非公平锁
    ReentrantLock lock = new ReentrantLock();
    // 获取锁操作
    lock.lock();
    try {
        // 执行代码逻辑
    } catch (Exception ex) {
        // ...
    } finally {
        // 解锁操作
        lock.unlock();
    }
}
```



new ReentrantLock() 构造函数默认创建的是非公平锁 NonfairSync

```java
public ReentrantLock() {
    sync = new NonfairSync();
}
```





同时也可以在创建锁构造函数中传入具体参数创建公平锁 FairSync

```java
ReentrantLock lock = new ReentrantLock(true);
--- ReentrantLock
// true 代表公平锁，false 代表非公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```



FairSync、NonfairSync 代表公平锁和非公平锁，两者都是 ReentrantLock 静态内部类，只不过实现不同锁语义

**公平锁 FairSync**

1. 公平锁是指多个线程按照申请锁的顺序来获取锁，线程直接进入队列中排队，队列中的第一个线程才能获得锁
2. 公平锁的优点是等待锁的线程不会饿死。缺点是整体吞吐效率相对非公平锁要低，等待队列中除第一个线程以外的所有线程都会阻塞，CPU 唤醒阻塞线程的开销比非公平锁大

**非公平锁 NonfairSync**

1. 非公平锁是多个线程加锁时直接尝试获取锁，获取不到才会到等待队列的队尾等待。但如果此时锁刚好可用，那么这个线程可以无需阻塞直接获取到锁
2. 非公平锁的优点是可以减少唤起线程的开销，整体的吞吐效率高，因为线程有几率不阻塞直接获得锁，CPU 不必唤醒所有线程。缺点是处于等待队列中的线程可能会饿死，或者等很久才会获得锁



两者的都继承自 ReentrantLock 静态抽象内部类 Sync，**Sync 类继承自 AQS**，这里就有个疑问

这些锁都没有直接继承 AQS，而是定义了一个 **Sync 类去继承 AQS**，为什么要这样呢？

因为 **锁面向的是使用用户**，**同步器面向的则是线程控制**，那么在锁的实现中聚合同步器而不是直接继承 AQS 就可以很好的 **隔离二者所关注的事情**

通过对不同锁种类的讲解以及 ReentrantLock 内部结构的解析，根据上下级关系继承图，加深其理解

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201106104059148.png)

这里以非公平锁举例，查看加锁的具体过程，详细信息下文会详细说明

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201117222132211.png)



看一下非公平锁加锁方法 lock 内部怎么做的

```java
ReentrantLock lock = new ReentrantLock();
lock.lock();
--- ReentrantLock
public void lock() {
    sync.lock();
}
--- Sync
abstract void lock();
```



**Sync#lock** 为抽象方法，最终会调用其子类非公平锁的方法 lock 

```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```



非公平加锁方法有两个逻辑

1. 通过比较并替换 State（同步状态）成功与否决定是否获得锁，设置 State 为 1表示成功获取锁，并将当前线程设置为独占线程
2. 修改 State 值失败则进入尝试获取锁流程，acquire 方法为 AQS 提供的方法



compareAndSetState 以 CAS 比较并替换的方式将 State 值设置为 1，表示同步状态被占用

```java
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, StateOffset, expect, update);
}
```



setExclusiveOwnerThread 设置当前线程为独占锁拥有线程

```java
protected final void setExclusiveOwnerThread(Thread thread) {    exclusiveOwnerThread = thread;}
```



acquire 对整个 AQS 做到了承上启下的作用，通过 tryAcquire 模版方法进行尝试获取锁，获取锁失败包装当前线程为 Node 节点加入等待队列排队

```java
public final void acquire(int arg) {    if (!tryAcquire(arg) &&            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))        selfInterrupt();}
```



tryAcquire 是 AQS 中抽象模版方法，但是内部会有默认实现，虽然默认的方法内部抛出异常，**为什么不直接定义为抽象方法呢?**

因为 AQS 不只是对独占锁实现了抽象，同时还包括共享锁；不同锁定义了不同类别的方法，共享锁就不需要 tryAcquire，如果定义为抽象方法，继承 AQS 子类都需要实现该方法

```java
protected boolean tryAcquire(int arg) {    throw new UnsupportedOperationException();}
```



NonfairSync 类中有 tryAcquire 重写方法，继续查看具体如何进行非公平方式获取锁

```java
protected final boolean tryAcquire(int acquires) {    return nonfairTryAcquire(acquires);}final boolean nonfairTryAcquire(int acquires) {    final Thread current = Thread.currentThread();    int c = getState();  	// State 等于0表示此时无锁    if (c == 0) {      	// 再次使用CAS尝试获取锁, 表现为非公平锁特性        if (compareAndSetState(0, acquires)) {          	// 设置线程为独占锁线程            setExclusiveOwnerThread(current);            return true;        }    // 如果当前线程等于已获取锁线程, 表现为可重入锁特性    } else if (current == getExclusiveOwnerThread()) {        int nextc = c + acquires;        if (nextc < 0) // overflow            throw new Error("Maximum lock count exceeded");      	// 设置 State        setState(nextc);        return true;    }  	// 如果state不等于0并且独占线程不是当前线程, 返回 false    return false;}
```



由于 tryAcquire 做了取反，如果设置 state 失败并且独占锁线程不是自己本身返回 false，通过取反会进入接下来的流程

```java
public final void acquire(int arg) {    if (!tryAcquire(arg) &&            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))        selfInterrupt();}
```



### Node 入队流程

尝试获得锁失败，接下来会将线程组装成为 Node 进行入队流程



Node 是 AQS 中最基本的数据结构，也是 CLH 变体队列中的节点，Node 有 **SHARED（共享）、EXCLUSIVE（独占）** 两种模式，文章主要介绍 EXCLUSIVE 模式，不相关的属性和方法不予介绍

下面列出关于 Node EXCLUSIVE 模式的一些关键方法以及状态信息

| 关键方法与属性 |              对应含义               |
| :------------: | :---------------------------------: |
|   waitStatus   |    当前节点在队列中处于什么状态     |
|     thread     |         表示节点对应的线程          |
|      prev      |  前驱指针，指向本节点的上一个节点   |
|      next      |  后继指针，指向本节点的下一个节点   |
|  predecessor   | 返回前驱节点，没有的话抛出 NPE 异常 |



Node 中独占锁相关的 waitStatus 属性分别有以下几种状态

|  属性值   |                     值含义                     |
| :-------: | :--------------------------------------------: |
|     0     |            Node 被初始化后的默认值             |
| CANCELLED |       值为1，由于中断或超时，节点被取消        |
|  SIGNAL   |      值为-1，表示节点的后继节点即将被阻塞      |
| CONDITION | 值为-2，表示节点在等待队列中，节点线程等待唤醒 |



介绍完 Node 相关基础知识，看一下请求锁线程如何被包装为 Node，又是如何初始化入队的

```java
private Node addWaiter(Node mode) {    Node node = new Node(Thread.currentThread(), mode);    // 获取等待队列的尾节点    Node pred = tail;  	// 如果尾节点不为空, 将 node 设置为尾节点, 并将原尾节点 next 指向 新的尾节点node    if (pred != null) {        node.prev = pred;        if (compareAndSetTail(pred, node)) {            pred.next = node;            return node;        }    }  	// 尾部为空，enq 执行    enq(node);    return node;}
```

pred 为队列的尾节点，根据尾节点是否为空会执行对应流程

1. 尾节点不为空，证明队列已被初始化，那么需要将对应的 node（当前线程）设置为新的尾节点，也就是入队操作；将 node 节点的前驱指针指向 pred（尾节点），并将 node 通过 CAS 方式设置为 AQS 等待队列的尾节点，替换成功后将原来的尾节点后继指针指向新的尾节点
2. 尾节点为空，证明还没有初始化队列，执行 enq 方法进行初始化队列



enq 方法执行初始化队列操作，等待队列中虚拟化的头节点也是在这里产生

```java
private Node enq(final Node node) {    for (; ; ) {        Node t = tail;        if (t == null) {          	// 虚拟化一个空Node, 并将head指向空Node            if (compareAndSetHead(new Node()))              	// 将尾节点等于头节点                tail = head;        } else {          	// node上一条指向尾节点            node.prev = t;          	// 设置node为尾节点            if (compareAndSetTail(t, node)) {              	// 设置原尾节点的下一条指向node                t.next = node;                return t;            }        }    }}
```

**执行 enq 方法的前提就是队列尾节点为空，为什么还要再判断尾节点是否为空？**



因为 enq 方法中是一个死循环，循环过程中 t 的值是不固定的。假如执行 enq 方法时队列为空，for 循环会执行两遍不同的处理逻辑

1. 尾节点为空，虚拟化出一个新的 Node 头节点，这时队列中只有一个元素，为了保证 AQS 队列结构的完整性，会将尾节点指向头节点，第一遍循环结束
2. 第二遍不满足尾节点为空条件，执行 else 语句块，node 节点前驱指针指向尾节点，并将 node 通过 CAS 设置为新的尾节点，成功后设置原尾节点的后继指针指向 node，至此入队成功。返回的 t 无意义，只是为了终止死循环



画两张图来理解 enq 方法整体初始化 AQS 队列流程，假设T1、T2两个线程争取锁，T1成功获得锁，T2进行入队操作

1. T2进行入队操作，循环第一遍，尾节点为空。开始初始化头节点，并将尾节点指向头节点，最终队列形式是这样纸滴

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201114213221013.png)

2. 循环第二遍，需要将 node 设置为新的尾节点。逻辑如下：尾节点不为空，设置 node 前驱指针指向尾节点，并将 node 设置为尾节点，原尾节点 next 指针指向 node



addWaiter 方法就是为了让 Node 入队，并且维护出一个双向队列模型

入队执行成功后，会在 acquireQueued 再次尝试竞争锁，竞争失败后会将线程阻塞

```java
public final void acquire(int arg) {    if (!tryAcquire(arg) &&            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))        selfInterrupt();}
```



acquireQueued 方法会尝试自旋获取锁，获取失败对当前线程实施阻塞流程，这也是为了避免无意义的自旋，对比 CLH 队列性能优化的体现

```java
final boolean acquireQueued(final Node node, int arg) {    boolean failed = true;    try {        boolean interrupted = false;        for (; ; ) {          	// 获取node上一个节点            final Node p = node.predecessor();          	// 如果node为头节点 & 尝试获取锁成功            if (p == head && tryAcquire(arg)) {              	// 此时当前node线程获取到了锁              	// 将node设置为新的头节点                setHead(node);              	// help GC                p.next = null;                failed = false;                return interrupted;            }            if (shouldParkAfterFailedAcquire(p, node) &&                    parkAndCheckInterrupt())                interrupted = true;        }    } finally {        if (failed)            cancelAcquire(node);    }}
```



通过 node.predecessor() 获取节点的前驱节点，前驱节点为空抛出空指针异常

```java
final Node predecessor() throws NullPointerException {    Node p = prev;    if (p == null)        throw new NullPointerException();    else        return p;}
```



获取到前驱节点后进行两步逻辑判断

1. 判断前驱节点 p 是否为头节点，为 true 进行尝试获取锁，获取锁成功设置当前节点为新的头节点，并将原头节点的后驱指针设为空
2. 前驱节点不是头节点或者尝试加锁失败，执行线程休眠阻塞操作



如果 node 获得锁后，setHead 将节点设置为队列头，从而实现出队效果，出于 GC 的考虑，清空未使用的数据

```java
private void setHead(Node node) {    head = node;    node.thread = null;    node.prev = null;}
```



shouldParkAfterFailedAcquire 需要重点关注下，流程相对比较难理解

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {    int ws = pred.waitStatus;    if (ws == Node.SIGNAL)        return true;    if (ws > 0) {        do {            node.prev = pred = pred.prev;        } while (pred.waitStatus > 0);        pred.next = node;    } else {        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);    }    return false;}
```



ws 表示为当前申请锁节点前驱节点的等待状态，代码中包含三个逻辑，分别是：

1. ws == Node.SIGNAL，表示需要将申请锁节点进行阻塞
2. ws > 0，表示等待队列中包含被取消节点，需要调整队列
3. 如果 ws == Node.SIGNAL || ws >0 都为 false，使用 CAS 的方式将前驱节点等待状态设置为 Node.SIGNAL



> 设置当前节点的前置节点等待状态为 Node.SIGNAL，表示当前节点获取锁失败，需要进行阻塞操作



还是通过几张图来理解流程，假设此时 T1、T2 线程来争夺锁

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201114213609335.png)

T1 线程获得锁，T2 进入 AQS 等待队列排队，并通过 CAS 将 T2 节点的前驱节点等待状态置为 SIGNAL

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201109140133492.png)

执行切换前驱节点等待状态后返回 false，继续进行循环尝试获取同步状态

> 这一步操作保证了线程能进行多次重试，尽量避免线程状态切换



如果 T1 线程没有释放锁，T2 线程第二次执行到 shouldParkAfterFailedAcquire 方法，因为前驱节点已设置为 SIGNAL，所以会直接返回 true，执行线程阻塞操作

```java
private final boolean parkAndCheckInterrupt() {  	// 将当前线程进行阻塞    LockSupport.park(this);  	// 方法返回了当前线程的中断状态，并将当前线程的中断标识设置为false    return Thread.interrupted();}
```



LockSupport.park 方法将当前等待队列中线程进行阻塞操作，线程执行一个从 RUNNABLE 到 WAITING 状态转变

如果线程被唤醒，通过执行 Thread.interrupted 查看中断状态，这里的中断状态会被传递到 acquire 方法

```java
public final void acquire(int arg) {    if (!tryAcquire(arg) &&            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))      	// 如果线程被中断, 这里会再次设置中断状态      	// 因为如果线程中断, 调用 Thread.interrupted 虽然会返回 true, 但是会清除线程中断状态        selfInterrupt();}
```

即使线程从 park 方法中唤醒后发现自己被中断了，但是不影响接下来的获取锁操作，如果需要设置线程中断来影响流程，可以使用 lockInterruptibly 获得锁，抛出检查异常 InterruptedException





### cancelAcquire

取消排队方法是 AQS 中比较难的知识点，不容易被理解

当线程因为自旋或者异常等情况获取锁失败，会调用此方法进行取消正在获取锁的操作

```java
private void cancelAcquire(Node node) {    // 不存在的节点直接返回    if (node == null)        return;    node.thread = null;    /**     * waitStatus > 0 代表节点为取消状态     * while循环会将node节点的前驱指针指向一个非取消状态的节点     * pred等于当前节点的前驱节点（非取消状态）     */    Node pred = node.prev;    while (pred.waitStatus > 0)        node.prev = pred = pred.prev;    // 获取过滤后的前驱节点的后继节点    Node predNext = pred.next;    // 设置node等待状态为取消状态    node.waitStatus = Node.CANCELLED;    // 🚩步骤一，如果node是尾节点，使用CAS将pred设置为新的尾节点  	    if (node == tail && compareAndSetTail(node, pred)) {      	// 设置pred（新tail）的后驱指针为空        compareAndSetNext(pred, predNext, null);    } else {        int ws;      	// 🚩步骤二，node的前驱节点pred（非取消状态）!= 头节点        if (pred != head             	/**            	 * 1. pred等待状态等于SIGNAL            	 * 2. ws <= 0并且设置pred等待状态为SIGNAL            	 */            	&& ((ws = pred.waitStatus) == Node.SIGNAL || (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL)))             	// pred中线程不为空            	&& pred.thread != null) {            Node next = node.next;          	/**          	 * 1. 当前节点的后继节点不为空          	 * 2. 后继节点等待状态<=0（表示非取消状态）          	 */            if (next != null && next.waitStatus <= 0)              	// 设置pred的后继节点设置为当前节点的后继节点                compareAndSetNext(pred, predNext, next);        } else {          	// 🚩步骤三，如果当前节点为头节点或者上述条件不满足, 执行唤醒当前节点的后继节点流程            unparkSuccessor(node);        }        node.next = node; // help GC    }}
```



逻辑稍微复杂一些，比较重要是以下三个逻辑

1. 步骤一当前节点为尾节点的话，设置 pred 节点为新的尾节点，成功设置后再将 pred 后继节点设置为空（尾节点不会有后继节点）

2. 步骤二需要满足以下四个条件才会将前驱节点（非取消状态）的后继指针指向当前节点的后继指针

   1）当前节点不等于尾节点

   2）当前节点前驱节点不等于头节点

   3）前驱节点的等待状态不为取消状态

   4）前驱节点的拥有线程不为空

3. 如果不满足步骤二的话，会执行步骤三相关逻辑，唤醒后继节点







**步骤一：**

假设当前取消节点为尾节点并且前置节点无取消节点，现有等待队列如下图，执行下述逻辑

```java
if (node == tail && compareAndSetTail(node, pred)) {    compareAndSetNext(pred, predNext, null);}
```

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201112180208027.png)

将 pred 设置为新的尾节点，并将 pred 后继节点设置为空，因为尾节点不会有后继节点了

T4 线程所在节点因无引用指向，会被 GC 垃圾回收处理

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201114141417895.png)

**步骤二：**

如果当前需要取消节点的前驱节点为取消状态节点，如图所示

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201114140258406.png)

设置 pred（非取消状态）的后继节点为 node 的后继节点，并设置 node 的 next 为 自己本身

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201114141329010.png)

**线程T2、T3所在节点因为被T4所直接或间接指向，如何进行GC?**

AQS 等待队列中取消状态节点会在 shouldParkAfterFailedAcquire 方法中被 GC 垃圾回收

```java
if (ws > 0) {    do {        node.prev = pred = pred.prev;    } while (pred.waitStatus > 0);    pred.next = node;}
```



T4 线程所在节点获取锁失败尝试停止时，会执行上述代码，执行后的等待队列如下图所示

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201114144243192.png)

等待队列中取消状态节点就可以被 GC 垃圾回收了，至此加锁流程也就结束了，下面继续看如何解锁



## 独占解锁源码解析

解锁流程相对于加锁简单了很多，调用对应API-lock.unlock()

```java
--- ReentrantLockpublic void unlock() {    sync.release(1);}--- AQSpublic final boolean release(int arg) {  	// 尝试释放锁    if (tryRelease(arg)) {        Node h = head;        if (h != null && h.waitStatus != 0)            unparkSuccessor(h);        return true;    }    return false;}
```



### 释放锁同步状态

tryRelease 是定义在 AQS 中的抽象方法，通过 Sync 类重写了其实现

```java
protected final boolean tryRelease(int releases) {    int c = getState() - releases;  	// 如果当前线程不等于拥有锁线程, 抛出异常    if (Thread.currentThread() != getExclusiveOwnerThread())        throw new IllegalMonitorStateException();    boolean free = false;    if (c == 0) {        free = true;      	// 将拥有锁线程设置为空        setExclusiveOwnerThread(null);    }  	// 设置State状态为0, 解锁成功    setState(c);    return free;}
```



### 唤醒后继节点

此时 State 值已被释放，对于头节点的判断这块流程比较有意思

```java
Node h = head;if (h != null && h.waitStatus != 0)		unparkSuccessor(h);
```



**什么情况下头节点为空**，当线程还在争夺锁，队列还未初始化，头节点必然是为空的

当头节点等待状态等于0，证明后继节点还在自旋，不需要进行后继节点唤醒



如果同时满足上述两个条件，会对等待队列头节点的后继节点进行唤醒操作

```java
private void unparkSuccessor(Node node) {  	// 获取node等待状态    int ws = node.waitStatus;    if (ws < 0)        compareAndSetWaitStatus(node, ws, 0);  	// 获取node的后继节点    Node s = node.next;  	// 如果下个节点为空或者被取消, 遍历队列查询非取消节点    if (s == null || s.waitStatus > 0) {        s = null;      	// 从队尾开始查找, 等待状态 <= 0 的节点        for (Node t = tail; t != null && t != node; t = t.prev)            if (t.waitStatus <= 0)                s = t;    }  	// 满足 s != null && s.waitStatus <= 0  	// 执行 unpark    if (s != null)        LockSupport.unpark(s.thread);}
```



**为什么查找队列中未被取消的节点需要从尾部开始？**

这个问题有两个原因可以解释，分别是 Node 入队和清理取消状态的节点

1. 先从 addWaiter 入队时说起，compareAndSetTail(pred, node)、pred.next = node 并非原子操作，如果在执行 pred.next = node 前进行 unparkSuccessor，就没有办法通过 next 指针向后遍历，所以才会从后向前找寻非取消的节点

2. cancelAcquire 方法也有导致使用 head 无法遍历全部 Node 的因素，因为先断开的是 next 指针，prev 指针并未断开



### 唤醒阻塞后流程

当线程获取锁失败被 park 后进入了阻塞模式，前驱节点释放锁后会进行唤醒 unpark，被阻塞线程状态回归 RUNNABLE 状态

```java
private final boolean parkAndCheckInterrupt() {  	// 从此位置唤醒    LockSupport.park(this);    return Thread.interrupted();}
```



被唤醒线程检查自身是否被中断，返回自身中断状态到 acquireQueued

```java
final boolean acquireQueued(final Node node, int arg) {    boolean failed = true;    try {        boolean interrupted = false;        for (; ; ) {            final Node p = node.predecessor();            if (p == head && tryAcquire(arg)) {                setHead(node);                p.next = null; // help GC                failed = false;                return interrupted;            }            if (shouldParkAfterFailedAcquire(p, node) &&                    parkAndCheckInterrupt())                interrupted = true;        }    } finally {        if (failed)            cancelAcquire(node);    }}
```



假设自身被中断，设置 interrupted = true，继续通过循环尝试获取锁，获取锁成功后返回 interrupted 中断状态

**中断状态本身并不会对加锁流程产生影响**，被唤醒后还是会不断进行获取锁，直到获取锁成功进行返回，返回中断状态是为了后续补充中断纪录



如果线程被唤醒后发现中断，成功获取锁后会将中断状态返回，补充中断状态

```java
public final void acquire(int arg) {    if (!tryAcquire(arg) &&            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))        selfInterrupt();}
```



selfInterrupt 就是对线程中断状态的一个补充，补充状态成功后，流程结束

```java
static void selfInterrupt() {    Thread.currentThread().interrupt();}
```



## 阅读源码小技巧

**1、从全局掌握要阅读的源码提供了什么功能**

这也是我一直推崇的学习源码方式，学习源码的关键点是抓住主线流程，在了解主线之前不要最开始就研究到源码实现细节中，否则很容易迷失在细枝末节的代码中

以文章中的 AQS 举例，当你知道了它是一个抽象队列同步器，使用它可以更简单的构造锁和同步器等实现

然后从中理解 tryAcquire、tryRelease 等方法实现，这样是不是可以更好的理解与 AQS 与其子类相关的代码



**2、把不易理解的源码粘贴出来，整理好格式打好备注**

一般源码中的行为格式和我们日常敲代码是不一样的，而且 JDK 源码中的变量命名实在是惨不忍睹

所以就应该将难以理解的源码粘贴出，标上对应注释以及调整成易理解的格式，这样对于源码的阅读就会轻松很多



## 后记

平常工作中接触到 AQS 相关知识还是很多的，知其然知其所以然，文章以 ReentrantLock 作为切入点，讲述了其公平锁和非公平锁的概念，以及对应 AQS 中 CLH、AOS 等不容易被发现的概念

针对 ReentrantLock 以及 AQS 加锁、解锁、排队等流程进行了详细说明，以图文并茂的方式讲述了其流程源码实现细节，这里希望在看的小伙伴都能收获 AQS 相关知识



**由于作者水平有限, 欢迎大家能够反馈指正文章中错误不正确的地方, 感谢 🙏**

小伙伴的喜欢就是对我最大的支持, 如果读了文章有所收获, 希望能够 **点赞、评论、关注三连!**



参考资料

- https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html
- https://tech.meituan.com/2018/11/15/java-lock.html
- https://mp.weixin.qq.com/s/y_e3ciU-hiqlb5vseuOFyw
- https://zhuanlan.zhihu.com/p/197840259?utm_source=wechat_session
- https://blog.csdn.net/java_lyvee/article/details/98966684
- https://zhuanlan.zhihu.com/p/197840259?utm_source=wechat_session

