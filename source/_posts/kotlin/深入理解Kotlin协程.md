---
title: 深入理解Kotlin协程
urlname: undestanding_kotlin
date: 2021-12-11 13:36:51
tags: Kotlin
categories: Kotlin
description: Kotlin协程实践
---

#### 1. suspend关键字

suspend关键字用在function前面，表示一个function在执行期间可以被paused或resumed。

```kotlin
suspend fun sample() {
}
```

注意：一个suspend函数只能在coroutine或者另一个suspend函数中被调用。

#### 2. Coroutine的生命周期

![Coroutine的状态](/images/coroutine_state.JPG)

#### 3. CoroutineContext

CoroutineContext是CoroutineScope的上下文（context）。CoroutineContext是封裝在CoroutineScope中的一个属性，在实现CoroutineBuilders时会用到，CoroutineBuilders是定义在CoroutineScope上的扩展。通常情况下，不推荐直接使用coroutineContext，除了访问Job实例进行一些高级用法之外。

一个CoroutineContext通过使用如下的四种元素来定义coroutine的行为：

* Job：控制coroutine的生命周期；
* CoroutineDispatcher：将任务分发到相应的线程上，默认值为Dispatchers.Default；
* CoroutineName：coroutine的名称，调试时很有用，默认值为"coroutine"；
* CoroutineExceptionHandler：处理未捕获的异常 ，默认值为None。

每个元素的默认值如下：

![CoroutineContext的元素](/images/coroutine_context.JPG)

##### 3.1 Job

Job是coroutine的句柄（handle），每一个使用launch或者async创建的coroutine会返回一个Job实例，用来唯一标识该coroutine、并且管理其生命周期。

##### 3.2 Dispatchers

用于指定操作是在哪个线程上执行。Android系统有是三个Dispatchers。

* Dispatchers.Main
* Dispatchers.Default
* Dispatchers.IO

通过使用withContext来在不同的Dispatchers中切换。

CoroutineContext的获取：

![CoroutineContext获取](/images/coroutine_context_default.JPG)

##### 3.3 CoroutineExceptionHandler

![Coroutine Exception Handler](/images/coroutine_exception_handler.JPG)

###### 3.3.1 在构建CoroutineScope时，传入handler可以捕捉到异常

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    println("Caught $exception")
}
val scope = CoroutineScope(Job() + handler)
val job = scope.launch(handler) {
    launch {
        throw Exception("Failed coroutine")
    }
}
job.join()
```

###### 3.3.2 在scope的launch函数中安装handler，可以捕捉到异常

```kotlin
 val handler = CoroutineExceptionHandler { _, exception ->
    println("Caught $exception")
}

val scope = CoroutineScope(Job())
val job = scope.launch(handler) {
    launch {
        throw Exception("Failed coroutine")
    }
}
job.join()
```

###### 3.3.3 handler被安装到内部协程时，无法捕获到异常

 异常不会被捕获的原因是因为 handler 没有被安装给正确的 CoroutineContext。内部协程会在异常出现时传播异常并传递给它的父级，由于父级并不知道 handler 的存在，异常就没有被捕获。 

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    println("Caught $exception")
}

val scope = CoroutineScope(Job())
val job = scope.launch {
    //WARN: handler被安装到内部协程时，无法捕获异常
    launch(handler) {
        throw Exception("Failed coroutine")
    }
}
job.join()
```

##### coroutineScope vs supervisorScope

（1）使用coroutineScope

