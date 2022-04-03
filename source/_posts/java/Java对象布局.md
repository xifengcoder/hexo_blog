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

####  一、 对象头（Object Header）

Object Header包含两部分：

1. MarkWord：32位占4字节，64位占8字节；

2. KClass Pointer：32位占4字节，64字节开启压缩占4字节、关闭压缩占8字节

   如果是数组实例（可能是基本类型数组，也可能是对象数组），则包含第3个：

2. Array length：4字节。

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

另外参见注释 [MarkOop](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/tip/src/share/vm/oops/markOop.hpp#l3)

```c++
// The markOop describes the header of an object.
//
// Note that the mark is not a real oop but just a word.
// It is placed in the oop hierarchy for historical reasons.
//
// Bit-format of an object header (most significant first, big endian layout below):
//
//  32 bits:
//  --------
//             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//             size:32 ------------------------------------------>| (CMS free block)
//             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
//
//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
//  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
//  size:64 ----------------------------------------------------->| (CMS free block)
//  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
//  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
//  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
//  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
//
//  - hash contains the identity hash value: largest value is
//    31 bits, see os::random().  Also, 64-bit vm's require
//    a hash value no bigger than 32 bits because they will not
//    properly generate a mask larger than that: see library_call.cpp
//    and c1_CodePatterns_sparc.cpp.
//
//  - the biased lock pattern is used to bias a lock toward a given
//    thread. When this pattern is set in the low three bits, the lock
//    is either biased toward a given thread or "anonymously" biased,
//    indicating that it is possible for it to be biased. When the
//    lock is biased toward a given thread, locking and unlocking can
//    be performed by that thread without using atomic operations.
//    When a lock's bias is revoked, it reverts back to the normal
//    locking scheme described below.
//
//    Note that we are overloading the meaning of the "unlocked" state
//    of the header. Because we steal a bit from the age we can
//    guarantee that the bias pattern will never be seen for a truly
//    unlocked object.
//
//    Note also that the biased state contains the age bits normally
//    contained in the object header. Large increases in scavenge
//    times were seen when these bits were absent and an arbitrary age
//    assigned to all biased objects, because they tended to consume a
//    significant fraction of the eden semispaces and were not
//    promoted promptly, causing an increase in the amount of copying
//    performed. The runtime system aligns all JavaThread* pointers to
//    a very large value (currently 128 bytes (32bVM) or 256 bytes (64bVM))
//    to make room for the age bits & the epoch bits (used in support of
//    biased locking), and for the CMS "freeness" bit in the 64bVM (+COOPs).

//  - the two lock bits are used to describe three states: locked/unlocked and monitor.
//
//    [JavaThread* | epoch | age | 1 | 01]       biased lock, lock is biased toward given thread
//    [0           | epoch | age | 1 | 01]       biased lock, lock is anonymously biased
//    [header                    | 0 | 01]       unlocked(normal object), regular object header
//    [ptr                           | 00]       locked, ptr points to real header on stack
//    [ptr                           | 10]       monitor, inflated lock (header is wapped out)
//    [ptr                           | 11]       marked, used by markSweep to mark an object
```

其中低三位bit的值如下：

```c++
enum { 
    locked_Value = 0, // 00 lightweight lock
    unlocked_Value = 1, // 0 01 no lock,
    monitor_Value = 2, // 10 heavyweight lock
    marked_Value = 3, // 11 GC flag
    biased_lock_Pattern = 5 // 1 01 bias lock
};
```

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

#### 三、锁升级

```java
public class JOLSample_14_FatLocking {
    /*
     * This is the example of fat locking.
     *
     * If VM detects contention on thread, it needs to delegate the
     * access arbitrage to OS. That involves associating the object
     * with the native lock, i.e. "inflating" the lock.
     *
     * In this example, we need to simulate the contention, this is
     * why we have the additional thread. You can see the fresh object
     * has the default mark word, the object before the lock was already
     * acquired by the auxiliary thread, and when the lock was finally
     * acquired by main thread, it had been inflated. The inflation stays
     * there after the lock is released. You can also see the lock is
     * "deflated" after the GC (the lock cleanup proceeds in safepoints,
     * actually).
     */

    public static void main(String[] args) throws Exception {
        //print(VMSupport.vmDetails());
        TimeUnit.SECONDS.sleep(5);

        final A a = new A();
        ClassLayout layout = ClassLayout.parseClass(A.class);

        print("**** Fresh object");
        //加了5s的sleep后, Fresh Object为偏向锁状态.
        print(layout.toPrintable(a)); //偏向锁状态, 无threadId(101)

        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (a) {
                    print("**** Thread lock:");
                    print(layout.toPrintable(a)); //偏向锁状态带threadId(101)
                    try {
                        TimeUnit.SECONDS.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        t.start();
        TimeUnit.SECONDS.sleep(1);

        synchronized (a) {
            print("**** With the lock");
            print(layout.toPrintable(a)); //重量级锁010, inflated lock.
        }

        //3. 重量级锁仍旧呆在这儿（010）.
        print("**** After the lock");
        print(layout.toPrintable(a));

        System.gc();

        //4. GC之后锁被重置（001）, deflated lock.
        print("**** After System.gc()");
        print(layout.toPrintable(a));
    }

    public static class A {
        // no fields
    }
}
```

