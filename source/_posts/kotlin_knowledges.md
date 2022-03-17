---
title: Kotlin 知识集合
date: 2022-03-17
categories:
- Kotlin
tags:
- Kotlin
---
## 问题1
### 为什么 Java 匿名内部类访问局部变量需要标记为 final，而 Kotlin 不用？
![](/imgs/java_i_cannot_compile.png)
![](/imgs/kotlin_i_ok.png)
### 解答
#### 1. 为什么 Java 需要标记 final？
对于 Java 来说，对于闭包的处理是直接将引用的局部变量 copy 一份，也就是说，Runnable 内部的 i，和外部的局部变量 i，已经不是同一个变量了，因此如果外部 i，改变了，内部 i 还是不变的，反之亦然，这样十分容易出现错误，因为开发者会质疑为何变量不同步。
为了解决这种疑惑，Java 语言直接在编译期就限制外部变量必须为 final，不可修改，也就不会有不一致的问题。
#### 2. 为什么 kotlin 不用？
Kotlin 其实也是一样的，之所以不用加 final （使用val），是因为 kotlin 在编译的时候悄悄把 i 的类型替换了，我们可以使用 IDE 反编译 kotlin class 为 Java，然后查看代码：
![](/imgs/kotlin_i_decompile.png)
我们发现Kotlin其实将 i 替换为 IntRef 类型了，而且也是加了 final 的，如此一来便只是修改 IntRef 内部的值，而不修改 i 本身，这也是 kotlin 不需要 final 的原因。

