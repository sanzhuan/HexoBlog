title: AbstractQueuedSynchronizer源码分析
date: 2018-01-20 22:21:43
tags: [Java]
categories: Java
---
## 背景和简介
AbstractQueuedSynchronizer简称AQS是一个抽象同步框架，可以用来实现一个依赖状态的同步器。
JDK1.5中提供的java.util.concurrent包中的大多数的同步器(Synchronizer)如Lock, Semaphore, Latch, Barrier等，这些类之间大多可以互相实现，如使用Lock实现一个Semaphore或者反过来，但是它们都是基于java.util.concurrent.locks.AbstractQueuedSynchronizer这个类的框架实现的，理解了这个稍微复杂抽象的类再去理解其他的同步器就很轻松了。

等待条件发生的操作是开发中常见的需求，例如实现一个生产者消费者，当队列为空时，消费者获取消息可以立即返回并不断轮询，另外一种方式是阻塞直到有新的消息，后者对调用方更加友好一些。

## 为什么需要AQS
那么为什么需要一个AQS框架，而不是用已有的监视器锁来实现这些同步器呢？
上面也提到，这些同步器间可以互相实现，但是它们间并没有一个比其他都更底层更抽象，也就是没有谁更令人信服。
下面实现一个简单的Semaphore

```
private static class MySemaphore {
        private Lock lock = new ReentrantLock();
        private Condition available = lock.newCondition();
        private int permit;
        public MySemaphore(int permit) {
            this.permit = permit;
        }
        public void acquire() {
            lock.lock();
            try {
                while (permit <= 0) {
                    try {
                        available.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                permit--;
            } finally {
                lock.unlock();
            }
        }
        public void release() {
            lock.lock();
            try {
                permit++;
                if (permit > 0) {
                    available.signalAll();
                }
            } finally {
                lock.unlock();
            }
        }
    }
    ```
    
另外基于synchronized关键字的监视器锁模式在性能上的优化更针对于几乎只有一个线程、无竞争的加锁的情景，减少加锁内存开销和锁获取时间延迟，如偏向锁、轻量级锁等优化。而上面的同步器在大多数情况下都是在多线程多处理器的情况下使用的，所以并不适用。相对于内存开销和获取延迟，AQS的性能目标更针对于扩张性(scalability)，也就是说无论有多少线程尝试使用同步器，需要同步的开销都应该是常量级的。当然也要考虑其他的开销不能太大，例如spinlock的锁获取时间比blocking lock更小，但是会消耗更多的CPU和更多的内存争用。

## 设计与实现
现在考虑设计这样一个同步框架需要哪些基本的功能和具备哪些元素。

同步框架提供两个基本类型的操作acquire和release，acquire表示根据同步器状态请求获取，获取不成功则需要排队等待，获取成功并且完成操作后需要进行release释放状态让给其他线程。
另外还有非阻塞获取、设置获取等待超时时间、可中断的获取等需求场景、排他和共享模式等。

要完成这些，可以抽象出一些基本功能。
首先，需要保存状态提供给子类去表示具体的含义，并且提供查询、设置、原子修改的操作。
其次，要能够阻塞、唤醒线程。
然后，还需要有一个容器来存放等待的线程。

这样获取和释放两个操作就可以用如下伪代码表示
假设AQS抽象出tryAcquire(arg)和tryRelease(arg)两个方法给子类去实现表示具体的获取和释放的逻辑。
```
Acquire:
     while (!tryAcquire(arg)) {
        enqueue thread if it is not already queued;
        possibly block current thread;
     }
     dequeue current thread if it was queued
Release:
     if (tryRelease(arg))
        unblock the first queued thread;
        ```
### 初步设计
AQS内部通过一个int类型的state字段表示同步状态，状态的具体含义可以子类来定义，例如ReentrantLock中用state表示线程重入的次数，Semaphore表示可用的许可的数量等。使用int是由于int能够应对大部分的场景，而long在很多平台需要使用额外锁来保证一致性的读取。

