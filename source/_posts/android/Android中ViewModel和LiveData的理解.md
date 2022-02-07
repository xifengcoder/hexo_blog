---
title: Android应用架构之MVVM模式
urlname: android_mvvm
date: 2022-01-27 17:59:12
tags: mvvm
categories: Android
description: Android中MVVM的理解，分别从ViewModel、LiveData、LifeCycle等几部分展开分析...
---

#### 一、ViewModel部分

##### 1. ViewModelProvider

使用方法：

```kotlin
val viewmodel = ViewModelProvider(this).get(MyViewModel::class.java)
val viewModel2 = ViewModelProvider(this, Injection.provideViewModelFactory(this))
            .get(MyViewModel::class.java)
```

对应的ViewModelProvider构造方法：

```java
package androidx.lifecycle;

import android.app.Application;

import androidx.annotation.MainThread;
import androidx.annotation.NonNull;

import java.lang.reflect.InvocationTargetException;

public class ViewModelProvider {
    private static final String DEFAULT_KEY =
            "androidx.lifecycle.ViewModelProvider.DefaultKey";
  
    public interface Factory {
        //创建创建一个类型为T的实例，其中T必须继承自ViewModel类。
        @NonNull
        <T extends ViewModel> T create(@NonNull Class<T> modelClass);
    }
  
    private final Factory mFactory;
    private final ViewModelStore mViewModelStore;
  
    //构造方法1
    public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
        this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
                ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
                : NewInstanceFactory.getInstance());
    }
  
    //构造方法2
    public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory) {
        this(owner.getViewModelStore(), factory);
    }
  
    //构造方法3
    public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
        mFactory = factory;
        mViewModelStore = store;
    }
  
    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        if (canonicalName == null) {
            throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
        }
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }
  
    @NonNull
    @MainThread
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            if (mFactory instanceof OnRequeryFactory) {
                ((OnRequeryFactory) mFactory).onRequery(viewModel);
            }
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) mFactory).create(key, modelClass);
        } else {
            viewModel = mFactory.create(modelClass);
        }
        mViewModelStore.put(key, viewModel);
        return (T) viewModel;
    }
    
}
```

##### 2. ViewModelStoreOwner

单一方法接口，用于获取一个ViewModelStore实例。

```java
package androidx.lifecycle;

import androidx.annotation.NonNull;

public interface ViewModelStoreOwner {
    @NonNull
    ViewModelStore getViewModelStore();
}
```

androidx.activity.ComponentActivity中对该接口的实现如下：

首先通过父类android.app.Activity的getLastNonConfigurationInstance()方法获取一个自定义的NonConfigurationInstances实例。

注意ComponentActivity类的onRetainNonConfigurationInstance()方法被定义为final的，即不允许子类override。但是，又重新定义了一个onRetainCustomNonConfigurationInstance()，供子类使用，功能与onRetainNonConfigurationInstance()完全一样。

下面重点关注一下

```java
package androidx.activity;

public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        LifecycleOwner,
        ViewModelStoreOwner,
        SavedStateRegistryOwner,
        OnBackPressedDispatcherOwner {
          
    static final class NonConfigurationInstances {
        Object custom;
        ViewModelStore viewModelStore;
    }
    
    private ViewModelStore mViewModelStore;
    
    /**
     * 获取ViewModelStore实例：
     * 1. 如果成员变量mViewModelStore中有值，则直接返回mViewModelStore;
     * 2. 从getLastNonConfigurationInstance()中获取，如果有值，则赋给mViewModelStore并返回；
     * 3. 如果getLastNonConfigurationInstance()中无值，则new一个ViewModelStore实例，赋给mViewModelStore并返回。
     */
    @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        if (mViewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                // Restore the ViewModelStore from NonConfigurationInstances
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    }
    
    /**
     * 当configuration放生变化时，onRetainNonConfigurationInstance()会被系统调用，处于onStop和onDestroy之间，可以用来保存任何Object的对象实例，当新的Activity实例被创建时通过getLastNonConfigurationInstance()获取上次保存的对象。
     * 1. 如果成员变量mViewModelStore已经有值，则构造一个NonConfigurationInstances实例返回；
     * 2. 如果mViewModelStore为空，则尝试从getLastNonConfigurationInstance()中获取：
     *   1) getLastNonConfigurationInstance()无值、且onRetainCustomNonConfigurationInstance()中也无值，返回null；
     *   2) 构造一个NonConfigurationInstances实例返回。
     */
    @Override
    @Nullable
    public final Object onRetainNonConfigurationInstance() {
        Object custom = onRetainCustomNonConfigurationInstance();

        ViewModelStore viewModelStore = mViewModelStore;
        if (viewModelStore == null) {
            // No one called getViewModelStore(), so see if there was an existing
            // ViewModelStore from our last NonConfigurationInstance
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                viewModelStore = nc.viewModelStore;
            }
        }

        if (viewModelStore == null && custom == null) {
            return null;
        }

        NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.custom = custom;
        nci.viewModelStore = viewModelStore;
        return nci;
    }
}
```

