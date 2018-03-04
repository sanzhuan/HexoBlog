title: CountDownLatch源码分析
date: 2018-01-20 22:35:22
tags: [Java,源码解读]
categories: Java
---
## CountDownLatch简介
闭锁CountDownLatch可以简单理解成栅栏，是一种线程同步机制。官方文档是这样说的
> A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.

## CountDownLatch使用
下面代码使用了两个CountDownLatch。创建了一组工作线程，

* 第一个CountDownLatch是主线程防止driver准备好之前工作线程开始执行。
* 第二个CountDownLatch是主线程为了等待工作线程全部结束才结束。
 
```
class Driver {
     void main() throws InterruptedException {
         CountDownLatch startSignal = new CountDownLatch(1);
         CountDownLatch doneSignal = new CountDownLatch(N);

         for (int i = 0; i < N; ++i) // create and start threads
             new Thread(new Worker(startSignal, doneSignal)).start();

         doSomethingElse();            // don't let run yet
         startSignal.countDown();      // let all threads proceed
         doSomethingElse();
         doneSignal.await();           // wait for all to finish
     }
 }

 class Worker implements Runnable {
     private final CountDownLatch startSignal;
     private final CountDownLatch doneSignal;
     Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
         this.startSignal = startSignal;
         this.doneSignal = doneSignal;
     }
     public void run() {
         try {
             startSignal.await();
             doWork();
             doneSignal.countDown();
         } catch (InterruptedException ex) {} // return;
     }

     void doWork() { ... }
 }}
```
另外一个闭锁的典型应用场景是把一个任务分给worker线程执行，然后主线程等待任务结束。

```
 class Driver2 {
             void main() throws InterruptedException {
                 CountDownLatch doneSignal = new CountDownLatch(N);
                 Executor e = ...

                     for (int i = 0; i < N; ++i) // create and start threads
                         e.execute(new WorkerRunnable(doneSignal, i));

                 doneSignal.await();           // wait for all to finish
             }
         }

 class WorkerRunnable implements Runnable {
     private final CountDownLatch doneSignal;
     private final int i;
     WorkerRunnable(CountDownLatch doneSignal, int i) {
         this.doneSignal = doneSignal;
         this.i = i;
     }
     public void run() {
         try {
             doWork(i);
             doneSignal.countDown();
         } catch (InterruptedException ex) {} // return;
     }

     void doWork() { ... }
 }}
 
```
## 源码实现分析
CountDownLatch基于AQS实现，使用AQS state作为计数器。
```
public class CountDownLatch {
 	private final Sync sync;
 	
    private static final class Sync extends AbstractQueuedSynchronizer {

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
        	//state == 0意味者不必等待，否则就在AQS同步队列上等待
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases){
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                	 // 计算器为0时才唤醒所有等待线程
                    return nextc == 0;
            }
        }
    }
    
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
    
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    
    public void countDown() {
        sync.releaseShared(1);
    }
 ```

## 参考
- <http://gee.cs.oswego.edu/dl/papers/aqs.pdf>
- 《Java并发编程实战》
- JDK8 源码 
 