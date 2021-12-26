---
title: 深入理解Kotlin协程
urlname: undestanding_kotlin
date: 2021-12-11 13:36:51
tags: Kotlin
categories: Kotlin
description: Kotlin协程实践
---

### Kotlin协程实践

#### 1. suspend

suspend关键字用在function前面，表示一个function在执行期间可以被paused或resumed。

```kotlin
suspend fun sample() {
}
```

注意：一个suspend函数只能在coroutine或者另一个suspend函数中被调用。

#### 2. Job

Job是coroutine的句柄（handle），每一个使用launch或者async创建的coroutine会返回一个Job实例，用来唯一标识该coroutine、并且管理其生命周期。

#### 3. Dispatchers

用于指定操作是在哪个线程上执行。Android系统有是三个Dispatchers。

* Dispatchers.Main
* Dispatchers.Default
* Dispatchers.IO

通过使用withContext来在不同的Dispatchers中切换。

#### 4. CoroutineScope

* 管理Coroutines的生命周期

  

  一个CoroutineScope定义了在其上启动的Coroutines的生命周期。CoroutineScope在一旦创建后就会启动，在它被取消时、或者它关联的Job或 SupervisorJob完成时结束。

* CoroutineScope的创建

  

  如果传入的context参数不包含Job元素，则会使用默认的Job( )，否则会使用传入的Job对象。
```kotlin
@Suppress("FunctionName")
public fun CoroutineScope(context: CoroutineContext): CoroutineScope =
    ContextScope(if (context[Job] != null) context else context + Job())
```
举例：
```kotlin
val scope1 = CoroutineScope(Dispatchers.Main + SupervisorJob())
val scope2 = MainScope();
```
另外一种用法：

```kotlin
class SomethingWithALifecycle : CoroutineScope {
    override val coroutineContext = Dispatchers.IO + SupervisorJob()
    ...
}

//或者
class SomethingWithALifecycle {
    val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    ...
    fun destroy() {
        scope.cancel()
    }
}
```

GlobalScope: 没有绑定任何Job的Scope。

```kotlin
public object GlobalScope : CoroutineScope {
    /**
     * Returns [EmptyCoroutineContext].
     */
    override val coroutineContext: CoroutineContext
        get() = EmptyCoroutineContext
}
```

#### 5. CoroutineContext



是CoroutineScope的上下文（context）。CoroutineContext封裝在CoroutineScope中的一个属性，在实现CoroutineBuilders时会用到，CoroutineBuilders是定义在CoroutineScope上的扩展。通常情况下，不推荐直接使用coroutineContext，除了访问Job实例进行一些高级用法。

一个CoroutineContext通过使用如下的四种元素来定义coroutine的行为：

* Job：控制coroutine的生命周期；
* CoroutineDispatcher：将任务分发到相应的线程上；
* CoroutineName：coroutine的名称，调试时很有用；
* CoroutineExceptionHandler：处理未捕获的异常 。


####  6. 协程的创建



启动一个新的协程, 常用的主要有以下几种方式，它们被称为`coroutine builders`。

1. **launch**
launch为定义在CoroutineScope上的扩展函数，接受一个CoroutineContext参数（默认为EmptyCoroutineContext）、一个CoroutineStart参数（默认为CoroutineStart.DEFAULT），以及一个定义在CoroutineScope上的suspend函数，类型为suspend CoroutineScope.() -> Unit，返回类型为Job。

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

2. **async**
async为定义在CoroutineScope上的扩展函数，接受一个CoroutineContext参数（默认为EmptyCoroutineContext）、一个CoroutineStart参数（默认为CoroutineStart.DEFAULT），以及一个定义在CoroutineScope上的suspend函数，类型为suspend CoroutineScope.() -> Unit，返回类型为Deferred<T>。

```kotlin
public fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T> {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyDeferredCoroutine(newContext, block) else
        DeferredCoroutine<T>(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

3.  **runBlocking**

runBlocking用来连接non-coroutine世界的代码和协程代码的桥梁。
runBlocking可以建立一个阻塞当前线程的协程. 所以它主要被用来在main函数中或者测试中使用, 作为连接函数。

#### 7. 上下文切换withContext

withContext是一个suspend函数，它会等待代码块中所有的corotines都完成以后再结束。如果其中任一coroutine失败，则整个代码块会抛出异常、并会自动取消代码块中其他的coroutines。但是withContext外的coroutines不受影响。

#### 8. Kotlin协程中的异常处理

Coroutine Builders处理异常一般分两种情况：

1. 自动传播异常（通过launch启动时）；
2. 将异常抛给用户（通过async启动时）：依赖用户去捕获该异常，否则程序会崩溃；



举例：

```kotlin
fun main() = runBlocking {
    val job = GlobalScope.launch { // root coroutine with launch
        println("Throwing exception from launch")
        throw IndexOutOfBoundsException() // Will be printed to the console by Thread.defaultUncaughtExceptionHandler
    }
    job.join()
    println("Joined failed job")
    val deferred = GlobalScope.async { // root coroutine with async
        println("Throwing exception from async")
        throw ArithmeticException() // Nothing is printed, relying on user to call await
    }
    try {
        deferred.await()
        println("Unreached") //不会被打印出来，因为会走到catch中。
    } catch (e: ArithmeticException) {
        println("Caught ArithmeticException")
    }
}
```


![coroutine_exception_example](/coroutine_exception_example.png)