类似模板模式，AQS暴露给子类一些方法实现(如tryAcquire，tryRelease), 获取操作通常是依赖state状态的，当状态允许时获取成功否则加入等待队列一直等待到允许的状态发生时重新获取，例如，Semaphore中当state(即许可数量)小于等于0时，获取不能成功。释放操作通过会修改状态并判断是否能让其他等待的线程能够重新获取，例如ReentrantLock中释放会减少重入次数并检查重入次数是否达到了0，达到0说明已经解锁，这时会通知其他等待线程重新获取。所以AQS的acquire和release会通过组合应用到不同的同步器实现中实现不同的语义，如Reentrant.lock, Semaphore.acquire, CountDownLatch.await, FutureTask.get等。
另外AQS还提供其他功能，如非阻塞的获取tryAcquire, 带有超时时间的获取，可以中断的获取等。
AQS可以根据具体的场景提供exclusive模式和shared模式，在exclusive模式下，同一时刻最多只能有一个线程能够处于成功获取的状态，排他锁是一个exclusive模式的例子，shared模式则可以多个线程一起获取成功，如多个许可的Semaphore。
另外在java.util.concurrent包中还定义了Condition接口，用来提供监视器风格的等待通知操作，可以替换Object中基于synchronized、监视器锁的wait和notify机制。

实现具体的同步器时，需要将实现类作为具体同步器的内部类，然后调用实现类的acquire、acquireShared、release等方法。

### 更具体的设计
同步状态synchronization管理
state字段可以通过volatile修饰，这样get和set方法具有了voaltile的相关语义，通过可以通过AtomicInteger或Unsafe类来实现原子CAS更新操作，基于减少indirection的考虑，JDK中一般都是使用Unsafe类。

阻塞、唤醒线程
除了基于内置的监视器的synchronizer机制外，唯一可用的似乎是已经Deprecated的Thread.stop、Thread.suspend、Thread.resume, 而这几个个方法有严重的安全问题比如可能造成死锁。JDK1.5增加了一个LockSupport类来解决这个问题, LockSupport给每个线程绑定一个类似Semaphore的permit许可数量，不过permit最大为1，初始时permit为0。park()操作阻塞一直到permit为1并且将permit减1。unpark则进行加1，不过unpark并不会累加permit。
而且如果先调用unpark()的话，之后调用park()会立即返回。
park()返回的几个情况是

之前有其他线程调用剩余的unpark()或在park()之后其他调用了unpark()
其他线程中断了目标线程
调用虚假返回（类似Object.wait()的伪通知, 所以调用返回时需要检查是否等待条件已经达成
hotspot的LockSupport最终还是使用了Unsafe的功能
```
public static void park() {
    unsafe.park(false, 0L);
}
public static void unpark(Thread thread) {
    if (thread != null)
        unsafe.unpark(thread);
}
```
更具体的实现上，在openjdk中unsafe.cpp是Unsafe的入口, 最终会使用各自平台的相关库来实现。

线程等待队列
AQS的核心是一个线程等待队列，采用的是一个先进先出FIFO队列。用来实现一个非阻塞的同步器队列有主要有两个选择Mellor-Crummey and Scott (MCS) locks和Craig, Landin, and Hagersten (CLH) locks的变种。CLH锁更适合处理取消和超时，所以AQS基于CLH进行修改作为线程等待队列。
CLH队列使用pred引用前节点形成一个队列，入队enqueue和出队dequeue操作都可以通过原子操作完成。
clh
初始化时，head和tail都会指向一个dummy节点
CLH队列的入队一个新的Node可以用伪代码来表示
```
do {
    node.pred = tail;
} while(!tail.compareAndSet(pred, node));
```
每个节点的释放状态由前置节点存放，所以spinlock的自旋可以表示为
```
while(node.pred.status != RELEASED) {
    // spin
}
```
出队可以通过更新head来完成，通过将head设置为刚刚获得锁的节点
head = node
CLH队列有很多优点，包括入队和出队非常快、无锁、非阻塞，并且是否有线程等待只需要判断head和tail是否相等。
CLH中并没有只有向后的指针引用后继节点，每个节点只需要修改自己的状态就能通知后继节点。但是在AQS这样的阻塞同步器中，需要主动唤醒(unpark)后继节点。所以在AQS中增加了next引用引用后继节点，但是并没有合适的方法使用CAS进行无锁插入双向链表的方法，所以节点的插入并不是原子完成的，需要在CAS替换tail完成后调用pred.next=node。
```
while(true) {
    t = tail;
    node.pred = t;
    if (tail.compareAndSet(t, node)) {
        t.next = node;
        break;
    }
}
```
所以next节点为null并不能表示这个节点是尾节点，next引用只能当做是一个优化路径，当next节点不存在或是被撤销时，需要从tail开始使用pred进行反向遍历来进行精确的check。

