---
title: Java动态代理的深入理解
date: 2021-12-26 17:33:23
tags: Java
categories: Java
description:
---

每一个Proxy实例对象都有一个与之相关联的InvocationHandler实例，当在Proxy对象上调用代理的方法时，该方法就会被分发（dispatched）到这个InvocationHandler对象的invoke()方法上。

<!-- more -->

```java
package java.lang.reflect;

/**
 * {@code InvocationHandler} is the interface implemented by
 * the <i>invocation handler</i> of a proxy instance.
 *
 * <p>Each proxy instance has an associated invocation handler.
 * When a method is invoked on a proxy instance, the method
 * invocation is encoded and dispatched to the {@code invoke}
 * method of its invocation handler.
 *
 * @author      Peter Jones
 * @see         Proxy
 * @since       1.3
 */
public interface InvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}

```
### 一、入门示例
#### 1. 目标类和代理类需要实现的共同接口IPayment 
```java
package com.yxf.dynamicproxy;
public interface IPayment {
    long pay(int amount);
    long price();
}
```
#### 2. 目标类CreditCard
```java
package com.yxf.dynamicproxy;

public class CreditCard implements IPayment {

    @Override
    public long pay(int amount) {
        return price() * amount;
    }

    @Override
    public long price() {
        return 99;
    }
}
```

#### 3. 调用者需要提供的InvocationHandler实例
```java
package com.yxf.dynamicproxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class MyInvocationHandler implements InvocationHandler{
    private final Object target;

    public MyInvocationHandler(Object object) {
        this.target = object;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if(target == null) {
            return null;
        }

        return method.invoke(target, args);
    }
}
```
#### 4. 使用方法
```java
package com.yxf.dynamicproxy;

import java.lang.reflect.Proxy;

public class Client {
    public static void main(String[] args) {
        testDynamicProxy();
    }

    private static void testDynamicProxy() {
        IPayment target = new CreditCard();
        //System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        //sun.misc.Launcher$AppClassLoader
        System.out.println("IPayment.class.getClassLoader: " + IPayment.class.getClassLoader().getClass().getName());
        IPayment proxy = null;
        try {
            proxy = (IPayment) Proxy.newProxyInstance(IPayment.class.getClassLoader(), new Class[]{IPayment.class},
                    new MyInvocationHandler(target));
            System.out.println("proxy: " + proxy.getClass().getName()); //com.sun.proxy.$Proxy0
            long money = proxy.pay(2000);
            System.out.println("proxy.price: " + money);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
#### 五、生成的Proxy实例class
```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.sun.proxy;

