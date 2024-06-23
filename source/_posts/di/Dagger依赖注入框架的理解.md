---
title: Dagger依赖注入框架的理解
urlname: dependency_injection
date: 2024-02-28 22:40:36
tags: dependency injection
categories:
description:
---

##### Injection的类型

- **Constructor** injection

     ```kotlin
     class Server(private val repository: Repository) {
       fun receive(data: Date) {
         repository.save(date)
       }
     }
     ```

    如果你可以控制依赖关系中组件的创建的话，**Constructor** injection 是最佳的依赖注入类型。

- **Field** injection

```kotlin
class Server () {
  lateinit var repository: Repository // HERE

  fun receive(data: Date) {
    repository.save (date)
  }
}
```

```kotlin
fun main() {
  // 1
  val repository = RepositoryImpl()
  // 2
  val server = Server()
  // 3  Field injection
  server.repository = repository
  // ...
  val data = Data()
  server.receive(data)
  // ...
}
```

- **Method** injection

```kotlin
class Server() {
  private var repository: Repository? = null

  fun receive(data: Date) {
    repository?.save(date)
  }

  fun fixRepo(repository: Repository) {
    this.repository = repository
  }
}
```

```kotlin
fun main() {
  val repository = RepositoryImpl()
  val server = Server()
  server.fixRepo(repository) // Method injection
  // ...
  val data = Data()
  server.receive(data)
  // ...
}
```

Method injection支持同一个Method中注入多个类对象，这是不同于Field injection的地方。

举例：

```kotlin
class Dependent() {
  private var dep1: Dep1? = null
  private var dep2: Dep2? = null
  private var dep3: Dep3? = null

  //同时注入3个类型参数
  fun fixDep(dep1: Dep1, dep2: Dep2, dep3: Dep3) {
    this.dep1 = dep1
    this.dep2 = dep2
    this.dep3 = dep3
  }
}
```

##### `@Binds`

`@Binds`用于绑定抽象类或接口与其具体实现类之间的关系。

##### @Component.Builder

```java
@Component(modules = [AppModule::class, NetworkModule::class])
interface AppComponent {
    fun inject(activity: SplashActivity)

    fun inject(activity: MainActivity)

    fun inject(fragment: BusStopFragment)

    fun inject(fragment: BusArrivalFragment)
      
  	@Component.Builder
  	interface Builder {

    		@BindsInstance
		    fun activity(activity: Activity): Builder

    		fun build(): AppComponent
  }
}
```

Component.Builder中使用@BindsInstance标记的方法表示将该方法的参数（比如Activity）传递到依赖关系图中（dependency graph），该方法必须返回Builder实例。

这样，该Component所包含的Module都可以使用该Activity对象。也即provide方法所需的参数。

同时，Component.Builder中还得有个方法返回AppComponent对象,  名字可以自己起，一般叫build()或者create()。

MainActivity中就可以这么使用：

```kotlin
class MainActivity : AppCompatActivity() {
		override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        DaggerAppComponent
            .builder()
            .activity(this)
            .build()
            .inject(this)
    }
}
```

还有一种实现方式：

```kotlin
@Component(modules = [AppModule::class, NetworkModule::class])
interface AppComponent {
    fun inject(activity: SplashActivity)

    fun inject(activity: MainActivity)

    fun inject(fragment: BusStopFragment)

    fun inject(fragment: BusArrivalFragment)

    // 1
    @Component.Builder
    interface Builder {

        fun appModule(appModule: AppModule): Builder

        fun networkModule(networkModule: NetworkModule): Builder

        fun build(): AppComponent
    }

```

Component.Builder中提供该Component所包含的Module的方法（注该方法返回的是Builder对象），

这样就退化为不使用Builder的形式，通过将Activity作为构造方法的参数传递给Module。

```kotlin
class MainActivity : AppCompatActivity() {
    lateinit var comp: AppComponent // 1

  	override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
      
        comp = DaggerAppComponent
            .builder()
            .appModule(AppModule(this))
            .networkModule(NetworkModule(this))
            .build().apply {
                inject(this@MainActivity)
            }

        if (savedInstanceState == null) {
            mainPresenter.goToBusStopList()
        }
    }
}

val Context.comp: AppComponent?
    get() = if (this is MainActivity) comp else null
```



