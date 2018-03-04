title: ReentrantReadWriteLock源码解读
date: 2018-01-28 17:49:18
tags: [Java,源码解读]
categories: Java
---

## ReentrantReadWriteLock简介
重入锁ReentrantLock是排他锁，在同一时刻仅有一个线程可以进行访问，而在大多数场景下，都是读多写少，如果一个线程在读时禁止其他线程读就会导致性能降低。而读写锁ReentrantReadWriteLock很好地解决了这种问题。

读写锁维护着一对锁，一个读锁和一个写锁，在同一时间可以允许多个读线程同时访问，但是在写线程访问时，所有读线程和写线程都会被阻塞，通过分离读锁和写锁，并发性比一般的排他锁有了较大的提升。

## 源码解读
如上所述，读写锁ReentrantReadWriteLock内部维护着一个读锁和一个写锁，与ReentrantLock一样，它的读锁、写锁都是依靠AQS来实现的。代码主框架如下：
```
    /** 内部类  读锁 */
    private final ReentrantReadWriteLock.ReadLock readerLock;
    /** 内部类  写锁 */
    private final ReentrantReadWriteLock.WriteLock writerLock;

    final Sync sync;

    /** 使用默认（非公平）的排序属性创建一个新的 ReentrantReadWriteLock */
    public ReentrantReadWriteLock() {
        this(false);
    }

    /** 使用给定的公平策略创建一个新的 ReentrantReadWriteLock */
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

    /** 返回用于写入操作的锁 */
    public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
    /** 返回用于读取操作的锁 */
    public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }

	abstract static class Sync extends AbstractQueuedSynchronizer {
        /**
         * 省略其余源代码
         */
    }
    public static class WriteLock implements Lock, java.io.Serializable{
        /**
         * 省略其余源代码
         */
    }

    public static class ReadLock implements Lock, java.io.Serializable {
        /**
         * 省略其余源代码
         */
    }
```
使用AQS state的高16bit表示读线程数目（线程的重入数目通过threadlocal维护），低16位表示写线程的持有数目（写线程是互斥的，因此这里的数目是指当前持有锁的线程的重入数目），同步状态的操作通过位操作完成。比如当前同步状态为S，那么写状态等于 S & 0x0000FFFF（将高16位全部抹去），读状态等于S >>> 16(无符号补0右移16位)。代码如下所示：

```
        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

### 写锁
写锁是支持重入的排他锁。写锁的获取代码如下：
```
    public void lock() {
    		//调用AQS的acquire 如下
            sync.acquire(1);
    }
    
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
写锁的获取首先调用AQS的acquire,AQS最终调用ReentrantReadWriteLock syn定义的tryAcquire，代码如下：
```
        protected final boolean tryAcquire(int acquires) {
            Thread current = Thread.currentThread();
            int c = getState();
            //写锁
            int w = exclusiveCount(c);
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // 当前线程持有写锁 Reentrant acquire
                setState(c + acquires);
                return true;
            }
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

#### 写锁的释放

```
    public void unlock() {
        sync.release(1);
    }

    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    } 
    
    protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            // state为0 释放写锁
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
    }                    
```

### 读锁
#### 读锁的获取
读锁为一个可重入的共享锁，因此调用acquireShared使得可以有多个共享的读者线程。代码如下：

```
        protected final int tryAcquireShared(int unused) {
            Thread current = Thread.currentThread();
            int c = getState();
            // 存在写锁  
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            // 读线程数
            int r = sharedCount(c);
            // c + SHARED_UNIT 相当于＋1
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                	 //第一个读线程
                    firstReader = current;
                    //第一个读线程重入数
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                //第一个读线程重入数 ＋ 1
                    firstReaderHoldCount++;
                } else {
                // 最后一个成功获取读锁数据缓存 包含重入数目和线程id
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                    // 最后一个读线程为空 或者最后一个读线程不是当前线程
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                    	 //readHolds为threadlocal,维护每个线程的重入数和线程id
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```


```
        final int fullTryAcquireShared(Thread current) {
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```

#### 读锁的释放

```
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                	// 当前线程计数
                    rh = readHolds.get();
                //获取当前读线程重入数
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();
                // 相当于 state － 1
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    //没有读锁 可以唤醒写锁
                    return nextc == 0;
            }
        }
```

### 锁降级
所谓的锁降级是指先获取写锁，再获取读锁，最后释放写锁。实现代码为：

```
// 持有写锁并且持有写锁的是当前线程 获取读锁并不会立即失败
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
```


## 参考
- <http://gee.cs.oswego.edu/dl/papers/aqs.pdf>
- 《Java并发编程实战》 
- JDK8 源码