---
title: 探索WebView的实现内幕
urlname: android_webview
date: 2022-03-19 17:42:58
tags: webview
categories: web view
description: 这篇文章将带你一起探索WebView的实现内幕...
---

[/frameworks/base/core/java/android/webkit/WebView.java]

#### 一、WebView的创建

##### 构造函数

```java
package android.webkit;

import android.util.SparseArray;
import android.view.DragEvent;
import android.view.KeyEvent;
import android.view.MotionEvent;
import android.view.View;
import android.view.ViewDebug;
import android.view.ViewGroup;
import android.view.ViewHierarchyEncoder;
import android.view.ViewStructure;
import android.view.ViewTreeObserver;
import android.view.accessibility.AccessibilityEvent;
import android.view.accessibility.AccessibilityNodeInfo;
import android.view.accessibility.AccessibilityNodeProvider;
import android.view.autofill.AutofillValue;
import android.view.inputmethod.EditorInfo;
import android.view.inputmethod.InputConnection;
import android.view.textclassifier.TextClassifier;
import android.widget.AbsoluteLayout;


// Implementation notes.
// The WebView is a thin API class that delegates its public API to a backend WebViewProvider
// class instance. WebView extends {@link AbsoluteLayout} for backward compatibility reasons.
// Methods are delegated to the provider implementation: all public API methods introduced in this
// file are fully delegated, whereas public and protected methods from the View base classes are
// only delegated where a specific need exists for them to do so.
@Widget
public class WebView extends AbsoluteLayout
        implements ViewTreeObserver.OnGlobalFocusChangeListener,
        ViewGroup.OnHierarchyChangeListener, ViewDebug.HierarchyHandler {
          
    public WebView(Context context) {
        this(context, null);
    }
          
    public WebView(Context context, AttributeSet attrs) {
        this(context, attrs, com.android.internal.R.attr.webViewStyle);
    }
    
    public WebView(Context context, AttributeSet attrs, int defStyleAttr) {
        this(context, attrs, defStyleAttr, 0);
    }

    public WebView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        this(context, attrs, defStyleAttr, defStyleRes, null, false);
    }
          
    protected WebView(Context context, AttributeSet attrs, int defStyleAttr,
            Map<String, Object> javaScriptInterfaces, boolean privateBrowsing) {
        this(context, attrs, defStyleAttr, 0, javaScriptInterfaces, privateBrowsing);
    }
     
    /**
     * @hide
     */
    @SuppressWarnings("deprecation")  // for super() call into deprecated base class constructor.
    protected WebView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes,
            Map<String, Object> javaScriptInterfaces, boolean privateBrowsing) {
        super(context, attrs, defStyleAttr, defStyleRes);

        // WebView is important by default, unless app developer overrode attribute.
        if (getImportantForAutofill() == IMPORTANT_FOR_AUTOFILL_AUTO) {
            setImportantForAutofill(IMPORTANT_FOR_AUTOFILL_YES);
        }

        if (context == null) {
            throw new IllegalArgumentException("Invalid context argument");
        }
        if (mWebViewThread == null) {
            throw new RuntimeException(
                "WebView cannot be initialized on a thread that has no Looper.");
        }
        sEnforceThreadChecking = context.getApplicationInfo().targetSdkVersion >=
                Build.VERSION_CODES.JELLY_BEAN_MR2;
        checkThread();

        ensureProviderCreated();
        mProvider.init(javaScriptInterfaces, privateBrowsing);
        // Post condition of creating a webview is the CookieSyncManager.getInstance() is allowed.
        CookieSyncManager.setGetInstanceIsAllowed();
    }
    
    private void jian ensureProviderCreated() {
        checkThread();
        if (mProvider == null) {
            // As this can get called during the base class constructor chain, pass the minimum
            // number of dependencies here; the rest are deferred to init().
            mProvider = getFactory().createWebView(this, new PrivateAccess());
        }
    }
          
}
```
创建WebViewProvider

```java
mProvider = getFactory().createWebView(this, new PrivateAccess());
```

其中getFactory()是WebView的静态方法。

```java
 public class WebView extends AbsoluteLayout
        implements ViewTreeObserver.OnGlobalFocusChangeListener,
        ViewGroup.OnHierarchyChangeListener, ViewDebug.HierarchyHandler {
    
    ...  
    private static WebViewFactoryProvider getFactory() {
        return WebViewFactory.getProvider();
    }
}
```

#### 二、通过WebViewFactory#getProvider方法创建WebViewFactoryProvider实例

[/frameworks/base/core/java/android/webkit/WebViewFactory.java]

