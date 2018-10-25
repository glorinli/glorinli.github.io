---
title: 【Android】NDK b便宜，缺少gnu-libstdc++ 相关的资源
date: 2018-10-25
---
# 问题
使用 JNI 编译时，提示找不到 gnu-libstdc++ 相关的资源，NDK 版本是 18

# 原因
在 NDK 18中，确实缺少了相关的资源，不知道是不是谷歌有意去除。

# 解决方法
下载 16 版本的 ndk: android-ndk-r16b， 然后copy相关组件到自己的ndk library中，位于
android-ndk-r16b/sources/cxx-stl目录中。

