---
title: 【Android】ARouter 入门
date: 2019-11-14 07:39:49
tags:
- Android
- 组件化
categories:
- Android
---
ARouter 是阿里巴巴开源的一款 Android 路由组件，可以用来控制 Android App 的界面跳转等。
本文基于 arouter-api 1.4.1, arouter-compiler 1.2.2 版本。

### Gradle 配置
首选是选择要引入 ARouter 的模块，这边直接使用 app 模块，修改 build.gradle:

``` groovy
apply plugin: 'com.android.application'
android {
    compileSdkVersion 28
    defaultConfig {
        applicationId "li.glorin.arouterdemo"
        // 省略
        // Arounter
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [AROUTER_MODULE_NAME: project.getName()]
            }
        }
    }
}
dependencies {
    // 省略
    // ARouter
    implementation ('com.alibaba:arouter-api:1.4.1') {
        exclude group: 'com.android.support'
    }
    annotationProcessor 'com.alibaba:arouter-compiler:1.2.2'
}
```
这边要注意，**compiler 的版本和 api 的版本不一样**，我们经常使用的如 ButterKnife 等，annotationProcessor 版本经常都是和 api 版本一致的，这是容易犯的一个想当然的小错误，可能笔者比较粗心，直接将compiler也设置成1.4.1，结果编译总是报错： Could not find com.alibaba:arouter-compiler: 1.4.1。

### 初始化
在 Application 的 onCreate 里面初始化：
``` java
public class App extends Application {
    @Override
    public void onCreate() {
        super.onCreate();

        // Init ARouter
        if (BuildConfig.DEBUG) {
            // Should set log and debug before init
            ARouter.openLog();
            ARouter.openDebug();
        }

        ARouter.init(this);
    }
}
```
这边注意，openLog 和 openDebug 都要**在 init 之前**调用，才会生效。
### 简单使用
假设我们有两个 Activity：MainActivity 和 TestActivity。
在TestActivity中，使用注解配置路由路径：
``` java
@Route(path = "/test/activity")
public class TestActivity extends AppCompatActivity {
    private static final String TAG = "TestActivity";

    @Autowired(name = "date")
    String mDate;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // Call this inject method to autowire data
        ARouter.getInstance().inject(this);

        Log.d(TAG, "Date = " + mDate);
    }
}
```
在 MainActivity 的点击事件中，使用 ARouter 跳转到 TestActivity:
``` java
public void onClick(View view) {
        switch (view.getId()) {
            case R.id.buttonTest:
                ARouter.getInstance().build("/test/activity")
                        .withString("date", "Today")
                        .navigation();
                break;
        }
    }
```
如此即可完成简单的跳转。
### 简单参数传递
在上面的示例代码中，我们发现在 TestActivity 中，有一个由 Autowired 注解的 Field mData， 它的 ``` name = "date"```，而在 MainActivity 启动 Test 的代码中，我们有一个 ``` .withString("date", "Today")```的调用，简单猜测一下，也知道这个是用来传递参数的，至于具体的实现，后续再行分析。

ARouter的入门就先介绍到这，感谢各位看官阅读。水平有限，如有错漏，烦请不吝指教。
