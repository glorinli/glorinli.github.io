---
title: [WIP]【Android】自定义 Lint 检查（二）
date: 2022-01-18
categories:
- Android
tags:
- Android
---
# 背景
上一篇文章我们讲了如何自定义一个简单的 lint 检查规则，这篇文章我们讲讲 lint 规则如何发布与集成。
# 问题
上一篇文章中，我们编写的 lint 规则，通过 `implementation project(':wrapper')` 在主工程中引入我们的规则，但是在一般实际生产的场景中，我们很少通过源码的方式去依赖，而是更偏向于使用 gradle 依赖来实现，本文就来讲讲如何将我们的自定义 lint 规则发布到 maven，从而使其他工程可以直接用 gradle 依赖的方式来使用。
# 升级 Gradle
在写这篇文章的时候，gradle 和 Android Gradle Plugin 都有了较大的版本升级，这里有一个限制需要注意： 自定义 lint 工程的 AGP，必须和使用 lint 规则的工程（主工程）的 AGP 版本一致，否则可能出现无法正常使用的问题。
由于我的主工程的 AGP 是 7.0.3，因此我们将 Lint 工程的 AGP 也同步升级到 7.0.3.
# 代码
本文工程代码: https://github.com/glorinli/CustomLintChecker
