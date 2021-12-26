---
title: Reified关键字
urlname: kotlin_reified
date: 2021-12-15 13:36:51
tags: Kotlin
categories: Kotlin
description: Kotlin中的reified关键字的使用...
---
Kotlin提供了一种使用 `reified` 关键字获取泛型类型参数的运行时类的方法，该关键字只能与 `inline` 函数一起使用 。

```kotlin
inline fun <reified T> test(value: T) {
    //T::class.java: class java.lang.Integer
    //T::class.javaClass: class kotlin.reflect.jvm.internal.KClassImpl
    println("value: " + value + ", type:" + T::class.java +
                ", javaClass: " + T::class.javaClass)
}

fun main(args: Array<String>) {
    test(10);
}

```

如果不使用reified关键字时，只能通过Class<T>参数才能获取T的类型。

```kotlin
fun <T> test(clazz: Class<T>, value: T) {
    //编译报错: Cannot use 'T' as reified type. Use a class instead.
    //println(T::class.java)
    println(value);
    println("Type of T: ${clazz}")
}

fun main(args: Array<String>) {
    test<Int>(Int::class.java, 100)
}
```

下面再看一个复杂的例子：

```kotlin
package com.yxf.kotlin

import kotlin.reflect.KFunction1

sealed class Animal(val name: String) {
    fun eat() {
    }

    fun sleep() {
    }

    open fun canSwim(): Boolean {
        return false
    }
}

data class Cat(
    val catName: String, var catWeight: Int
) : Animal(catName)

data class Panda(val pandaName: String) : Animal(pandaName)

data class Dog(val dogName: String) : Animal(dogName) {
    override fun canSwim(): Boolean {
        return true
    }
}

fun Animal.knownSpeciesCount(): Int {
    return when (this) {
        is Cat -> 2
        is Dog -> 3
        is Panda -> 5
    }
}

fun animalFactCheck(mammal: Animal, factCheck: KFunction1<Animal, Int>): Int {
    return factCheck(mammal)
}

/**
 * 过滤T类型的动物
 */
fun <T : Animal> printAnimalsFiltered(clazz: Class<T>, list: List<Animal>, factCheck: Animal.() -> Int) {
    if (list.isNotEmpty()) {
        list.forEach {
            if (clazz.isInstance(it)) {
                println("${it.javaClass.name} - ${it.factCheck()}")
            }
        }
    }
}

/**
 * 扩展函数版本：过滤T类型的动物
 */
fun <T : Animal> List<Animal>.printAnimalsExtensionFiltered(
    clazz: Class<T>,
    factCheck: Animal.() -> Int
): List<Animal> {
    if (this.isNotEmpty()) {
        this.filter { clazz.isInstance(it) }
            .forEach {
                println("${it.javaClass.name} - ${it.factCheck()}")
            }
    }
    return this
}

/**
 * reified版本：过滤T类型的动物
 */
inline fun <reified T : Animal> printAnimalsFiltered(
    list: List<Animal>,
    factCheck: Animal.() -> Int
): List<Animal> {

    if (list.isNotEmpty()) {
        list.filterIsInstance<T>()
            .forEach { println("${it.javaClass.name} - ${it.factCheck()}") }
    }
    return list
}

/**
 * 扩展函数的reified版本：过滤T类型的动物
 */
inline fun <reified T : Animal> List<Animal>.printAnimalsExtensionFiltered(
    factCheck: Animal.() -> Int
): List<Animal> {
    if (this.isNotEmpty()) {
        this.filterIsInstance<T>()
            .forEach {
                println("${it.javaClass.name} - ${it.factCheck()}")
            }
    }
    return this
}

fun main() {
    val animals = listOf(
        Cat("Jerry", 15),
        Panda("Tegan"),
        Dog("Manny")
    )

    //过滤Panda类型的动物
    animals.filterIsInstance<Panda>()
        .forEach {
            println(
                "${it.javaClass.name} - " +
                        "${animalFactCheck(it, Animal::knownSpeciesCount)}"
            )
        }

    println("------ Class -------")
    printAnimalsFiltered(Cat::class.java, animals, Animal::knownSpeciesCount)
    animals.printAnimalsExtensionFiltered(Cat::class.java, Animal::knownSpeciesCount)

    println("------ Reified -------")
    printAnimalsFiltered<Dog>(animals, Animal::knownSpeciesCount)
    animals.printAnimalsExtensionFiltered<Dog>(Animal::knownSpeciesCount)
}
```





