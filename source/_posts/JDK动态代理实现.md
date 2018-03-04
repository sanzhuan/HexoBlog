title: JDK动态代理实现
date: 2017-12-31 23:19:08
tags: [Java]
categories: Java
---
## 问题
 翻看spring代码时候经常能看到InvocationHandler，这个接口只有一个方法
```
public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
```
JDK对这个方法是这样介绍的
>  Processes a method invocation on a proxy instance and returns the result.  This method will be invoked on an invocation handler when a method is invoked on a proxy instance that it is associated with.

看的也是云里雾里，invoke是谁调用的，什么时候调用？
另外，我们都知道JDK动态代码实现AOP有个限制就是代理类必须实现接口，这又是为啥？

## JDK动态代理的使用
废话少说，先看一下JDK动态代理是怎么用的
```  
package dynamicProxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

/**
 * @athor wuxiaomin
 * @created 2/9/2017.
 */
public class MyInvocationHandler implements InvocationHandler {
    private Object target;

    public MyInvocationHandler(Object target) {
        super();
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before doSomeThing");
        Object ret = method.invoke(target, args);
        System.out.println("after doSomeThing");
        return ret;
    }

    public Object getProxy() {
        return Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), target.getClass().getInterfaces(), this);
    }
}
```
代理对象实现及接口

``` 
package dynamicProxy;

/**
 * @athor wuxiaomin
 * @created 2/9/2017.
 */
public interface ISubject {

    void doSomeThing();
}
```

```
package dynamicProxy.impl;

import dynamicProxy.ISubject;

/**
 * @athor wuxiaomin
 * @created 2/9/2017.
 */
public class ISubjectImpl implements ISubject {
    public void doSomeThing() {
        System.out.println("doSomeThing");
    }
}
```
测试类

```java
package dynamicProxy;

import dynamicProxy.impl.ISubjectImpl;

/**
 * @athor wuxiaomin
 * @created 2/9/2017.
 */
public class DynamicProxyTest {

    public static void main(String[] args) {
        ISubject ISubject = new ISubjectImpl();
        MyInvocationHandler handler = new MyInvocationHandler(ISubject);
        ISubject proxy = (ISubject) handler.getProxy();
        proxy.doSomeThing();
    }
}
```
运行结果

```
before doSomeThing
doSomeThing
after doSomeThing
```
其实这里就是AOP的一个简单实现了，在目标对象的方法执行之前和执行之后进行了增强。Spring的AOP实现其实也是用了Proxy和InvocationHandler这两个东西的。

## JDK动态代理的原理与实现
先来看看Proxy类的newProxyInstance是怎么生成代理类的
```java
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```

核心逻辑就三步：1.Class<?> cl = getProxyClass0(loader, intfs);生成代理对象；2.
final Constructor<?> cons = cl.getConstructor(constructorParams);获取代理类的构造函数，这里值得注意的是代理类构造函数参数是InvocationHandler对象（constructorParams定义是private static final Class<?>[] constructorParams =
        { InvocationHandler.class };）；3.return cons.newInstance(new Object[]{h});返回代理类对象
        
这里比较重要的是第一步生成代理类
```
    private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        return proxyClassCache.get(loader, interfaces);
    }
```

getProxyClass0只是从缓存中拿代理类，缓存中要是没有在通过地工厂生产。这个生产过程涉及到缓存比较复杂，不做具体研究，我们直接定位到ProxyClassFactory中看看是怎么产生的

