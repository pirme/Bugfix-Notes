## 每日一言

真爱的第一个征兆，在男孩身上是 **胆怯**，在女孩身上是 **大胆**。——雨果《悲惨世界》





## Thread.yield() 是什么

通过 **java.lang.Thread** 类中的 **yield()** 方法可以实现让当前正在执行的线程让出 CPU 时间片

线程状态 **Running（运行中）** 执行后会变为 **Ready（就绪）** 状态

此时其它处于 **Ready 状态** 的线程可能获取到 CPU 时间片，也有可能是调用 **yield()** 方法的线程再次获得



## 线程状态

我们查看下线程 Thread 中 State 枚举中代表线程状态的值

```java
public enum State {
    NEW,

    RUNNABLE,

    BLOCKED,

    WAITING,
    
    TIMED_WAITING,

    TERMINATED;
}
```

### NEW

表示刚创建的线程，还没有调用 **Thread.start()** ，这种线程还没开始执行

```java
Thread thread = new Thread();
```



### RUNNABLE

当线程调用了 **Thread.start()** ，表示线程进入了可运行状态，**即表示 Running 状态 & Ready 状态**

```java
Thread thread = new Thread();
thread.start();
```

**可运行不代表运行状态**，可以理解为 Ready 就绪状态，等待获取到 CPU 时间片，才会变为 Running 运行状态



### BLOCKED

在线程执行中，没有获得 **synchronized** 块/方法 & 进入 **synchronized** 块/方法后调用 **wait()** 被唤醒/超时后重新进入 **synchronized** 块/方法  ，会进入阻塞状态

阻塞状态的线程会暂定执行，**不占用 CPU 时间片资源**，直到获取锁



进入 **BLOCKED** 状态的线程，不会感知到 **interrupt() 中断方法**

### WAITING、TIMED_WAITING

两种状态都表示等待状态，不同的是 **WAITING** 是无限期等待，需要另一个事件进行唤醒；**TIMED_WAITING** 等到一个时限点时自我唤醒



**WAITING**

```java
Object.wait();
Thread.join();
LockSupport.park();
```

如果没有 **被唤醒或等待的线程没有结束**，那么将一直等待，当前状态的线程 **不会被分配CPU资源和持有锁**



**TIMED_WAITING**

```java
Thread.sleep(long);
Object.wait(long);
Thread.join(long);
LockSupport.parkNanos(long);
LockSupport.parkUntil(long);
```





### TERMINATED

当线程执行完毕后，会进入 **TERMINATED** 终止、结束的状态



### 注意⚠️

从 **NEW** 状态执行后，线程不能再回到 **NEW** 状态，同样，处于 **TERMINATED** 的线程也不能再回到 **RUNNABLE** 状态



## 源码声明

首先来看一下源码中 **yield()** 的方法声明

```java
/**
 * A hint to the scheduler that the current thread is willing to yield
 * its current use of a processor. The scheduler is free to ignore this
 * hint.
 *
 * <p> Yield is a heuristic attempt to improve relative progression
 * between threads that would otherwise over-utilise a CPU. Its use
 * should be combined with detailed profiling and benchmarking to
 * ensure that it actually has the desired effect.
 *
 * <p> It is rarely appropriate to use this method. It may be useful
 * for debugging or testing purposes, where it may help to reproduce
 * bugs due to race conditions. It may also be useful when designing
 * concurrency control constructs such as the ones in the
 * {@link java.util.concurrent.locks} package.
 */
public static native void yield();
```



从上述描述和方法声明中可以获取以下几点信息：

1、**yield()** 是 Thread 类的静态方法，同时也是 **native 修饰的本地方法**，无返回值

2、调度程序提示 **当前线程愿意让出当前使用的处理器**，但是调度程序可能 **随意忽略此提示**

3、**正常情况下很少使用此方法**。对于调试或测试目的，它可能是有用的，在某些情况下，它可能由于 **竞争条件而有助于重现错误**

4、当设计并发控制程序时，尽量使用 **JUCL 包下的程序**



### 小结一下

注释告诉我们，**yield()** 只是向调度器发起让出 CPU 的请求，但是 **调度器可能不鸟你**

应该谨慎使用 **yield()**，如果出于测试的目的可以使用来复现错误，如果设计并发控制程序，还是要用 JUCK 包下的程序



## 小程序测试释放 CPU

下面这段小程序的执行目的是为了 **验证 yield() 方法释放线程 CPU 后可能会再次获取到 CPU**



1、创建两个线程，分别是 **麻花 & 百万**

2、每个线程 for 循环 40 次，每执行 10 次会执行 **Thread.yield()**

3、正常情况下会出现两种情况，分别是：

- 执行到符合 % 运算条件的线程 **下一个执行是自己本身**
- 执行到符合 % 运算条件的线程 **下一个执行是另外一个线程**



大家可以跑一下程序自己试试，结合实际情况理解 **yield()** 语义

```java
public class YieldTest {
    public static void main(String[] args) {
        int max = 40;
        Runnable runnable = () -> {
            for (int i = 0; i <= max; i++) {
                System.out.println(
                        String.format("当前线程 :: %s, 执行进度 :: %d ", Thread.currentThread().getName(), i));
                if (i % 10 == 0) Thread.yield();
            }
        };
        new Thread(runnable, "百万").start();
        new Thread(runnable, "麻花").start();
    }
}
```



## 面试题：yield() 是否释放 synchronized 锁

以一道面试题作为切入口：

多个线程竞争 **synchronized 同步块** 执行权时，获得执行权线程在同步块内部执行 **yield()**，是否会释放锁？

**答：不会释放锁**



我们还是以实际的小程序来演绎这个问题的过程，先来说下执行过程

1、创建两个线程，分别是 **麻花 & 百万**

2、“麻花” 启动首先获取锁，睡眠 50 毫秒等待 “百万” 竞争锁

3、“百万” 在 “麻花” 启动后，等待 10ms 后执行启动步骤，“百万” 竞争失败进入 **阻塞状态**

4、“麻花” 睡眠 50 ms 后，执行 **Thread.yield()** 释放 CPU

5、“麻花” 再次获取 CPU 时间片打印日志后释放锁，“百万” 获得锁



```java
public class YieldTest {
    
    @SneakyThrows
    public static void main(String[] args) {
        Object lock = new Object();

        new Thread(() -> {
            synchronized (lock) {
                System.out.println(
                        String.format("当前线程 :: %s, 获取锁并准备执行 Thread.yield() ", Thread.currentThread().getName())
                );
                try {
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    // ignore
                }
                Thread.yield();
                System.out.println(
                        String.format("当前线程 :: %s, 执行 Thread.yield() 后", Thread.currentThread().getName())
                );
            }
        }, "麻花").start();
        Thread.sleep(10);

        new Thread(() -> {
            System.out.println(
                    String.format("当前线程 :: %s, 开始竞争锁 ", Thread.currentThread().getName())
            );
            synchronized (lock) {
                System.out.println(
                        String.format("当前线程 :: %s, 获取锁 ", Thread.currentThread().getName())
                );
            }
        }, "百万").start();
    }
}
```



**查看执行结果与上文答案相呼应**，在 “麻花” 执行 **Thread.yield()** 后，“百万” 并没有直接获取锁，而是在 **“麻花” 执行完同步块后获取到的锁**

```java
/** * 当前线程 :: 麻花, 获取锁并准备执行 Thread.yield()  * 当前线程 :: 百万, 开始竞争锁  * 当前线程 :: 麻花, 执行 Thread.yield() 后 * 当前线程 :: 百万, 获取锁  */
```
