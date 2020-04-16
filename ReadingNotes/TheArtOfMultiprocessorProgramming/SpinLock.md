# 自旋锁

任何互斥协议都会面临一个问题：当无法获得锁时，该怎么做？有两种方案：一种是继续进行尝试，这种锁称为 **自旋锁（spin lock）**，反复检测锁的这个过程被称为自旋（spining）或忙等待（busy waiting），当平均获取锁所花时间较短时，非常适合使用自旋锁。另外一种方案是挂起自己，请求操作系统将另一个线程调度到当前的处理器上，这种方式被称为 **阻塞（blocking）**，由于线程的上下文切换产生的开销很大，所以阻塞适合在获取锁所花时间较长的场景中使用。

## Test-And-Set Locks
`java.util.concurrent`包中的`AtomicBoolean`类提供了用新值替换旧值并将旧值返回的`getAndSet(boolean newValue)`方法，传统的`testAndSet()`指令就相当于是对`getAndSet(true)`的一次调用。`TASLock`类描述了一个基于`testAndSet()`指令的锁算法：
```Java
public class TASLock implements Lock {
    AtomicBoolean state = new AtomicBoolean(false);
    
    public void lock() {
        while (state.getAndSet(true)) {}
    }

    public void unlock() {
        state.set(false);
    }
}
```

为了说明`TASLock`的缺陷，考虑一种基于总线进行通信的多处理器架构：处理器和存储控制器都可以在总线上广播，但是一个时刻只能有一个处理器（或存储控制器）在总线上广播。所有的处理器（或存储控制器）都可以监听。每个处理器都有一个高速缓存（cache），当处理器读取某个地址的数据时，首先会检查该地址及其所存储的数据是否已在它的cache中。如果cache命中，就直接读取，否者需要在其它处理器的cache或内存中查找这个数据。为了在其它处理器的cache中找到需要的数据，处理器会在总线上广播这个地址，如果监听总线的处理器中有一个在自己的cache中发现了这个地址，就会广播该地址来进行响应。如果所有的处理器中都没有发现这个地址，则会去读取内存中该地址所对应的值。

在`TASLock`中，每个`getAndSet()`调用相当于总线上的一次广播，由于所有的线程都需要通过总线和内存来进行通信，所以`getAndSet()`会延迟所有的线程（包括那些没有等待这个锁的线程）。更加糟糕的是，`getAndSet()`强制其它处理器丢弃自己cache内锁的副本，<u>因此每个自旋的线程几乎每次都会遭遇cache缺失，然后必须从总线获取新的但未改变的值</u>。当持有锁的线程尝试释放锁时，也有可能会因总线被其它自旋的线程独占而被延迟。

`TTASLock`是一种改进后的算法，它没有直接调用`getAndSet()`，而是由线程反复地查询锁地状态直到锁是空闲的（即直到`get()`返回`false`），这种技术被称为：Test-Test-And-Set:
```Java
public class TTASLock implements Lock {
    AtomicBoolean state = new AtomicBoolean(fasle);

    public void lock() {
        while (true) {
            while (state.get()) {}
            if (!state.getAndSet(true)) {
                return;
            }
        }
    }

    public void unlock() {
        state.set(false);
    }
}
```

`TTASLock`的性能比`TASLock`要好，当锁被线程A持有后，线程B在第一次检查锁的状态时会发生cache缺失，从而阻塞并等待值被载入它的cache中。只要A不释放锁，B就会一直读取同一个值并且每次都命中cache，这样就不会产生总线流量，也不会降低其它线程的内存访问速度。此外，释放锁的线程也不会被正在该锁上自旋的线程后延。

## 竞争（Contention）
当多个线程同时尝试获取一个锁时，就会发生竞争。同时争抢同一个锁的线程越多，竞争就越激烈，为了减轻竞争程度，可以让获取锁的线程主动“沉默”一段时间，然后再尝试获取锁。那么线程应该“沉默”多久呢？有一个好的准则：<u>尝试获取锁的失败次数越多，竞争可能就越激烈，“沉默”的时间就应该越长</u>。

## 队列锁（Queue Locks）
基于`Test-And-Set`的锁存在两个问题：
* Cache-coherence Traffic：所有在同一个共享位置上自旋的线程在成功访问锁之后都会带来cache-coherence traffic。
* Critical Section Underutilization：所有的线程都会出现不必要的延迟，这回导致临界区资源利用不足。

可以将线程组织为一个队列来克服这些缺点。在队列中，每个线程通过检测前一个线程是否完成来判断是否轮到自己，这样一来，每个线程在不同的存储单元上自旋，降低了cache-coherence traffic。此外，每个线程不需要主动判断何时访问临界区，因为队列中的前一个线程会通知它，这提高了临界区资源的利用率。