另外一个改动是使用Node中的status状态来控制阻塞而不是像CLH中控制自旋，但是在AQS中这个状态并不能表示这个线程能从acquire操作中返回，在AQS中排队的线程只有在通过了子类实现的tryAcquire方法后才能从acquire中返回，所以这个阻塞状态只能说明这个活跃线程队列的头结点的时候可以去调用tryAcquire方法，如果失败了还需要重新阻塞。

所以，修改后的AQS中的CLH变种实现一个独占、不可中断的获取操作可以表示为
```
if (!tryAcquire(arg)) {
    node = create and enqueue new node
    pred = node's effective predecessor
    while (pred is not head node || !tryAcquire(arg)) {
        if (pred's signal bit is set) {
            park()
        } else {
            compareAndSet pred's signal bit to true
        } 
        pred = node's effective predecessor
    }
}
```
释放操作可以表示为
```
if (tryRelease(arg) && head node's signal bit is set) {
    compareAndSet head's signal bit to false
    unpark head's successor, if one exist
}
```

### ConditionObject
AQS还提供了一个ConditionObject类来表示监视器风格的等待通知操作，主要用在Lock中，和传统的监视器的区别在于一个Lock可以创建多个Condition。ConditionObject使用相同的内部队列节点，但是维护在一个单独的条件队列中，当收到signal操作的时候将条件队列的节点转移到等待队列。

## 代码实现
### AQS继承结构
AQS继承于AbstractOwnableSynchronizer, AbstractOwnableSynchronizer用于表示在排他模式下存储获取当前持有线程。目前在ReentrantLock、ReentrantReadWriteLock、ThreadPoolExecutor.Worker中有用到。
```
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer {
}
```
AbstractOwnableSynchronizer中的exclusiveOwnerThread没有使用volatile修饰，提供的get、set方法也没有使用同步或Unsafe等方式获取字段。
这个基于几个前提和使用条件
首先，exclusive模式下调用setExclusiveOwnerThread设置为自己的线程和之后将其清空为null肯定是同一个线程，另外调用getExclusiveOwnerThread方法的用途通常为判断自己是不是当前同步器持有线程，假设t1正在持有同步器，t2调用getExclusiveOwnerThread方法得到的一定不是自己，因为由于线程内的happen-before关系，如果设置过t2，肯定是在t2线程内部，对自己是可见的。
```
/**
 * The current owner of exclusive mode synchronization.
 */
private transient Thread exclusiveOwnerThread;
/**
 * Sets the thread that currently owns exclusive access. A
 * <tt>null</tt> argument indicates that no thread owns access.
 * This method does not otherwise impose any synchronization or
 * <tt>volatile</tt> field accesses.
 */
protected final void setExclusiveOwnerThread(Thread t) {
    exclusiveOwnerThread = t;
}
/**
 * Returns the thread last set by
 * <tt>setExclusiveOwnerThread</tt>, or <tt>null</tt> if never
 * set.  This method does not otherwise impose any synchronization
 * or <tt>volatile</tt> field accesses.
 * @return the owner thread
 */
protected final Thread getExclusiveOwnerThread() {
    return exclusiveOwnerThread;
}
```
### AQS大体内部结构和方法
```
class AbstractQueuedSynchronizer {
    static final class Node{}
    private transient volatile Node head;
    private transient volatile Node tail;
    private volatile int state;
    protected final int getState() {
        return state;
    }
    protected final void setState(int newState) {
        state = newState;
    }
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
    private Node enq()
    private void setHead(Node node)
    private void unparkSuccessor(Node node)
    private void doReleaseShared()
    final boolean acquireQueued(final Node node, int arg) 
    private void doAcquireInterruptibly(int arg)
    private boolean doAcquireNanos(int arg, long nanosTimeout)
    private void doAcquireShared(int arg) 
    private void doAcquireSharedInterruptibly(int arg)
    private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
    // 需要子类去实现的方法
    protected boolean tryAcquire(int arg)
    protected boolean tryRelease(int arg) 
    protected int tryAcquireShared(int arg) 
    protected boolean tryReleaseShared(int arg)
    // 对外提供的public方法
    public final void acquire()
    public final void acquireInterruptibly(int arg)()
    public final boolean tryAcquireNanos()
    public final boolean release(int arg)()
    public final void acquireShared(int arg)()
    public final void acquireSharedInterruptibly(int arg)()
    public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)()
    public final boolean releaseShared(int arg) ()
    ...
}
```
### Node结构
Node表示一个等待线程的节点
如上面设计部分讲到，AQS中的等待队列节点使用pred节点控制等待状态，并且具有pred和next指针。

