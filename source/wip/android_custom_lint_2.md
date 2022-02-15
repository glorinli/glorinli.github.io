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
# 解决方法
## 升级 Gradle 和 lint library
在写这篇文章的时候，gradle 和 Android Gradle Plugin 都有了较大的版本升级，这里有一个限制需要注意：**自定义 lint 工程的 AGP，必须和使用 lint 规则的工程（主工程）的 AGP 版本一致，否则可能出现无法正常使用的问题**。
由于我的主工程的 AGP 是 7.0.3，因此我们将 Lint 工程的 AGP 也同步升级到 7.0.3.
这边有一个官方文档没有说得很清楚的点，就是 lint library 的版本，应该和 AGP 的版本是一一对应的，具体的对应规则就是：
**如果 AGP 版本是 x.y.z，那么 lint 的版本应该是 (x+23).y.z**
所以对应 AGP 7.0.3 的 lint library 版本就是 30.0.3，我们修改相应的依赖如下：
``` groovy
compileOnly "com.android.tools.lint:lint-api:$LINT_VERSION"
compileOnly "com.android.tools.lint:lint-checks:$LINT_VERSION"
```
其中 LINT_VERSION = 30.0.3
## 发布到 Maven
接下来就是发布到 maven repo 了，其实就是将 wrapper module 发布到 maven，如果你已经很熟悉 gradle maven 的发布，那下面的内容就可以不用看了。
要发布到maven，我们只需要两个步骤：
### 1. 配置 maven plugin
由于新版本的 AGP 已经集成了 maven plugin，所以我们不需要再使用第三方的插件了，直接在 wrapper module 下新建一个 maven.gradle，内容如下：
``` groovy
apply plugin: 'maven-publish'

group = GROUP
version = VERSION
def repositoryUrl = version.endsWith('SNAPSHOT') ? SNAPSHOT_REPOSITORY_URL : RELEASE_REPOSITORY_URL

afterEvaluate {
    publishing {
        publications {
            // Creates a Maven publication called "release".
            release(MavenPublication) {
                // Applies the component for the release build variant.
                from components.release

                // You can then customize attributes of the publication as shown below.
                groupId = GROUP
                artifactId = ARTIFACT_ID
                version = VERSION
            }
        }

        repositories {
            maven {
                allowInsecureProtocol true
                url = repositoryUrl
            }
        }
    }

    publish {
        doLast {
            println "Custom Lint Checker Uploaded"
        }
    }
}

```
其中有几个参数是定义在项目根目录下的 gradle.properties 里面的：
* GROUP: maven 组，例如 xyz.glorin.lint
* ARTIFACT_ID: 产物的ID，例如 customlint
* VERSION: 版本号
最终我们在主工程使用的时候，完整的依赖就是:
`$GROUP:$ARTIFACT_ID:$VERSION (xyz.glorin.lint:customlint:1.0.0)`
### 2. 打包发布
只需要执行: `./gradlew :wrapper:publish` 即可发布成功。
# 代码
本文工程代码: https://github.com/glorinli/CustomLintChecker