```java
package android.webkit;

@SystemApi
public final class WebViewFactory {
    private static final String CHROMIUM_WEBVIEW_FACTORY_METHOD = "create";

    public static final String CHROMIUM_WEBVIEW_VMSIZE_SIZE_PROPERTY =
            "persist.sys.webview.vmsize";

    private static final String LOGTAG = "WebViewFactory";
  
    // Cache the factory both for efficiency, and ensure any one process gets all webviews from the
    // same provider.
    private static WebViewFactoryProvider sProviderInstance;

    private static WebViewFactoryProvider sProviderInstance;
    private static final Object sProviderLock = new Object();
    
    static WebViewFactoryProvider getProvider() {
        synchronized (sProviderLock) {
            if (sProviderInstance != null) return sProviderInstance;

            final int uid = android.os.Process.myUid();
            if (uid == android.os.Process.ROOT_UID ||
                    uid == android.os.Process.SYSTEM_UID ||
                    uid == android.os.Process.PHONE_UID ||
                    uid == android.os.Process.NFC_UID ||
                    uid == android.os.Process.BLUETOOTH_UID) {
                throw new UnsupportedOperationException(
                        "For security reasons, WebView is not allowed in privileged processes");
            }

            if (!isWebViewSupported()) {
                // Device doesn't support WebView; don't try to load it, just throw.
                throw new UnsupportedOperationException();
            }

            if (sWebViewDisabled) {
                throw new IllegalStateException(
                        "WebView.disableWebView() was called: WebView is disabled");
            }

            //获取名为"com.android.webview.chromium.WebViewChromiumFactoryProviderForP"的Class对象。
            Class<WebViewFactoryProvider> providerClass = getProviderClass();
            Method staticFactory = null;
            try {
                //获取WebViewChromiumFactoryProviderForP类中的create()方法。
                staticFactory = providerClass.getMethod(
                        CHROMIUM_WEBVIEW_FACTORY_METHOD, WebViewDelegate.class);
            } catch (Exception e) {
            }

            try {
                //反射调用WebViewChromiumFactoryProviderForP的create()方法。
                sProviderInstance = (WebViewFactoryProvider)
                        staticFactory.invoke(null, new WebViewDelegate());
                return sProviderInstance;
            } catch (Exception e) {
                Log.e(LOGTAG, "error instantiating provider", e);
                throw new AndroidRuntimeException(e);
            }
        }
    }
}
```

##### 1. 获取名为"com.android.webview.chromium.WebViewChromiumFactoryProviderForP"的Class对象。

首先加载.so动态库

```java
WebViewLibraryLoader.loadNativeLibrary(clazzLoader,
                                       getWebViewLibrary(sPackageInfo.applicationInfo));
```

```java
package android.webkit;

@SystemApi
public final class WebViewFactory {
     // visible for WebViewZygoteInit to look up the class by reflection and call preloadInZygote.
    /**
     * @hide
     */
    private static final String CHROMIUM_WEBVIEW_FACTORY =
            "com.android.webview.chromium.WebViewChromiumFactoryProviderForP";
    /**
     * @hide
     */
    public static Class<WebViewFactoryProvider> getWebViewProviderClass(ClassLoader clazzLoader)
            throws ClassNotFoundException {
        return (Class<WebViewFactoryProvider>) Class.forName(CHROMIUM_WEBVIEW_FACTORY,
                true, clazzLoader);
    }
}

```

##### 2. 通过反射调用WebViewChromiumFactoryProviderForP#create方法

其中，传递的参数为new WebViewDelegate()对象。

WebViewFactoryProvider

[/frameworks/base/core/java/android/webkit/WebViewFactoryProvider.java]

```java
package android.webkit;

import android.annotation.NonNull;
import android.annotation.SystemApi;
import android.content.Context;
import android.content.Intent;
import android.net.Uri;

import java.util.List;

/**
 * This is the main entry-point into the WebView back end implementations, which the WebView
 * proxy class uses to instantiate all the other objects as needed. The backend must provide an
 * implementation of this interface, and make it available to the WebView via mechanism TBD.
 * @hide
 */
@SystemApi
public interface WebViewFactoryProvider {
    /**
     * This Interface provides glue for implementing the backend of WebView static methods which
     * cannot be implemented in-situ in the proxy class.
     */
    interface Statics {
        String findAddress(String addr);

        String getDefaultUserAgent(Context context);

        void freeMemoryForTests();

        void setWebContentsDebuggingEnabled(boolean enable);

        void clearClientCertPreferences(Runnable onCleared);

        void enableSlowWholeDocumentDraw();

        Uri[] parseFileChooserResult(int resultCode, Intent intent);

        void initSafeBrowsing(Context context, ValueCallback<Boolean> callback);

        void setSafeBrowsingWhitelist(List<String> hosts, ValueCallback<Boolean> callback);

        @NonNull
        Uri getSafeBrowsingPrivacyPolicyUrl();
    }

    Statics getStatics();

    /**
     * Construct a new WebViewProvider.
     *
     * @param webView       the WebView instance bound to this implementation instance. Note it will not
     *                      necessarily be fully constructed at the point of this call: defer real initialization to
     *                      WebViewProvider.init().
     * @param privateAccess provides access into WebView internal methods.
     */
    WebViewProvider createWebView(WebView webView, WebView.PrivateAccess privateAccess);

    GeolocationPermissions getGeolocationPermissions();

    CookieManager getCookieManager();

    TokenBindingService getTokenBindingService();

    TracingController getTracingController();

    ServiceWorkerController getServiceWorkerController();

    WebIconDatabase getWebIconDatabase();

    WebStorage getWebStorage();

    WebViewDatabase getWebViewDatabase(Context context);

    ClassLoader getWebViewClassLoader();
}
```

##### 3. WebViewChromiumFactoryProviderForP

该类位于chromium中。

[android_webview/glue/java/src/com/android/webview/chromium/WebViewChromiumAwInit.java]

```java
package com.android.webview.chromium;

public class WebViewChromiumFactoryProviderForP extends WebViewChromiumFactoryProvider {
  
    public static WebViewChromiumFactoryProvider create(android.webkit.WebViewDelegate delegate) {
        return new WebViewChromiumFactoryProviderForP(delegate);
    }

    protected WebViewChromiumFactoryProviderForP(android.webkit.WebViewDelegate delegate) {
        super(delegate);
    }
}
```

#### 三、WebDelegate
[/frameworks/base/core/java/android/webkit/WebViewDelegate.java]

