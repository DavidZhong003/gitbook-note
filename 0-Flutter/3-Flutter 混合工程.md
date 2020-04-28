# Flutter 混合工程搭建

# 背景

现有的原生移动项目(Android ，IOS)中引入flutter ?

在官方预览指导[Add Flutter to existing apps](<https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps>) 中 提供了两种引入方式。

# 方案

flutter module 与flutter app  不同，通常它是以模块形式嵌入原生的app 中，通过以下命令创建：

`flutter create - t module 模块名`

而在IDE 中更可以看见flutter module 与flutter app 的区别。

![IDE 创建flutter module](https://tva1.sinaimg.cn/large/006y8mN6gy1g8n61lavlkj30m80lajsj.jpg)

flutter module 的目录结构

```
➜  flutter_module git:(master) ✗ tree -L 2 -a
.
├── .android
│   ├── Flutter
│   ├── app
│   ├── ...
├── .gitignore
├── .ios
│   ├── Config
│   ├── Flutter
│   ├── ...
│   └── Runner.xcworkspace
├── lib
│   └── main.dart
├── pubspec.lock
├── pubspec.yaml
└── test
    └── widget_test.dart

```
与直接创建flutter项目不同的是，在.android 与.ios 下有个flutter 目录。而这个目录可以生成aar 、framework 库。
## 依赖源码
是指Android 和 IOS 的项目工程中直接依赖flutter module 工程源码。

### 步骤

1. Android

- 项目下创建flutter 模块
- 修改setting.gradle文件
```gradle
// MyApp/settings.gradle
include ':app'                                     // assumed existing content
setBinding(new Binding([gradle: this]))                                 // new
evaluate(new File(                                                      // new
  settingsDir.parentFile,                                               // new
  'my_flutter/.android/include_flutter.groovy'                          // new
)) 
```

- 修改build.gradle 文件
```gradle
dependencies {
  implementation project(':flutter')
}
```

2. iOS

- 原生项目引入flutter module
- podfile 添加podhelper.rb
- Xcode的build phases 添加xcode_backend.sh 编译脚本。

### 协同开发

协同开发中原生使用git submodule 管理.
具体步骤如下：

a. 原生项目中添加子模块
`git submodule add {flutter module 仓库地址}`
`git submodule update`

b. 生成.android 和.ios 
cd 到flutter module 模块内，执行`flutter packages get` 命令

c. 运行。

## 依赖产物

这个方案是指原生工程依赖flutter module 编译后产物。在Android 中依赖aar ，而在iOS 中依赖 .framework 内容。

这种方式使原生与flutter的耦合更低。

### 步骤

其实无论是Android 还是iOS平台，使用产物依赖的方式进行flutter混合开发，一般是经过下面3个步骤。这里以Android 为例。

1. 编译产物

   在flutter模块内执行命令`flutter build aar`，生成aar 包。路径如下：

   ```tex
   build/host/outputs/repo
   └── com
       └── example
           └── my_flutter
               └── flutter_release
                   ├── 1.0
                   │   ├── flutter_release-1.0.aar
                   │   ├── flutter_release-1.0.aar.md5
                   │   ├── flutter_release-1.0.aar.sha1
                   │   ├── flutter_release-1.0.pom
                   │   ├── flutter_release-1.0.pom.md5
                   │   └── flutter_release-1.0.pom.sha1
                   ├── maven-metadata.xml
                   ├── maven-metadata.xml.md5
                   └── maven-metadata.xml.sha1
   ```

   

2. 上传远端

   可通过相关脚本上传至依赖仓库。Android 中可通过Nexus 搭建maven 私有库。flutter module 工程可通过可持续集成工具发布release 版本的__arr__包至maven 私有库，而原生这边可以添加远程私有库进行依赖。

   

3. 原生依赖

   这里简单介绍下Android 中远程依赖配置。

   1. 项目下的build.gradle 文件：

      ```groovy
      allprojects {
          repositories {
              ...
              maven {
                  url "远程仓库地址"
                  // 配置用户名和密码
                  credentials {
                      username = "用户名"
                      password = "密码"
                  }
              }
          }
      }
      ```

   2. app下的build.gradle文件

      `implementation 'group name:modulelibrary name:版本@aar'`

      

      

# 比对

下面在难易度，侵入性，可维护性，调试角度进行两个方案的比对。

1. 难易度

   依赖源码显然比依赖产物简单多了。对于依赖产物，还需要进行远程依赖仓库搭建以及上传脚本等一些工作。

2. 侵入性

   在对原生工程的侵入性考虑，依赖源码对原生的侵入性较大。而依赖产物方式相当于把flutter module 作为一个__组件__接入原生工程。

3. 维护性

   对于flutter module 来看，两个方案下，它都是作为一个独立的工程，所以维护是一样的。而对于原生工程来说。依赖产物的混合工程更好维护，而依赖源码的工程还需要小心对待git submodule。

4. 调试

   对于需要联合调试来说，依赖源码的方式更方便一点。在纯flutter 工程，直接点击热重载flutter 页面机会刷新，而在混合工程中却不行，若想实现热重载，源码依赖的混合工程也很简单。

   1. 运行原生工程。
   2. 命令行进入flutter module 工程。
   3. 输入命令：`flutter attach`

   但依赖源码方式却会导致原生工程编译时间增加。

综上之可得，对于flutter 模块与原生耦合较低的混合工程适用依赖产物方式，而若flutter 模块与原生工程存在较大耦合，则源码依赖的方式更好一些。

| 方案     | 难易 | 侵入     | 维护     | 调试     |
| -------- | ---- | -------- | -------- | -------- |
| 依赖源码 | 简单 | 侵入性大 | 维护较难 | 调试简单 |
| 依赖产物 | 难   | 侵入小   | 维护简单 | 调试难   |

# 问题

## 各个成员flutter 版本不一致

多人开发过程中无法保证所有成员的flutter SDK版本一致，若各个flutter SDK 版本不一致可能会造成dart api 兼容 以及 dart 虚拟机不一致等问题。解决办法：各个成员使用同个版本SDK，或者参考[flutterw](https://github.com/passsy/flutter_wrapper)解决方案。

## growingIO 等第三方统计工具

一些第三方埋点统计工具暂不支持flutter ，所以在选择用flutter实现的页面 需要和产品方面确认。

## 启动Flutter 白屏

在AOT 模式下情况会好一点，可仿照闲鱼添加loading 对话框。

```kotlin
loading.show()
super.onCreate(savedInstanceState)
val route ="route"
val flutterView = Flutter.createView(this, lifecycle, route)
flutterView.addFirstFrameListener { 
    loading.dismiss()
}
```

