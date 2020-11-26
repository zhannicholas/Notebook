# 线程池
线程池是管理一组同构工作线程的资源池。线程池与工作队列（Work Queue）密切相关，其中工作队列中保存了所有等待执行的任务。工作者线程（Worker Thread）的任务很简单：从工作队列中取出一个任务并执行，然后返回线程池并等待下一个任务。

与“为每个任务都创建一个线程”相比，<u>使用线程池不仅可以平摊线程在创建和销毁过程中产生的巨大开销，还能提高程序的响应性</u>（当请求到达时，工作线程通常已经存在，可以节省创建线程的时间）。通过调整线程池的大小，可以创建足够多的线程以使CPU保持忙碌状态，同时还可以防止多线程相互竞争资源而使应用程序耗尽内存或失败。

## 设置线程池的大小
线程池的理想大小取决于被提交的任务的类型以及所部署系统的特性。在代码中通常不应该固定线程池的大小，而应该通过某种配置机制来提供，或者根据`Runtime.availableProcessors()`来获取可用处理器的数量之后动态计算。

如果线程池过大，那么大量的线程将在相对很少的CPU和内存资源上发生竞争，这不仅会导致更高的内存使用量，而且还可能耗尽资源。如果线程池过小，那么将导致许多空闲的处理器无法执行工作，从而降低吞吐率。

对于计算密集型任务，在拥有`N`个处理器的系统上，当线程池的大小为 **`N+1`**时，通常能实现最优的利用率（即使当计算密集型的线程偶尔由于缺页故障或者其他原因而暂停时，这个“额外”的线程也能确保CPU的时钟周期不被浪费）。对于包含I/O操作或其它阻塞操作的任务，由于线程并不会一直执行，因此线程池的规模应该更大。要正确的设置线程池的大小，必须估算出任务的等待时间与计算时间的比值。

## 线程池的创建和销毁

