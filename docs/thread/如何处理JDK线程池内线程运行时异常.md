## 前言

上次写了 [【剑指 Offer】如何解决 JDK 线程池中不超过最大线程数下快速消费任务](https://mp.weixin.qq.com/s/sk5-MAtm-fmmOCMzTuznbg)

宽哥瞅了后没有被怼, 心想不枉我上周天看了一天的线程池源码啊 🤓

**场景太过温馨, 必须记录下来**

![](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20200913135614302.png)

不过还好, 后面没有像往常一样被怼 🙃️

言归正传哈, 本篇 《如何处理 JDK 线程池内线程执行异常》 文章适合哪些小伙伴阅读呢

> 工作中使用线程池却不知异常的处理流程, 以及不知如何正确处理抛出的异常

## 1. 带着问题看文章

1、线程池如何输出打印运行任务时抛出的异常?

2、线程池 execute()、submit() 处理异常是否一致?

3、都有哪些方式可以处理任务异常?

根据上述问题, 通过示例代码以及源码共同解析

> 如无特别标注, 文章阅读以 JDK 1.8 为准

## 2. 如何处理运行任务时抛出的异常

这个问题我们以 execute() 为例, 先看下源码中是如何处理的

> 如果看过前面两篇线程池文章的小伙伴对第一个任务执行流程是比较清晰的
>
> execute() -> addWorker() -> start() -> run() -> runWorker()

在这里直接放 **ThreadPoolExecutor#runWorker** 源码

```java
final void runWorker(Worker w) {
    ...
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
						...
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                  	/**
                  	 * 运行提交任务的run方法
                  	 * 如果抛出异常会被下面的catch块进行捕获
                  	 */
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x;
                    throw x;
                } catch (Error x) {
                    thrown = x;
                    throw x;
                } catch (Throwable x) {
                    thrown = x;
                    throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

**当提交到线程池的任务执行抛出异常时, 会被下方的 catch 块捕获, 向 JVM 抛出异常**

我们看下述测试代码, 逻辑比较简单, 创建一个线程池, 提交第一个任务时抛出运行时异常

> 不必关心示例代码中线程池构建参数

```java
@Test
public void executorTest() {
    ThreadPoolExecutor pool =
            new ThreadPoolExecutor(1, 3, 1000, TimeUnit.HOURS, new LinkedBlockingQueue(5));
    pool.execute(() -> {
        int i = 1/0;
    });

    pool.shutdown();
}
/**
 * Exception in thread "pool-1-thread-1" java.lang.ArithmeticException: / by zero
 * 	at cn.hsa.tps.ThreadPoolExceptionTest.lambda$executorTest$0(ThreadPoolExceptionTest.java:22)
 * 	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
 * 	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
 * 	at java.lang.Thread.run(Thread.java:748)
 */
```

抛出异常这个都可以理解, 但是线程池内部只是将异常进行 **throw 操作,** 异常信息是如何被打印的呢?

带着疑惑搜了万能的 "度娘", 最终得到答案是:

向上抛出的异常会由虚拟机 JVM 进行调用 **Thread#dispatchUncaughtException**

```java
/**
 * Dispatch an uncaught exception to the handler. This method is
 * intended to be called only by the JVM.
 */
private void dispatchUncaughtException(Throwable e) {
    getUncaughtExceptionHandler().uncaughtException(this, e);
}
```

我们查看注释也能得到相关的信息

> **向处理程序分配一个未捕获的异常, dispatchUncaughtException 方法仅能被 JVM 调取**

而这个处理未捕获异常的 **"程序"** 就是 **UncaughtExceptionHandler**

继续查看相关方法, **Thread#getUncaughtExceptionHandler**

```java
public Thread.UncaughtExceptionHandler getUncaughtExceptionHandler() {
    return uncaughtExceptionHandler != null ?
            uncaughtExceptionHandler : group;
}
```

这里会查看线程是否有未捕获处理策略, 没有则使用 **默认线程组的策略** 执行

```java
private ThreadGroup group;
public class ThreadGroup implements Thread.UncaughtExceptionHandler {...}
```

而线程组中牵扯到批量管理线程，如批量停止或挂起等概念, 这里不过多分析

获取到具体执行策略后, 我们查看下 **ThreadGroup#uncaughtException** 是如何处理的

```java
public void uncaughtException(Thread t, Throwable e) {
    if (parent != null) {
        parent.uncaughtException(t, e);
    } else {
        Thread.UncaughtExceptionHandler ueh =
                Thread.getDefaultUncaughtExceptionHandler();
        if (ueh != null) {
            ueh.uncaughtException(t, e);
        } else if (!(e instanceof ThreadDeath)) {
          	// 最终会调用到这里将一场堆栈信息进行打印
            System.err.print("Exception in thread \""
                    + t.getName() + "\" ");
            e.printStackTrace(System.err);
        }
    }
}
```

其实也就相当于 **将异常吞掉, 但是会打印出异常具体的异常信息**,到这里我们也能够明白, 线程池中抛出的异常最终落脚点在哪了

## 3. execute()、submit() 处理异常是否一致

还是上面的程序, 只不过这次将 execute() 换成了 submit()

```java
@Test
public void executorTest() {
    ThreadPoolExecutor pool =
            new ThreadPoolExecutor(1, 3, 1000, TimeUnit.HOURS, new LinkedBlockingQueue(5));
    pool.submit(() -> {
        int i = 1/0;
    });

    pool.shutdown();
}
```

明了的小伙伴相信看到这里也会知道, **这里不会进行异常信息打印**

为什么不会打印? 这个问题跟着源码我们一起看

![ThreadPoolExecutor#runWorker](https://images-machen.oss-cn-beijing.aliyuncs.com/image-20200913000652674.png)

可以看出 task 已不再是相关的 Runnable 对象, 而是 FutureTask

继续查看 FutureTask 源代码是如何执行的

```java
public void run() {
    if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                    null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
              	// 这里内部也是调用的run方法
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        ...
}
```

通过源码得知, **执行任务流程抛出的异常会被 catch 住, 所以不会将异常信息默认打印**

那如何能够知道 submit() 是否抛出了异常?

submit() 返回值是 Future 接口, 默认实现是 FutureTask, 内部有一个 get() 方法, 既可以获取返回值, 同时也可以抛出对应异常

```java
public V get() throws InterruptedException, ExecutionException {    int s = state;  	// 如果新创建或者未完成的Future会被阻塞    if (s <= COMPLETING)        s = awaitDone(false, 0L);  	// 🚩 重点    return report(s);}private V report(int s) throws ExecutionException {    Object x = outcome;  	// 正常完成返回结果    if (s == NORMAL)        return (V) x;  	// 任务被取消抛出异常    if (s >= CANCELLED)        throw new CancellationException();  	// 这里就是任务内部执行出错返回    throw new ExecutionException((Throwable) x);}
```



在任务发生异常时, 会将异常进行报错, 通过 get() 方法可以返回任务执行异常

## 4. 都有哪些方式可以处理任务异常

**1、任务内部将可能会抛出异常的步骤进行 try catch**

```java
@Testpublic void executorTest() {    ThreadPoolExecutor pool =            new ThreadPoolExecutor(1, 3, 1000, TimeUnit.HOURS, new LinkedBlockingQueue(5));    pool.execute(() -> {        try {            int i = 1/0;        } catch (Exception ex) {            ...        }    });    pool.shutdown();}
```

在 catch 块中对异常进行处理, 重试处理或者异常报警等操作

submit 同上

**2、重写线程 UncaughtExceptionHandler**

一般我们创建线程池时都会使用线程工厂, 在创建线程工厂时可以指定 UncaughtExceptionHandler 处理未捕获异常策略

```java
@Testpublic void executorTest() {    Thread.UncaughtExceptionHandler sceneHandler = new Thread.UncaughtExceptionHandler() {        @Override        public void uncaughtException(Thread t, Throwable e) {            // 仅举例, 不同场景各不相同            log.error("  >>> 线程 :: {}, 异常处理... ", t.getName(), e);        }    };    ThreadFactory threadFactory = new ThreadFactoryBuilder()            .setUncaughtExceptionHandler(sceneHandler)            .setDaemon(true).setNameFormat("根据线程池作用命名").build();    ThreadPoolExecutor pool =            new ThreadPoolExecutor(1, 3, 1000, TimeUnit.HOURS, new LinkedBlockingQueue(5), threadFactory);    pool.execute(() -> {        int i = 1 / 0;    });    pool.shutdown();}
```

示例代码也比较简单, 自定义 UncaughtExceptionHandler 处理策略, 创建线程工厂时指定自定义处理策略, 将线程工厂赋值线程池



**3、重写线程池的 afterExecute 方法**

```java
@Testpublic void executorExceptionAfterExecuteTest() {    ThreadPoolExecutor pool =            new ThreadPoolExecutor(1, 3, 1000, TimeUnit.HOURS, new LinkedBlockingQueue(5)) {                @Override                public void afterExecute(Runnable r, Throwable t) {                    // 仅举例, 不同场景各不相同                    log.error("  >>> 异常处理... ", t);                }            };    ...}
```

### 4.1 异常处理总结

以上三种方式由于第二种第三种的粒度以及处理不友好, 所以在日常工作中 **直接使用第一种任务内 try catch 即可**



## 后记

**由于作者水平有限, 希望大家能够反馈指正文章中错误不正确的地方, 感谢 🙏**
