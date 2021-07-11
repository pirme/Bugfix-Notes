## 前言

使用线程池时会忽略死锁问题, 但是只要代码写的"六"没啥是不可能的

文章代码及部分理念引自 [线程池使用不当也会死锁](https://www.cnblogs.com/caoshenglu/p/9461567.html "线程池使用不当也会死锁")

## 什么是死锁

死锁是指两个或两个以上的进程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去

此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程

> 死锁定义来源: [死锁-百度百科](https://baike.baidu.com/item/%E6%AD%BB%E9%94%81/2196938?fr=aladdin "死锁")

简化来说就是: **一组相互竞争资源的线程因为互相等待，导致"永久"阻塞的现象**

我们通过一个 Demo 简单理解下

```java
public class DeadlockTest {

    public static void main(String[] args) throws InterruptedException {
        Object o1 = new Object();
        Object o2 = new Object();
        new Thread(() -> {
            synchronized (o1) {
                try {
                    System.out.println(String.format("  >>> %s 获取 o1 锁 ", Thread.currentThread().getName()));
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (o2) {
                    System.out.println(String.format("  >>> %s 获取 o2 锁 ", Thread.currentThread().getName()));
                }
            }
        }, "线程一").start();

        Thread.sleep(10);

        new Thread(() -> {
            synchronized (o2) {
                try {
                    System.out.println(String.format("  >>> %s 获取 o2 锁 ", Thread.currentThread().getName()));
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (o1) {
                    System.out.println(String.format("  >>> %s 获取 o1 锁 ", Thread.currentThread().getName()));
                }
            }
        }, "线程二").start();
    }
    /**
     *   >>> 线程一 获取 o1 锁 
     *   >>> 线程二 获取 o2 锁 
     */
}
```



在程序中, 线程一首先获取 o1 对象锁, 其后睡眠 50ms, 醒后获取 o2 对象锁

线程二首先获取 o2 对象锁, 其后睡眠 50ms, 醒后获取 o1 对象锁

这样就很微妙的达到了死锁的产生, 这里不再讲述死锁相关的细节

## 线程池死锁产生的条件

通过代码来模拟线程池产生死锁的前因后果, 有兴趣的小伙伴可以把代码拉到本地跑一下

这段代码表现的场景如下:

1、创建一个 JDK 线程池, 并添加一个线程三秒后打印线程池的信息

2、循环调取 innerFutureAndOutFuture 方法

3、innerFutureAndOutFuture 运行过程为提交十 Callable 运行

4、但是在提交 Callable 内部另外依赖了 Callable 任务, 并需要获取执行结果

**理想情况是 while 循环会不断执行, 并自增 loop**

> **建议电脑观看代码, 或粘贴到编辑器中运行**

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

public class HungryDeadLockTest {

    private static ThreadPoolExecutor executor;

    public static void main(String[] args) throws InterruptedException, ExecutionException {
        TimeUnit unit = TimeUnit.HOURS;
        BlockingQueue workQueue = new LinkedBlockingQueue();
        executor = new ThreadPoolExecutor(5, 5, 1000, unit, workQueue);

        new Thread(() -> {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(executor);
        }).start();

        int loop = 0;
        while (true) {
            System.out.println("loop start. loop = " + (loop));
            innerFutureAndOutFuture();
            System.out.println("loop end. loop = " + (loop++));
            Thread.sleep(10);
        }
    }

    public static void innerFutureAndOutFuture() throws ExecutionException, InterruptedException {
        Callable<String> innerCallable = new Callable<String>() {
            @Override
            public String call() throws Exception {
                Thread.sleep(100);
                return "inner callable";
            }
        };

        Callable<String> outerCallable = new Callable<String>() {
            @Override
            public String call() throws Exception {
                Thread.sleep(10);
                Future<String> innerFuture = executor.submit(innerCallable);
                String innerResult = innerFuture.get();
                Thread.sleep(10);
                return "outer callable. inner result = " + innerResult;
            }
        };

        List<Future<String>> futures = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            System.out.println("submit : " + i);
            Future<String> outerFuture = executor.submit(outerCallable);
            futures.add(outerFuture);
        }
        for (int i = 0; i < 10; i++) {
            String outerResult = futures.get(i).get();
            System.out.println(outerResult + ":" + i);
        }
    }
}
```



查看上述代码, 我们先来梳理下线程池提交任务后的状态

我们先来线程池三个核心参数

- 核心线程数: 5
- 最大线程数: 5
- 阻塞队列: 无界队列

1、for 循环中添加执行 outerCallable, 添加十个任务

> 这个时候, 核心线程数五个任务已经跑满, 剩余的五个任务会添加到无界队列中

2、而在 outerCallable 中会创建 innerCallable 任务, 提交同一个线程池

> 因为 innerCallable 是在 outerCallable 的 call 方法中执行的, 相当于也会将十个任务在线程池中执行

3、因为在 outerCallable 任务执行前睡眠了 10 ms, 能够保证 outerCallable 全部得到执行

![图1 运行流程](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20200918135858502.png)

我们先来看一下程序的运行结果

```java
    /**
     * loop start. loop = 0
     * submit : 0
     * submit : 1
     * submit : 2
     * submit : 3
     * submit : 4
     * submit : 5
     * submit : 6
     * submit : 7
     * submit : 8
     * submit : 9
     * java.util.concurrent.ThreadPoolExecutor@5b6dc686[Running, pool size = 5, active threads = 5, queued tasks = 10, completed tasks = 0]
     */
```



结合运行结果和运行流程图, 我们来分析下, 并提出一些问题

**1、outerCallable、innerCallable 各自提交了十次, 为什么 queued tasks 里只有十个任务**

> 想一下哈: 我们核心线程是五个, 根据代码看会提交二十个任务, 理论上不应该在阻塞队列中有十五个任务么?

我们在 for 循环中执行了十次提交 outerCallable, outerCallable 内部执行了 10ms 的睡眠

这时线程池的状态为: 核心线程跑满五个, 阻塞队列任务五个, 此时五个核心线程运行的 outerCallable 从 10 ms 睡眠醒过来

这个时候会通过占有五个核心线程的 outerCallable 执行提交任务 innerCallable

因为只有五个核心线程, 所以阻塞队列中的十个任务是 **五个 outerCallable 和五个 innerCallable**

**2、为什么会导致线程池死锁**

通过上面的讲解, 答案已经出来了

五个核心线程运行的 outerCallable 在等待在阻塞队列中的 innerCallable 返回结果

达到了死锁的产生标准, 资源互相等待执行



## 如何预防线程池死锁

**核心点在于: 不要在线程池里等待另一个在池里执行的任务**

- 不要在线程池中的同步方法执行异步任务
- 不要在异步方法中阻塞等待异步方法

当然, 上面代码死锁的问题, 通过增大线程池线程数也能解决

可以把核心线程数改为 40, 最大线程数改为 50, 看看还能不能死锁

这种解决思路固然能解决, 但是最好还是不要有

因为线程池的参数设置要合理化, 不能为了解决某类问题而设置

## 文末总结

这里以一个小例子描述了线程池可能发生死锁一种可能

希望大家能通过上述的描述, 能够在项目中避免线程池死锁的可能