```kotlin
fun main() {
    runBlocking {
        val handler = CoroutineExceptionHandler { _, exception ->
            println("Caught $exception")
        }

        val scope = coroutineScope {
            launch(handler) {
                throw Exception("Failed coroutine")
            }
        }
        scope.join();
        println("Done!")
    }
}

//output:
Exception in thread "main" java.lang.Exception: Failed coroutine
	at com.yxf.kotlin.coroutines.exception_handling.CoroutineExceptionHandlerKt$main$1$scope$1$1.invokeSuspend(CoroutineExceptionHandler.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(Dispatched.kt:233)
	at kotlinx.coroutines.EventLoopImplBase.processNextEvent(EventLoop.kt:116)
	at kotlinx.coroutines.BlockingCoroutine.joinBlocking(Builders.kt:76)
	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking(Builders.kt:53)
	at kotlinx.coroutines.BuildersKt.runBlocking(Unknown Source)
	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking$default(Builders.kt:35)
	at kotlinx.coroutines.BuildersKt.runBlocking$default(Unknown Source)
	at com.yxf.kotlin.coroutines.exception_handling.CoroutineExceptionHandlerKt.main(CoroutineExceptionHandler.kt:8)
	at com.yxf.kotlin.coroutines.exception_handling.CoroutineExceptionHandlerKt.main(CoroutineExceptionHandler.kt)

Process finished with exit code 1
```

（2）使用supervisorScope

```kotlin
fun main() {
    runBlocking {
        val handler = CoroutineExceptionHandler { _, exception ->
            println("Caught $exception")
        }

        val scope = supervisorScope {
            launch(handler) {
                throw Exception("Failed coroutine")
            }
        }
        scope.join();
        println("Done!")
    }
}
//output:
Caught java.lang.Exception: Failed coroutine
Done!

Process finished with exit code 0
```

#### 4. CoroutineScope

CoroutineScope会追踪每一个您通过launch或async创建的协程，任何时候都可以通过scope.cancel()来取消正在运行的协程。

- 管理Coroutines的生命周期

​    一个CoroutineScope定义了在其上启动的Coroutines的生命周期。CoroutineScope在一旦创建后就会启动，在它被取消时、或者它关联的Job或 SupervisorJob完成时结束。

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


####  5. 协程的启动

启动一个新的协程, 常用的主要有以下几种方式，它们被称为`coroutine builders`。

##### 5.1 launch

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

##### 5.2 async

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

##### 5.3 runBlocking

runBlocking用来连接non-coroutine世界的代码和协程代码的桥梁。
runBlocking可以建立一个阻塞当前线程的协程. 所以它主要被用来在main函数中或者测试中使用, 作为连接函数。

##### 5.4 coroutineScope

##### 5.5 supervisorScope

#### 6. 上下文切换withContext

withContext是一个suspend函数，它会等待代码块中所有的corotines都完成以后再结束。如果其中任一coroutine失败，则整个代码块会抛出异常、并会自动取消代码块中其他的coroutines。但是withContext外的coroutines不受影响。

#### 7. Kotlin协程中的异常处理

##### 7.1 launch或async对异常的默认处理

Coroutine Builders处理异常一般分两种情况：

1. 自动传播异常（通过launch启动时）；
2. 将异常抛给用户（通过async启动时）：依赖用户去捕获该异常，否则程序会崩溃；

举例：

