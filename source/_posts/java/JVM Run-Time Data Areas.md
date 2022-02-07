---
title: JVM运行时数据区域
urlname: jvm_runtime_data
date: 2022-01-30 17:57:34
tags: Java
categories: Java
description: JVM运行时数据区域的理解...
---

### 一、JVM Run-Time Data Areas

#### 1. The `pc` Register

PC Register是线程私有的数据区域，存储当前线程正在执行的指令地址，如果正在执行的方法是native方法，则PC Register的值是未定义的(undefined)。

#### 2. Java Virtual Machine Stacks

JVM Stacks是线程私有的数据区域，用于存储栈帧（frames）。每一个JVM Thread拥有一个私有的JVM Stack。

当JVM中的Thread被创建的时候，JVM Stack会随之一起创建。

#### 3. Heap

Heap区域被所有的JVM线程共享，在JVM启动时创建。所有的Class实例和数组（arrays）在Heap中创建。

#### 4. Method Area

Method Area是线程共享的内存区域，用于存储类信息（类的名称、方法信息、字段信息等）、常量(constants)、静态常量(static-final constants)、静态变量(static variables)以及JIT编译后的代码（code compiled by the jit compiler）等，此外Method Area中还包含一块特殊的区域——运行时常量池（Run-Time Constant Pool），在第5点单独分析。

Method Area包含的信息一共有：

##### 4.1 Loaded class bytecode

JVM将class的bytecode加载至内存的Method Area，bytecode可以是编译生成的.class文件、通过网络传输获取的、或者cglib动态生成的。

##### 4.2 Metadata objects such as class/method/field

###### ① Type information

对于每个已加载的类型(class, interface, enum, annotation)，JVM必须保存该类型的以下信息：

- 完整、有效的类型名称(packageName + \<typeName>.class)
- 完整、有效的直接父类的类型名称
- 类型修饰符(public, abstract, final等)

###### ② Field information：

JVM必须保存所有的fileds信息、以及它们的声明顺序：

fields信息包括：filed name、filed type、filed modifier等；

###### ③ Method information：

JVM必须保存所有的method信息、以及他们的声明顺序：

- Method名称.
- Method返回值
- Method 参数的类型和个数
- Method 修饰符 (public, private, protected, static, final, synchronized, native, abstract的子集)
- Method bytecodes, operand stack, local variable table and size (除了 abstract和 native方法)
- Exception table (除了 abstract和 native方法)，每个 exception handling的起始位置和结束位置, PC计数器中code handling 的偏移地址, and the constant pool index of the caught exception class.

##### 4.3 Non-final variables(static variales和class variables)

```java
public class MethodAreaTest {
    public static void main(String[] args) {
        Order order = null;
        order.hello(); //NO crash
        System.out.println(order.count);
    }
}

class Order {
    public static int count = 1;
    public static final int number = 2;


    public static void hello() {
        System.out.println("hello!");
    }
}
```

##### 4.4 Compilation result of jit compiler

#### 5. Run-Time Constant Pool

Run-Time Constant Pool是Method Area中的一部分，用于存储编译器生成的各种字面量(literals)和符号引用(symbol references)，主要有：numeric literals、string literals、class references、field references、method references。当class或interface被JVM创建时，它们的run-time constant pool就会被构建出来。

#### 6. Native Method Stacks

Native Method Stack是线程私有的数据区域，主要用于执行native(non-java)的方法。

### 二、HotSpot中Method Area的变迁

关于如何实现method area，JVM规范并没有统一的要求。在Hotsport中，在jdk7及以前，通常称Method Area为永久代(permanent generation)。从jdk8起，永久代(permanent generation)被meta-space所替代。

注意：永久代只是Hotspot中的Method Area的实现，在BEA的JRockit和IBM的J9上没有永久代这个概念。

Method Area的大小参数设置：

jdk7及以前(permanent generation)：

-XX:PermSize 设置永久代初始分配空间，默认是20.75M；

-XX:MaxPermSize 设置永久代最大可分配空间，默认32-bit机器上是64M, 64-bit机器上是82M。

