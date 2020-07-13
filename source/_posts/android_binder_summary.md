---
title: 【Android】Android Binder 总结
date: 2020-07-13
tags:
- Android
- Binder
categories:
- Android
---
通过最近对Binder的源码阅读，以及拜读了各路大神的文章后，再次对自己的理解做一个小小的记录。
## Binder 是什么
Binder 是 Android 系统中的一中进程间通信的机制，前身是 OpenBinder，Binder 是 Android 系统最重要的组件之一，也是整个系统的基石。
同时，Binder 也是一个系统设备 /dev/binder 对应的是 Binder 驱动，是 Binder 的核心。
从实现机制上来说，Binder是一个token，用来标识一个服务，这个token可以跨进程地传递，拿到binder对象，就可以发起远程访问。
## Linux 常见的进程间通信方式及特点
### 共享内存
无需拷贝
逻辑复杂，需要应用程序处理进程间数据同步
### 信号量
适合事件通知，不适合传递大量数据
### Socket
速度慢，两次拷贝，适合低速通信，或跨设备通信
### 文件共享
速度慢
## 为什么采用 Binder
* 效率高，仅需一次拷贝
* 封装完善，由系统统一管理进程间的数据同步、线程池管理等等，使用方便
* 语法简洁，可以以同步的方式完成ipc通信，使程序逻辑更简单
* 支持 Java 层，Native 层
* 由系统统一处理uid，方便进行权限控制，安全性高
* AIDL 工具支持直接生成Binder代码，方便使用
## Binder 主要概念
### IBinder
代表一个可以远程访问的对象，所谓远程访问，就是跨进程访问。
IBinder主要有几个方法：
* getInterfaceDescriptor(): 获取这个对象的名称
* pingBinder(): 检查这个对象是否还存在
* isBinderAlive()：检查提供这个Binder对象的进程是否还存活
* queryLocalInterface(@NonNull String descriptor): 查询本地接口，对于同进程使用 Binder，这个方法会直接返回 Binder 的实现类，对于跨进程的场景，这个方法返回 null
* transact(int code, @NonNull Parcel data, @Nullable Parcel reply, int flags): Binde r的核心方法，用来向远程服务发送数据
* linkToDeath: 死亡监听
### IInterface
Binder服务的基础接口，只有一个方法：asBinder，用来获取对应的Binder对象。IInterface 主要是用来作为服务端和客户端的一个协议规范，即服务端实现相应的方法，客户端来调用，这个规范就是一个IInterface。
### Parcel
Parcel 是一个支持数据序列化的组件，我们知道，在进程间传递消息，由于进程间对象、类型都不共用，因此必须序列化成通用的数据，才能进行传递，Parcel就是用来做这个事情的，我们常用的Parcelable，就是基于Parcel实现的。在 binder transact 过程中使用。
### Binder
这里我们说的是Java层的Binder类，这个类是IBinder的一个基本实现，实现了 Binder 的本地调用（同进程），即 transact 方法直接调用 onTransact 方法，也是在这里引入了 onTransact方法，我们在使用 AIDL 的时候，Stub 类就是 Binder 的子类。
对于 Native 层的 Binder，作用也是一样的，只不过是 Native 的实现，用来给 Native 代码使用。
### BinderInternal
这里提供一些 Binder 的内部 API（不给app使用的），常见的比如：
joinThreadPool: 将调用线程加入Binder线程池
getContextObject: 获取系统的“全局上下文”（Global Context），其实就是 0 号 Binder，也就是 ServiceManager，这个方法在 ServiceManager 的 getIServiceManager 方法里面有使用到，就是用来获取 ServiceManager 的 Binder 对象，这个后续 ServiceManager 中会介绍。
setMaxThreads:
### BinderProxy
很重要的一个类，是 Java 层对 Native Binder 的一个代理，由于 Binder 的主要实现在 Native 层，如果 Java 层要使用，就要通过这个 BinderProxy，这个类的对象是由 Native 的 javaObjectforIBinder 方法创建的，在 Java 层，不会创建这个对象。
BinderProxy 主要是定义了一堆的 native 方法，用来实现 Java 层和 Native 层的调用，其中最主要的是 transactNative 方法，这个方法就是 Java 层 transact 的根本所在，也就是走到了 Native 层的 transact 逻辑。
对应的 Native 实现，在 android_util_Binder.cpp 这个文件中。
### BBinder
BBinder 就是 Binder 的 Native 实现，也就是说，这个是在提供服务的地方创建的，用来处理 Binde r的逻辑，这个在上面 Binder 的地方有提到过了。
### BpBinder
BpBinder  是 Native 层最重要的类之一，我们知道，Binder 是一中 C/S 架构，而 BpBinder 就是 Client 端真正做事的那个人，不管是 Java 的 BinderProxy，还是直接 Native 使用 Binder，最终发送和接收数据的就是 BpBinder，transact 最终的实现就是在这里，BpBinder 调用 IPCThreadState，实现 IPC 请求，和结果的监听。可以看看 transact 方法的代码：
``` cpp
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
```
### ProcessState
这个是 Binder 环境的初始化类，看名字是进程相关的，这个类是单例的，在创建的时候，会打开 Binder 驱动，并完成和 Binder 驱动的关系的建立，也就是通过 mmap 来做一个内存映射，这样 Binder 驱动就可以对这个进程发送 transact 数据。
### IPCThreadState
这个是 Binder 线程相关的，最终和 Binder 驱动交换数据的工作是由它来完成的。
### Binder Driver
Binder 驱动，Binder 通信机制的核心，主要的工作就是
1. 维护进程间的 binder 对象，保存引用
2. 在进程间传递消息
### ServiceManager
ServiceManager 是一个管理器，可以理解为 binder 服务的一个管理处，主要是两个作用：
1. 注册服务
2. 获取服务