```
    private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // prefix for all proxy class names
        private static final String proxyClassNamePrefix = "$Proxy";

        // next number to use for generation of unique proxy class names
        private static final AtomicLong nextUniqueNumber = new AtomicLong();

        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            for (Class<?> intf : interfaces) {
                /*
                 * Verify that the class loader resolves the name of this
                 * interface to the same Class object.
                 */
                Class<?> interfaceClass = null;
                try {
                    interfaceClass = Class.forName(intf.getName(), false, loader);
                } catch (ClassNotFoundException e) {
                }
                if (interfaceClass != intf) {
                    throw new IllegalArgumentException(
                        intf + " is not visible from class loader");
                }
                /*
                 * Verify that the Class object actually represents an
                 * interface.
                 */
                if (!interfaceClass.isInterface()) {
                    throw new IllegalArgumentException(
                        interfaceClass.getName() + " is not an interface");
                }
                /*
                 * Verify that this interface is not a duplicate.
                 */
                if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                    throw new IllegalArgumentException(
                        "repeated interface: " + interfaceClass.getName());
                }
            }

            String proxyPkg = null;     // package to define proxy class in
            int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

            /*
             * Record the package of a non-public proxy interface so that the
             * proxy class will be defined in the same package.  Verify that
             * all non-public proxy interfaces are in the same package.
             */
            for (Class<?> intf : interfaces) {
                int flags = intf.getModifiers();
                if (!Modifier.isPublic(flags)) {
                    accessFlags = Modifier.FINAL;
                    String name = intf.getName();
                    int n = name.lastIndexOf('.');
                    String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                    if (proxyPkg == null) {
                        proxyPkg = pkg;
                    } else if (!pkg.equals(proxyPkg)) {
                        throw new IllegalArgumentException(
                            "non-public interfaces from different packages");
                    }
                }
            }

            if (proxyPkg == null) {
                // if no non-public proxy interfaces, use com.sun.proxy package
                proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
            }

            /*
             * Choose a name for the proxy class to generate.
             */
            long num = nextUniqueNumber.getAndIncrement();
            String proxyName = proxyPkg + proxyClassNamePrefix + num;

            /*
             * Generate the specified proxy class.
             */
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
                /*
                 * A ClassFormatError here means that (barring bugs in the
                 * proxy class generation code) there was some other
                 * invalid aspect of the arguments supplied to the proxy
                 * class creation (such as virtual machine limitations
                 * exceeded).
                 */
                throw new IllegalArgumentException(e.toString());
            }
        }
    }
```

核心逻辑是调用ProxyGenerator的generateClassFile生成代理类二进制字节码,然后调用defineClass0定义代理类class对象
```java
    public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2) {
        ProxyGenerator var3 = new ProxyGenerator(var0, var1, var2);
        final byte[] var4 = var3.generateClassFile();
        if(saveGeneratedFiles) {
            AccessController.doPrivileged(new PrivilegedAction() {
                public Void run() {
                    try {
                        int var1 = var0.lastIndexOf(46);
                        Path var2;
                        if(var1 > 0) {
                            Path var3 = Paths.get(var0.substring(0, var1).replace('.', File.separatorChar), new String[0]);
                            Files.createDirectories(var3, new FileAttribute[0]);
                            var2 = var3.resolve(var0.substring(var1 + 1, var0.length()) + ".class");
                        } else {
                            var2 = Paths.get(var0 + ".class", new String[0]);
                        }

                        Files.write(var2, var4, new OpenOption[0]);
                        return null;
                    } catch (IOException var4x) {
                        throw new InternalError("I/O exception saving generated file: " + var4x);
                    }
                }
            });
        }

        return var4;
    }

```

正式生成字节码在generateClassFile函数中，比较繁琐，简单的逻辑是遍历代理接口中的方法，生成相应方法的字节码
到这里还是不知道代码类长啥样，因为saveGeneratedFiles（sun.misc.ProxyGenerator.saveGeneratedFiles的属性值）为false,并没有把代理类保存到磁盘。我们可以直接ProxyGenerator生成字节码保存到磁盘

```
package dynamicProxy;

import dynamicProxy.impl.ISubjectImpl;
import sun.misc.ProxyGenerator;

import java.io.FileOutputStream;

/**
 * @athor wuxiaomin
 * @created 2/9/2017.
 */
public class ProxyGeneratorTest {

    public static void main(String[] args){
        byte[] bytes = ProxyGenerator.generateProxyClass("$Proxy11",ISubjectImpl.class.getInterfaces());
        FileOutputStream out = null;
        try {
            out = new FileOutputStream("$Proxy11.class");
            out.write(bytes);
            out.flush();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}
```

反编译出来$Proxy11.class（用IDEA打开$Proxy11.class就能反编译出来）

```
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

import dynamicProxy.ISubject;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy11 extends Proxy implements ISubject {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public $Proxy11(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void doSomeThing() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{Class.forName("java.lang.Object")});
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
            m3 = Class.forName("dynamicProxy.ISubject").getMethod("doSomeThing", new Class[0]);
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

doSomeThing是我们代理接口的方法，Proxy的newProxyInstance返回的是$Proxy11的实例,调用测试类的proxy.doSomeThing()就是调用$Proxy11的doSomeThing方法，而$Proxy11的doSomeThing 方法（```super.h.invoke(this, m3, (Object[])null);```）调用了MyInvocationHandler的invoke方法


另外一个问题，因为代理类本身已经extends了Proxy，而java是不允许多重继承的，所以JDK动态代理只能代理接口
