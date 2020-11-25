# 线程
进程是资源分配的基本单元，而线程是程序执行的基本单元。一个进程可以包含多个线程，多个线程之间共享进程的资源。和进程相比，线程更加轻量化，所以线程又叫*轻量级进程*。

## 创建一个新线程的方法
1. 继承`Thread`类，重写`run()`方法。
2. 实现`Runnable`接口。
3. 实现`Callable`接口。

## 线程的状态
Java线程在其生命周期中可能处于6种状态（这6中状态指的线程在JVM中的状态，并不是操作系统中线程的状态），在一个特定的时刻，线程只能处于其中的一个状态。

| 状态          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| NEW           | A thread that has not yet started is in this state.          |
| RUNNABLE      | A thread executing in the Java virtual machine is in this state.             |
| BLOCKED       | A thread that is blocked waiting for a monitor lock is in this state.  |
| WAITING       | A thread that is waiting indefinitely for another thread to perform a particular action is in this state. |
| TIMED_WAITING | A thread that is waiting for another thread to perform an action for up to a specified waiting time is in this state. |
| TERMINATED    | A thread that has exited is in this state.                           |

```java
public enum State {
    /**
        * Thread state for a thread which has not yet started.
        */
    NEW,

    /**
        * Thread state for a runnable thread.  A thread in the runnable
        * state is executing in the Java virtual machine but it may
        * be waiting for other resources from the operating system
        * such as processor.
        */
    RUNNABLE,

    /**
        * Thread state for a thread blocked waiting for a monitor lock.
        * A thread in the blocked state is waiting for a monitor lock
        * to enter a synchronized block/method or
        * reenter a synchronized block/method after calling
        * {@link Object#wait() Object.wait}.
        */
    BLOCKED,

    /**
        * Thread state for a waiting thread.
        * A thread is in the waiting state due to calling one of the
        * following methods:
        * <ul>
        *   <li>{@link Object#wait() Object.wait} with no timeout</li>
        *   <li>{@link #join() Thread.join} with no timeout</li>
        *   <li>{@link LockSupport#park() LockSupport.park}</li>
        * </ul>
        *
        * <p>A thread in the waiting state is waiting for another thread to
        * perform a particular action.
        *
        * For example, a thread that has called {@code Object.wait()}
        * on an object is waiting for another thread to call
        * {@code Object.notify()} or {@code Object.notifyAll()} on
        * that object. A thread that has called {@code Thread.join()}
        * is waiting for a specified thread to terminate.
        */
    WAITING,

    /**
        * Thread state for a waiting thread with a specified waiting time.
        * A thread is in the timed waiting state due to calling one of
        * the following methods with a specified positive waiting time:
        * <ul>
        *   <li>{@link #sleep Thread.sleep}</li>
        *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
        *   <li>{@link #join(long) Thread.join} with timeout</li>
        *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
        *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
        * </ul>
        */
    TIMED_WAITING,

    /**
        * Thread state for a terminated thread.
        * The thread has completed execution.
        */
    TERMINATED;
}

```

以下是线程之间的状态迁移图：

![Java thread state transfering](images/thread-state-transfering-diagram.png)


## 实现多线程同步的方法
Java主要提供了3中实现同步机制的方法：

1. 使用`synchronized`关键字。见[`Synchronized`关键字](./Synchronized.md)。
2. 使用`wait()`与`notify()`方法。
3. 使用`Lock`。见[Locks](./Locks.md)。

## Java中的原子操作
在Java中可以通过锁和循环CAS的方式实现原子操作。

JVM中的CAS操作是利用处理器提供的 **`COMPXCHG`**指令实现的。自旋CAS实现的基本思路是循环进行CAS操作直到成功为止。但CAS存在三个问题：
* **ABA问题**。可以使用版本号来解决。
* **循环时间长开销大**。这一般出现在自旋CAS长时间不成功的情况下。
* **只能保证一个共享变量的原子操作**。对于多个共享变量的原子操作，一般采用锁来解决，也可以将多个共享变量封装进一个对象，然后使用`AtomicReference`类来解决。

JVM内部实现了很多锁机制，有意思的是除了偏向锁，JVM实现锁的方式都用了循环CAS，即当一个线程进入同步代码块时使用循环CAS获取锁，离开同步代码块时使用循环CAS释放锁。

## 线程优先级
`Thread`类定义了三个和线程优先级的属性：
```java
/**
* The minimum priority that a thread can have.
*/
public static final int MIN_PRIORITY = 1;

/**
* The default priority that is assigned to a thread.
*/
public static final int NORM_PRIORITY = 5;

/**
* The maximum priority that a thread can have.
*/
public static final int MAX_PRIORITY = 10;
```

优先级越高的线程获得CPU时间片的概率越大，但这并不是意味着优先级越高的线程一定越先执行。Java中的线程的优先级可以通过方法`setPriority(int newPriority)`来设置，有效的优先级范围为：`1-10`。

## 常用方法