##### Component.Factory

```kotlin
@Component(modules = [AppModule::class, NetworkModule::class])
interface AppComponent {
  // ...
  @Component.Factory // 1
  interface Factory {
    // 2
    fun create(@BindsInstance activity: Activity): AppComponent
  }
}
```





#####  `@Singleton`

同一个@Component的实例总是返回同一个给定类型的实例。

但是不同的@Component实例是可以返回不同类型的实例的。





##### Component Dependencies

如果某个Component使用dependencies依赖了其它的Component，就需要在@Component.Factory的create()方法中添加它所依赖的Component参数。



##### ApplicationComponent

```kotlin

@Component(modules = [ApplicationModule::class])
@Singleton
interface ApplicationComponent {

    fun locationObservable(): Observable<LocationEvent> //开放给依赖ApplicationComponent的Component使用

    fun bussoEndpoint(): BussoEndpoint //开放给依赖ApplicationComponent的Component使用

    @Component.Factory
    interface Builder {

        fun create(@BindsInstance application: Application): ApplicationComponent // 3
    }
}
```



###### ActivityComponent

```kotlin
@Component(
    modules = [ActivityModule::class],
    dependencies = [ApplicationComponent::class] // 1
)
@ActivityScope
interface ActivityComponent {

  fun inject(activity: SplashActivity)

  fun inject(activity: MainActivity)

  fun navigator(): Navigator //开放给依赖ActivityComponent的Component使用

  @Component.Factory
  interface Factory {
    fun create(
        @BindsInstance activity: Activity,
        applicationComponent: ApplicationComponent // 2
    ): ActivityComponent
  }
}

```

在@Component.Factory的create()方法中添加ApplicationComponent参数，注意这里没有@BindsInstance注解。

##### FragmentComponent

```kotlin
@Component(
    modules = [FragmentModule::class],
    dependencies = [ActivityComponent::class, ApplicationComponent::class] // 1
)
@FragmentScope
interface FragmentComponent {

    fun inject(fragment: BusStopFragment) // 3

    fun inject(fragment: BusArrivalFragment) // 3

    @Component.Factory
    interface Factory {
        // 4
        fun create(
            applicationComponent: ApplicationComponent,
            activityComponent: ActivityComponent
        ): FragmentComponent
    }
}
```





### @Subcomponents

##### ApplicationComponent

相较Component Dependencies来说，ApplicationComponent不用发布公开方法出来了。但是需要为它的children定义factory方法；

```kotlin
@Component(modules = [ApplicationModule::class])
@Singleton
interface ApplicationComponent {
  
    fun activityComponentFactory(): ActivityComponent.Factory //需要为自己的子组件定义获取方法

    @Component.Factory
    interface Factory {

        fun create(@BindsInstance application: Application): ApplicationComponent
    }
}
```

##### ActivityComponent

```kotlin
@Subcomponent(
    modules = [ActivityModule::class]
)
@ActivityScope
interface ActivityComponent {

    fun inject(activity: SplashActivity)

    fun inject(activity: MainActivity)

    fun fragmentComponent(): FragmentComponent //需要为自己的子组件定义获取方法

    @Subcomponent.Factory
    interface Factory {
        fun create(
            @BindsInstance activity: Activity
        ): ActivityComponent
    }
}
```

##### FragmentComponent

```kotlin
@Subcomponent(
    modules = [FragmentModule::class]
)
@FragmentScope
interface FragmentComponent {

  fun inject(fragment: BusStopFragment)

  fun inject(fragment: BusArrivalFragment)
}
```

##### ApplicationComponent的创建

```kotlin
class Main : Application() {

  lateinit var appComponent: ApplicationComponent

  override fun onCreate() {
    super.onCreate()
    // 3
    appComponent = DaggerApplicationComponent
        .factory()
        .create(this)
  }
}

val Context.appComp: ApplicationComponent
  get() = (applicationContext as Main).appComponent
```

##### ActivityComponent的创建

