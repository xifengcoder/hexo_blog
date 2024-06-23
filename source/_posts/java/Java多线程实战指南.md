---
title: Java多线程实战指南
urlname:  multi_thread
date: 2024-05-01 22:44:59
tags: 多线程
categories: 多线程
description: Java多线程编程指南
---

内存屏障（Memory Barrier）

加载屏障（Load Barrier）

刷新处理器缓存，用于保证一个处理器上的读线程能够读取到其它处理器上的线程对共享变量所做的更新；



存储屏障（Store Barrier）

冲刷处理器缓存，用于保证一个处理器上的写线程对共享变量所做的更新能够同步到其它处理器上的线程。





获取屏障（Acquire Barrier）

在一个读操作之后插入该内存屏障。其作用是禁止该读操作与其后的任何读写操作之间进行重排序。





释放屏障（Release Barrier）

在一个写操作之前插入该屏障。其作用是禁止该写操作与其前面的任何读写操作之间进行重排序。





