---
title: Java对象布局
urlname: java_object_layout
date: 2022-04-01 14:23:19
tags: Java
categories: Java
description: Java对象布局初探
---

在HotSpot虚拟机中，对象在内存中存储的布局可以分为3块区域：

- 对象头（Object Header）

- 实例数据（Instance Data)

- 对齐填充（Padding）

#### 一、 对象头（Object Header）

Object Header包含两部分：

1. MarkWord： 32位占4字节，64位占8字节；
2. KClass Pointer： 32位占4字节，64字节开启压缩占4字节、关闭压缩占8字节。

添加如下VM Options来关闭压缩：

```
-XX:-UseCompressedOops 
```

![object_header_32.png](/images/object_header_32.png)

<center><font size=1  color=gray>图1：Hotspot 32-bit对象布局</font></center>

![object_header_64](/images/object_header_64.png)
<center><font size=1  color=gray>图2：Hotspot 64-bit对象布局</font></center>

![object_header_64_coops](/images/object_header_64_coops.png)
<center><font size=1  color=gray>图3：Hotspot 64-bit带压缩的对象布局</font></center>

参考资料：https://gist.github.com/arturmkrtchyan/43d6135e8a15798cc46c

#### 二、代码测试：

```java
package net.lelyak.courses.jol;

import org.openjdk.jol.info.ClassLayout;
import org.openjdk.jol.util.VMSupport;

import static net.lelyak.mindview.util.Print.print;

public class JOLSample_12_ThinLocking {
    /*
     * This is another dive into the mark word.
     *
     * Among other things, mark words store locking information.
     * We can clearly see how the mark word contents change when
     * we acquire the lock, and release it subsequently.
     *
     * This one is the example of thin (displaced) lock. The data
     * in mark word when lock is acquired is the reference to the
     * displaced object header, allocated on stack. Once we leave
     * the lock, the displaced header is discarded, and mark word
     * is reverted to the default value.
     */

    public static void main(String[] args) throws Exception {
        print(VMSupport.vmDetails());

        final A a = new A();

        ClassLayout layout = ClassLayout.parseClass(A.class);

        print("**** Fresh object");
        print(layout.toPrintable(a));

        synchronized (a) {
            print("**** With the lock");
            print(layout.toPrintable(a));
        }

        print("**** After the lock");
        print(layout.toPrintable(a));
    }
    public static class A {
        // no fields
    }
}
```

##### 1. 开启指针压缩(默认)

```
/Library/Java/JavaVirtualMachines/jdk1.8.0_321.jdk/Contents/Home/bin/java

Running 64-bit HotSpot VM.
Using compressed oop with 3-bit shift.
Using compressed klass with 3-bit shift.
Objects are 8 bytes aligned.
Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]

**** Fresh object
net.lelyak.courses.jol.JOLSample_12_ThinLocking.A object internals:
 OFFSET  SIZE  TYPE DESCRIPTION                    VALUE
      0     4       (object header)                01 00 00 00 (0000 0001 0000 0000 0000 0000 0000 0000)
      4     4       (object header)                00 00 00 00 (0000 0000 0000 0000 0000 0000 0000 0000)
      8     4       (object header)                f0 24 01 f8 (1111 0000 0010 0100 0000 0001 1111 1000)
     12     4       (loss due to the next object alignment)
Instance size: 16 bytes (reported by Instrumentation API)
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** With the lock
net.lelyak.courses.jol.JOLSample_12_ThinLocking.A object internals:
 OFFSET  SIZE  TYPE DESCRIPTION                    VALUE
      0     4       (object header)                d8 f9 c4 05 (1101 1000 1111 1001 1100 0100 0000 0101)
      4     4       (object header)                03 00 00 00 (0000 0011 0000 0000 0000 0000 0000 0000)
      8     4       (object header)                f0 24 01 f8 (1111 0000 0010 0100 0000 0001 1111 1000)
     12     4       (loss due to the next object alignment)
Instance size: 16 bytes (reported by Instrumentation API)
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** After the lock
net.lelyak.courses.jol.JOLSample_12_ThinLocking.A object internals:
 OFFSET  SIZE  TYPE DESCRIPTION                    VALUE
      0     4       (object header)                01 00 00 00 (0000 0001 0000 0000 0000 0000 0000 0000)
      4     4       (object header)                00 00 00 00 (0000 0000 0000 0000 0000 0000 0000 0000)
      8     4       (object header)                f0 24 01 f8 (1111 0000 0010 0100 0000 0001 1111 1000)
     12     4       (loss due to the next object alignment)
Instance size: 16 bytes (reported by Instrumentation API)
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

![object_header_sample](/images/object_header_sample.png)

##### 2. 关闭指针压缩

```txt
/Library/Java/JavaVirtualMachines/jdk1.8.0_321.jdk/Contents/Home/bin/java -XX:-UseCompressedOops

Running 64-bit HotSpot VM.
Objects are 8 bytes aligned.
Field sizes by type: 8, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
Array element sizes: 8, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]

**** Fresh object
net.lelyak.courses.jol.JOLSample_12_ThinLocking.A object internals:
 OFFSET  SIZE  TYPE DESCRIPTION                    VALUE
      0     4       (object header)                05 00 00 00 (0000 0101 0000 0000 0000 0000 0000 0000)
      4     4       (object header)                00 00 00 00 (0000 0000 0000 0000 0000 0000 0000 0000)
      8     4       (object header)                c0 7f e9 23 (1100 0000 0111 1111 1110 1001 0010 0011)
     12     4       (object header)                01 00 00 00 (0000 0001 0000 0000 0000 0000 0000 0000)
Instance size: 16 bytes (reported by Instrumentation API)
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

**** With the lock
net.lelyak.courses.jol.JOLSample_12_ThinLocking.A object internals:
 OFFSET  SIZE  TYPE DESCRIPTION                    VALUE
      0     4       (object header)                05 f8 80 a8 (0000 0101 1111 1000 1000 0000 1010 1000)
      4     4       (object header)                a6 7f 00 00 (1010 0110 0111 1111 0000 0000 0000 0000)
      8     4       (object header)                c0 7f e9 23 (1100 0000 0111 1111 1110 1001 0010 0011)
     12     4       (object header)                01 00 00 00 (0000 0001 0000 0000 0000 0000 0000 0000)
Instance size: 16 bytes (reported by Instrumentation API)
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

**** After the lock
net.lelyak.courses.jol.JOLSample_12_ThinLocking.A object internals:
 OFFSET  SIZE  TYPE DESCRIPTION                    VALUE
      0     4       (object header)                05 f8 80 a8 (0000 0101 1111 1000 1000 0000 1010 1000)
      4     4       (object header)                a6 7f 00 00 (1010 0110 0111 1111 0000 0000 0000 0000)
      8     4       (object header)                c0 7f e9 23 (1100 0000 0111 1111 1110 1001 0010 0011)
     12     4       (object header)                01 00 00 00 (0000 0001 0000 0000 0000 0000 0000 0000)
Instance size: 16 bytes (reported by Instrumentation API)
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

![object_header_sample_nocompress](/images/object_header_sample_nocompress.png)
