## 前言

上一篇文章 [《通过 JDK 原子并发类 AtomicInteger 彻底掌握 CAS 无锁算法》](https://mp.weixin.qq.com/s/QIkYOcgCMzz_OPQH9L_Vfw)

和大家聊了聊 Atomic 相关的概念, 也说了下 AtomicInteger 的实现原理

同时也说了 Atomic 的不足: 

1、多线程高并发对共享资源读写操作会导致自旋过度

2、**"ABA" 问题**



今天这篇文章就来聊一聊如何解决 "ABA" 问题



## ABA 问题背景

AtomicInteger 存在的一个问题, 也是大部分 Atomic 相关类存在的, 就是 ABA 问题

简短来说, 就是线程一获取到 AtomicInteger 的 value 为 0, 在准备做修改之前

线程二对 AtomicInteger 的 value 做了两次操作, 一次是将值修改为 1, 然后又将值修改为原来的 0

此时线程一进行 CAS 操作, 发现内存中的值依旧是 0, OK, 更新成功, 结合下图了解

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20201001170629285.png)

**1、为什么线程二能够在线程一获取最新值后进行操作?**

我们以 **AtomicInteger#getAndIncrement** 说明

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
      	// 标记1
        var5 = this.getIntVolatile(var1, var2);
    } while (!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```



因为我们的 CPU 执行是 **抢占式的, 时间片分配并不固定**

可能在线程一读取完标记1处之后, 时间片就分配执行了线程二, 此时线程一等待时间片的分配

等线程二做完两步操作之后, 时间片分配到线程一, 这时才会继续执行



**2、ABA 会引发什么样的问题?**

大家应该都知道了 ABA 的行为是如何发生的, 列举网上的一个小例子, 大致意思如下:

1、小明银行卡账户余额1万元, 去取款机取钱5千元, 正常应该发起一次请求, 因为网络或机器故障发起了两次请求

> 各位大哥大姐不要抬杠哈,  不要说什么后端接口幂等, 取款机防重复提交之类的, **例子是为了说明问题**, 感谢 😄

2、线程一「由取款机发起」: 获取当前银行卡余额值1万元, 期望值5千元

3、线程二「由取款机发起」: 获取当前银行卡余额值1万元, 期望值5千元

4、线程一成功了, 银行卡内余额剩余五千,  线程二未分配时间片, 获取到余额后被阻塞

5、此时线程三「支付宝转入银行卡」: 获取当前银行卡余额5千元, 期望值1万元, 修改成功

6、线程二获得时间片, 发现卡内余额是1万, 成功将卡内余额减少至5千

7、卡内余额原本应是1万-5千+5千=1万, 最终因为 ABA 问题导致 1万-5千+5千-5千=5千



当然正式业务中, 可能不会存在此类问题, 不过平常自己业务中使用到了原子类, 还是会埋下潜在隐患

我们先通过小程序代码来看下, ABA 问题是如何复现的

```java
public static void main(String[] args) throws InterruptedException {
    AtomicInteger atomicInteger = new AtomicInteger(100);
    new Thread(() -> {
        atomicInteger.compareAndSet(100, 101);
        atomicInteger.compareAndSet(101, 100);
    }).start();

    Thread.sleep(1000);

    new Thread(() -> {
        boolean result = atomicInteger.compareAndSet(100, 101);
        System.out.println(String.format("  >>> 修改 atomicInteger :: %s ", result));
    }).start();
    /**
     *   >>> 修改 atomicInteger :: true 
     */
}
```



1、创建一个初始值为100的 AtomicInteger, 线程一将100-> 101, 再从101->100

2、休眠1000ms, 防止时间片分配线程二提前执行

3、线程二从100->101, 修改成功

## AtomicStampedReference

先来说下解决 ABA 的思路吧, 也就是 **AtomicStampedReference** 的原理

内部维护了一个 **Pair<T> 对象**, 存储了 value 值和一个版本号, 每次更新除了 value 值还会更新版本号

```java
private static class Pair<T> {
  	// 存储值, 相当于上文的值100
    final T reference;
  	// 类似于版本号的概念
    final int stamp;

    private Pair(T reference, int stamp) {
        this.reference = reference;
        this.stamp = stamp;
    }
		// 创建一个新的Pair对象, 每次值变化时都会创建一个新的对象
    static <T> AtomicStampedReference.Pair<T> of(T reference, int stamp) {
        return new AtomicStampedReference.Pair<T>(reference, stamp);
    }
}
```



我们还是先通过一个小程序来了解下 **AtomicStampedReference** 运行机制

```java
@SneakyThrows
public static void main(String[] args) {
    AtomicStampedReference stampedReference = new AtomicStampedReference(100, 0);

    new Thread(() -> {
        Thread.sleep(50);
        stampedReference.compareAndSet(100,
                101,
                stampedReference.getStamp(),
                stampedReference.getStamp() + 1);

        stampedReference.compareAndSet(101,
                100,
                stampedReference.getStamp(),
                stampedReference.getStamp() + 1);
    }).start();


    new Thread(() -> {
        int stamp = stampedReference.getStamp();
        Thread.sleep(500);
        boolean result = stampedReference.compareAndSet(100, 101, stamp, stamp + 1);
        System.out.println(String.format("  >>> 修改 stampedReference :: %s ", result));
    }).start();
  	/**
 			*   >>> 修改 atomicInteger :: false 
 			*/
}
```



1、创建 AtomicStampedReference, 并设置初始值100, 以及版本号0

2、线程一睡眠50ms, 然后对value做出操作100->101, 101->100, 版本+1

> 为什么要睡眠50ms? 为了模拟多线程并发抢占, 让线程二先获取到版本号

3、线程二睡眠500ms, 等待线程一执行完毕, 开始将100->101, 版本号+1

**不出意外, 线程二一定会修改失败**, 虽然值相对应, 但是预期的版本号与 Pair 中的已不一致



### compareAndSet

来看下它的 compareAndSet(...) 是如何做的

```java
/**
 * 比较并设置
 *
 * @param expectedReference 预期值
 * @param newReference      期望值
 * @param expectedStamp     预期版本号
 * @param newStamp          期望版本号
 * @return									是否成功
 */
public boolean compareAndSet(V expectedReference,
                             V newReference,
                             int expectedStamp,
                             int newStamp) {
    // 获取当前 pair 引用
    AtomicStampedReference.Pair<V> current = pair;
    // 预期值对比 pair value
    return expectedReference == current.reference &&
      			// 预期版本号对比 pair stamp
            expectedStamp == current.stamp &&
      			// 期望值对比 pair value
            ((newReference == current.reference &&
              			// 期望版本号对比 pair stamp
                    newStamp == current.stamp) ||
             				// casPair
                    casPair(current, Pair.of(newReference, newStamp)));
}
```



如果 compareAndSet(...) 为 True, 必须满足上述条件表达式中3个条件

1、预期值等于 pair value

2、预期版本号等于 pair stamp

3、期望值等于 pair value 并且期望版本号等于 pair stamp

> 这是 value 和 版本号未发生变化时的场景

4、当第一个条件和第二个条件满足, 但是不满足第三个条件, 证明值和版本号发生了变化, 创建 Pair  进行 CAS 比较替换



**上述条件必须满足1、2、3或者满足1、2、4均可返回 True**



将当前 Pair 原子引用切换为新 Pair, 与 **AtomicReference 思路一致**, 将对象引用进行原子转换

```java
private boolean casPair(AtomicStampedReference.Pair<V> cmp, AtomicStampedReference.Pair<V> val) {
    return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val);
}
```



### 使用 Integer value 的坑

小伙伴应该都知道, 如果你写出下面的代码, 是不会在堆内创建对象的

```java
Integer i1 = 100;
Integer i2 = 101;
```



因为 JVM 会在常量池内存储两个对象, 而超过 -128～127 的数值则会在堆中创建对象

我们 Pair 对象的 reference 是范型, 传递的 int 类型的数值会被转型, 变为 Integer

使用 Integer 类型并且值不在 -128～127 范围内, 会出现数据比对出错的情况, 大家看一下 compareAndSet(...) 就明白了



### AtomicMarkableReference

大家看到上面的版本号两次操作必须保持不一致, 得自增或、自减或者别的操作, 相对而言也比较繁琐

Doug Lea 大师同时也为我们提供了一个省事的类库 **AtomicMarkableReference**



API 接口以及实现思路与上述的基本一致, 只不过把版本号 int 类型替换为了 boolean 类型, 其它无区别



不过, **AtomicMarkableReference 并不能完全解决 ABA 问题, 只是能够小概率预防**

