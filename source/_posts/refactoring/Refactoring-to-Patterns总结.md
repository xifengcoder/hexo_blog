---
title: Refactoring to Patterns总结
urlname: refactoring
date: 2023-06-24 10:57:28
tags: 重构
categories: 重构
description: 重构
---

### Chapter 6: Creation

#### 6.1 Replace Constructors with Creation Methods

Constructors on a class make it hard to decide which constructor to call during development.

*Replace the constructors with intention-revealing Creation Methods that return object instances.*

***BENEFITS AND LIABILITIES***

&emsp;&emsp;\+ Communicates what kinds of instances are available better than constructors.

&emsp;&emsp;\+ Bypasses constructor limitations, such as the inability to have two constructors with the same number and type of arguments.

&emsp;&emsp;\+ Makes it easier to find unused creation code.

&emsp;&emsp;– Makes creation nonstandard: some classes are instantiated using new, while others use *Creation Methods*.

#### 6.2 Move Creation Knowledge to Factory

#### 6.3 Encapsulate Classes with Factory

##### Descriptio:

Clients directly instantiate classes that reside in one package and implement a common interface.

Make the class constructors non-public and let clients create instances of them using a Factory.

- Description

    Data and code used to instantiate a class is sprawled across numerous classes.

- Action

​        Move the creation knowledge into a single Factory class.



- Description

Constructors on a class make it hard to decide which constructor to call during development.

##### Encapsulate Classes with Factory

Clients directly instantiate classes that reside in one package and implement a common interface.

Make the class constructors non-public and let clients create instances of them using a Factory.



##### 6.5 Encapsulate Composite with Builder

Building a Composite is repetitive, complicated, or error-prone.

*Simplify the build by letting a Builder handle the details.*



##### 6.6 Inline Singleton

Code needs access to an object but doesn’t need a global point of access to it.

*Move the Singleton’s features to a class that stores and provides access to the object. Delete the Singleton.*

### Chapter 7: Simplification

#### 7.1 Compose Method

####  7.2 Replace Conditional Logic with Strategy

##### Description

Conditional logic in a method controls which of several variants of a calculation are executed.

Create a Strategy for each variant and make the method delegate the calculation to a Strategy instance.

#### Replace Implicit Tree with Composite

##### Description

You implicitly form a tree structure, using a primitive representation, such as a String.

Replace your primitive representation with a Composite.

####  Replace Conditional Dispatcher with Command

#### Replace State-Altering Conditionals with State

**Description**：

The conditional expressions that control an object’s state transitions are complex.

Replace the conditionals with State classes that handle specific states and transitions between them.

### Chapter 8. Generalization

#### 8.1 Form Template Method

Two methods in subclasses perform similar steps in the same order, yet the steps are different.

**BENEFITS AND LIABILITIES**

&emsp;&emsp;\+ Removes duplicated code in subclasses by moving invariant behavior to a superclass.

&emsp;&emsp;\+ Simplifies and effectively communicates the steps of a general algorithm.

&emsp;&emsp;\+ Allows subclasses to easily customize an algorithm.

&emsp;&emsp;– Complicates a design when subclasses must implement many methods to flesh out the algorithm.

#### 8.2 Extract Composite

Subclasses in a hierarchy implement the same Composite.

Extract a superclass that implements the Composite.

#### 8.3 Replace One/Many Distinctions with Composite

A class processes single and multiple objects using separate pieces of code.

*Use a Composite to produce one piece of code capable of handling single or multiple objects.*

####  8.4 Replace Hard-Coded Notifications with Observer

Subclasses are hard-coded to notify a single instance of another class.

*Remove the subclasses by making their superclass capable of notifying one or more instances of any class that implements an Observer interface.*

#### 8.5 Unify Interfaces with Adapter

使用Adapter统一接口

Clients interact with two classes, one of which has a preferred interface.

Unify the interfaces with an Adapter.

#### 8.6 Extract Adapter

One class adapts multiple versions of a component, library, API, or other entity.

Extract an Adapter for a single version of the component, library, API, or other entity.

**BENEFITS AND LIABILITIES**

&emsp;&emsp;\+  Isolates differences in versions of a component, library, or API.

&emsp;&emsp;\+  Makes classes responsible for adapting only one version of something.

&emsp;&emsp;\+  Provides insulation from frequently changing code.

&emsp;&emsp;–  Can shield a client from important behavior that isn’t available on the Adapter.

### 8.7 Replace Implicit Language with Interpreter

Numerous methods on a class combine elements of an implicit language.

*Define classes for elements of the implicit language so that instances may be combined to form interpretable expressions.*

### Chapter 9. Protection



### References

| Smell                       | Refactoring |
| --------------------------- | ----------- |
| 重复代码（Duplicated Code） |             |
| 过长函数（Long Method）     |             |
|                             |             |