### `start()`与`run()`
从`Thread`类的源码来看，`start()`为`Thread`自身的一个方法，通过`synchronized`来保证线程安全，并且只能调用一次：
```java
/**
* Causes this thread to begin execution; the Java Virtual Machine
* calls the {@code run} method of this thread.
* <p>
* The result is that two threads are running concurrently: the
* current thread (which returns from the call to the
* {@code start} method) and the other thread (which executes its
* {@code run} method).
* <p>
* It is never legal to start a thread more than once.
* In particular, a thread may not be restarted once it has completed
* execution.
*
* @throws     IllegalThreadStateException  if the thread was already started.
* @see        #run()
* @see        #stop()
*/
public synchronized void start() {
    /**
        * This method is not invoked for the main method thread or "system"
        * group threads created/set up by the VM. Any new functionality added
        * to this method in the future may have to also be added to the VM.
        *
        * A zero status value corresponds to state "NEW".
        */
    if (threadStatus != 0)
        throw new IllegalThreadStateException();

    /* Notify the group that this thread is about to be started
        * so that it can be added to the group's list of threads
        * and the group's unstarted count can be decremented. */
    group.add(this);

    boolean started = false;
    try {
        start0();
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {
            /* do nothing. If start0 threw a Throwable then
                it will be passed up the call stack */
        }
    }
}
```

而`run()`是`Runnable`接口声明的一个方法，`Thread`只是重写了它：
```java
/**
* If this thread was constructed using a separate
* {@code Runnable} run object, then that
* {@code Runnable} object's {@code run} method is called;
* otherwise, this method does nothing and returns.
* <p>
* Subclasses of {@code Thread} should override this method.
*
* @see     #start()
* @see     #stop()
* @see     #Thread(ThreadGroup, Runnable, String)
*/
@Override
public void run() {
    if (target != null) {
        target.run();
    }
}
```

另外，`start()`方法会影响线程的状态（由NEW变为RUNNABLE），只能调用一次，而`run()`只是一个普通方法，并且可以被多次调用。

### `wait()`与`notify()`
若`synchronized`关键字修饰的某个共享资源R的锁已经被线程T1获得，其它需要资源R的锁才能运行的线程就会被阻塞直至T1释放R上的锁。一般情况下，这个锁是在同步代码块执行完才被释放的。

但是在同步代码块执行期间，已持有锁的线程T1可以调用资源R的`wait()`方法释放锁，然后进入等待状态，当前线程被挂起。锁的持有者可以调用`notify()`方法随机唤醒一个处于等待状态的线程，或`notifyAll()`方法去唤醒所有处于等待状态的线程。

只有持有与共享资源R相关联的`monitor`的锁的线程才应该去调用`notify()`或`notifyAll()`方法。一个线程可以通过3种方式获得与资源R相关联的`monitor`的锁，这三种方式刚好对应于同步代码块的三种形式。因为只有持有`monitor`的锁只有，才能进入同步代码块。

### `sleep()`与`wait()`
`sleep()`与`wait()`都可以暂停线程执行，但它们还有一些区别：

1. 原理不同。`sleep()`是`Thread`类的静态方法，是线程自己用来控制自身执行的，它可以使线程自己暂停一段时间，把执行的机会让给其它线程，等睡眠时间一到，便会自动“醒来”。而`wait()`是`Object`类的静态方法，用于线程间的通信，这个方法会使当前持有对象锁的线程释放锁并进入等待状态，直至其它线程调用对象的`notify()`或`notifyAll()`方法才会醒来。`wait()`方法也支持设置超时时间。
2. 对锁的处理机制不同。调用`sleep()`方法只是让当前线程暂停一段时间，不涉及线程间通信，也不会释放锁。而调用`wait()`方法会释放锁。
3. 使用区域不同。`sleep()`方法可以在任何区域使用。而由于调用`wait()`方法前必须先获得对象锁，因此只能在同步代码块内使用。
4. 对异常的处理方式不同。调用`sleep()`方法时必须处理`InterruptedException`异常。而调用`wait()`方法时不用关心异常处理。

### `sleep()`与`yield()`
`sleep()`方法与`yield()`方法都属于`Thread`类，它们的区别主要体现在：

1. 前者被调用后会立马进入阻塞状态，在一段时间内不会再执行。而后者只是使当前线程重新回到`RUNNABLE`状态，因此可能马上又被执行。
2. 前者在方法声明上抛出了`InterruptedException`异常，而后者没有声明任何异常。

### `join()`
JDK中对`join()`方法的解释为：Waits for this thread to die.

实际上，当线程A调用了线程B的`join()`方法后，线程A会让出CPU的执行权给线程B。直到线程B执行完成或者过了超时时间，线程A才会继续执行。

## 参考资料
1. Brian Goetz, Tim Peierls, Joshua Bloch, Joseph Bowbeer, David Holmes, and Doug Lea. <i>Java Concurrency in Practice</i>. Addison-Wesley Professional, 2006.
2. 方腾飞, 魏鹏, 程晓明. Java并发编程的艺术. 机械工业出版社, 2015.