jdk8及以后 (meta space):
-XX:MetaspaceSize和-XX:MaxMetaspaceSize分别对应前面的-XX:PermSize和-XX:MaxPermSize。
默认值视平台而定：
在windows上，-XX: MetaspaceSize是21M；-XX:MaxMetaspaceSize是-1，表示没有上限；
与永久代不同的是，如果没有指定值的话，虚拟机默认将会用光所有可用的系统内存；如果metadata溢出时，虚拟机会抛出OutOfMemoryError: Metaspace异常。

### 三、使用实例

下面通过例子来加深理解：

```java
package com.yxf.sample;

public class ConstantPool {
    private final int a = 10; //编译后会添加ConstantValue: int 10
    private int b = 1;
    private static int c = 2;

    private static final String d = "TAG"; //编译后会添加ConstantValue: String TAG
    private final String e = "final variable"; //编译后添加ConstantValue: String final variable
    private final Order order = new Order();

    public void sayHello() {
        System.out.println("Hello World");
    }
}
```
编译完成后，查看class文件
```java
$ javap -p -v out/production/JavaSample/com/yxf/sample/ConstantPool.class 
Classfile out/production/JavaSample/com/yxf/sample/ConstantPool.class
  Last modified 2022-1-23; size 898 bytes
  MD5 checksum 33e618f45ee1428896943e4b58e6a4b1
  Compiled from "ConstantPool.java"
public class com.yxf.sample.ConstantPool
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #14.#38        // java/lang/Object."<init>":()V
   #2 = Fieldref           #13.#39        // com/yxf/sample/ConstantPool.a:I
   #3 = Fieldref           #13.#40        // com/yxf/sample/ConstantPool.b:I
   #4 = String             #41            // final variable
   #5 = Fieldref           #13.#42        // com/yxf/sample/ConstantPool.e:Ljava/lang/String;
   #6 = Class              #43            // com/yxf/sample/Order
   #7 = Methodref          #6.#38         // com/yxf/sample/Order."<init>":()V
   #8 = Fieldref           #13.#44        // com/yxf/sample/ConstantPool.order:Lcom/yxf/sample/Order;
   #9 = Fieldref           #45.#46        // java/lang/System.out:Ljava/io/PrintStream;
  #10 = String             #47            // Hello World
  #11 = Methodref          #48.#49        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #12 = Fieldref           #13.#50        // com/yxf/sample/ConstantPool.c:I
  #13 = Class              #51            // com/yxf/sample/ConstantPool
  #14 = Class              #52            // java/lang/Object
  #15 = Utf8               a
  #16 = Utf8               I
  #17 = Utf8               ConstantValue
  #18 = Integer            10
  #19 = Utf8               b
  #20 = Utf8               c
  #21 = Utf8               d
  #22 = Utf8               Ljava/lang/String;
  #23 = String             #53            // TAG
  #24 = Utf8               e
  #25 = Utf8               order
  #26 = Utf8               Lcom/yxf/sample/Order;
  #27 = Utf8               <init>
  #28 = Utf8               ()V
  #29 = Utf8               Code
  #30 = Utf8               LineNumberTable
  #31 = Utf8               LocalVariableTable
  #32 = Utf8               this
  #33 = Utf8               Lcom/yxf/sample/ConstantPool;
  #34 = Utf8               sayHello
  #35 = Utf8               <clinit>
  #36 = Utf8               SourceFile
  #37 = Utf8               ConstantPool.java
  #38 = NameAndType        #27:#28        // "<init>":()V
  #39 = NameAndType        #15:#16        // a:I
  #40 = NameAndType        #19:#16        // b:I
  #41 = Utf8               final variable
  #42 = NameAndType        #24:#22        // e:Ljava/lang/String;
  #43 = Utf8               com/yxf/sample/Order
  #44 = NameAndType        #25:#26        // order:Lcom/yxf/sample/Order;
  #45 = Class              #54            // java/lang/System
  #46 = NameAndType        #55:#56        // out:Ljava/io/PrintStream;
  #47 = Utf8               Hello World
  #48 = Class              #57            // java/io/PrintStream
  #49 = NameAndType        #58:#59        // println:(Ljava/lang/String;)V
  #50 = NameAndType        #20:#16        // c:I
  #51 = Utf8               com/yxf/sample/ConstantPool
  #52 = Utf8               java/lang/Object
  #53 = Utf8               TAG
  #54 = Utf8               java/lang/System
  #55 = Utf8               out
  #56 = Utf8               Ljava/io/PrintStream;
  #57 = Utf8               java/io/PrintStream
  #58 = Utf8               println
  #59 = Utf8               (Ljava/lang/String;)V
{
  private final int a;
    descriptor: I
    flags: ACC_PRIVATE, ACC_FINAL
    ConstantValue: int 10

  private int b;
    descriptor: I
    flags: ACC_PRIVATE

  private static int c;
    descriptor: I
    flags: ACC_PRIVATE, ACC_STATIC

  private static final java.lang.String d;
    descriptor: Ljava/lang/String;
    flags: ACC_PRIVATE, ACC_STATIC, ACC_FINAL
    ConstantValue: String TAG

  private final java.lang.String e;
    descriptor: Ljava/lang/String;
    flags: ACC_PRIVATE, ACC_FINAL
    ConstantValue: String final variable

  private final com.yxf.sample.Order order;
    descriptor: Lcom/yxf/sample/Order;
    flags: ACC_PRIVATE, ACC_FINAL

  public com.yxf.sample.ConstantPool();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: bipush        10
         7: putfield      #2                  // Field a:I
        10: aload_0
        11: iconst_1
        12: putfield      #3                  // Field b:I
        15: aload_0
        16: ldc           #4                  // String final variable
        18: putfield      #5                  // Field e:Ljava/lang/String;
        21: aload_0
        22: new           #6                  // class com/yxf/sample/Order
        25: dup
        26: invokespecial #7                  // Method com/yxf/sample/Order."<init>":()V
        29: putfield      #8                  // Field order:Lcom/yxf/sample/Order;
        32: return
      LineNumberTable:
        line 3: 0
        line 4: 4
        line 5: 10
        line 9: 15
        line 10: 21
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      33     0  this   Lcom/yxf/sample/ConstantPool;

  public void sayHello();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #10                 // String Hello World
         5: invokevirtual #11                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 13: 0
        line 14: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lcom/yxf/sample/ConstantPool;

  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: iconst_2
         1: putstatic     #12                 // Field c:I
         4: return
      LineNumberTable:
        line 6: 0
}
SourceFile: "ConstantPool.java"

```

