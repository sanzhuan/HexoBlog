title: 'Java 1:1线程模型实现源码解读'
date: 2018-05-13 21:03:10
tags: [Java,源码解读]
categories: [Java,源码解读]
---
## Java的线程模型
线程模型按照内核态线程和用户态线程的关系可以粗略地分为N：M、1:N和1:1线程模型。Java线程在JDK 1.2之前是基于Green Threads实现的，而在1.2中，线程模型替换成基于操作系统原生线程模型来实现，在Window和Linux中都是采用1:1的线程模型
## 源码实现
Java的1:1线程模型即一个Java线程映射为一个操作系统的线程。首先看java.lang.Thread的源码

``` 
public
class Thread implements Runnable {

    public synchronized void start() {
        if (threadStatus != 0)
            throw new IllegalThreadStateException();
            
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

    private native void start0();
}
```
调用线程start会掉用start0这个本地方法，然后掉用线程的Runnable接口的run方法，那么run方法是怎么掉用的呢

start0这个native方法在src/share/native/java/lang/Thread.c这个文件中定义。可以看到start0调用的是c++ 的JVM_StartThread这个函数

```
static JNINativeMethod methods[] = {
    {"start0",           "()V",        (void *)&JVM_StartThread},
    {"stop0",            "(" OBJ ")V", (void *)&JVM_StopThread},
    {"isAlive",          "()Z",        (void *)&JVM_IsThreadAlive},
    {"suspend0",         "()V",        (void *)&JVM_SuspendThread},
    {"resume0",          "()V",        (void *)&JVM_ResumeThread},
    {"setPriority0",     "(I)V",       (void *)&JVM_SetThreadPriority},
    {"yield",            "()V",        (void *)&JVM_Yield},
    {"sleep",            "(J)V",       (void *)&JVM_Sleep},
    {"currentThread",    "()" THD,     (void *)&JVM_CurrentThread},
    {"countStackFrames", "()I",        (void *)&JVM_CountStackFrames},
    {"interrupt0",       "()V",        (void *)&JVM_Interrupt},
    {"isInterrupted",    "(Z)Z",       (void *)&JVM_IsInterrupted},
    {"holdsLock",        "(" OBJ ")Z", (void *)&JVM_HoldsLock},
    {"getThreads",        "()[" THD,   (void *)&JVM_GetAllThreads},
    {"dumpThreads",      "([" THD ")[[" STE, (void *)&JVM_DumpThreads},
    {"setNativeName",    "(" STR ")V", (void *)&JVM_SetNativeThreadName},
};

JNIEXPORT void JNICALL
Java_java_lang_Thread_registerNatives(JNIEnv *env, jclass cls)
{
    (*env)->RegisterNatives(env, cls, methods, ARRAY_LENGTH(methods));
}

```
而JVM_StartThread这个函数在src/share/vm/prims/jvm.cpp这个文件中定义。

```
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_StartThread");
  JavaThread *native_thread = NULL;

  {
    MutexLocker mu(Threads_lock);

    if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) != NULL) {
      throw_illegal_thread_state = true;
    } else {

      jlong size =
             java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));

      size_t sz = size > 0 ? (size_t) size : 0;
      native_thread = new JavaThread(&thread_entry, sz);

      if (native_thread->osthread() != NULL) {
        native_thread->prepare(jthread);
      }
    }
  }

  if (throw_illegal_thread_state) {
    THROW(vmSymbols::java_lang_IllegalThreadStateException());
  }

  assert(native_thread != NULL, "Starting null thread?");

  if (native_thread->osthread() == NULL) {
    delete native_thread;
    if (JvmtiExport::should_post_resource_exhausted()) {
      JvmtiExport::post_resource_exhausted(
        JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_THREADS,
        "unable to create new native thread");
    }
    THROW_MSG(vmSymbols::java_lang_OutOfMemoryError(),
              "unable to create new native thread");
  }

  Thread::start(native_thread);

JVM_END

```
其中native_thread = new JavaThread(&thread_entry, sz);这行代码创建一个c++ JavaThread对象。thread_enty内容为：

```
static void thread_entry(JavaThread* thread, TRAPS) {
  HandleMark hm(THREAD);
  Handle obj(THREAD, thread->threadObj());
  JavaValue result(T_VOID);
  JavaCalls::call_virtual(&result,
                          obj,
                          KlassHandle(THREAD, SystemDictionary::Thread_klass()),
                          vmSymbols::run_method_name(),
                          vmSymbols::void_method_signature(),
                          THREAD);
}
```
其中vmSymbols::run_method_name()在vmSymbols.hpp中定义为

```
class vmSymbolHandles: AllStatic {   
   …  
    template(run_method_name,"run")  
   …  
}  
```
也就是调用操作系统API以后就直接调用run函数，这就是调用start函数以后立即调用run函数的魔法所在。
而在linux下调用的创建线程的API其实就是pthread API

```
JavaThread::JavaThread(ThreadFunction entry_point, size_t stack_sz) :
  Thread()
{
  if (TraceThreadEvents) {
    tty->print_cr("creating thread %p", this);
  }
  initialize();
  _jni_attach_state = _not_attaching_via_jni;
  set_entry_point(entry_point);
  os::ThreadType thr_type = os::java_thread;
  thr_type = entry_point == &compiler_thread_entry ? os::compiler_thread :
                                                     os::java_thread;
  os::create_thread(this, thr_type, stack_sz);
  _safepoint_visible = false;
}
```
os::create_thread实现了操作系统相关的实际的线程创建操作。

```
bool os::create_thread(Thread* thread, ThreadType thr_type, size_t stack_size) {

  OSThread* osthread = new OSThread(NULL, NULL);
  if (osthread == NULL) {
    return false;
  }

  osthread->set_thread_type(thr_type);

  // Initial state is ALLOCATED but not INITIALIZED
  osthread->set_state(ALLOCATED);

  thread->set_osthread(osthread);

  // init thread attributes
  pthread_attr_t attr;
  pthread_attr_init(&attr);
  pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

  // glibc guard page
  pthread_attr_setguardsize(&attr, os::Linux::default_guard_size(thr_type));

  ThreadState state;

  {
    // Serialize thread creation if we are running with fixed stack LinuxThreads
    bool lock = os::Linux::is_LinuxThreads() && !os::Linux::is_floating_stack();
    if (lock) {
      os::Linux::createThread_lock()->lock_without_safepoint_check();
    }

    pthread_t tid;
    int ret = pthread_create(&tid, &attr, (void* (*)(void*)) java_start, thread);

    pthread_attr_destroy(&attr);

    ...
  }
  return true;
}
```
## 参考
+ <https://medium.com/@unmeshvjoshi/how-java-thread-maps-to-os-thread-e280a9fb2e06>
+ <https://blog.csdn.net/u012109105/article/details/75047470?locationNum=4&fps=1>
+ <https://www.ibm.com/developerworks/cn/java/j-lo-processthread/#icomments>
+ <https://juejin.im/entry/5960852cf265da6c2e0f8a31>
+ <http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/3ad03712ea43/src/share/native/java/lang/Thread.c>
+ <http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/os/linux/vm/os_linux.cpp>
