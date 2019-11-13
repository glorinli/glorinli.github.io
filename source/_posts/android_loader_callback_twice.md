---
title: 【Android】LoaderManager.LoaderCallbacks.onLoadFinished 被多次调用的问题
date: 2018-09-13

categories:
- Android
tags:
- Android
---
### 问题
在小米 MIX 上使用 LoaderManager 的时候，发现在 onCreate 的时候发起一个 Loader，结果每次从下一个 Activity 返回的时候，onLoadFinished 都会被调用一次，导致界面异常刷新。
StackOverFlow了一下，发现可能是 Loader 框架本身的BUG，或者是手机厂商的锅。

### 解决方法
规避掉，即添加一个变量控制是否处理 onLoadFinished方法，在每次发起的时候置为 true，在 onLoadFinished 方法中，判断并重置。

``` java
public class Activity... {
    private boolean mWaitingForData;

    public void load {
        mWaitingForData = true;
        // Start load
    }

    public void onLoadFinished(...) {
        if(mWaitingForData) {
            // Handle data
            mWaitingForData = false;
        }
    }
}
```
