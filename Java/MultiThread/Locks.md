# Locks接口
`Lock`接口中定义了一组抽象的加锁操作。与内置加锁机制不同的是，`Lock`提供了一种无条件的、可轮询的、定时的以及可中断的锁获取操作，所有加锁和解锁的方法都是显式的。`Lock`接口的定义如下：
```Java
public interface Lock {
	void lock();
	void lockInterruptibly() throws InterruptedException;
	boolean tryLock();
	boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException;
	void unlock();
	Condition newCondition();
}
```

`ReentrantLock`实现了`Lock`接口，并提供了与`synchronized`相同的互斥性和内存可见性。获取`ReentrantLock`的语义与进入同步代码块一致，释放`RenentrantLock`的语义也同退出同步代码块一致。与`synchronized`一样，`ReentrantLock`也提供了可重入的加锁语义。但`ReentrantLock`比`synchronized`更加灵活。最典型的就是前者支持非公平和公平的锁获取机制，而后者只支持公平的锁获取机制。

以下是`Lock`接口的标准使用形式，它比内置锁更复杂，并且必须在`finally`块中释放锁。否则，如果在被保护的代码中抛出了异常，那么这个锁永远都无法释放。
```Java
Lock lock = new ReentrantLock();
…
lock.lock();
try {
	// update object state
	// catch exceptions and restore invariants if necessary
} finally {
	lock.unlock();
}
```

## `synchronized`与`Lock`的区别
1. `synchronized`使用的是`Object`对象的`wait()`、`notify()`、`notifyAll()`调度机制，而`Lock`使用的是`Condition`接口的`await()`、`signal()`、`signalAll()`完成线程之间的调度。
2. `sychronized`的作用限于同步代码块，而`Lock`需要手动给出锁的起始和结束位置。前者的执行是托管给JVM的，后者是通过代码手动控制的。
3. 前者能够自动释放锁，后者需要手动释放锁。
4. 前者只能以阻塞的方式获得锁，后者可以使用非阻塞的方式去获得锁。

## AbstractQueuedSynchronizer(AQS)
`Lock`接口的实现基本都是通过聚合一个`AbstractQueuedSynchronizer`的子类来完成线程访问控制的。

**`AQS`**是一个用于构建锁和同步器的框架，`ReentrantLock`、`Semaphore`、`CountDownLatch`、`ReentrantReadWriteLock`、`SynchronousQueue`等都是基于它实现的。

在基于`AQS`构建的同步器类中，最基本的操作包括各种形式的获取操作和释放操作。获取操作是一种依赖状态的操作，通常也会阻塞。相反，释放操作并不是一个阻塞操作，当执行释放操作时，所有在请求时被阻塞的线程都会开始执行。

`AQS`负责管理同步器类中的状态，它用一个int类型的字段 **`state`**来保存状态信息，可以通过`getState()`、`setState()`以及`CompareAndSetState()`等方法来进行操作。`state`可以表示任意状态。例如：`ReentrantLock`用它来表示所有者线程已经重复获取该锁的次数；`Semaphore`用它表示剩余的许可数量；`CountDownLatch`用它来表示计数器当前的值；`ReentrantReadWriteLock`用`state`的高16位表示读状态，也就是获取读锁的次数，低16位表示获取写锁的次数。<u>对于`AQS`来说，线程同步的关键就是对状态值`state`进行操作</u>。

`AQS`是一个FIFO的双向队列，内部通过节点`head`和`tail`记录队首和队尾元素，队列元素的类型为`Node`。

### `AQS`的实现分析

#### 同步队列
`AQS`依赖内部的同步队列（一个FIFO的双向队列）来完成同步状态的管理，当前线程获取同步状态失败时，`AQS`会将当前线程以及等待状态等信息构成一个节点（`Node`）并将其加入同步队列，同时阻塞当前线程，当同步状态释放时，会把队首节点中的线程唤醒，使其再次尝试获取同步状态。

##### `Node`节点
`Node`节点是构成同步队列的基础，`AQS`拥有首节点（`head`）和尾节点（`tail`）。`head`节点是获取同步状态成功的节点，它在释放同步状态时，会唤醒后继节点，而被唤醒的后继节点会在获取同步状态成功时将自己设置为`head`节点。

