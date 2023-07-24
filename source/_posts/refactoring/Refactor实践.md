---
title: Refactor实践
urlname: refactoring
date: 2023-05-29 23:00:29
tags: 重构
categories: 重构
description: 重构
---

### Refactoring: Improving the Design of Existing Code

重构的第一步

每当我们要进行重构的时候，第一个步骤永远相同：为即将修改的代码建立一组可靠的测试环境。

### Chapter 6: Composing Method

#### 6.1 Extract Method(提炼函数)

You have a code fragment that can be grouped together.

Turn the fragment into a method whose name explains the purpose of the method.

#### 6.2 Inline Method(内联函数)

A method's body is just as clear as its name.
Put the method's body into the body of its callers and remove the method.

####  6.3 Inline Temp(内敛临时变量)

You have a temp that is assigned to once with a simple expression, and the temp is getting in the
way of other refactorings.
Replace all references to that temp with the expression.

#### 6.4 Replace Temp with Query(以查询取代临时变量)

You are using a temporary variable to hold the result of an expression.
Extract the expression into a method. Replace all references to the temp with the expression. The
new method can then be used in other methods.

#### 6.5 Introduce Explaning Variable(引入解释性变量)

You have a complicated expression.
Put the result of the expression, or parts of the expression, in a temporary variable with a name that explains the purpose.

#### 6.6 Split Temporary Variable(分解临时变量)

You have a temporary variable assigned to more than once, but is not a loop variable nor a collecting temporary variable.
Make a separate temporary variable for each assignment.

#### 6.7 Remove Assignments to Parameters(移除对参数的赋值)

The code assigns to a parameter.
Use a temporary variable instead.

### Chapter 7: Moving Features Between Objects

#### 7.1 Move Method

A method is, or will be, using or used by more features of another class than the class on which it is defined.
Create a new method with a similar body in the class it uses most. Either turn the old method into a simple delegation, or remove it altogether.

当我需要使用源类的某个特性(feature)时，我有四种选择：

1)  move this feature to the target class as well;
2)  create or use a reference from the target class to the source;
3)  pass the source object as a parameter to the method;
4)  if the feature is a variable, pass it in as a parameter.

#### 7.2 Move Field

A field is, or will be, used by another class more than the class on which it is defined.
Create a new field in the target class, and change all its users.

#### 7.3 Extract Class

You have one class doing work that should be done by two.

Create a new class and move the relevant fields and methods from the old class into the new class.

#### 7.4 Inline Class

A class isn't doing very much.
Move all its features into another class and delete it.

#### 7.5 	Hide Delegate

A client is calling a delegate class of an object.
Create methods on the server to hide the delegate.

### Chapter 8: Organizing Data

### Chapter 9: Simplifying Conditional Expressions

### Chapter 10: Making Method Calls Simpler

### Chapter 11: Dealing with Generalization

### Chapter 12: Big Refactorings