import com.yxf.dynamicproxy.IPayment;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements IPayment {
    private static Method m1;
    private static Method m4;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final long price() throws  {
        try {
            return (Long)super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
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

    public final long pay(int var1) throws  {
        try {
            return (Long)super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m4 = Class.forName("com.yxf.dynamicproxy.IPayment").getMethod("price");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.yxf.dynamicproxy.IPayment").getMethod("pay", Integer.TYPE);
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```
### 二、Proxy的newProxyInstance()方法
```java
public class Proxy implements java.io.Serializable {
    private static final long serialVersionUID = -2222568056686623797L;

    /** parameter types of a proxy class constructor */
    private static final Class<?>[] constructorParams =
        { InvocationHandler.class };

    ...

    @CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h) throws IllegalArgumentException {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         */
        //获取代理类的class对象。
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            //获取代理类的构造方法，其中constructorParams为构造方法的参数。
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
            //使用Constructor反射的方式创建代理类的实例，其中构造方法的参数为new Object[]{h}.
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

    /**
     * A factory function that generates, defines and returns the proxy class given
     * the ClassLoader and array of interfaces.
     */
    //给定一个ClassLoader和一个Interface的Class[]数组，生成代理类。
    private static final class ProxyFactory
            implements BiFunction<ClassLoader, Class<?>[], Class<?>> {
        // prefix for all proxy class names
        private static final String proxyClassNamePrefix = "$Proxy";

        // next number to use for generation of unique proxy class names
        private static final AtomicLong nextUniqueNumber = new AtomicLong();

        public Class<?> apply(ClassLoader classLoader, Class<?>[] classes) {
            //...

            String proxyPkg = null;     // package to define proxy class in

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
                throw new IllegalArgumentException(e.toString());
            }
        }
    }

    //JVM native方法, 用于将字节流转化为Class对象。
    private static native Class<?> defineClass0(ClassLoader loader, String name,
                                                byte[] b, int off, int len);
}
```
### 三、ProxyGenerator类的generateClassFile()方法
关键代码：
[sun/misc/ProxyGenerator.java]
```java
package sun.misc;
/**
 * ProxyGenerator contains the code to generate a dynamic proxy class
 * for the java.lang.reflect.Proxy API.
 *
 * The external interfaces to ProxyGenerator is the static
 * "generateProxyClass" method.
 *
 * @author      Peter Jones
 * @since       1.3
 */
public class ProxyGenerator {

    /* 预加载的java.lang.Object类的Method实例。*/
    private static Method hashCodeMethod;
    private static Method equalsMethod;
    private static Method toStringMethod;
    static {
        try {
            hashCodeMethod = Object.class.getMethod("hashCode");
            equalsMethod = Object.class.getMethod("equals", new Class<?>[] { Object.class });
            toStringMethod = Object.class.getMethod("toString");
        } catch (NoSuchMethodException e) {
            throw new NoSuchMethodError(e.getMessage());
        }
    }

    /** 保存生成class文件的调试开关 */
    private final static boolean saveGeneratedFiles =
            java.security.AccessController.doPrivileged(
                    new GetBooleanAction(
                            "sun.misc.ProxyGenerator.saveGeneratedFiles")).booleanValue();


    /**
     * Generate a proxy class given a name and a list of proxy interfaces.
     *
     * @param name        the class name of the proxy class
     * @param interfaces  proxy interfaces
     * @param accessFlags access flags of the proxy class
     */
    public static byte[] generateProxyClass(final String name,
                                            Class<?>[] interfaces,
                                            int accessFlags)
    {
        ProxyGenerator gen = new ProxyGenerator(name, interfaces, accessFlags);
        final byte[] classFile = gen.generateClassFile();

        if (saveGeneratedFiles) {
            //调试使用
        }

        return classFile;
    }

    /**
     * 生成Proxy类的.class文件
     *
     * @return
     */
    private byte[] generateClassFile() {
        //生成Java.lang.Object的hashCode、equals和toString方法.
        addProxyMethod(hashCodeMethod, Object.class);
        addProxyMethod(equalsMethod, Object.class);
        addProxyMethod(toStringMethod, Object.class);

        for (Class<?> intf : interfaces) {
            for (Method m : intf.getMethods()) {
                addProxyMethod(m, intf);
            }
        }

        for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
            checkReturnTypes(sigmethods);
        }

        try {
            methods.add(generateConstructor());
            for (List<ProxyMethod> sigmethods : proxyMethods.values()) {
                for (ProxyMethod pm : sigmethods) {
                    //添加static的Method字段变量
                    fields.add(new FieldInfo(pm.methodFieldName,
                            "Ljava/lang/reflect/Method;",
                            ACC_PRIVATE | ACC_STATIC));

                    methods.add(pm.generateMethod());
                }
            }
            methods.add(generateStaticInitializer());
        } catch (IOException e) {
            throw new InternalError("unexpected I/O Exception", e);
        }

        if (methods.size() > 65535) {
            throw new IllegalArgumentException("method limit exceeded");
        }
        if (fields.size() > 65535) {
            throw new IllegalArgumentException("field limit exceeded");
        }

        cp.getClass(dotToSlash(className));
        cp.getClass(superclassName);
        for (Class<?> intf : interfaces) {
            cp.getClass(dotToSlash(intf.getName()));
        }

        cp.setReadOnly();

        ByteArrayOutputStream bout = new ByteArrayOutputStream();
        DataOutputStream dout = new DataOutputStream(bout);

        try {
            dout.writeInt(0xCAFEBABE); // u4 magic;
            dout.writeShort(CLASSFILE_MINOR_VERSION); // u2 minor_version;
            dout.writeShort(CLASSFILE_MAJOR_VERSION); // u2 major_version;
            cp.write(dout);  // (write constant pool)

            dout.writeShort(accessFlags); // u2 access_flags;
            dout.writeShort(cp.getClass(dotToSlash(className))); // u2 this_class;
            dout.writeShort(cp.getClass(superclassName)); // u2 super_class;
            dout.writeShort(interfaces.length); // u2 interfaces_count;

            // u2 interfaces[interfaces_count];
            for (Class<?> intf : interfaces) {
                dout.writeShort(cp.getClass(
                        dotToSlash(intf.getName())));
            }

            // u2 fields_count;
            dout.writeShort(fields.size());
            // field_info fields[fields_count];
            for (FieldInfo f : fields) {
                f.write(dout);
            }

            // u2 methods_count;
            dout.writeShort(methods.size());
            // method_info methods[methods_count];
            for (MethodInfo m : methods) {
                m.write(dout);
            }

            // u2 attributes_count;
            dout.writeShort(0); // (no ClassFile attributes for proxy classes)
        } catch (IOException e) {
            throw new InternalError("unexpected I/O Exception", e);
        }

        return bout.toByteArray();
    }
}

参见代码：
https://github.com/JetBrains/jdk8u_jdk/blob/master/src/share/classes/sun/misc/ProxyGenerator.java
```