### 基于数组的队列锁
以下是一个基于数组的简单队列锁`ALock`：
```Java
public class ALock implements Lock {
    ThreadLocal<Integer> slotIndex = new ThreadLocal<Integer>(){
        protected Integer initialValue() {
            return 0;
        }
    };
    AtomicInteger tail;
    boolean[] flag;
    int size;

    public ALock(int capacity) {
        size = capacity;
        tail = new AtomicInteger(0);
        flag = new boolean[capacity];
        flag[0] = true;
    }

    public void lock() {
        int slot = tail.getAndIncrement() % size;
        slotIndex.set(slot);
        while (!flag[slot]) {};
    }

    public void unlock() {
        int slot = slotIndex.get();
        flag[slot] = false;
        flag[(slot + 1) % size] = true;
    }
}
```
在`ALock`中，`tail`域被所有线程共享，其初始值为0。为了获得锁，每个线程原子的增加`tail`域，所得结果值称为槽，槽将作为布尔数组`flag`的索引。若`flag[slot]=true`，那么槽为`slot`的线程就有权获得锁。初始状态下，`flag[0]=true`。为了获得锁，线程不断自旋，直到它的槽在`flag`数组中变为`true`。释放锁时，线程将它的槽在`flag`数组中对应的位置设置为`false`，并将下一个槽位在`flag`数组中的对应位置设置为`true`。

### CLH队列锁
`ALock`保证了先来先服务的公平性，但空间利用率却不怎么好。它要求事先知道并发线程的最大数量`n`，并且为每一个锁都分配一个大小为`n`的数组。因此，同步`L`个不同的对象需要`O(Ln)`的空间，即使一个线程一次只访问一把锁也是这样。

`CLHLock`是另一种不同类型的队列锁，它提高了空间利用率：
```Java
public class CLHLock implements Lock {
    AtomicReference<QNode> tail;
    ThreadLocal<QNode> prev;
    ThreadLocal<QNode> node;

    public CLHLock() {
        tail = new AtomicReference<QNode>(new QNode());
        prev = new ThreadLocal<QNode>() {
            protected QNode initialValue() {
                return new QNode();
            }
        };
        node = new ThreadLocal<QNode>() {
            protected QNode initialValue() {
                return null;
            }
        };
    }

    public void lock() {
        QNode qNode = node.get();
        qNode.locked = true;
        QNode prevNode = tail.getAndSet(qNode);
        prev.set(prevNode);
        while (prevNode.locked) {}
    }

    public void unlock() {
        QNode qNode = node.get();
        qNode.locked = false;
        node.set(prev.get());
    }
}
```
类中的`QNode`对象的布尔型`locked`域中记录了对应线程的状态，若为`true`，则表示相应的线程要么已经获得锁，要么正在等待锁，若为`false`，则表示相应的线程已经释放锁。锁本身被表示成一个由`QNode`对象构成的虚拟链表。这里的链表是 **隐式**的：每个线程通过`Threadlocal`类型的`prev`变量引用其前驱，共享变量`tail`指向的是最新加入队列的节点。

若要获得锁，线程将其`QNode`的`locked`域设置为`true`，表示该线程不准备释放锁。线程在`tail`域上执行`getAndSet()`并使自己的`node`成为队列中的`tail`，与此同时，线程还会获得一个指向前驱`QNode`的引用。最后，线程会监视其前驱，在其前驱的`QNode`的`locked`域上自旋，等待锁被释放。

若要释放锁，线程将其`node`的`locked`域设为`false`，这样线程的后继就能够获得锁了。然后回收前驱的`QNode`，等待下次使用。

如果有`L`个锁，且每个线程每次最多访问一个锁，与`ALock`所需要的`O(Ln)`空间相比，`CLHLock`只需要`O(L+n)`的空间。

### MCS队列锁
`MCSLock`的锁也被表示为`QNode`对象的链表，与`CLHLock`不同的是，`MCSLock`中的链表不是隐式的：整个链表通过`QNode`对象里的`next`域连接。
```Java
public class MCSLock implements Lock {
    AtomicReference<QNode> tail;
    ThreadLocal<QNode> node;

    public MCSLock() {
        tail = new AtomicReference<QNode>(null);
        node = new ThreadLocal<QNode>() {
            protected QNode initialValue() {
                return null;
            }
        };
    }

    public void lock() {
        QNode qNode = node.get();
        QNode prev = tail.getAndSet(qNode);
        if (prev != null) {
            qNode.locked = true;
            prev.next = qNode;
            while (qNode.locked) {}
        }
    }

    public void unlock() {
        QNode qNode = node.get();
        if (qNode.next == null) {
            if (tail.compareAndSet(qNode, null)) return;
            while (qNode.next == null) {}
        }
        qNode.next.locked = false;
        node.next = null;
    }

    class QNode {
        boolean locked = false;
        QNode next = null;
    }
}
```
若要获得锁，线程将自己的`QNode`添加到链表的尾部。若队列原先不为空，则将其前驱`QNode`的`next`域设置为当前线程自己的`QNode`，然后在自己`QNode`的`locked`域上自旋等待，直到前驱将该域设为`false`为止。

释放锁时，线程会先检查`QNode`的`next`域是否为空，如果是，则表示当前没有其它线程正在争用该锁，或当前存在一个争用该锁的线程，但这个线程运行得很慢。为了区分这两种情况，令`qNode`为当前线程的`QNode`，对`tail`域调用`compareAndSet(qNode, null)`，如果调用成功，则表示没有其它线程正在试图获得锁，方法会`tail`域会设置为`null`后直接返回。否则，说明是第二种情况，方法会自旋等待，直至结束。在任何一种情况下，一旦出现了后继，就将后继节点的`locked`域设置为`false`，表明当前锁是空闲的。

`MCSLock`具有`CLHLock`的优点，空间复杂度也为`O(L+n)`。和`CLHLock`相比，`MCSLock`在释放锁时也需要自旋，指令条数也更多。