### 线程池的创建
我们可以使用`ThreadPoolExecutor`来创建一个线程池：
```Java
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
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
`ThreadPoolExecutor`的构造函数包含7个核心参数：
* `corePoolSize`：线程池的常驻核心线程数量。如果设置过小，可能导致频繁的创建和销毁线程，如果设置过大，又会造成系统资源的浪费，开发者应该根据实际业务场景来调整此值。
* `maximumPoolSize`：池内所允许的最大线程数量。
* `keepAliveTime`：当池内线程数量大于`corePoolSize`或当`allowCoreThreadTimeOut`设置为`true`时，空闲线程在被销毁前的最大存活时间。
* `unit`：指定`keepAliveTime`参数的单位。
* `workQueue`：线程池执行的任务队列。当线程池中所有的线程都在处理任务时，新到来的任务就会在`workQueue`中排队等待执行。
* `threadFactory`：执行器创建新线程时所使用的工厂，池中所有的线程都由它创建（通过`addWorker()`方法）。默认的线程工厂是`DefaultThreadFactory`，它为线程指定了名字和优先级。我们可以通过实现`ThreadFactory`接口来实现更多自定义操作。
* `handler`：用来执行线程池的拒绝（饱和）策略。当`workQueue`满了并且线程池不能再创建线程来执行新提交的任务时，就会对新任务执行拒绝策略。拒绝策略其实是一种限流保护机制。

此外，Java类库提供了一些灵活的线程池创建方案，可以通过`Executors`中的静态工厂方法来创建一个线程池：
* `newFixedThreadPool()`。创建一个固定容量的线程池，每提交一个任务就创建一个线程，直到达到线程池的容量，这时线程池的规模将不再变化（若某个线程因为发生了Exception而结束，线程池将补充一个新线程）。
* `newCachedThreadPool()`。创建一个可缓存的线程池，如果线程池的当前规模超过了处理需求时，就回收空闲线程，而当需求增加时，就添加新的线程。线程池的规模不做任何限制。
* `newSingleTreadExecutor()`。创建单个工作线程来执行任务，如果这个线程异常结束，则会创建另一个线程来替代。它能确保任务按照某种顺序串行执行（例如FIFO、LIFO、优先级）。
* `newScheduledTreadPool()`。创建固定容量的线程池，并且以延迟或者定时任务的方式来执行任务。

以上4个方法的底层都是通过`ThreadPoolExecutor`实现的。阿里巴巴的《Java开发手册》中为了规避资源耗尽的风险，禁止使用`Executors`类去创建线程池。这是因为`Executors`创建出来的线程池有以下弊端：
* `newFixedThreadPool()`和`newSingleThreadPool()`创建出来的线程池允许的请求队列（`workQueue`）长度为`Integer.MAX_VALUE`，可能堆积大量的请求，从而导致OOM。
* `newCachedThreadPool()`和`newScheduledThreadPool()`创建出来的线程池允许的线程的最大数量（`maximumPoolSize`）为`Inteter.MAX_VALUE`，可能会创建大量的线程，从而导致OOM。

### 线程池的销毁
可以通过调用线程池的`shutdown()`或`shutdownNow()`方法来关闭线程池。它们的原理都是通过遍历线程池中的工作线程，然后调用线程的`interrup()`方法来中断线程，所以无法响应中断的任务可能永远无法终止。但它们之间存在一定的区别：`shutdown()`方法将执行<u>平缓</u>的关闭过程：不再接受新任务，同时等待已经提交的任务执行完成（包括已提交但还未执行的任务）。`shutdownNow()`方法将执行粗暴的关闭过程：尝试取消所有运行中的任务，并且不再启动队列中尚未开始执行的任务。等所有任务都完成后，线程池就转入`Terminated(已终止)`状态。


## 线程的创建和销毁
线程池的基本大小（corePoolSize）、最大大小（maximumPoolSize）以及存活时间（keepAliveTime）等因素共同负责线程的创建和销毁。基本大小也是线程池的目标大小，即在没有任务执行时线程池的大小，并且只有在工作队列满了的情况下才会创建超出这个数量的线程。线程池的最大大小表示可以同时活动的线程数量的上限。如果某个线程的空闲时间超过了存活时间，那么将被标记为可回收的，当线程池的当前大小超过了基本大小时，这个线程将被终止。

值得注意的是：在创建`TreadPoolExecutor`的初期，线程并不会立即启动，而是等到有任务提交时才会启动，除非调用`prestartCoreThreads()`。

## Java线程池的实现原理
`ThreadPoolExecutor`通过`execute(Runnable command)`方法来执行一个线程：
```Java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```
当提交一个新任务到线程池时，线程池的处理流程如下：

1. 如果线程池中正在执行的线程数少于线程池的基本大小（`corePoolSize`），则创建一个新的线程来执行任务。否则，进入下一步。
2. 尝试向往工作队列里加入一个线程，如果工作队列满了，进入下一步。
3. 尝试创建一个新线程来执行任务。如果失败，则采用给定的饱和策略来处理这个任务。

`execute()`方法内部调用了`addWorker(Runnable firstTask, boolean core)`方法，它的参数说明如下：
* `firstTask`：线程应首先运行的任务，如果没有则可以设置为 `null`；
* `core`：判断是否可以创建线程的阀值（最大值），如果等于 `true` 则表示使用 `corePoolSize` 作为阀值，`false` 则表示使用 `maximumPoolSize` 作为阀值。

## 饱和策略
使用者可以通过调用`ThreadPoolExecutor`的`setRejectedExecutionHandler(RejectedExecutionHandler handler)`来修改其饱和策略。JDK自带了几种不同的`RejectedExecutionHandler`实现，每种实现都对应着有不同的饱和策略：
* **`AbortPolicy`**：终止策略。这是默认的策略，该策略会抛出未检查的`RejectedExecutionException`。调用者可以捕获这个异常，然后根据需求编写自己的处理代码。
* **`DiscardPolicy`**：丢弃策略。当新提交的任务无法保存到队列中等待执行时，直接丢弃掉该任务。
* **`DiscardOldestPolicy`**：丢弃最老的任务。丢弃下一个将被执行的任务，然后尝试重新提交新的任务。
* **`CallerRunsPolicy`**：由调用者执行。不丢弃任务，也不抛出异常，而是将任务退回给调用者。它不会在线程池中的某个线程中执行新提交的任务，而是在调用了`execute(Runnable command)`方法的线程（调用者线程）中继续执行该任务。


## 参考资料
1. Brian Goetz, Tim Peierls, Joshua Bloch, Joseph Bowbeer, David Holmes, and Doug Lea. <i>Java Concurrency in Practice</i>. Addison-Wesley Professional, 2006.
2. 方腾飞, 魏鹏, 程晓明. Java并发编程的艺术. 机械工业出版社, 2015.