##### 3. ViewModelStore

ViewModelStore用来保存ViewModel实例。

ViewModelStore的实例在configuration发生变化时，必须继续保留(retained)；

如果一个ViewModelStore的owner由于configuration的 变化被重新创建时，新的owner实例应该继续持有同一个ViewModelStore实例。

如果一个ViewModelStore的owner被destroyed、并且不准备被重新创建，应该调用ViewModelStore的clear()方法通知该ViewModelStore。

```java
package androidx.lifecycle;

import java.util.HashMap;
import java.util.HashSet;
import java.util.Set;

public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```

##### 4. ViewModel

ViewModel负责准备、管理android.app.Activity或androidx.fragment.app.Fragment的数据，用于UI展示。

当LifeCyclerOwner由于configuration的变化(屏幕旋转)被destroyed时，ViewModel不会被destroyed，新的owner实例获取的仍然是原有的ViewModel实例。

千万不要在ViewModel中访问View对象、或者持有Activity或者Fragment的引用。

#### 二、Lifecycle部分

##### 1. LifecycleOwner

```java
package androidx.lifecycle;

import androidx.annotation.NonNull;

/**
 * A class that has an Android lifecycle. These events can be used by custom components to
 * handle lifecycle changes without implementing any code inside the Activity or the Fragment.
 *
 * @see Lifecycle
 * @see ViewTreeLifecycleOwner
 */
@SuppressWarnings({"WeakerAccess", "unused"})
public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}
```

##### 2. LifecycleObserver

```java
package androidx.lifecycle;
/**
 * 空方法的接口，依赖于OnLifecycleEvent注解
 */
@SuppressWarnings("WeakerAccess")
public interface LifecycleObserver {

}
```

##### 3. DefaultLifecycleObserver

```java
package androidx.lifecycle;

import androidx.annotation.NonNull;

/**
 * Callback interface for listening to {@link LifecycleOwner} state changes.
 * <p>
 * If you use Java 8 language, <b>always</b> prefer it over annotations.
 */
@SuppressWarnings("unused")
public interface DefaultLifecycleObserver extends FullLifecycleObserver {

    @Override
    default void onCreate(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onStart(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onResume(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onPause(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onStop(@NonNull LifecycleOwner owner) {
    }

    @Override
    default void onDestroy(@NonNull LifecycleOwner owner) {
    }
}
```

如果某一个组件需要感知生命周期的话，则直接去继承 `LifecycleObserver` 接口。

如下:

```java
//CustomComponent希望感知生命周期
public class CustomComponent implements DefaultLifecycleObserver {
  @Override
    public void onCreate(@NonNull LifecycleOwner owner) {
    }

    @Override
    public void onStart(@NonNull LifecycleOwner owner) {
    }

    @Override
    public void onResume(@NonNull LifecycleOwner owner) {
    }

    @Override
    public void onPause(@NonNull LifecycleOwner owner) {
    }
}
```

同时在Activity或Fragment中添加：

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
      
        //现在CustomComponent可以感知MainActivity的生命周期了。
        getLifecycle().addObserver(new CustomComponent());
    }
}
```

##### 4. ReportFragment

版本号：androidx.lifecycle:lifecycle-livedata:2.3.1

生命周期的观察，分为Android 9及以上、Android 9以下两种情况：

1. Build.VERSION.SDK_INT >= 29
通过activity.registerActivityLifecycleCallbacks来感知生命周期；
2. Build.VERSION.SDK_INT < 29
通过给activity绑定一个Fragment（名字为ReportFragment），然后使用Fragment的回调方法来感知声明周期。

```java
package androidx.lifecycle;

import android.app.Activity;
import android.app.Application;
import android.os.Build;
import android.os.Bundle;

import androidx.annotation.NonNull;
import androidx.annotation.Nullable;
import androidx.annotation.RequiresApi;
import androidx.annotation.RestrictTo;

@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP_PREFIX)
public class ReportFragment extends android.app.Fragment {
    private static final String REPORT_FRAGMENT_TAG = "androidx.lifecycle"
            + ".LifecycleDispatcher.report_fragment_tag";
    private ActivityInitializationListener mProcessListener;

