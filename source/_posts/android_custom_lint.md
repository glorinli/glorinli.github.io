---
title: 【Android】自定义 Lint 检查（一）
date: 2020-11-01
categories:
- Android
tags:
- Android
---
# 问题
Android lint 是一个静态代码扫描工具，sdk 本身已经支持了一些检查规则，但是有时候我们需要根据自己的业务或者代码规范自定义一些检查规则，这时候就需要用到自定义 lint 的功能，本文介绍如何自定义一个 lint 检查规则。

# 环境
MacOS Catalina + Android Studio 4.1 + Gralde 6.5

# 工程结构
首先创建一个 Android 工程，我将它命名为 CustomLintChecker，默认带有一个 app 模块，我们另外创建两个模块：
1. checker 模块，是一个 **Java Library**
2. wrapper 模块，是一个 **Android Library**

最终工程结构如下:

```
CustomLintChecker
+ app
+ wrapper
+ checker
```

# Checker 模块
## Gradle 配置
Checker 模块是 lint 的检查逻辑，首先我们在 build.gradle 中引入 lint-api 及 `com.android.lint` 插件:
``` groovy
plugins {
    id 'java-library'
    id 'kotlin'
    id 'com.android.lint'
}

...

dependencies {
    compileOnly "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"

    def lint_version = '27.1.0'
    compileOnly "com.android.tools.lint:lint-api:$lint_version"
    compileOnly "com.android.tools.lint:lint-checks:$lint_version"
}
```
值得注意的是这边的依赖全部用 compileOnly，否则可能会导致依赖冲突。
## Detector
配置完 gradle 之后，我们便着手编写我们的 lint 检查规则，此处以一个 log 检查为示例，我们的目标是检测到代码中使用了  `android.util.Log` 时，提示用户修改为 `Timber`

Detector 代码如下，相关代码作用见注释
``` kotlin
// 集成 Detector类，并实现 SourceScanner 接口
class AndroidLogDetector : Detector(), SourceCodeScanner {
    // 获取感兴趣的方法名，可以理解为让 lint 规则应用于返回的方法名列表，因为我们不可能检测每一个方法，所以此处要限定范围。
    override fun getApplicableMethodNames(): List<String>? {
        return listOf("wtf", "v", "d", "i", "w", "e")
    }

    // visitMethodCall 回调，当 lint 运行到上面的方法调用时，便会调用这个方法，我们在这里处理检测逻辑
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        // 判断方法是否是 android.util.Log 的方法
        if (context.evaluator.isMemberInClass(method, "android.util.Log")) {
            // 报告问题，第一个参数 ISSUE 是问题的数据，在后面定义，第二个参数是要报告的位置，可以理解为代码的位置，后面是一个 Message
            context.report(ISSUE, context.getLocation(node), "Don't use Android log directly")
        }
    }

    companion object {
        // 此处便是上面用到的 ISSUE，很容易理解相关参数的含义，此处不再赘述
        val ISSUE = Issue.create(
            "AndroidLogDeprecated",
            "Please don't use android log directly, use Timber instead",
            "Please don't use android log directly, use Timber instead",
            Category.LINT,
            5,
            Severity.WARNING,
            Implementation(AndroidLogDetector::class.java, Scope.JAVA_FILE_SCOPE)
        )
    }
}
```

## Registry
编写完 Detector 后，我们要对规则进行注册，这样 IDE 或者 gradle lint，才能加载我们的 Detector。此处我们需要一个 Registry 类，代码如下：
``` kotlin
package xyz.glorin.customlint.checker

import com.android.tools.lint.client.api.IssueRegistry
import xyz.glorin.customlint.checker.detectors.AndroidLogDetector

class CustomRegistry : IssueRegistry() {
    override val issues = listOf(
        AndroidLogDetector.ISSUE
    )
}
```
这个类非常简单，继承了 `IssueRegistry`，并修改了 `issues` 字段，很容易理解这是在注册这个 ISSUE。
## META-INF 配置
为了完成注册，我们还需要配置一个 IssueRegistry 文件，这样 lint 框架才能读取到我们的 Registry，这个类的路径是: `checker/src/main/resources/META-INF/services/com.android.tools.lint.client.api.IssueRegistry`，文件内容页很简单：
```
xyz.glorin.customlint.checker.CustomRegistry
```
其实就是咱们的 Registry 类的完整类名。
如此我们的 checker 模块算是完成了。
# Wrapper
checker 模块是一个 Java 模块，编译完成后，是一个 jar，传统的应用 lint 规则的方式是，把这个 jar 放在 `~/.android/lint` 目录下，但是这样十分不方便维护，且会应用到所有项目中，于是我们需要寻求其他方式。

好在 gradle 支持读取 aar 中的 lint.jar 并应用相应规则，且在新版 gradle 中新增了 `lintPublish` 方法支持更快捷地打包 lint 检测规则，我们要做的很简单，就是修改 wrapper 模块的 build.gradle 文件，内容如下：
``` groovy
...
dependencies {
    lintPublish project(':checker')
}
```
由于这只是一个简单的 wrapper 项目，因此可以考虑将 dependencies 其他内容都删除。
# App
wrapper 模块也编写完成，其实 lint 检测规则的工程已经配置完成了，我们接下来就要试试效果，在 app 模块的 build.gradle 文件中，依赖 wrapper 模块：
``` groovy
implementation project(':wrapper')
```
然后在 `MainActivity.kt` 中，写下 Log 代码，此时 lint 不一定即时生效，可以在 terminal 中，运行一下命令：
``` sh
./gradlew :app:lint
```
运行完成后，会输出 lint 检查报告，不出意外地话，IDE 也会出现相应的提示了。
# 代码
本文工程代码: https://github.com/glorinli/CustomLintChecker