```
class Node {
        volatile int waitStatus;
        volatile Node prev;
        volatile Node next;
        volatile Thread thread;
        Node nextWaiter;
        ...
}
```
其中waitStatus有如下几个值,
CANCELED表示线程等待已经取消，是唯一一个大于0的状态。
SINALG表示需要唤醒next节点
CONDITION表明线程正在等待一个条件
PROPAGATE用于acquireShared中向后传播

```
/** waitStatus value to indicate thread has cancelled */
static final int CANCELLED =  1;
/** waitStatus value to indicate successor's thread needs unparking */
static final int SIGNAL    = -1;
/** waitStatus value to indicate thread is waiting on condition */
static final int CONDITION = -2;
/**
 * waitStatus value to indicate the next acquireShared should
 * unconditionally propagate
 */
static final int PROPAGATE = -3;
```
### state状态相关操作
state用volatile修饰, 原子操作使用CAS
```
private volatile int state;
protected final int getState() {
    return state;
}
protected final void setState(int newState) {
    state = newState;
}
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```
acquire
独占模式的acquire
首先尝试一次tryAcquire, 如果不成功则添加一个Node节点到等待队列

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
addWaiter先判断tail如果不为空则进行一次快速的插入，否则使用enq进行可能包括初始化的入队操作。

```
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
acquireQueued则在循环中判断node的前继节点是不是head，如果是则继续尝试tryAcquire，如果acquire成功则说明成功通过了acquire，则将自己设置为新的head，并设置一些null值防止多余引用导致一些内存GC不掉。否则将前继节点的waitStatus设置为SIGNAL。
```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
共享模式的acquire
shared模式调用的是acquireShared, 需要子类实现tryAcquireShared(arg), 需要子类实现tryAcquireShared返回值小于0说明获取失败，等于0表示获取成功，但是接下来的acquireShared不会成功，大于0说明acquireShared获取成功并且接下来的acquireShared也可能成功。
```
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```
主要逻辑在doAcquireShared中，和acquireQueued比较相似，先addWaiter，然后循环中根据前继是否是head判断是否进行tryAcquireShared。tryAcquireShared成功后（结果>=0)，调用的是setHeadAndPropagate，以下种情况都会尝试unparkSuccessor。

tryAcquireShared返回值大于0
head为null或head的waitStatus小于0
这两种情况都要继续判断一下，当next为null或next是共享类型的时候，基于保守的策略都去unpark后继节点。
```
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    if (propagate > 0 || h == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```
### release
独占模式的release
如果tryRelease返回了true,说明可以唤醒其他线程，则判断head不为null并且waitStatus不为0的情况下去unpark后继节点。
unparkSuccessor中当node的next为null并不能表明自己是尾节点，next的waitStatus > 0说明
next已经取消，如果小于等则清除signal状态。这两种情况都需要降级从tail向前遍历找到离node最近的没有取消的节点进行unpark。

```
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
共享模式release
shared模式，tryRelease调用子类实现的tryReleaseShared，tryReleaseShared返回true说明允许一个waiting acquire能够成功。

```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```
Node入队列
```
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.pred = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
## 参考
- <http://gee.cs.oswego.edu/dl/papers/aqs.pdf>
- <http://www.cs.rochester.edu/wcms/research/systems/high_performance_synch/>
- 《Java并发编程实战》 
- 《Java并发编程的艺术》