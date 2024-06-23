---
title: Hilt的使用实践
urlname: dependency_injection
date: 2024-04-04 18:51:59
tags: dependency injection
categories:
description:
---



#### Enabling Hilt in Application

自定义一个Application，在其上使用@HiltAndroidApp注解。@HiltAndroidApp用于告诉Dagger这是整个项目依赖关系图（dependency graph）的入口点（entry point）。

##### @InstallIn



@InstallIn注解用于将指定Module提供的对象实例添加到Component中。

举例：

```kotlin
@Module(
  includes = [
    LocationModule::class,
    NetworkModule::class,
    AndroidSupportInjectionModule::class
  ]
)
@InstallIn(ApplicationComponent::class) // HERE
object ApplicationModule {

  @Provides
  fun provideNetworkingConfiguration(): NetworkingConfiguration =
    BussoConfiguration
}
```

使用@InstallIn(ApplicationComponent::class)时会告诉Dagger你希望将ApplicationModule中定义的所有bindings添加到ApplicationComponent的dependency graph中。其中ApplicationComponent在Hilt中进行了预定义，具体路径dagger.hilt.android.components.ApplicationComponent。

备注：Hilt中预置的Components有：

```kotlin
dagger.hilt.android.components.ApplicationComponent
dagger.hilt.android.components.ActivityComponent
dagger.hilt.android.components.FragmentComponent
```

对应的Scope有，定义在目录dagger.hilt.android.scopes下。

```
ActivityScoped
FragmentScoped
ActivityRetainedScoped
ServiceScoped
ViewScoped
```



##### @AndroidEntryPoint

@AndroidEntryPoint用来将一个标准的Android组件标记为一个可能的dependency target。**dependency targets**包括：