    public static void injectIfNeededIn(Activity activity) {
        if (Build.VERSION.SDK_INT >= 29) {
            LifecycleCallbacks.registerIn(activity);
        }
    
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            manager.executePendingTransactions();
        }
    }
  
    //获取ReportFragment
    static ReportFragment get(Activity activity) {
        return (ReportFragment) activity.getFragmentManager().findFragmentByTag(
                REPORT_FRAGMENT_TAG);
    }
  
    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatchCreate(mProcessListener);
        dispatch(Lifecycle.Event.ON_CREATE);
    }
  
    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }
  
    @Override
    public void onResume() {
        super.onResume();
        dispatchResume(mProcessListener);
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onStop() {
        super.onStop();
        dispatch(Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        dispatch(Lifecycle.Event.ON_DESTROY);
        // just want to be sure that we won't leak reference to an activity
        mProcessListener = null;
    }
  
    void setProcessListener(ActivityInitializationListener processListener) {
        mProcessListener = processListener;
    }
  
   interface ActivityInitializationListener {
        void onCreate();
        void onStart();
        void onResume();
    }
  
    @RequiresApi(29)
    static class LifecycleCallbacks implements Application.ActivityLifecycleCallbacks {

        static void registerIn(Activity activity) {
            activity.registerActivityLifecycleCallbacks(new LifecycleCallbacks());
        }

        @Override
        public void onActivityCreated(@NonNull Activity activity,
                @Nullable Bundle bundle) {
        }

        @Override
        public void onActivityPostCreated(@NonNull Activity activity,
                @Nullable Bundle savedInstanceState) {
            dispatch(activity, Lifecycle.Event.ON_CREATE);
        }

        @Override
        public void onActivityStarted(@NonNull Activity activity) {
        }

        @Override
        public void onActivityPostStarted(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_START);
        }

        @Override
        public void onActivityResumed(@NonNull Activity activity) {
        }

        @Override
        public void onActivityPostResumed(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_RESUME);
        }

        @Override
        public void onActivityPrePaused(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_PAUSE);
        }

        @Override
        public void onActivityPaused(@NonNull Activity activity) {
        }

        @Override
        public void onActivityPreStopped(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_STOP);
        }

        @Override
        public void onActivityStopped(@NonNull Activity activity) {
        }

        @Override
        public void onActivitySaveInstanceState(@NonNull Activity activity,
                @NonNull Bundle bundle) {
        }

        @Override
        public void onActivityPreDestroyed(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_DESTROY);
        }

        @Override
        public void onActivityDestroyed(@NonNull Activity activity) {
        }
    }
}
```

##### 5. LifecycleRegistry

```java

    @MainThread
    @NonNull
    public abstract State getCurrentState();

    @SuppressWarnings("WeakerAccess")
    public enum Event {
        ON_CREATE,
        ON_START,
        ON_RESUME,
        ON_PAUSE,
        ON_STOP,
        ON_DESTROY,
        ON_ANY;

        /**
         * 使得参数State降低一级时所需的Event。
         */
        @Nullable
        public static Event downFrom(@NonNull State state) {
            switch (state) {
                case CREATED:
                    return ON_DESTROY;
                case STARTED:
                    return ON_STOP;
                case RESUMED:
                    return ON_PAUSE;
                default:
                    return null;
            }
        }

        /**
         * 降级成为参数的State时所需的Event
         */
        @Nullable
        public static Event downTo(@NonNull State state) {
            switch (state) {
                case DESTROYED:
                    return ON_DESTROY;
                case CREATED:
                    return ON_STOP;
                case STARTED:
                    return ON_PAUSE;
                default:
                    return null;
            }
        }

        /**
         * 从参数State升级为更高一级的State时所需的Event。
         */
        @Nullable
        public static Event upFrom(@NonNull State state) {
            switch (state) {
                case INITIALIZED:
                    return ON_CREATE;
                case CREATED:
                    return ON_START;
                case STARTED:
                    return ON_RESUME;
                default:
                    return null;
            }
        }

        /**
         * 升级为参数的State时所需的Event。
         */
        @Nullable
        public static Event upTo(@NonNull State state) {
            switch (state) {
                case CREATED:
                    return ON_CREATE;
                case STARTED:
                    return ON_START;
                case RESUMED:
                    return ON_RESUME;
                default:
                    return null;
            }
        }

        /**
         * 返回上报该Event后的新的State。
         */
        @NonNull
        public State getTargetState() {
            switch (this) {
                case ON_CREATE:
                case ON_STOP:
                    return State.CREATED;
                case ON_START:
                case ON_PAUSE:
                    return State.STARTED;
                case ON_RESUME:
                    return State.RESUMED;
                case ON_DESTROY:
                    return State.DESTROYED;
                case ON_ANY:
                    break;
            }
            throw new IllegalArgumentException(this + " has no target state");
        }
    }
```



```java

    @SuppressWarnings("WeakerAccess")
    public enum State {
        DESTROYED,
        INITIALIZED,
        CREATED,
        STARTED,
        RESUMED;
      
        //是否不低于参数所指的State。
        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }

```

#### 三、LiveData部分

LiveData是一个可被观测的数据的持有类(data holder class)。

LifecycleOwner用于表示一个类拥有Lifecycle，而Observer则可以通过注册一个listener去监听生命周期的变化。

使用LiveData的observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer)方法将一个Observer对象附加（attach）到该LiveData对象上。

当LifecycleOwner的状态是STARTED或者RESUMED时，被认为是活跃状态。Observers可以观察到处于活跃状态的owner的数据变化，如果owner进入DESTROYED状态，则observers会被自动移除。