---
title: Java位运算的理解
urlname: bit_operation
date: 2021-12-26 17:57:34
tags: Java
categories: Java
description: Java中的位操作的理解...
---

#### 一、有符号整数的存储

有符号整数通常用补码来表示和存储，补码的特征：

* 正整数的补码与原码相同，负整数的原码为除符号位以外的所有位取反后加1；

#### 二、Java中的负数存储

举例: 
-20
调用Integer.toHexString(-20)返回结果：ffffffec。
其中toHexString(int i)返回将整型参数作为无符号整数时的16进制字符串表示形式。这里的无符号数取值规则为：

1. 如果实参为负数，则对应的无符号数为其值加上2^32；
2. 否则，无符号数为实参本身。
```java
/**
 * Returns a string representation of the integer argument as an unsigned integer in base 16.
 */
public static String toHexString(int i);
```
Java中的无符号数是以补码的形式存储的。

![Java位运算1](/images/Java位运算1.jpg)


#### 三、>>与>>>
##### 3.1 >>: 带符号位的右移
带符号位右移时，保留符号位，正数右移高位补0，负数右移高位补1。
a = -20;
a >> 2后：

![Java位运算2](/images/Java位运算2.png)


```
int a = -20;
int b = a >> 2;
System.out.println(b); //-5
System.out.println(Integer.toHexString(b)); //fffffffb
```

##### 3.2 >>>: 不带符号位的右移
不带符号位右移时，高位统统补0。
a = -20;
a >>> 2后，变为正数。

![Java位运算3](/images/Java位运算3.png)

```java
int a = -20;
int b = a >>> 2;
System.out.println(b); //1073741819
System.out.println(Integer.toHexString(b)); //3ffffffb
```