对于比如 ActivityManagerService、WindowManagerService 等系统服务，都是有在 ServiceManager 中注册过的，因此他们叫做“实名Binder”，而未注册的就叫“匿名Binder”
有一种说法很形象，Binder 作为一个 C/S 架构的通信机制，binder 就是 每个服务的 IP 和端口，而 ServiceManager 就是一个 DNS 服务器，将 名称 映射到对应的binder，方便客户端通过名称访问服务。
## Binder 一次拷贝的原理
最根本的就是使用了mmap进行了内存的映射，上面有提到，在ProcessState初始化的时候，会进行一个mmap，这是服务端进程（用户空间）到binder驱动（内核空间）的一个映射，而在binder驱动内部，也采用mmap对不同进程的缓冲区做了mmap，于是就相当于从客户端的内核空间的一块内存，直接映射到了服务端的用户空间的一块内存，因此只需要从客户端拷贝一次数据到内核空间，数据就会自动到达服务端的用户空间，这就是只需要一次拷贝的原因。

具体可参考此文章：https://blog.csdn.net/AndroidStudyDay/article/details/93749470
## Binder 调用过程
这个问题要区分客户端和服务端
### 客户端
对于客户端，调用过程如下: 业务方法 -> BinderProxy.transact -> BpBinder.transact -> IPCThreadState.transact -> talkWithDriver -> binder_ioctl (驱动部分省略)
### 服务端
对于服务端，主要是Binder线程不停地轮询，调用IPCThreadState.getAndExcuteCommand -> talkWithDriver -> binder_ioctl -> executeCommand -> BBinder.transact -> BBinder.onTransact
而 BBinder 是怎么调用到Java层的Binder对象的方法呢，秘密在android_util_Binder的JavaBBinder中：
``` cpp
const char* const kBinderPathName = "android/os/Binder";

static int int_register_android_os_Binder(JNIEnv* env)
{
    jclass clazz = FindClassOrDie(env, kBinderPathName);
    ...
    gBinderOffsets.mExecTransact = GetMethodIDOrDie(env, clazz, "execTransact", "(IJJI)Z");
    ...
}
```
也就是说是通过这边，实现Native到Java层的调用的。

所以服务端的完整流程是: IPCThreadState.getAndExcuteCommand -> talkWithDriver -> binder_ioctl -> executeCommand -> BBinder.transact -> BBinder.onTransact -> JavaBBinder.onTransact -> Binder.execTrasact -> Binder.onTransact
## Java Binder 和 Native Binder 联系
Java Binder 是对 Native Binder的封装，方便Java 层使用，两者交互的逻辑主要在 android_util_Binder.cpp 中。
## 实名 Binder，匿名 Binder 区别与联系
上面有提到过，在ServiceManager中登记过的，就叫实名Binder，其他的叫匿名Binder，对于匿名Binder，需要有实名Binder作为媒介，才能建立通信，否则Binder无法在进程间传递。
## 参考
http://gityuan.com/2016/09/04/binder-start-service/
https://blog.csdn.net/universus/article/details/6211589
https://blog.csdn.net/AndroidStudyDay/article/details/93749470