`Node`中的`thread`变量用来记录存放进`AQS`队列里面的线程。`Node`节点内部的`SHARED`用来标记该线程在共享模式阻塞，`EXCLUSIVE`用来标记线程在互斥模式阻塞。`waitStatus`记录当前节点的等待状态：
* **CANCELLED**：由于在同步队列中等待的线程等待超时或者被中断，需要从同步队列中取消等待，节点进入该状态后将不在变化
* **SIGNAL**：后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行
* **CONDITION**：节点在等待队列中，节点线程等待在`Condition`上，当其它线程调用了`Condition`的`signal()`方法后，该节点将会从等待队列移动到同步队列中
* **PROPAGATE**：表示下一次共享式同步状态获取将会无条件地被传播下去
* **0**：不处于以上任何一个状态
`prev`表示当前节点的前驱节点（当节点从队尾加入同步队列时被设置），`next`表示当前节点的后继节点。

#### 独占式同步状态的获取与释放
通过调用`AQS`的`acquire(int arg)`方法，可以以互斥的形式获取同步状态并忽略中断。`acquire(int arg)`方法的代码如下：
```Java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
这段代码主要完成了获取同步状态、构造节点、加入同步队列以及在同步队列中自旋等待的相关操作。主要逻辑是：首先调用`AQS`子类自定义的`tryAcquire(int arg)`方法获取同步状态，若获取失败，则构造同步节点（`Node.EXCLUSIVE`）并通过`addWaiter(Node node)`方法将该节点加入到同步队列的队尾，最后调用`acquireQueued(Node node, int arg)`方法使该节点以自旋的方式获取同步状态（只有前驱节点为头节点时才能够尝试获取同步状态）。如果获取不到则阻塞节点中的线程，而被阻塞线程的唤醒主要<u>依靠前驱节点的出队或阻塞线程被中断来实现</u>。

释放同步状态则是通过`release(int arg)`方法来完成的：
```Java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
这段代码执行时，会释放同步状态并唤醒后继节点（后继节点成为`head`节点，继而重新尝试获取同步状态）中的线程，具体唤醒等待线程时通过`unparkSuccessor(Node node)`方法来完成的，它调用了`LockSupport`的`unpark(Thread thread)`方法。

#### 共享式同步状态获取与释放
共享式获取同步状态与独占式获取最主要的区别在于<u>同一时刻是否能有多个线程同时获取到同步状态</u>。通过调用`AQS`的`acquireShared(int arg)`方法可以以共享的方式获取同步状态，该方法的代码如下：
```Java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean interrupted = false;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node))
                interrupted |= parkAndCheckInterrupt();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    } finally {
        if (interrupted)
            selfInterrupt();
    }
}
```
在`acquireShared(int arg)`方法中，调用的`tryAcquireShared(int arg)`返回值若不小于0，则表示获取同步状态成功。若获取失败，就进入`doAcquireShared(int arg)`方法自旋，如果当前节点的前驱为`head`节点，再次尝试获取同步状态，获取成功便退出自旋，否则继续尝试。

同步状态的释放是通过`releaseShared(int arg)`方法来完成的：
```Java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```
不同于`release(int arg)`方法，`releaseShared(int arg)`方法会唤醒后继节点并确保释放操作向后传播。

## ReentrantLock
`ReentrantLock`支持一个线程对资源重复加锁，还支持获取锁是的公平与非公平性选择。

`synchronized`关键字本是是支持重入的（隐式的），而`ReentrantLock`的重入需要显式进行，即调用`lock()`方法。

### 重入
**重入**是指任意线程在获取锁之后能够再次获取该锁而不会被锁阻塞。这一特性的实现需要解决以下两个问题：
1. 线程再次获取同一个锁。这个时候需要锁去识别线程是否为当前拥有锁的线程，如果是，则再次获取成功，完成重入。
2. 锁的最终释放。需要保证线程重复获取n次锁并释放n次锁之后，其它线程能够获取到该锁。