```kotlin
class MainActivity : AppCompatActivity() {

    @Inject
    lateinit var mainPresenter: MainPresenter

    lateinit var comp: ActivityComponent

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        comp = application.appComp  //借助该Activity所依附的Application来创建
            .activityComponentFactory()
            .create(this)
            .apply {
                inject(this@MainActivity)
            }
        if (savedInstanceState == null) {
            mainPresenter.goToBusStopList()
        }
    }
}

val Context.activityComp: ActivityComponent
    get() = (this as MainActivity).comp
```

##### FragmentComponent的创建

```kotlin
class BusArrivalFragment : Fragment() {
  	override fun onAttach(context: Context) {
        context.activityComp  //借助该Fragment所依附的Activity的ActivityComponent来创建
            .fragmentComponent()
            .inject(this)
        super.onAttach(context)
    }
}

```

#### @Subcomponents VS Component Dependencies

**Component Dependencies:**

1. 当前Component需要分享给其它Component的对象，需要在该Component中提供factory methods进行显式的发布；

2. 当前Component所依赖的Components组件，必须在该@Component的dependencies属性中列出；

3. 依赖关系不具备可传递性，如果`@Component` **A** 依赖 **B** ，**B** 依赖 **C**，并不意味着**A**依赖**C**；

4. 当有存在的对象时，可以使用`@Component.Builder` and `@Component.Factory`；

5. 当使用**dependencies**属性进行依赖管理时，不支持**multibinding**。



**@Subcomponents**

1. 每一个@Subcomponent会从它的父@Component或者父@Subcomponent组件继承所有的对象，因此不需要显式的发布；

2. 依赖关系具备可传递性：如果@Subcomponent **A** 继承自 **B** 并且 **B** 继承自 **C**, 那么 **A** 继承自 **C**；

3. @Subcomponent组件的实例通过父@Component或@Subcomponent组件的factory方法来创建；
4. 当有存在的对象时，可以使用@Subcomponent.Builder和@Subcomponent.Factory，父组件会为它的children组件定义相应的factory方法；

4. **multibinding**可以和@Subcomponent结合使用；

#### MultiBinding

##### @ElementsIntoSet VS @IntoSet

@IntoSet用于将单个对象添加到Set中

举例：

```java
@Module(includes = [WhereAmIModule.Bindings::class])
object WhereAmIModule {
  //...
  @Provides // 1
  @ApplicationScope // 2
  @Named(WHEREAMI_INFO_NAME) // 3
  fun provideWhereAmISpec(endpoint: WhereAmIEndpointImpl): InformationPluginSpec = object : InformationPluginSpec { // 4
    override val informationEndpoint: InformationEndpoint
      get() = endpoint
    override val serviceName: String
      get() = "WhereAmI"

  }
}
```



@ElementsIntoSet用于将多个对象添加到Set中；
举例:

provideWeatherSpec()返回了一个Set<InformationPluginSpec\>，Dagger会将返回的集合添加到Set中。

```kotlin
@Module(includes = [WhereAmIModule::class, WeatherModule::class])
object InformationSpecsModule {

    @Provides
    @ElementsIntoSet // 1
    @ApplicationScope
    fun provideWeatherSpec(
        @Named(WHEREAMI_INFO_NAME) whereAmISpec: InformationPluginSpec, // 2
        @Named(WEATHER_INFO_NAME) weatherSpec: InformationPluginSpec // 2
    ): Set<InformationPluginSpec> { // 3
        return mutableSetOf<InformationPluginSpec>().apply {
            add(whereAmISpec)
            add(weatherSpec)
        }
    }
}
```

#### 14. Multibinding With Maps

使用Map进行Multibinding，Dagger支持的keys包括：

`@StringKey`

`@ClassKey`

`@IntKey` 

`@LongKey`





1. `@IntoMap`用于告诉Dagger你使用`@Provides`提供的对象会作为Map的一部分。

2. `@StringKey(WEATHER_INFO_NAME)`用于告诉Dagger使用`@Provides`提供的对象的key是String对象。





使用`@ClassKey`





##### 使用 @MapKey

