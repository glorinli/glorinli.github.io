---
title: 【Android】关于Library Project中so库打包的问题
date: 2017-05-16
---

## 问题
大家知道，Android Studio是支持library类型的module的，这些module可以直接被别的module引用，也可以导出为aar，供别的project使用，如果在library Project中含有native的so库，其中还是有点小坑的。

目前Android Studio对native的支持还不是很完美，或者说，很多项目还是使用旧的ndk-build方式进行管理的，而ndk-build生成的so库默认是放在<module location>/libs/abi(armeabi等)下面的。

而gradle默认的jni库，应该是放在jniLibs目录下的。

如果是直接在工程中使用，只需要在gradle文件的android节点，指定jni lib的位置即可：

``` gradle
sourceSets.main.jniLibs.srcDirs = ['libs']
```

这样gradle就会去libs目录下寻找so库。

但是如果要让其他module引用，或者导出为aar（两者本质上一样），就会出现so库没有被编译到aar包和目标apk的问题。

## 方案
经过我的试验，发现只有把so库放到src/main/jniLibs目录下，才能被正常编译到aar和apk中，因此我们需要在ndk-build之后，手动移动一下so库的位置。

当然每次手动移动效率也太慢了，我们可以在ndk-build的时候指定输出的位置，如

``` bash
ndk-build NDK_LIBS_OUT=../jniLibs
```
NDK_LIBS_OUT： 指定so库的输出位置，需要ndk9版本支持

NDK_OUT： 指定obj的位置，需要ndk7c版本支持

## 其他
以上基于Android Studio 2.3 和 Gradle 2.3.1版本

参考[http://stackoverflow.com/questions/30865110/change-ndk-build-output-locations](http://stackoverflow.com/questions/30865110/change-ndk-build-output-locations)
