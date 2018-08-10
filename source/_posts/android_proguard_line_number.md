---
title: 【Android】Proguard 混淆时保留行号
date: 2018-08-10
---
# 问题
在使用Proguard进行混淆的时候，如果没有保留行号，在Retrace的时候就会难以判断对应的方法，这是因为不同的方法，只要参数列表不一样，都有可能被混淆成相同的变量名，因此如果没有行号，Retrace也就无法正确还原。

# 解决方法
在proguard配置中加入以下两行，即可保留行号

``` bash
# Keep line numbers
-renamesourcefileattribute XXX # XXX 会替换混淆后的源文件名，可以自行指定
-keepattributes SourceFile,LineNumberTable
```
