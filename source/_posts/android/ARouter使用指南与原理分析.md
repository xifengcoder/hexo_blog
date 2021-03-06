---
title: ARouter使用指南与原理分析
urlname: deep_in_arouter
date: 2021-12-26 17:15:54
tags: 路由组件
categories: Android
description: ARouter使用总结，以及原理的探讨分析...
---
Arouter通过APT技术生成的文件如下：
package: com.alibaba.android.arouter.routes;
* ARouter\$\$Group$$\<groupName>.java
* ARouter\$\$Interceptors$$\<moduleName>.java
* ARouter\$\$Providers$$\<moduleName>.java
* ARouter\$\$Root$$\<moduleName>.java

如果使用AutoWired的话，会在原有的package下生成：
* \<ClassName\>$$ARouter$$Autowired.java

下面分别举例说明：
1. ARouter$$Group$$\<groupName>.java
```java
package com.alibaba.android.arouter.routes;

import com.alibaba.android.arouter.facade.enums.RouteType;
import com.alibaba.android.arouter.facade.model.RouteMeta;
import com.alibaba.android.arouter.facade.template.IRouteGroup;
import com.yxf.arouter.other.OtherActivity;
import java.lang.Override;
import java.lang.String;
import java.util.Map;

/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Group$$other implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/other/otherActivity", RouteMeta.build(RouteType.ACTIVITY, OtherActivity.class, "/other/otheractivity", "other", null, -1, -2147483648));
  }
}
```
2. ARouter$$Providers$$\<moduleName>.java
```java
package com.alibaba.android.arouter.routes;

import com.alibaba.android.arouter.facade.model.RouteMeta;
import com.alibaba.android.arouter.facade.template.IProviderGroup;
import java.lang.Override;
import java.lang.String;
import java.util.Map;

/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Providers$$moduleother implements IProviderGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> providers) {
  }
}
```
3. ARouter\$\$Root$$\<moduleName>.java
```java
package com.alibaba.android.arouter.routes;

import com.alibaba.android.arouter.facade.template.IRouteGroup;
import com.alibaba.android.arouter.facade.template.IRouteRoot;
import java.lang.Class;
import java.lang.Override;
import java.lang.String;
import java.util.Map;

/**
 * DO NOT EDIT THIS FILE!!! IT WAS GENERATED BY AROUTER. */
public class ARouter$$Root$$moduleother implements IRouteRoot {
  @Override
  public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
    routes.put("other", ARouter$$Group$$other.class);
  }
}
```
#### WareHouse的数据结构：
![test.png](https://img2020.cnblogs.com/blog/368055/202110/368055-20211006214111989-1436169959.jpg)
![](https://img2020.cnblogs.com/blog/368055/202110/368055-20211006214135774-245463483.jpg)
![](https://img2020.cnblogs.com/blog/368055/202110/368055-20211006214149891-914672834.jpg)

#### Arouter的初始化：
```java
Arouter.init(this);
```
主要工作：
```java
//LogisticsCenter.init(mContext, executor);
/**
     * LogisticsCenter init, load all metas in memory. Demand initialization
     */
    public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
        mContext = context;
        executor = tpe;

        try {
            long startInit = System.currentTimeMillis();
            //load by plugin first
            loadRouterMap();
            if (registerByPlugin) {
                logger.info(TAG, "Load router map by arouter-auto-register plugin.");
            } else {
                Set<String> routerMap;

                // It will rebuild router map every times when debuggable.
                if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
                    routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
                    if (!routerMap.isEmpty()) {
                        context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, routerMap).apply();
                    }

                    PackageUtils.updateVersion(context);    // Save new version name when router map update finishes.
                } else {
                    routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, Context.MODE_PRIVATE).getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
                }

                startInit = System.currentTimeMillis();

                for (String className : routerMap) {
                    if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                        //以com.alibaba.android.arouter.routes.ARouter$$Root开头。
                        ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
                    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
                        //以com.alibaba.android.arouter.routes.ARouter$$Interceptors开头。
                        ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
                    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
                        //以com.alibaba.android.arouter.routes.ARouter$$Providers开头。
                        ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
                    }
                }
            }
        } catch (Exception e) {
            throw new HandlerException(TAG + "ARouter init logistics center exception! [" + e.getMessage() + "]");
        }
    }
```
其中使用的executor为DefaultPoolExecutor.getInstance()。
1. 扫描com.alibaba.android.arouter.routes包名下的所有className，存放入HashSet中；
2. 遍历该HashSet, 分三种情况：
（1）针对ARouter$$Root开头的className，实例化IRouteRoot并调用loadInfo()加载到Warehouse.groupsIndex（Map<String, Class<? extends IRouteGroup>>类型）；
（2）针对ARouer$$Interceptors开头的className，实例化IInterceptorGroup并调用loadInfo()加载到Warehouse.interceptorsIndex（Map<Integer, Class<? extends IInterceptor>>类型）；
（3）针对ARouer$$Providers开头的className，实例化IProviderGroup并调用loadInfo()加载到Warehouse.providersIndex（Map<String, RouteMeta>类型）。

#### Arouter.getInstance().build(String path);
通过path和group构建一个Postcard实例；
### Postcard$navigation()
-> Postcard#navigation(Context)
-> Postcard#navigation(Context context, NavigationCallback callback)
-> Arouter#navigation(Context mContext, Postcard postcard, int requestCode, NavigationCallback callback)
-> LogisticsCenter#completion(Postcard postcard)

下面重点分析一下LogisticsCenter#completion(Postcard)方法。

1. 根据postcard的path，从Warehouse.routes(Map<String, RouteMeta>)中查找RouteMeta实例routeMeta；
2. 如果routeMeta为空，则先从Warehouse.groupsIndex中查找postcard所属的group对应的Class<? extends IRouteGroup>实例，然后通过反射构造出IRouteGroup实例，然后调用iGroupInstance.loadInto(Warehouse.routes)，然后再将postcard所属的group从Warehouse.groupsIndex中移除。然后再次调用LogisticsCenter#completion(Postcard)方法。
3. 设置PostCard的destination、type、priority和extra。

```java
public class LogisticsCenter {
    ...
    /**
     * Completion the postcard by route metas
     *
     * @param postcard Incomplete postcard, should complete by this method.
     */
    public synchronized static void completion(Postcard postcard) {
        if (null == postcard) {
            throw new NoRouteFoundException(TAG + "No postcard!");
        }

        RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
        if (null == routeMeta) {    // Maybe its does't exist, or didn't load.
            Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());  // Load route meta.
            if (null == groupMeta) {
                throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
            } else {
                // Load route and cache it into memory, then delete from metas.
                try {
                    ...
                    IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
                    iGroupInstance.loadInto(Warehouse.routes);
                    Warehouse.groupsIndex.remove(postcard.getGroup());
                    ...
                } catch (Exception e) {
                    throw new HandlerException(TAG + "Fatal exception when loading group meta. [" + e.getMessage() + "]");
                }

                completion(postcard);   // Reload
            }
        } else {
            postcard.setDestination(routeMeta.getDestination());
            postcard.setType(routeMeta.getType());
            postcard.setPriority(routeMeta.getPriority());
            postcard.setExtra(routeMeta.getExtra());

            Uri rawUri = postcard.getUri();
            if (null != rawUri) {   // Try to set params into bundle.
                Map<String, String> resultMap = TextUtils.splitQueryParameters(rawUri);
                Map<String, Integer> paramsType = routeMeta.getParamsType();

                if (MapUtils.isNotEmpty(paramsType)) {
                    // Set value by its type, just for params which annotation by @Param
                    for (Map.Entry<String, Integer> params : paramsType.entrySet()) {
                        setValue(postcard,
                                params.getValue(),
                                params.getKey(),
                                resultMap.get(params.getKey()));
                    }

                    // Save params name which need auto inject.
                    postcard.getExtras().putStringArray(ARouter.AUTO_INJECT, paramsType.keySet().toArray(new String[]{}));
                }

                // Save raw uri
                postcard.withString(ARouter.RAW_URI, rawUri.toString());
            }

            switch (routeMeta.getType()) {
                case PROVIDER:  // if the route is provider, should find its instance
                    // Its provider, so it must implement IProvider
                    Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                    IProvider instance = Warehouse.providers.get(providerMeta);
                    if (null == instance) { // There's no instance of this provider
                        IProvider provider;
                        try {
                            provider = providerMeta.getConstructor().newInstance();
                            provider.init(mContext);
                            Warehouse.providers.put(providerMeta, provider);
                            instance = provider;
                        } catch (Exception e) {
                            throw new HandlerException("Init provider failed! " + e.getMessage());
                        }
                    }
                    postcard.setProvider(instance);
                    postcard.greenChannel();    // Provider should skip all of interceptors
                    break;
                case FRAGMENT:
                    postcard.greenChannel();    // Fragment needn't interceptors
                default:
                    break;
            }
        }
    }
}
```
4. 调用ARouter#_navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback)实现跳转功能。

    如果是Activity的跳转，则会调用ARouter#startActivity(int requestCode, Context currentContext, Intent intent, Postcard postcard, NavigationCallback callback)。
```java
final class _ARouter {
    ...
    private Object _navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        final Context currentContext = null == context ? mContext : context;

        switch (postcard.getType()) {
            case ACTIVITY:
                // Build intent
                final Intent intent = new Intent(currentContext, postcard.getDestination());
                intent.putExtras(postcard.getExtras());

                // Set flags.
                int flags = postcard.getFlags();
                if (-1 != flags) {
                    intent.setFlags(flags);
                } else if (!(currentContext instanceof Activity)) {    // Non activity, need less one flag.
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }

                // Set Actions
                String action = postcard.getAction();
                if (!TextUtils.isEmpty(action)) {
                    intent.setAction(action);
                }

                // Navigation in main looper.
                runInMainThread(new Runnable() {
                    @Override
                    public void run() {
                        startActivity(requestCode, currentContext, intent, postcard, callback);
                    }
                });

                break;
            case PROVIDER:
                return postcard.getProvider();
            case BOARDCAST:
            case CONTENT_PROVIDER:
            case FRAGMENT:
                Class fragmentMeta = postcard.getDestination();
                try {
                    Object instance = fragmentMeta.getConstructor().newInstance();
                    if (instance instanceof Fragment) {
                        ((Fragment) instance).setArguments(postcard.getExtras());
                    } else if (instance instanceof android.support.v4.app.Fragment) {
                        ((android.support.v4.app.Fragment) instance).setArguments(postcard.getExtras());
                    }

                    return instance;
                } catch (Exception ex) {
                    logger.error(Consts.TAG, "Fetch fragment instance error, " + TextUtils.formatStackTrace(ex.getStackTrace()));
                }
            case METHOD:
            case SERVICE:
            default:
                return null;
        }

        return null;
    }

    /**
     * Start activity
     *
     * @see ActivityCompat
     */
    private void startActivity(int requestCode, Context currentContext, Intent intent, Postcard postcard, NavigationCallback callback) {
        if (requestCode >= 0) {  // Need start for result
            if (currentContext instanceof Activity) {
                ActivityCompat.startActivityForResult((Activity) currentContext, intent, requestCode, postcard.getOptionsBundle());
            } else {
                logger.warning(Consts.TAG, "Must use [navigation(activity, ...)] to support [startActivityForResult]");
            }
        } else {
            ActivityCompat.startActivity(currentContext, intent, postcard.getOptionsBundle());
        }

        if ((-1 != postcard.getEnterAnim() && -1 != postcard.getExitAnim()) && currentContext instanceof Activity) {    // Old version.
            ((Activity) currentContext).overridePendingTransition(postcard.getEnterAnim(), postcard.getExitAnim());
        }

        if (null != callback) { // Navigation over.
            callback.onArrival(postcard);
        }
    }
}
```
### 扫描dex文件中包名com.alibaba.android.arouter.routes下的所有class文件
ClassUtils.getFileNameByPackageName()方法返回指定包名下的所有class文件名。

举例：

```java
com.alibaba.android.arouter.routes.ARouter$$Root$$app
com.alibaba.android.arouter.routes.ARouter$$Root$$moduleother，
com.alibaba.android.arouter.routes.ARouter$$Root$$modulethird,
com.alibaba.android.arouter.routes.ARouter$$Root$$moduleservice,
com.alibaba.android.arouter.routes.ARouter$$Root$$arouterapi,

com.alibaba.android.arouter.routes.ARouter$$Group$$arouter, 
com.alibaba.android.arouter.routes.ARouter$$Group$$main, 
com.alibaba.android.arouter.routes.ARouter$$Group$$other, 
com.alibaba.android.arouter.routes.ARouter$$Group$$third, 
com.alibaba.android.arouter.routes.ARouter$$Group$$service, 

com.alibaba.android.arouter.routes.ARouter$$Providers$$arouterapi,
com.alibaba.android.arouter.routes.ARouter$$Providers$$app, 
com.alibaba.android.arouter.routes.ARouter$$Providers$$moduleother,
com.alibaba.android.arouter.routes.ARouter$$Providers$$modulethird, 
com.alibaba.android.arouter.routes.ARouter$$Providers$$moduleservice,

com.alibaba.android.arouter.routes.ARouter$$Interceptors$$app
```

具体代码：

```java
//ClassUtils.java

public static Set<String> getFileNameByPackageName(Context context, final String packageName)
            throws PackageManager.NameNotFoundException, IOException, InterruptedException {
        final Set<String> classNames = new HashSet<>();

        List<String> paths = getSourcePaths(context);
        final CountDownLatch parserCtl = new CountDownLatch(paths.size());

        for (final String path : paths) {
            DefaultPoolExecutor.getInstance().execute(new Runnable() {
                @Override
                public void run() {
                    DexFile dexfile = null;

                    try {
                        //path: /data/app/com.yxf.arouter.sample-c4N-3bLkk7jsTmTOlMbi-w==/base.apk
                        if (path.endsWith(EXTRACTED_SUFFIX)) {
                            //NOT use new DexFile(path), because it will throw "permission error in /data/dalvik-cache"
                            dexfile = DexFile.loadDex(path, path + ".tmp", 0);
                        } else {
                            dexfile = new DexFile(path);
                        }

                        Enumeration<String> dexEntries = dexfile.entries();
                        while (dexEntries.hasMoreElements()) {
                            String className = dexEntries.nextElement();
                            if (className.startsWith(packageName)) {
                                classNames.add(className);
                            }
                        }
                    } catch (Throwable ignore) {
                        Log.e("ARouter", "Scan map file in dex files made error.", ignore);
                    } finally {
                        if (null != dexfile) {
                            try {
                                dexfile.close();
                            } catch (Throwable ignore) {
                            }
                        }

                        parserCtl.countDown();
                    }
                }
            });
        }

        parserCtl.await();

        Log.d(Consts.TAG, "Filter " + classNames.size() + " classes by packageName <" + packageName + ">");
        return classNames;
    }
```

#### arouter-register
会扫描所有以下三种的接口实例：
1. IRouteRoot
2. IInterceptorGroup
3. IProviderGroup

然后添加到

例如：
```
private static void loadRouterMap() {
    registerByPlugin = true;
    register("com.alibaba.android.arouter.routes.ARouter$$Root$$arouterapi");
    register( "com.alibaba.android.arouter.routes.ARouter$$Root$$moduleother");
    register("com.alibaba.android.arouter.routes.ARouter$$Root$$modulethird");
    register("com.alibaba.android.arouter.routes.ARouter$$Root$$app");
    register("com.alibaba.android.arouter.routes.ARouter$$Providers$$arouterapi");
    register( "com.alibaba.android.arouter.routes.ARouter$$Providers$$moduleother");
    register("com.alibaba.android.arouter.routes.ARouter$$Providers$$modulethird");
    register("com.alibaba.android.arouter.routes.ARouter$$Providers$$app");
}
```