#### Class文件常量池

上述javap输出的Constant Pool是class文件的常量池。包含：

- Methodref：以冒号隔开的一对值，例如com/yxf/sample/Order."<init>":()V，第一部分是方法名，第二部分是方法类型；

- Fieldref：以冒号隔开的一对值，例如com/yxf/sample/ConstantPool.f:Ljava/lang/String;，第一部分是字段名，第二部分是字段类型

- String:  指向Utf8中的字面量；

- Class：指向Utf8中的类名全路径。例如com/yxf/sample/ConstantPool；

- Integer：整形字面量，就是用final修饰的int类型；

- Utf8：类名、字段名、字段类型、函数名、函数类型、this指针、LineNumberTable、LocalVariableTable、ConstantValue等；

- NameAndType： 以冒号隔开的一对值，即\<name>:\<type>。注意这里的名字可以是变量名，也可以是函数名。如果是函数名的话，类型就是函数签名。举例：

```java
private final int a = 10; //对应的NameAndType是"a:I"
private int b = 1; //对应的NameAndType是"b:I"
private static final String d = "TAG"; //对应的NameAndType是"d:Ljava/lang/String;"
System.out.println("Hello World"); //这里有两个NameAndType, out对应的是："out:Ljava/io/PrintStream;", println对应的是"println:(Ljava/lang/String;)V"。
```

#### 全局字符串常量池

Java中创建字符串对象的两种方式：

```java
String s0 = "Hello";
String s1 = new String("Hello");
```

Class文件常量池的大部分数据会被加载到Run-time Constant Pool中，包括String字面量。但"Hello"字符串的一个引用会被存储在Non-Heap区域的"字符串常量池"中，而"Hello"本身还是和其它对象一样，在Heap中创建。

当创建s1时，JVM会先从字符串池中查找是否有等于"Hello"的String，如果相等，就把字符串池中的"Hello"赋值给

s1；如果不相等，就会在Heap中创建一个新对象，同时把引用驻留在字符串池中，在把引用赋值给s1。

##### 参考文档：

[彻底弄懂java中的常量池] https://cloud.tencent.com/developer/article/1450501

