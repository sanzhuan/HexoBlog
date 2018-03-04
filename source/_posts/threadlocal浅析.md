title: threadlocal浅析
date: 2018-01-02 23:30:30
tags: [Java]
categories: Java
---
## 什么是ThreadLocal
java对ThreadLocal的解释是

This class provides thread-local variables.  These variables differ from their normal counterparts in that each thread that accesses one (via its {@code get} or {@code set} method) has its own, independently initialized copy of the variable.  {@code ThreadLocal} instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).

其实ThreadLocal并非是一个线程的本地实现版本，它并不是一个Thread，而是threadlocalvariable(线程局部变量)。也许把它命名为ThreadLocalVar更加合适。线程局部变量(ThreadLocal)其实的功用非常简单，就是为每一个使用该变量的线程都提供一个变量值的副本，是Java中一种较为特殊的线程绑定机制，是每一个线程都可以独立地改变自己的副本，而不会和其它线程的副本冲突。

## ThreadLocal怎么使用
ThreadLocal可以看做线程相关的register，存进去的数据在同一个线程都可以使用。注意ThreadLocal 有protected方法initialValue()可以设置ThreadLocal的初始值，在jdk1.8中，ThreadLocal提供了withInitial()方法，可以使用FunctionalInterface Supplier对ThreadLocal提供初始值

```
private static final ThreadLocal<String> threadContext = new ThreadLocal<>();
  
threadContext.set("test");
threadContext.get();// 同一个线程中都可以获取字符串test
```

## ThreadLocal的原理
ThreadLocal有一个ThreadLocalMap静态内部类，你可以简单理解为一个MAP，这个Map为每个线程复制一个变量的‘拷贝’存储其中。

当线程调用ThreadLocal.get()方法获取变量时,首先获取当前线程引用，以此为key去获取响应的ThreadLocalMap，如果此‘Map’不存在则初始化一个，否则返回其中的变量，代码如下：

```
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
调用get方法如果此Map不存在首先初始化，创建此map，将线程为key，初始化的vlaue存入其中，注意此处的initialValue，我们可以覆盖此方法，在首次调用时初始化一个适当的值。setInitialValue代码如下：

```
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```
set方法相对比较简单如果理解以上俩个方法，获取当前线程的引用，从map中获取该线程对应的map，如果map存在更新缓存值，否则创建并存储，代码如下：

```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```
很明显，ThreadLocal的变量值都是存放在ThreadLocalMap中的

```
void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
}
```

这个ThreadLocalMap 类是ThreadLocal中定义的内部类，但是它的实例却用在Thread类中，这说明ThreadLocal访问的是Thread里面的变量

```
public class Thread implements Runnable {
    ......
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
    ......
}
```

## ThreadLocal的内存管理
一个非常自然想法是用一个线程安全的 Map<Thread,Object> 实现：

```
class ThreadLocal {
  private Map values = new ConcurrentHashMap();
  public Object get() {
    Thread curThread = Thread.currentThread();
    Object o = values.get(curThread);
    if (o == null && !values.containsKey(curThread)) {
      o = initialValue();
      values.put(curThread, o);
    }
    return o;
  }
  public void set(Object newValue) {
    values.put(Thread.currentThread(), newValue);
  }
}
```
这样做法的最大问题是当线程结束了，Tread在map中还有引用，导致线程和value都无法回收

```
static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```
![threadlocal](/images/threadlocal.png)

每个thread中都存在一个map, map的类型是ThreadLocal.ThreadLocalMap. Map中的key为一个threadlocal实例. 这个Map的确使用了弱引用,不过弱引用只是针对key. 每个key都弱引用指向threadlocal. 当把threadlocal实例tl置为null以后,没有任何强引用指向threadlocal实例,所以threadlocal将会被gc回收. 但是,我们的value却不能回收,因为存在一条从current thread连接过来的强引用. 只有当前thread结束以后, current thread就不会存在栈中,强引用断开, Current Thread, Map, value将全部被GC回收。


## ThreadLocal产生的问题
在WEB服务器环境下，由于容器有一个线程池的概念，即接收到一个请求后，直接从线程池中取得线程处理请求；请求响应完成后，这个线程本身是不会结束，而是进入线程池，这样可以减少创建线程、启动线程的系统开销。

由于线程池的原因，最初使用的”线程局部变量”保存的值，在下一次请求依然存在（同一个线程处理）

解决方案：处理完成后主动调用该业务treadLocal的remove()方法，将”线程局部变量”清空，避免本线程下次处理的时候依然存在旧数据。
