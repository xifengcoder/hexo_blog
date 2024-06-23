---
title: Android中常用的设计模式总结
urlname: Android中常用的设计模式总结
date: 2024-02-26 22:55:32
tags:
categories:
description:
---

### MVP Vs MVVM

1. **解耦：**
    - 两者都旨在实现解耦，以便更容易测试和维护代码。
2. **分离关注点：**
    - 都倡导将关注点分离，以确保清晰的代码组织。
3. **单一职责原则：**
    - 两者都遵循单一职责原则，确保每个组件都有一个清晰的责任。

| MVP                                                          | MVVM                                |
| ------------------------------------------------------------ | ----------------------------------- |
| Presenter和View一般是1v1的关系，耦合度较高，难以实现复用     | 多个View可以关联一个ViewModel       |
| Presenter不会感知View的生命周期，需要Presenter手动管理生命周期，避免内存泄漏 | ViewModel可以自动感知View的生命周期 |
| Presenter会持有View的引用,                                   | ViewModel不持有View的引用           |
|                                                              | 支持DataBinding                     |