重入的一种实现方法是：为每个锁关联一个计数值和一个所有者线程，当计数值为0时，这个锁就被认为不被任何线程持有。当线程请求一个未被持有的锁时，JVM会记下锁的持有者并将计数值设置为1，如果同一个线程再次获取这个锁，计数值就加1，当线程退出同步代码块时，计数值减1。当计数值为0时，这个锁将被释放。

### 锁的公平性
在`ReentrantLock`的构造函数中提供了一个`fair`参数，用来指定所创建的锁是否公平，默认创建一个非公平的锁。在公平的锁上，线程将按照它们发出请求的顺序来获得锁，但在非公平的锁上，是允许“插队”的：当一个线程请求非公平的锁时，如果在发出请求的同时该锁的状态变为可用，那么这个线程将跳过队列中的所有等待线程并获得这个锁。

在激烈竞争的情况下，非公平锁的性能高于公平锁的一个原因是：<u>在恢复一个被挂起的线程与该线程真正开始运行之间存在着严重的延迟，而使用非公平锁减少了线程上下文切换的次数</u>。

## ReadWriteLock
ReadWriteLock的接口定义如下：
```Java
public interface ReadWriteLock {
	Lock readLock();
	Lock writeLock();
}
```
`ReadWriteLock`暴露了两个`Lock`对象，其中一个用于读操作，另一个用于写操作。要读取由`ReadWriteLock`保护的数据，必须首先获得读锁，当需要修改`ReadWriteLock`保护的数据时，必须首先获得写锁。

在`ReadWriteLock`实现的加锁策略中，允许多个读操作同时进行，但每次只允许一个写操作。对于频繁读取的场景，`ReadWriteLock`能够提高性能。

### ReentrantReadWriteLock
`ReentrantReadWriteLock`是`ReadWriteLock`的一个实现类，除接口方法外，它还加入了一些监控锁内部工作状态的方法。

`ReentrantReadWriteLock`的实现依赖于`AQS`，由于`AQS`内使用的是一个`int`类型的`state`变量来保存的同步状态，所以`ReentrantReadWriteLock`对`state`变量按位切割成了两部分使用：高16位表示读状态，低16位表示写状态。

## LockSupport
`LockSupport`是一个工具类，由`UnSafe`类实现，主要用来挂起和唤醒线程，它是创建锁和其它同步类的基础。`LockSupport`会为每个使用它的线程关联一个许可证（permit），在默认情况下调用`LockSupport`类的方法的线程是不持有许可证的。

如果调用`park()`方法的线程已经拿到了与`LockSupport`关联的许可证，则调用`LockSupport.park()`时会马上返回，否则调用线程会被禁止参与线程调度，也就是会被阻塞挂起。

在其它线程调用`unpark(Thread thread)`方法并将被阻塞线程作为参数时，因调用`park()`方法而被阻塞的线程会返回。如果其它线程调用了被阻塞线程的`interrupt()`方法，被阻塞线程也会返回，但不会抛出`InterruptedException`异常。

## `Contidion`接口
在某些情况下，当内置锁过于死板时，可以使用显式锁。正如`Lock`是一种广义的内置锁一样，`Condiiton`也是一种广义的内置条件队列。`Condition`接口的定义如下：
```Java
public interface Condition {
	void await() throws InterruptedException;
	boolean await(long time, TimeUnit unit) throws InterruptedException;
	long awaitNanos(long nanosTimeout) throws InterruptedException;
	void awaitUninterruptibly();
	boolean awaitUntil(Date deadlines) throws InterruptedException;
	void signal();
	void signalAll();
}
```
一个`Condition`和一个`Lock`关联在一起，就像一个条件队列和一个内置锁相关联一样。可以通过`Lock.newCondition()`方法来创建一个`Condition`。

特别注意：在`Condition`对象中，与`wait()`、`notify()`、`notifyAll()`方法对应的分别是`await()`、`signal()`和`signalAll()`。`Condition`对`Object`进行了扩展，因为它也包含`wait()`和`notify()`等方法。一定要确保使用正确的版本——`await()`和`signal()`。

## 参考文献
1. 《Java并发编程实战》及其原著.
2. 《Java并发编程的艺术》.