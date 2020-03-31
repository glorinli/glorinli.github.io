---
title: 【Android】Bad method handle type 7 解决办法
date: 2020-03-31 20:19:00
tags:
- Android
- 构建
- Gradle
categories:
- Android
---
今天在创建一个Demo项目的时候，某次编译运行后，发现一打开就闪退，查看闪退日志如下：
``` java
2020-03-31 19:24:40.776 29643-29643/xyz.glorin.gankyo E/AndroidRuntime: FATAL EXCEPTION: main
    Process: xyz.glorin.gankyo, PID: 29643
    java.lang.RuntimeException: Unable to instantiate application xyz.glorin.gankyo.App: java.lang.ClassNotFoundException: Didn't find class "xyz.glorin.gankyo.App" on path: DexPathList[[zip file "/data/app/xyz.glorin.gankyo-vTz_RQM4TTzoqJhZTkXffw==/base.apk"],nativeLibraryDirectories=[/data/app/xyz.glorin.gankyo-vTz_RQM4TTzoqJhZTkXffw==/lib/arm64, /system/lib64, /system/vendor/lib64]]
        at android.app.LoadedApk.makeApplication(LoadedApk.java:978)
        at android.app.ActivityThread.handleBindApplication(ActivityThread.java:5851)
        at android.app.ActivityThread.-wrap1(Unknown Source:0)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1690)
        at android.os.Handler.dispatchMessage(Handler.java:105)
        at android.os.Looper.loop(Looper.java:173)
        at android.app.ActivityThread.main(ActivityThread.java:6698)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:240)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:782)
     Caused by: java.lang.ClassNotFoundException: Didn't find class "xyz.glorin.gankyo.App" on path: DexPathList[[zip file "/data/app/xyz.glorin.gankyo-vTz_RQM4TTzoqJhZTkXffw==/base.apk"],nativeLibraryDirectories=[/data/app/xyz.glorin.gankyo-vTz_RQM4TTzoqJhZTkXffw==/lib/arm64, /system/lib64, /system/vendor/lib64]]
        at dalvik.system.BaseDexClassLoader.findClass(BaseDexClassLoader.java:93)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:379)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:312)
        at android.app.Instrumentation.newApplication(Instrumentation.java:1087)
        at android.app.LoadedApk.makeApplication(LoadedApk.java:972)
        at android.app.ActivityThread.handleBindApplication(ActivityThread.java:5851) 
        at android.app.ActivityThread.-wrap1(Unknown Source:0) 
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1690) 
        at android.os.Handler.dispatchMessage(Handler.java:105) 
        at android.os.Looper.loop(Looper.java:173) 
        at android.app.ActivityThread.main(ActivityThread.java:6698) 
        at java.lang.reflect.Method.invoke(Native Method) 
        at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:240) 
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:782) 
    	Suppressed: java.io.IOException: Failed to open dex files from /data/app/xyz.glorin.gankyo-vTz_RQM4TTzoqJhZTkXffw==/base.apk because: Failure to verify dex file '/data/app/xyz.glorin.gankyo-vTz_RQM4TTzoqJhZTkXffw==/base.apk': Bad method handle type 7
        at dalvik.system.DexFile.openDexFileNative(Native Method)
        at dalvik.system.DexFile.openDexFile(DexFile.java:353)
        at dalvik.system.DexFile.<init>(DexFile.java:100)
        at dalvik.system.DexFile.<init>(DexFile.java:74)
        at dalvik.system.DexPathList.loadDexFile(DexPathList.java:374)
        at dalvik.system.DexPathList.makeDexElements(DexPathList.java:337)
        at dalvik.system.DexPathList.<init>(DexPathList.java:157)
        at dalvik.system.BaseDexClassLoader.<init>(BaseDexClassLoader.java:65)
        at dalvik.system.PathClassLoader.<init>(PathClassLoader.java:64)
        at com.android.internal.os.PathClassLoaderFactory.createClassLoader(PathClassLoaderFactory.java:43)
        at android.app.ApplicationLoaders.getClassLoader(ApplicationLoaders.java:69)
        at android.app.ApplicationLoaders.getClassLoader(ApplicationLoaders.java:36)
        at android.app.LoadedApk.createOrUpdateClassLoaderLocked(LoadedApk.java:681)
        at android.app.LoadedApk.getClassLoader(LoadedApk.java:714)
        at android.app.LoadedApk.getResources(LoadedApk.java:941)
        at android.app.ContextImpl.createAppContext(ContextImpl.java:2254)
        at android.app.ActivityThread.handleBindApplication(ActivityThread.java:5755)
        		... 8 more
```

经过一番搜索，发现有人遇到相同问题: https://stackoverflow.com/a/50983181/4087988

解决方案就是在 app/build.gralde 的 android 配置下，增加

``` groovy
compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
}
```

原因的话，是因为有一些类库可能使用了Java8的新特性，我们知道，Android对待Java8，是使用desugar来进行处理的，还原为低版本Java的字节码。设置这个sourceCompatibility，就是告诉gradle，我们有Java8的代码，需要额外处理，否则运行就失败了。