```kotlin
package com.yxf.kotlin.coroutines.exception_handling

import kotlinx.coroutines.GlobalScope
import kotlinx.coroutines.async
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking
import org.apache.logging.log4j.LogManager
import org.apache.logging.log4j.Logger

val logger: Logger = LogManager.getLogger()

fun main() = runBlocking {
    val asyncJob = GlobalScope.launch {
        logger.info("1. Exception created via launch coroutine")
        //异常信息由Thread.defaultUncaughtExceptionHandler打印在控制台
        throw IndexOutOfBoundsException()
    }

    asyncJob.join()
    logger.info("2. Joined failed job")

    val deferred = GlobalScope.async {
        logger.info("Exception created via async coroutine")
        //不会有异常信息打印，依赖用户调用await()
        throw ArithmeticException()
    }

    try {
        deferred.await()
        logger.info("4. Unreachable, this statement is never executed")
    } catch (e: Exception) {
        //捕捉async()抛出的异常
        logger.error("5. Caught ${e.javaClass.simpleName}")
    }
}

//output:
2022-01-30 20:40:57.760[DefaultDispatcher-worker-2] INFO Main.kt[14]: 1. Exception created via launch coroutine
Exception in thread "DefaultDispatcher-worker-2" java.lang.IndexOutOfBoundsException
	at com.yxf.kotlin.coroutines.exception_handling.MainKt$main$1$asyncJob$1.invokeSuspend(Main.kt:17)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(Dispatched.kt:233)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:594)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.access$runSafely(CoroutineScheduler.kt:60)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:742)
2022-01-30 20:40:57.826[main] INFO Main.kt[21]: 2. Joined failed job
2022-01-30 20:40:57.835[DefaultDispatcher-worker-2] INFO Main.kt[24]: Exception created via async coroutine
2022-01-30 20:40:57.840[main] ERROR Main.kt[33]: 5. Caught ArithmeticException
```

##### 7.2 某个协程突然运行失败怎么办? —Job VS SupervisorJob

​    当某一个协程由于异常而运行失败时，默认情况下它会传播这个异常并传递给它父级。接下来，父级会进行下面的操作：

取消它自己的Child；

取消它自己；

将异常传播并传递给它的父级；

![Coroutine的状态](/images/coroutine_exception.gif)

​    如果想要在某一个协程出现错误时，不会退出父级和影响其他的兄弟协程，就需要使用 SupervisorJob 或 supervisorScope 。具体实现有三种：

(1) 在构造CoroutineScope时，使用SuperVisorJob()；

```kotlin
//使用SupervisorJob()可以在jobs1异常中止时，不影响job2的正常运行。
val scope = CoroutineScope(SupervisorJob())
        
val job1 = scope.launch {
    logger.info("Launching child1")
    delay(500L)
    throw ArithmeticException()
}

val job2 = scope.launch {
    logger.info("Launching child2")
    delay(5000L)
    logger.info("Child2 completed")
}

logger.info("Waiting for jobs finish...")
job1.join()
job2.join()
logger.info("Jobs Done...")
```

(2) 将SupervisorJob安装在launch函数上；

```kotlin
val scope = CoroutineScope(Job())
val supervisorJob = SupervisorJob()

// 将SupervisorJob安装在launch函数上；
val job1 = scope.launch(supervisorJob) {
    logger.info("Launching child1")
    delay(500L)
    throw ArithmeticException()
}

val job2 = scope.launch(supervisorJob) {
    logger.info("Launching child2")
    delay(5000L)
    logger.info("Child2 completed")
}

logger.info("Waiting for jobs finish...")
job1.join()
job2.join()
logger.info("Jobs Done...")
```

(3) 使用coroutineScope

```kotlin
runBlocking {
    //使用coroutineScope时，Child 1的异常不影响Child 2的正常执行完成。
    coroutineScope {
        launch {
            // Child 1
            logger.info("Launching child1")
            delay(500L)
            throw ArithmeticException()
        }
        launch {
            // Child 2
            logger.info("Launching child2")
            delay(5000L)
            logger.info("Child2 completed")
        }
    }
    logger.info("Jobs Done...")
}
```

##### 7.3 多个子任务同时抛出异常的处理

​    如果多个子任务都会抛出异常时，CoroutineExceptionHandler只会捕捉到最先抛出的异常，后面的异常会被屏蔽掉，通过Throwable的getSuppressed()方法可以查看被屏蔽的异常。

