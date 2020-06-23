---
title: 【Android】Android App 默认有哪些线程
date: 2020-06-23
tags:
- Android
- 多线程
categories:
- Android
---
今天偶然看到一个问题：一个Android App，有几个线程
首先这个问题就有问题，一个app，线程数量肯定是不固定的，但是我又觉得这个问题有点意思，于是转换一下思路，不如研究研究一个最简单的app，默认有几个线程。

## 创建一个最简单的app
用as创建一个最简单的app，然后打印出线程信息，我们使用Thread.getAllStackTraces方法，来获取线程信息，代码如下：
``` kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val sb = StringBuilder()

        sb.append("Current threads: \n\n")

        Thread.getAllStackTraces().forEach {
            sb.append(it.key.name)
            sb.append("\n")
        }

        tvMain.text = sb.toString()
    }
}
```

运行后，显示的结果如下:
![](/imgs/20200622114435940_19888.png =320x)

可以看出，一个最简单的app，有如下几个线程：
* main
* Binder:interceptor
* FinalizerDaemon
* FinalizerWatchdogDaemon
* HeapTaskDaemon
* ReferenceQueueDaemon

## 线程作用
经过一系列搜索，总结一下这几个默认创建的Thread的作用，其中，四个Daemon的代码都位于libcore/libart/src/main/java/java/lang/Daemons.java中。
### main
这个不必多说，主线程，也叫UI线程
### Binder:interceptor
### ReferenceQueueDaemon
这是一个守护进程，它的工作就是不停地把ReferenceQueue.unenqueued，通过ReferenceQueue.enqueuePending方法进行入队。
### FinalizerDaemon
我们知道，Java对象是有一个finalize方法的，如果一个类override了这个方法，并且方法体不为空，那么，gc的时候，gc会将对象加入FinalizerReference.queue中，而FinalizerDaemon就是负责不停的将queue里面的对象取出来，调用它的finalize方法，然后再释放掉。
### FinalizerWatchdogDaemon
这个Daemon线程是用来监视FinalizerDaemon的，如果FinalizerDaemon处理一个对象的finalize过程，超过了一个时间（10秒），就发送一个SIGQUIT，获取Native堆栈。而如果Thread没有设置UncaughtExceptionPreHandler，也没有设置DefaultUncaughtExceptionHandler，就直接System.exit(2)，杀死进程。
### HeapTaskDaemon
这个也是GC相关的，主要是用来调用VMRuntime.getRuntime().runHeapTasks();方法。

## 参考
[江义旺：滴滴出行安卓端 finalize time out 的解决方案](https://segmentfault.com/a/1190000019373275?utm_source=tag-newest)
[再谈Finalizer对象--大型App中内存与性能的隐性杀手](https://yq.aliyun.com/articles/225755)
