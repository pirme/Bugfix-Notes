## 前言

如果要聊原子类相关的话题, 可以先从基本的概念开始

**1、原子类为了解决什么样的问题?**

答: 为了解决并发场景下无锁的方式保证单一变量的数据一致性



**2、什么情况下存在并发问题?**

答: 多个线程同时读写同一个共享数据时存在多线程并发问题



解决并发安全问题的方式有很多种方式, **著名的就是 JDK 并发包 concurrent**, 为了并发而存在的



## 非原子计算

大家应该都知道, 类似于代码中的 i++ 操作, 虽然是一行, 但是执行时候是分为三步的

- 从主存获取变量 i
- 变量i值+1
- 新增后变量i值写回主存



写个小程序来更好的理解, 不论我们怎么运行下方程序, 99% 以上概率不会到达 100000000

```java
static int NUM = 0;
public static void main(String[] args) throws InterruptedException {
    for (int i = 0; i < 10000; i++) {
        new Thread(() -> {
            for (int j = 0; j < 10000; j++) {
                NUM++;
            }
        }).start();
    }
    Thread.sleep(2000);
    System.out.println(NUM);
    /**
     * 99149419
     */
}
```



可以使用 JDK 自带的 synchronized, 通过 **互斥锁的方式** 同步执行 NUM++ 这个代码块

```java
static int NUM = 0;
public static void main(String[] args) throws InterruptedException {
    for (int i = 0; i < 10000; i++) {
        new Thread(() -> {
            for (int j = 0; j < 10000; j++) {
                synchronized (Object.class) {
                    NUM++;
                }
            }
        }).start();
    }
    Thread.sleep(2000);
    System.out.println(NUM);
    /**
     * 100000000
     */
}
```



前面一直在说锁, 写着 AtomicInteger 在这说锁的问题, 咱不能挂着羊皮卖狗肉是吧