```java
package com.yxf.kotlin.coroutines.exception_handling

import kotlinx.coroutines.*
import org.apache.logging.log4j.LogManager
import org.apache.logging.log4j.Logger

fun main() {
    val logger: Logger = LogManager.getLogger()

    runBlocking {
        val handler = CoroutineExceptionHandler { context, exception ->
            //捕捉到的是Job2的IllegalStateException异常，Job1的ArithmeticException异常被屏蔽掉了，
            //通过Throwable的getSuppressed()方法可以查看。
            logger.info(
                "Caught $exception with suppressed " +
                        exception.suppressed?.contentToString()
            )
        }

        //Parent Job
        val parentJob = GlobalScope.launch(handler) {
            //Child Job 1
            launch {
                try {
                    delay(Long.MAX_VALUE)
                } catch (e: Exception) {
                    //JobCancellationException.
                    logger.error("${e.javaClass.simpleName} in Child Job 1")
                } finally {
                    throw ArithmeticException()
                }
            }
            
            //Child Job 2            
            launch {
                delay(100)
                throw IllegalStateException()
            }

            delay(Long.MAX_VALUE)
        }

        //Wait until parentJob completes
        parentJob.join()
        logger.info("All Jobs completed")
    }
}

//output:
2022-01-30 22:50:07.273[DefaultDispatcher-worker-2] ERROR Main04.kt[28]: JobCancellationException in Child Job 1
2022-01-30 22:50:07.333[DefaultDispatcher-worker-2] INFO CoroutineExceptionHandler.kt[101]: Caught java.lang.IllegalStateException with suppressed [java.lang.ArithmeticException]
2022-01-30 22:50:07.333[main] INFO Main04.kt[44]: All Jobs completed
```

#### 8. Coroutine的取消

A **cancelled** scope cannot start more coroutines.

A **cancelled** child doesn't affect other siblings.

A **cancelling** coroutine is not able to suspend.

join()/await() — Suspend until the job is completed.

join() — No action if completed

await() — Throws if completed

##### 8.1 join() 

① 先cancel( )后join( )

```kotlin
fun main() {
    val logger = LogManager.getLogger()

    runBlocking {
        val startTime = System.currentTimeMillis()
        val job = launch(Dispatchers.Default) {
            var nextPrintTime = startTime
            var i = 0;

            while(i < 5) {
                if(System.currentTimeMillis() - nextPrintTime > 500) {
                    logger.info("Hello $i, isActive: " + isActive)
                    nextPrintTime += 500
                    i++
                }
            }
        }

        delay(1000L)
        logger.info("Cancelling...")
        job.cancel()
        logger.info("Cancelled")
        job.join()
        logger.info("Joined")
    }
}

//output:
2022-02-05 12:23:08.635[DefaultDispatcher-worker-1] INFO Main.kt[19]: Hello 0, isActive: true
2022-02-05 12:23:09.123[DefaultDispatcher-worker-1] INFO Main.kt[19]: Hello 1, isActive: true
2022-02-05 12:23:09.162[main] INFO Main.kt[27]: Cancelling...
2022-02-05 12:23:09.165[main] INFO Main.kt[29]: Cancelled
2022-02-05 12:23:09.623[DefaultDispatcher-worker-1] INFO Main.kt[19]: Hello 2, isActive: false
2022-02-05 12:23:10.123[DefaultDispatcher-worker-1] INFO Main.kt[19]: Hello 3, isActive: false
2022-02-05 12:23:10.623[DefaultDispatcher-worker-1] INFO Main.kt[19]: Hello 4, isActive: false
2022-02-05 12:23:10.626[main] INFO Main.kt[31]: Joined

Process finished with exit code 0
```

② 先join( )后cancel( )

