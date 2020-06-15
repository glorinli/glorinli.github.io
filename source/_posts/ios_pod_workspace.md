---
title: 【iOS】Pod 错误: Could not automatically select an Xcode workspace
date: 2020-06-06
tags:
- iOS
- Cocoapods
categories:
- iOS
---
##问题
初尝 cocoapods，写好 Podfile 之后， pod install 一下，就出现了错误：
```
[!] Could not automatically select an Xcode workspace. Specify one in your Podfile like so:

    workspace 'path/to/Workspace.xcworkspace'

```
##解决办法
原因在于笔者看的pod使用文章太老旧了，新版需要新语法：
```
platform :ios, '10.0'
target 'XXX' do

pod 'Alamofire', '~> 5.2'

end
```

也就是这个 target 'XXX' do 的语法，别忘了最后还要一个end。
