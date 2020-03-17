---
title: 【Android】关于 Context 的一些想法
date: 2020-03-17
categories:
- Android
tags:
- Android
---
## Context 是什么
看注释:

``` java
/**
 * Interface to global information about an application environment.  This is
 * an abstract class whose implementation is provided by
 * the Android system.  It
 * allows access to application-specific resources and classes, as well as
 * up-calls for application-level operations such as launching activities,
 * broadcasting and receiving intents, etc.
 */
 ```
 
 翻译一下就是
 
 ``` java
 Context 是 App 全局信息的一个接口，这是一个抽象类，具体实现由 Android 系统提供，
 用来支持获取 App 相关的资源和类，以及调用一些 App 级别的操作，比如打开 Activity， 发送广播等。
 ```
 
## Activity 的继承关系
Activity -> ContextThemeWrapper -> ContextWrapper -> Context
## Application 的继承关系
Application -> ContextWrapper -> Context
## Service 的继承关系
Service -> ContextWrapper -> Context
## ContextWrapper
由 Activity 和 Application, Service 的继承关系，我们可以看到，Context 最主要的一个实现类就是 ContextWrapper，看看 ContextWrapper 的代码:

``` java
/**
 * Proxying implementation of Context that simply delegates all of its calls to
 * another Context.  Can be subclassed to modify behavior without changing
 * the original Context.
 */
public class ContextWrapper extends Context {
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }
    
    /**
     * Set the base context for this ContextWrapper.  All calls will then be
     * delegated to the base context.  Throws
     * IllegalStateException if a base context has already been set.
     * 
     * @param base The new base context for this wrapper.
     */
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
    ...
}
```

由注释和代码可以发现，ContextWrapper 真的是一个 Wrapper，操作都转发给这个 mBase了，所以这是个代理类，我们要一探究竟，还是要去看 mBase。
![](_v_images/20200317141508120_25543.png)

我们看到，在 Activity， Application， Service 中，都有去调用 attachBaseContext，这就是给 mBase 赋值的地方了，这时候用 Android Studio 就找不着哪里调用了，于是我们转去看 AOSP 源码。
我们先从 Activity 入手，已知 Activity 是在什么 ActivityThread 里面创建的（为什么已知，就要靠多研究啦，这里先不求甚解一下），搜了一下，我们发现了 attach 的地方：
``` java

2807    /**  Core implementation of activity launch. */
2808    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
            ... 省略
2826
2827        ContextImpl appContext = createBaseContextForActivity(r);
2828        Activity activity = null;

2829        ... 省略
2846
2847        try {
2848            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
                ... 省略
2857
2858            if (activity != null) {
2859                ... 省略

2872                appContext.setOuterContext(activity);
2873                activity.attach(appContext, this, getInstrumentation(), r.token,
2874                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
2875                        r.embeddedID, r.lastNonConfigurationInstances, config,
2876                        r.referrer, r.voiceInteractor, window, r.configCallback);
```

所以我们来看 createBaseContextForActivity:
``` java
2990    private ContextImpl createBaseContextForActivity(ActivityClientRecord r) {
2991        final int displayId;
2992        try {
2993            displayId = ActivityManager.getService().getActivityDisplayId(r.token);
2994        } catch (RemoteException e) {
2995            throw e.rethrowFromSystemServer();
2996        }
2997
2998        ContextImpl appContext = ContextImpl.createActivityContext(
2999                this, r.packageInfo, r.activityInfo, r.token, displayId, r.overrideConfig);
3000
3001        ...

3017        return appContext;
3018    }
```

破案了，原来这个 mBase 就是 ContextImpl。

## Activity，Service， Application 都是 Context的子类，有什么区别
其实就是问 ContextWrapper 和 ContextThemeWrapper 有什么区别，所以我们只要去看看 ContextThemeWrapper 就行了。总体来说就是加了主题、Resource 相关的一些东西。

## 一个 App有几个Context
我们会这样想，Activity， Service， Application 都是 Context 子类，所以 App 的 Context 个数是:
1 Application + n Activity  + n Service

再想想不对，应用不一定单进程，所以Application 可能有多个了，所以还要考虑多进程的问题。

再想想还是不对，Context 里面不是 还有个 mBase，这也是一个 Context 实例，所以恐怕还要 x2

再想想，除了这些场景，还会有其他地方创建 Context 吗？这个就不深究了。
