---
title: 【Android】Android Binder 01 - 从AIDL说起
date: 2020-06-18
tags:
- Android
- Binder
categories:
- Android
---
## 一个AIDL Demo
Demo代码：https://github.com/glorinli/AIDLDemo

### 首先定义AIDL接口:

AlbumManager.aidl
``` java
package xyz.glorin.aidlcommon.aidl;

import xyz.glorin.aidlcommon.aidl.Album;

interface AlbumManager {
    List<Album> getAlbums();
    void addAlbum(in Album album);
}
```
Album.aidl
``` java
package xyz.glorin.aidlcommon.aidl;

parcelable Album;
```

Build一下，自动生成Binder相关类
![](/imgs/20200617101425269_645.png)

这个自动生成的类就是核心代码了，后续我们来分析，现在先看看如何使用AIDL
### 使用AIDL

#### 首先是服务端：
```java
class AlbumService : Service() {
    private val albumList: MutableList<Album> = mutableListOf()

    override fun onBind(intent: Intent?): IBinder? {
        return AlbumServiceBinder()
    }

    inner class AlbumServiceBinder : AlbumManager.Stub() {
        override fun addAlbum(album: Album?) {
            album?.let { albumList.add(it) }
        }

        override fun getAlbums(): MutableList<Album> {
            return albumList
        }

    }
}
```

可以看出，我们重写了Service的onBind方法，返回一个IBinder，而这个IBinder就是一个AlbumManager.Stub的子类，我们返回去看看AlbumManager，发现它确实是一个IBinder
``` java
// 可以看出Stub继承了Binder，而Binder实现了IBinder，因此Stub是一个IBinder
public static abstract class Stub extends android.os.Binder implements xyz.glorin.aidlcommon.aidl.AlbumManager
```
#### 接着看看客户端
![](/imgs/20200617104944431_2883.png)

核心逻辑在onServiceConnected回调中，在这里，我们使用
``` java
AlbumManager.Stub.asInterface(service)
```
获取了一个AlbumManager对象，后续就可以调用 AlbumManager 对象进行跨进程调用了。
所以我们返回去看看 AlbumManager.Stub.asInterface 方法
``` java
/**
     * Cast an IBinder object into an xyz.glorin.aidlcommon.aidl.AlbumManager interface,
     * generating a proxy if needed.
     */
    public static xyz.glorin.aidlcommon.aidl.AlbumManager asInterface(android.os.IBinder obj)
    {
      if ((obj==null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof xyz.glorin.aidlcommon.aidl.AlbumManager))) {
        return ((xyz.glorin.aidlcommon.aidl.AlbumManager)iin);
      }
      return new xyz.glorin.aidlcommon.aidl.AlbumManager.Stub.Proxy(obj);
    }
```

可见返回值有两种情况
1. 直接强制转换 iin 为 AlbumManager
2. 返回一个Stub.Proxy对象

其中，Stub.Proxy 也实现了AlbumManager，所以asInterface方法最终是返回一个AlbumManager对象
``` java
private static class Proxy implements xyz.glorin.aidlcommon.aidl.AlbumManager
```
## 总结
1. 工具会根据aidl接口定义，自动生成对应的类，其中最重要的就是AlbumManager.Stub
2. 提供服务的Service，在onBind方法返回一个Stub实例。
3. 调用服务的Client，在onServiceConnect回调中，利用Stub.asInterface方法，将系统回传回来的service(IBinder)，转换为AlbumManager，即可发起跨进程调用。

可以看出，IBinder就是连接Service和Client的桥梁，后续文章我们将深入这个IBinder，以此来窥探Binder机制的本质。

