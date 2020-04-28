# Flutter混合工程踩坑记录

#背景

前几天把flutter 从1.9.1 升级到最新的1.12.13，其中碰到了一些坑。这里进行整理下。



## 1. CachedNetworkImageProvider

升级后在编译时出现下面的错误。

```The method 'CachedNetworkImageProvider.load' has fewer positional arguments than those of overridden method 'ImageProvider.load'```

这是flutter 在1.10.15后对**ImageProvider.load**进行api更改，而使用的第三方框架[cached_network_image](https://pub.dev/packages/cached_network_image)里面依赖的是旧的api ，为了适配新的api 使用__cached_network_image 2.0.0-rc__版本。



## 2. 集成方式

一般flutter 混合App 集成方式有两种，一种是依赖源码，另外一种是依赖产物。而为了减少对原生工程的侵入，这边采用依赖产物的形式。而在Android 这边，flutter 模块可以通过```flutter build aar```命令编译成aar 包。

升级后我编译后的aar 依旧放入lib 目录下，一编译，之前的Flutter,FlutterView类全没了。

查看编译后的aar后发现，新版本没有把之前的flutter 相关类以及so库编译进去。

![](https://tva1.sinaimg.cn/large/006tNbRwgy1ga2z73gkvpj30o00b8407.jpg)

，而这里只有GeneratedPluginRegistrant类了。



### 源码依赖下flutter.gradle 添加依赖

而源码依赖下也是只有GeneratedPluginRegistrant类确能够编译通过，为了搞清编译时候他帮我们做了什么，我们来查看下它的gradle 文件。先看下若使用源码依赖时候的flutter gradle 文件。

(.android/Flutter/build.gradle)

```groovy
// Generated file. Do not edit.

def localProperties = new Properties()
def localPropertiesFile = new File(buildscript.sourceFile.parentFile.parentFile, 'local.properties')
if (localPropertiesFile.exists()) {
    localPropertiesFile.withReader('UTF-8') { reader ->
        localProperties.load(reader)
    }
}

def flutterRoot = localProperties.getProperty('flutter.sdk')
if (flutterRoot == null) {
    throw new GradleException("Flutter SDK not found. Define location with flutter.sdk in the local.properties file.")
}

def flutterVersionCode = localProperties.getProperty('flutter.versionCode')
if (flutterVersionCode == null) {
    flutterVersionCode = '1'
}

def flutterVersionName = localProperties.getProperty('flutter.versionName')
if (flutterVersionName == null) {
    flutterVersionName = '1.0'
}

apply plugin: 'com.android.library'
apply from: "$flutterRoot/packages/flutter_tools/gradle/flutter.gradle"

group '................'
version '1.0'

android {
    compileSdkVersion 28

    defaultConfig {
        minSdkVersion 16
        targetSdkVersion 28
        versionCode flutterVersionCode.toInteger()
        versionName flutterVersionName
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }
}

flutter {
    source '../..'
}

dependencies {
    testImplementation 'junit:junit:4.12'
}
```

这里依赖了sdk 中的flutter.gradle文件，这里可以看见它定义了FlutterPlugin插件，在它apply函数下

```groovy
.....    
project.android.buildTypes.each this.&addFlutterDependencies
    project.android.buildTypes.whenObjectAdded this.&addFlutterDependencies
```
进行依赖添加，

```groovy
void addFlutterDependencies(buildType) {
    String flutterBuildMode = buildModeFor(buildType)
    if (!supportsBuildMode(flutterBuildMode)) {
        return
    }
    String repository = useLocalEngine()
        ? project.property('local-engine-repo')
        : MAVEN_REPO

    project.rootProject.allprojects {
        repositories {
            maven {
                url repository
            }
        }
    }
    // Add the embedding dependency.
    addApiDependencies(project, buildType.name,
            "io.flutter:flutter_embedding_$flutterBuildMode:$engineVersion")

    List<String> platforms = getTargetPlatforms().collect()
    // Debug mode includes x86 and x64, which are commonly used in emulators.
    if (flutterBuildMode == "debug" && !useLocalEngine()) {
        platforms.add("android-x86")
        platforms.add("android-x64")
    }
    platforms.each { platform ->
        String arch = PLATFORM_ARCH_MAP[platform].replace("-", "_")
        // Add the `libflutter.so` dependency.
        addApiDependencies(project, buildType.name,
                "io.flutter:${arch}_$flutterBuildMode:$engineVersion")
    }
}
```
这里主要添加了embedding 以及libflutter.so 依赖。



### 产物依赖下添加相关依赖

产物依赖模式下是怎么处理那些依赖的呢。我们在回到flutter build aar 命令下，执行后，

![](https://tva1.sinaimg.cn/large/006tNbRwgy1ga30nmkwjdj30pe0lrwg0.jpg)

新版本会提示我们集成步骤。

和之前版本有两个主要不同点：

- 添加flutter io 远程仓库

  ```groovy
      maven {
          url 'http://download.flutter.io'
      }
  ```

- 依赖项减少

  之前版本若flutter module 依赖其他第三方包，集成时候需要加上第三方的aar 文件，而现在只需要什么依赖flutter module 模块。

  原因是，执行命令后生成一个本地maven仓库，而其他第三方依赖生命在flutter module 的pom文件中。

来看下其中debug 版本下的pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd" xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <modelVersion>4.0.0</modelVersion>
  <groupId>flutter模块名</groupId>
  <artifactId>flutter_debug</artifactId>
  <version>1.0</version>
  <packaging>aar</packaging>
  <dependencies>
    <dependency>
      <groupId>io.flutter</groupId>
      <artifactId>flutter_embedding_debug</artifactId>
      <version>1.0.0-2994f7e1e682039464cb25e31a78b86a3c59b695</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>io.flutter</groupId>
      <artifactId>armeabi_v7a_debug</artifactId>
      <version>1.0.0-2994f7e1e682039464cb25e31a78b86a3c59b695</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>io.flutter</groupId>
      <artifactId>arm64_v8a_debug</artifactId>
      <version>1.0.0-2994f7e1e682039464cb25e31a78b86a3c59b695</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>io.flutter</groupId>
      <artifactId>x86_64_debug</artifactId>
      <version>1.0.0-2994f7e1e682039464cb25e31a78b86a3c59b695</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>io.flutter</groupId>
      <artifactId>x86_debug</artifactId>
      <version>1.0.0-2994f7e1e682039464cb25e31a78b86a3c59b695</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>io.github.ponnamkarthik.toast.fluttertoast</groupId>
      <artifactId>fluttertoast_debug</artifactId>
      <version>1.0</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>io.flutter.plugins.sharedpreferences</groupId>
      <artifactId>shared_preferences_debug</artifactId>
      <version>1.0</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>io.flutter.plugins.shared_preferences_web</groupId>
      <artifactId>shared_preferences_web_debug</artifactId>
      <version>1.0</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>io.flutter.plugins.shared_preferences_macos</groupId>
      <artifactId>shared_preferences_macos_debug</artifactId>
      <version>1.0</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>com.tekartik.sqflite</groupId>
      <artifactId>sqflite_debug</artifactId>
      <version>1.0</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>io.flutter.plugins.pathprovider</groupId>
      <artifactId>path_provider_debug</artifactId>
      <version>1.0</version>
      <scope>compile</scope>
    </dependency>
  </dependencies>
</project>

```

看见其中主要有之前的embedding ，libflutter.so 以及第三方依赖项。

而实际开发中我们可以把这些文件上传到maven私有库。并把这些flutter配置项声明在一个gradle 文件中。然后在app 模块中apply即可。



## 3. Android 集成

升级后相关api 也进行了更改。

- 原本集成

```java
 FlutterView flutterView = Flutter.createView(this, this.getLifecycle(), routeName);
 setContentView(flutterView);
```

通过Flutter 工具类创建出flutterview 然后设置成Activity 的contentview。

- 现集成

  升级后你会发现Flutter 这个类不见了。现在[官方的集成步骤](https://flutter.dev/docs/development/add-to-app/android/add-flutter-screen?tab=custom-activity-launch-kotlin-tab)如下：



1. 清单文件中添加FlutterActivity

   ```xml
   <activity
     android:name="io.flutter.embedding.android.FlutterActivity"
     android:theme="@style/LaunchTheme"
     android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
     android:hardwareAccelerated="true"
     android:windowSoftInputMode="adjustResize"
     />
   ```

   

2. 启动flutterActivity

   - 正常启动

     ```kotlin
     myButton.setOnClickListener {
       startActivity(
         FlutterActivity.createDefaultIntent(this)
       )
     }
     ```

     或者指定路由,使用新的flutter engine

     ```kotlin
     myButton.setOnClickListener {
       startActivity(
         FlutterActivity
           .withNewEngine()
           .initialRoute("/my_route")
           .build(this)
       )
     }
     ```

   - 使用缓存

     - 在Application 中预先创建flutter engine

     ```kotlin
     class MyApplication : Application() {
       lateinit var flutterEngine : FlutterEngine
     
       override fun onCreate() {
         super.onCreate()
     
         // Instantiate a FlutterEngine.
         flutterEngine = FlutterEngine(this)
     
         // Start executing Dart code to pre-warm the FlutterEngine.
         flutterEngine.dartExecutor.executeDartEntrypoint(
           DartExecutor.DartEntrypoint.createDefault()
         )
     
         // Cache the FlutterEngine to be used by FlutterActivity.
         FlutterEngineCache
           .getInstance()
           .put("my_engine_id", flutterEngine)
       }
     }
     ```

     - 使用缓存

       ```kotlin
       myButton.setOnClickListener {
         startActivity(
           FlutterActivity
             .withCachedEngine("my_engine_id")
             .build(this)
         )
       }
       ```

查看flutterActivity 源码我们发现，其实里面是通过intent 来指定cache ，route 等参数。要注意的是使用了cache engine 就不能指定initRoute 。来看下flutterActivity 帮我们做了什么。



### FlutterActivity 源码

#### 1.Activity 的 onCreate 方法



```java
  @Override
  protected void onCreate(@Nullable Bundle savedInstanceState) {
    // 配置主题
    switchLaunchThemeForNormalTheme();

    super.onCreate(savedInstanceState);

    lifecycle.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);

    // 创建代理
    delegate = new FlutterActivityAndFragmentDelegate(this);
    delegate.onAttach(this);
    delegate.onActivityCreated(savedInstanceState);
		// 配置backgroundMode
    configureWindowForTransparency();
    // 设置content view
    setContentView(createFlutterView());
    // 配置状态栏，这里配置的颜色为0x40000000
    configureStatusBarForFullscreenFlutterExperience();
  }
```

其中主要任务是：

- 创建FlutterActivityAndFragmentDelegate对象，执行onAttach，onActivityCreated方法。
- 设置contentView。



#### 2. FlutterActivityAndFragmentDelegate

FlutterActivityAndFragmentDelegate主要是FlutterActivity 以及FlutterFragment 的代理类。



##### 2.1 onAttach方法



```java
void onAttach(@NonNull Context context) {
  ensureAlive();
  if (flutterEngine == null) {
    // 创建flutter engine 
    setupFlutterEngine();
  }
  //创建platformPlugin
  platformPlugin = host.providePlatformPlugin(host.getActivity(), flutterEngine);

  if (host.shouldAttachEngineToActivity()) {

    Log.d(TAG, "Attaching FlutterEngine to the Activity that owns this Fragment.");
    flutterEngine.getActivityControlSurface().attachToActivity(
        host.getActivity(),
        host.getLifecycle()
    );
  }
	//配置flutter engine
  host.configureFlutterEngine(flutterEngine);
}
```

​                                 

这里主要是创建flutter engine 以及 PlatformPlugin对象。

##### 2.2 setupFlutterEngine

```
void setupFlutterEngine() {
    Log.d(TAG, "Setting up FlutterEngine.");

    String cachedEngineId = host.getCachedEngineId();
    if (cachedEngineId != null) {
      flutterEngine = FlutterEngineCache.getInstance().get(cachedEngineId);
      isFlutterEngineFromHost = true;
      if (flutterEngine == null) {
        throw new IllegalStateException("The requested cached FlutterEngine did not exist in the FlutterEngineCache: '" + cachedEngineId + "'");
      }
      return;
    }

    flutterEngine = host.provideFlutterEngine(host.getContext());
    if (flutterEngine != null) {
      isFlutterEngineFromHost = true;
      return;
    }

    Log.d(TAG, "No preferred FlutterEngine was provided. Creating a new FlutterEngine for"
        + " this FlutterFragment.");
    flutterEngine = new FlutterEngine(host.getContext(), host.getFlutterShellArgs().toArray());
    isFlutterEngineFromHost = false;
  }
```

这里创建flutter engine 有3段逻辑：

- 如果设置了cache engine id 则从cache engine 中获取。
- 如果host （也就是Activity 或者fragment ）提供了engine 则使用。
- 直接创建FlutterEngine。

#### 3. createFlutterView()

这里主要是调用Delegate 的onCreateView方法。

```java
View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
  Log.v(TAG, "Creating FlutterView.");
  ensureAlive();
  //创建flutterView
  flutterView = new FlutterView(host.getActivity(), host.getRenderMode(), host.getTransparencyMode());

  // 设置显示渲染监听
  flutterView.addOnFirstFrameRenderedListener(flutterUiDisplayListener);

  // 创建flutterSplashView
  flutterSplashView = new FlutterSplashView(host.getContext());
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
    flutterSplashView.setId(View.generateViewId());
  } else {

    flutterSplashView.setId(486947586);
  }

//flutterSplashView 与flutterview 关联
  flutterSplashView.displayFlutterViewWithSplash(flutterView, host.provideSplashScreen());

  return flutterSplashView;
}
```



这里主要是：

- 创建flutterview
- 设置显示监听
- 创建fluttersplashView

可以看见设置到Activity 中的view 不是flutter ，而是flutterSplashView。而flutterSplashView 是一个FrameLayout帧布局。而这个主要是解决flutter 启动白屏/黑屏问题。在启动flutter到渲染过程中，可能会出现黑屏或者白屏问题。而之前的解决方案一般是先显示个loading，然后在flutter 显示后的监听回调中隐藏这个loading 。

而在新版本中可以配置个SplashScreen进行显示。查看FlutterSplashView的displayFlutterViewWithSplash方法，其实内部也是通过flutterview的FirstFrameRenderedListener监听来控制flutterview 与splashview 显示与隐藏。



### onStart方法

FlutterActivity的onstart 方法主要交给Delegate去做。我们直接看下代理的onstart 方法。

```java
void onStart() {
  Log.v(TAG, "onStart()");
  ensureAlive();

  // We post() the code that attaches the FlutterEngine to our FlutterView because there is
  // some kind of blocking logic on the native side when the surface is connected. That lag
  // causes launching Activitys to wait a second or two before launching. By post()'ing this
  // behavior we are able to move this blocking logic to after the Activity's launch.
  // TODO(mattcarroll): figure out how to avoid blocking the MAIN thread when connecting a surface
  new Handler().post(new Runnable() {
    @Override
    public void run() {
      Log.v(TAG, "Attaching FlutterEngine to FlutterView.");
      flutterView.attachToFlutterEngine(flutterEngine);

      doInitialFlutterViewRun();
    }
  });
}
```



这里主要：

- flutterview 关联flutterengine。
- 调用doInitialFlutterViewRun初始化。

而上述操作为了避免阻塞onstart 方法，放置在Handler中进行操作。



#### 4.doInitialFlutterViewRun

```java
private void doInitialFlutterViewRun() {
  // 如果使用cache engine return（所以设置route id 不起作用）
  if (host.getCachedEngineId() != null) {
    return;
  }

  if (flutterEngine.getDartExecutor().isExecutingDart()) {
    // No warning is logged because this situation will happen on every config
    // change if the developer does not choose to retain the Fragment instance.
    // So this is expected behavior in many cases.
    return;
  }

  Log.d(TAG, "Executing Dart entrypoint: " + host.getDartEntrypointFunctionName()
      + ", and sending initial route: " + host.getInitialRoute());

  // The engine needs to receive the Flutter app's initial route before executing any
  // Dart code to ensure that the initial route arrives in time to be applied.
  if (host.getInitialRoute() != null) {
//初始化route 路径    
    flutterEngine.getNavigationChannel().setInitialRoute(host.getInitialRoute());
  }

  // Configure the Dart entrypoint and execute it.
  // 执行入口函数，默认为main
  DartExecutor.DartEntrypoint entrypoint = new DartExecutor.DartEntrypoint(
      host.getAppBundlePath(),
      host.getDartEntrypointFunctionName()
  );
  flutterEngine.getDartExecutor().executeDartEntrypoint(entrypoint);
}
```



这里我们可以看见如果我们使用cache engine ，engine 预先创建后，无法指定route 路径。



## 4.实际集成。

实际开发中我们一般自己创建flutterActivity ，主要是可以添加methodchannel 进行flutter 与原生交互。主要矛盾是，若设置了cache id 无法指定route 路径。解决方法是，可以创建一个routeMethodChannel 进行交互指定跳转路径。