```java
fun main() {
    val logger = LogManager.getLogger()

    runBlocking {
        val startTime = System.currentTimeMillis()
        val job = launch(Dispatchers.Default) {
            var nextPrintTime = startTime
            var i = 0;

            while(i < 5) {
                if(System.currentTimeMillis() - nextPrintTime > 500) {
                    logger.info("Hello $i, isActive: " + isActive)
                    nextPrintTime += 500
                    i++
                }
            }
        }

        delay(1000L)
        job.join()
        logger.info("Joined")
        logger.info("Cancelling...")
        job.cancel()
        logger.info("Cancelled")
    }
}

//output:
2022-02-05 12:16:59.058[DefaultDispatcher-worker-1] INFO Main.kt[19]: Hello 0, isActive: true
2022-02-05 12:16:59.544[DefaultDispatcher-worker-1] INFO Main.kt[19]: Hello 1, isActive: true
2022-02-05 12:17:00.044[DefaultDispatcher-worker-1] INFO Main.kt[19]: Hello 2, isActive: true
2022-02-05 12:17:00.544[DefaultDispatcher-worker-1] INFO Main.kt[19]: Hello 3, isActive: true
2022-02-05 12:17:01.044[DefaultDispatcher-worker-1] INFO Main.kt[19]: Hello 4, isActive: true
2022-02-05 12:17:01.047[main] INFO Main.kt[28]: Joined
2022-02-05 12:17:01.048[main] INFO Main.kt[29]: Cancelling...
2022-02-05 12:17:01.049[main] INFO Main.kt[31]: Cancelled

Process finished with exit code 0
```

#####  8.2 wait()

① 先await( )后cancel( )

```kotlin
fun main() {
    val logger = LogManager.getLogger()

    runBlocking {
        val job = async(Dispatchers.Default) {
            repeat(5) {
                logger.info("Hello $it")
                delay(500)
            }
        }

        delay(1000L)
        val result = job.await()
        logger.info("Awaited, result: $result")
        logger.info("Cancelling...")
        job.cancel()
        logger.info("Cancelled")
    }
}

//output:
2022-02-05 12:31:58.399[DefaultDispatcher-worker-2] INFO Main01.kt[15]: Hello 0
2022-02-05 12:31:58.915[DefaultDispatcher-worker-2] INFO Main01.kt[15]: Hello 1
2022-02-05 12:31:59.416[DefaultDispatcher-worker-3] INFO Main01.kt[15]: Hello 2
2022-02-05 12:31:59.924[DefaultDispatcher-worker-3] INFO Main01.kt[15]: Hello 3
2022-02-05 12:32:00.426[DefaultDispatcher-worker-4] INFO Main01.kt[15]: Hello 4
2022-02-05 12:32:00.932[main] INFO Main01.kt[22]: Awaited, result: kotlin.Unit
2022-02-05 12:32:00.934[main] INFO Main01.kt[23]: Cancelling...
2022-02-05 12:32:00.934[main] INFO Main01.kt[25]: Cancelled
```

② 先cancel( )后await( ) 

会抛出kotlinx.coroutines.JobCancellationException异常。

```kotlin
fun main() {
    val logger = LogManager.getLogger()

    runBlocking {
        val job = async(Dispatchers.Default) {
            repeat(5) {
                logger.info("Hello $it")
                delay(500)
            }
        }

        delay(1000L)
        logger.info("Cancelling...")
        job.cancel()
        logger.info("Cancelled")
        val result = job.await()
        logger.info("Awaited, result: $result")
    }
}

//output:
2022-02-05 12:39:49.933[DefaultDispatcher-worker-1] INFO Main01.kt[15]: Hello 0
2022-02-05 12:39:50.452[DefaultDispatcher-worker-1] INFO Main01.kt[15]: Hello 1
2022-02-05 12:39:50.924[main] INFO Main01.kt[21]: Cancelling...
2022-02-05 12:39:50.939[main] INFO Main01.kt[23]: Cancelled
Exception in thread "main" kotlinx.coroutines.JobCancellationException: Job was cancelled; job=DeferredCoroutine{Cancelled}@23986957

Process finished with exit code 1
```

参考资料：

1. [KotlinConf 2019: Coroutines! Gotta catch 'em all! ](https://www.bilibili.com/video/BV1E7411M7Sj/)
2. 协程中的取消和异常 | 异常处理详解  https://picture.iczhiku.com/weixin/message1595087419639.html