![](https://images-machen.oss-cn-beijing.aliyuncs.com/好羞耻啊.png)

如果看过 JDK 源码 JUC 包下的类库源代码, 关于 **Atomic 开头的类库** 应该不会陌生

> Atomic 英  [əˈtɒmɪk] 美  [əˈtɑːmɪk] 翻译 : 原子的



如果不使用锁来解决上面的非原子自增问题, 可以这么来写

```java
static AtomicInteger NUM = new AtomicInteger();
public static void main(String[] args) throws InterruptedException {
    for (int i = 0; i < 10000; i++) {
        new Thread(() -> {
            for (int j = 0; j < 10000; j++) {
              	// 🚩 重点哦, 自增并获取新值
                NUM.incrementAndGet();
            }
        }).start();
    }
    Thread.sleep(2000);
    System.out.println(NUM);
    /**
     * 100000000
     */
}
```



## AtomicInteger 简介

看到一个技术点或者框架之类比较新颖的知识, 一般会思考两个问题



### 它是什么

AtomicInteger 是 JDK 并发包下提供的 **操作 Integer 类型原子类**, 通过调用底层 **Unsafe 的 CAS 相关方法实现原子操作**

基于乐观锁的思想实现的一种无锁化原子操作, **保障了多线程情况下单一变量的线程安全问题**

> 关于 CAS 的概念下面会详细说明



### 有什么优点

AtmoicInteger 使用硬件级别的指令 CAS 来更新计数器的值, 机器直接支持的指令, **这样可以避免加锁**

比如像互斥锁 synchronized 在并发比较严重情况下, 会将锁 **升级到重量级锁**

唤醒与阻塞线程时会有一个 **用户态到内核态的一个转变**, 而转换状态是需要消耗很多时间的

> 写过程序进行测试, 尽管 synchronized 经过升级后, 性能有了大幅度提升, 但在 **一般并发场景下, CAS 无锁算法性能更高一些**



当然不可能会有尽善尽美的存在, 关于 CAS 无锁算法会在下面说明劣势所在



## 结构分析

AtomicInteger 有两个构造方法, 分别是一个无参构造及有参构造

- 无参构造的 value 就是 int 的默认值 0
- 有参构造会将 value 赋值

```java
public AtomicInteger() { }

public AtomicInteger(int initialValue) {
    value = initialValue;
}
```



AtomicInteger 有三个重要的变量, 分别是:

**Unsafe:** 可以理解它对于 Java 而言, 是一个 "BUG" 的存在, 在 AtomicInteger 里的最大作用就是直接操作内存进行值替换

**value:** 使用 int 类型存储 AtomicInteger 计算的值, 通过 volatile 进行修饰, **提供了内存可见性及防止指令重排序**

**valueOffset:** value 的内存偏移量

```java
// 获取Unsafe实例
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

// 静态代码块，在类加载时运行
static {
    try {
      	// 获取 value 的内存偏移量
        valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) {
        throw new Error(ex);
    }
}

private volatile int value;
```



这里罗列出一些常用API, 核心实现思路都是一致的, 会着重讲其中一个

```java
// 获取当前 value 值
public final int get();

// 取当前的值, 并设置新的值
public final int getAndSet(int newValue);

// 获取当前的值, 并加上预期的值
public final int getAndAdd(int delta);

// 获取当前值, 并进行自增1
public final int getAndIncrement();
  
// 获取当前值, 并进行自减1
public final int getAndDecrement();
```



**获取当前值并自增 #getAndIncrement()**

看一下具体在源码中是如何操作的

```java
public final int getAndIncrement() {
  	return unsafe.getAndAddInt(this, valueOffset, 1);
}

/**
 * unsafe.getAndAddInt
 *
 * @param var1 AtomicInteger 对象
 * @param var2 value 内存偏移量
 * @param var4 增加的值, 比如在原有值上 + 1
 * @return
 */
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        // 内存中 value 最新值
        var5 = this.getIntVolatile(var1, var2);
    } while (!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```



这里就是 CAS 体现无锁算法的地方, 先来说下这段代码的执行步骤

1、根据 AtomicInteger 对象 以及 value 内存偏移量获取对应 value 最新值

2、通过 compareAndSwapInt(...) 将内存中的值(var5)更改为期望的值 (var5+var4), 不存在多线程竞争成功修改返回 True 结束循环, Flase 继续执行循环



精髓的地方就在于 **compareAndSwapInt(...)**

```java
/**
 * 比较 var1 的 var2 内存偏移量处的值是否和 var4 相等, 相等则更新为 var5
 *
 * @param var1 AtomicInteger 对象
 * @param var2 value 内存偏移量
 * @param var4 value 原本的值
 * @param var5 期望将 value 设置的值
 * @return
 */
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```



由于是 native 关键字修饰, 我们无法查看其源码, 说明一下方法思路

1、通过 var1(AtomicInteger) 获取到 var2 (内存偏移量) 的 value 值

2、将 value(内存中值) 与 var4(线程内获取的value值) 进行比较

3、如果相等将 var5(期望值) 设置为内存中新的 value 并返回 True

4、不相等返回 False 继续尝试执行循环



## 图文分析 CAS

这里给出一组图片进一步理解 CAS

**Unsafe#getAndAddInt()** 以此方法为例

1、这个图片相当于 AtomicInteger 对象对应的 valueOffset 位置

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20200930201842632.png)

2、线程一执行 var5 = this.getIntVolatile(var1, var2)

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20200930202236455.png)

3、线程二执行 var5 = this.getIntVolatile(var1, var2)

此时在线程一、二的工作内存中 var5 都是 0

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20200930202400022.png)

4、线程一欲修改内存中的 value 为1, 通过比对 var4 与内存中的值相等, 内存值成功被设置成 1

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20200930202633665.png)

5、线程二也是要修改内存中对应的 value 值, 发现已经不相等了, 返回 False 继续尝试修改

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20200930202827344.png)

## 不足之处

CAS 虽然能够实现无锁编程, 在一般情况下对性能做出了提升, 但是并不是没有局限性或缺点

**1、CPU 自旋开销较大**

在高并发情况下, 自旋 CAS 如果长时间不成功, 会给 CPU 带来非常大的执行开销

> 如果是实现高并发下的计数, 可以使用 LongAdder, 设计的高并发思想真的强!



**2、著名的 "ABA" 问题**

CAS 需要在操作值的时候检查下值有没有发生变化, 如果没有发生变化则更新

但是如果一个值原来是A, 变成了B, 又变成了A, 那么使用 CAS 进行检查时会发现它的值没有发生变化, 但是实际上却变化了

如果感兴趣的小伙伴可以去看下 JUCA 原子包下的 **AtomicStampedReference**

