# Flutter引擎启动 (1.12)

# 前言

## 

本文记录分析flutter 启动再到  Dart  main 方法回调过程（基于 v1.12.13-hotfixes 版本）。



先看下Android 端如何启动一个Flutter 页面。以使用FlutterActivity 为例：

1. Android 端调用启动flutter 页面。

默认启动方式:

```kotlin
myButton.setOnClickListener {
  startActivity(
    FlutterActivity.createDefaultIntent(this)
  )
}

```

而其中FlutterActivity.createDefaultIntent方法其实是创建了一个intent 进行跳转。

```java
    public Intent build(@NonNull Context context) {
      return new Intent(context, activityClass)
          .putExtra(EXTRA_INITIAL_ROUTE, initialRoute)
          .putExtra(EXTRA_BACKGROUND_MODE, backgroundMode)
          .putExtra(EXTRA_DESTROY_ENGINE_WITH_ACTIVITY, true);
    }
```

主要携带三个信息：

- 初始路由，默认为"/"
- Backgroud_Mode，默认为opaque（不透明）
- 退出Activity时候是否销毁engine , 默认为true

# FlutterActivity.onCreate

(io.flutter.embedding.android.FlutterActivity.java)

```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
  switchLaunchThemeForNormalTheme();

  super.onCreate(savedInstanceState);
  // 关联生命周期
  lifecycle.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
  // 创建代理类
  delegate = new FlutterActivityAndFragmentDelegate(this);
  delegate.onAttach(this);
  delegate.onActivityCreated(savedInstanceState);

  configureWindowForTransparency();
  // 设置contentview
  setContentView(createFlutterView());
  configureStatusBarForFullscreenFlutterExperience();
}
```

   为了适配Activity 以及Fragment，这里很多工作其实交给代理类去实现，而Activity 的onCreate方法内主要做了两件事：

- 创建FlutterActivityAndFragmentDelegate类，调用其onAttach 以及onActivityCreated 方法

- 调用crateFlutterView方法创建并设置contentview

  

## FlutterActivityAndFragmentDelegate.onAttach

（io.flutter.embedding.android.FlutterActivityAndFragmentDelegate.java）

```java
void onAttach(@NonNull Context context) {
  ensureAlive();
  if (flutterEngine == null) {
    // 创建设置flutter 引擎
    setupFlutterEngine();
  }
  platformPlugin = host.providePlatformPlugin(host.getActivity(), flutterEngine);

  if (host.shouldAttachEngineToActivity()) {
    Log.d(TAG, "Attaching FlutterEngine to the Activity that owns this Fragment.");
    flutterEngine.getActivityControlSurface().attachToActivity(
        host.getActivity(),
        host.getLifecycle()
    );
  }
```

 setupFlutterEngine() 主要是创建FlutterEngine

```java
/* package */ void setupFlutterEngine() {
  // 如果使用了缓存engine 
  String cachedEngineId = host.getCachedEngineId();
  if (cachedEngineId != null) {
    flutterEngine = FlutterEngineCache.getInstance().get(cachedEngineId);
    isFlutterEngineFromHost = true;
    if (flutterEngine == null) {
      throw new IllegalStateException("The requested cached FlutterEngine did not exist in the FlutterEngineCache: '" + cachedEngineId + "'");
    }
    return;
  }

  // 如果host 提供了flutter engine
  flutterEngine = host.provideFlutterEngine(host.getContext());
  if (flutterEngine != null) {
    isFlutterEngineFromHost = true;
    return;
  }
  // 创建flutter engine
  Log.d(TAG, "No preferred FlutterEngine was provided. Creating a new FlutterEngine for"
      + " this FlutterFragment.");
  flutterEngine = new FlutterEngine(host.getContext(), host.getFlutterShellArgs().toArray());
  isFlutterEngineFromHost = false;
}
```

 这两个方法主要是：

- 创建FlutterEngine，遵循以下策略
  - 若设置了cache Engine 则复用
  - 若Host 提供了Engine 则使用
  - 创建全新的Flutter Engine
- 初始化PlatformPlugin

### FlutterEngine 创建

不管是使用cache Engine 还是new Engine 方式启动flutter 都涉及创建一个FlutterEngine 对象。而创建FlutterEngine 对象内部又做了什么呢？

（io.flutter.embedding.engine.FlutterEngine.java）

```java
public FlutterEngine(
    @NonNull Context context,
    @NonNull FlutterLoader flutterLoader,
    @NonNull FlutterJNI flutterJNI,
    @Nullable String[] dartVmArgs,
    boolean automaticallyRegisterPlugins
) {
  this.flutterJNI = flutterJNI;
  // 启动Flutter ，开始进行初始化
  flutterLoader.startInitialization(context);
  // 确保初始化完成
  flutterLoader.ensureInitializationComplete(context, dartVmArgs);
  
  flutterJNI.addEngineLifecycleListener(engineLifecycleListener);
  // 关联native
  attachToJni();
  // 注册platform message 通道
  this.dartExecutor = new DartExecutor(flutterJNI, context.getAssets());
  this.dartExecutor.onAttachedToJNI();

  this.renderer = new FlutterRenderer(flutterJNI);
  // 建立各种 method channel 
  .....

  this.pluginRegistry = new FlutterEnginePluginRegistry(
    context.getApplicationContext(),
    this,
    flutterLoader
  );
  // 内部反射调用类GeneratedPluginRegistrant的registerWith方法实现第三方的插件注册
  if (automaticallyRegisterPlugins) {
    registerPlugins();
  }
}
```

这里FlutterEngine 构造方法所做的事有：

- 创建FlutterJNI以及FlutterLoader 对象
- 调用flutterLoader 进行初始化
- 确保初始化完成
- attachJni 关联native
- 注册platform 各种插件
- 反射注册第三方插件

#### startInitialization 开始初始化

flutterLoader 开始进行初始化。

```java
public void startInitialization(@NonNull Context applicationContext, @NonNull Settings settings) {
  // 确保不重复初始化
    if (this.settings != null) {
      return;
    }
  // 确保在主线程
    if (Looper.myLooper() != Looper.getMainLooper()) {
      throw new IllegalStateException("startInitialization must be called on the main thread");
    }

    this.settings = settings;

    long initStartTimestampMillis = SystemClock.uptimeMillis();
    // 初始化清单文件中的配置
    initConfig(applicationContext);
  	// 清理、提取资源等操作
    initResources(applicationContext);
		// 加载flutter.so 库
    System.loadLibrary("flutter");

    VsyncWaiter
        .getInstance((WindowManager) applicationContext.getSystemService(Context.WINDOW_SERVICE))
        .init();
		
    long initTimeMillis = SystemClock.uptimeMillis() - initStartTimestampMillis;
  	// 记录初始化时间
    FlutterJNI.nativeRecordStartTimestamp(initTimeMillis);
}
```

Flutter加载器初始化里面主要做的工作是：

- 提取配置信息
- 清理、提取资源
- **加载flutter.so 库**
- 记录初始化耗费时间

##### flutter.so 库加载

在FlutterLoader.startInitialization 中我们看见加载了flutter.so 动态库，而动态库加载最终会调用到库中的JNI_OnLoad方法。

(flutter/shell/platform/android/library_loader.cc)

```c++
JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
  // Initialize the Java VM.
  fml::jni::InitJavaVM(vm);

  JNIEnv* env = fml::jni::AttachCurrentThread();
  bool result = false;

  // Register FlutterMain.
  result = flutter::FlutterMain::Register(env);
  FML_CHECK(result);

  // Register PlatformView
  result = flutter::PlatformViewAndroid::Register(env);
  FML_CHECK(result);

  // Register VSyncWaiter.
  result = flutter::VsyncWaiterAndroid::Register(env);
  FML_CHECK(result);

  return JNI_VERSION_1_4;
}
```

这里主要完成注册工作。

- 初始化JVM 虚拟机
- 注册FlutterMain
- 注册PlatformView
- 注册VSyncWaiter

###### 关联 JVM 虚拟机

（flutter/fml/platform/android/jni_util.cc）

```c++
void InitJavaVM(JavaVM* vm) {
  FML_DCHECK(g_jvm == nullptr);
  g_jvm = vm;
}
```

这里主要是关联当前进程创建的JVM 实例，并且保存成静态变量。

###### 注册FlutterMain

（flutter/shell/platform/android/flutter_main.cc）

```c++
bool FlutterMain::Register(JNIEnv* env) {
  static const JNINativeMethod methods[] = {
      {
          .name = "nativeInit",
          .signature = "(Landroid/content/Context;[Ljava/lang/String;Ljava/"
                       "lang/String;Ljava/lang/String;Ljava/lang/String;)V",
          .fnPtr = reinterpret_cast<void*>(&Init),
      },
      {
          .name = "nativeRecordStartTimestamp",
          .signature = "(J)V",
          .fnPtr = reinterpret_cast<void*>(&RecordStartTimestamp),
      },
  };

  jclass clazz = env->FindClass("io/flutter/embedding/engine/FlutterJNI");

  if (clazz == nullptr) {
    return false;
  }

  return env->RegisterNatives(clazz, methods, fml::size(methods)) == 0;
}
```

这里主要是注册了FlutterJNI(在1.9中是FlutterMain类)中的 nativeInit 以及nativeRecordStartTimestamp方法。

###### 注册PlatformView

（flutter/shell/platform/android/platform_view_android_jni.cc）

```c++
bool PlatformViewAndroid::Register(JNIEnv* env) {
  if (env == nullptr) {
    FML_LOG(ERROR) << "No JNIEnv provided";
    return false;
  }

  g_flutter_callback_info_class = new fml::jni::ScopedJavaGlobalRef<jclass>(
      env, env->FindClass("io/flutter/view/FlutterCallbackInformation"));
  if (g_flutter_callback_info_class->is_null()) {
    FML_LOG(ERROR) << "Could not locate FlutterCallbackInformation class";
    return false;
  }
  ...

  return RegisterApi(env);
}
```

这里也是进行方法注册调用。

###### 注册VSyncWaiter

(flutter/shell/platform/android/vsync_waiter_android.cc)

```c++
bool VsyncWaiterAndroid::Register(JNIEnv* env) {
  static const JNINativeMethod methods[] = {{
      .name = "nativeOnVsync",
      .signature = "(JJJ)V",
      .fnPtr = reinterpret_cast<void*>(&OnNativeVsync),
  }};

  jclass clazz = env->FindClass("io/flutter/embedding/engine/FlutterJNI");

  if (clazz == nullptr) {
    return false;
  }

  g_vsync_waiter_class = new fml::jni::ScopedJavaGlobalRef<jclass>(env, clazz);

  FML_CHECK(!g_vsync_waiter_class->is_null());

  g_async_wait_for_vsync_method_ = env->GetStaticMethodID(
      g_vsync_waiter_class->obj(), "asyncWaitForVsync", "(J)V");

  FML_CHECK(g_async_wait_for_vsync_method_ != nullptr);

  return env->RegisterNatives(clazz, methods, fml::size(methods)) == 0;
}
```

这里也是把FlutterJNI 的nativeOnVsync 方法与C++ 层的 OnNativeVsync 关联，让java 可以调用C++ 的方法、

把JAVA 层的 asyncWaitForVsync方法保存到C++ 的 g_async_wait_for_vsync_method_ 变量，让C++ 可以调用JAVA。



所以上面Flutter加载的初始化工作主要是进行资源整合，加载so 库，以及native 与 java 层的通信方法注册。

#### ensureInitializationComplete 初始化完成

上面调用只是注册了一些通信方法，实际上初始化工作还未完成。调用ensureInitializationComplete方法确保初始化完成。

```java
public void ensureInitializationComplete(@NonNull Context applicationContext, @Nullable String[] args) {
    ....
    try {
        if (resourceExtractor != null) {
            resourceExtractor.waitForCompletion();
        }

        List<String> shellArgs = new ArrayList<>();
        ...
        // 构建启动参数  
				...
        String appStoragePath = PathUtils.getFilesDir(applicationContext);
        String engineCachesPath = PathUtils.getCacheDirectory(applicationContext);
        // native 初始化
        FlutterJNI.nativeInit(applicationContext, shellArgs.toArray(new String[0]),
            kernelPath, appStoragePath, engineCachesPath);

        initialized = true;
    } catch (Exception e) {
        Log.e(TAG, "Flutter initialization failed.", e);
        throw new RuntimeException(e);
    }
}
```

其中做的事情是：

- 构建启动shell 参数列表
- 调用FlutterJNI.nativeInit 进行native 初始化工作。

##### nativeInit

###### FlutterJNI.nativeInit

```java
public static native void nativeInit(
    @NonNull Context context,
    @NonNull String[] args,
    @Nullable String bundlePath,
    @NonNull String appStoragePath,
    @NonNull String engineCachesPath
);
```

从上面的so 库加载可以看到nativeInit 方法映射到c++ 中是 FlutterMain::Init 方法

###### FlutterMain::Init

(flutter/shell/platform/android/flutter_main.cc)

```c++
void FlutterMain::Init(JNIEnv* env,
                       jclass clazz,
                       jobject context,
                       jobjectArray jargs,
                       jstring kernelPath,
                       jstring appStoragePath,
                       jstring engineCachesPath) {
  std::vector<std::string> args;
  args.push_back("flutter");
  for (auto& arg : fml::jni::StringArrayToVector(env, jargs)) {
    args.push_back(std::move(arg));
  }
  auto command_line = fml::CommandLineFromIterators(args.begin(), args.end());
  // 记录传入的参数
  auto settings = SettingsFromCommandLine(command_line);

  
  flutter::DartCallbackCache::SetCachePath(
      fml::jni::JavaStringToString(env, appStoragePath));

  fml::paths::InitializeAndroidCachesPath(
      fml::jni::JavaStringToString(env, engineCachesPath));
  // 磁盘中载入缓存
  flutter::DartCallbackCache::LoadCacheFromDisk();

  ...
  g_flutter_main.reset(new FlutterMain(std::move(settings)));

  g_flutter_main->SetupObservatoryUriCallback(env);
}
```



该方法主要是把传入的args 以及资源文件路径等信息保存在Setting 类中。



#### attachToJni 

在完成初始化后，flutter engine 还调用了attachToJni 方法 ，而内部是调用FlutterJNI的attachToNative方法：

```java
@UiThread
public void attachToNative(boolean isBackgroundView) {
  // isBackgroundView = false
  ensureRunningOnMainThread();
  ensureNotAttachedToNative();
  nativePlatformViewId = nativeAttach(this, isBackgroundView);
}
```

而nativeAttach 是个native 方法

在platform_view_android_jni.cc 中 

```c++
{
    .name = "nativeAttach",
    .signature = "(Lio/flutter/embedding/engine/FlutterJNI;Z)J",
    .fnPtr = reinterpret_cast<void*>(&AttachJNI),
}
```

关联了AttachJNI 方法

##### AttachJNI

（flutter/shell/platform/android/platform_view_android_jni.cc）

```c++
// Called By Java

static jlong AttachJNI(JNIEnv* env,
                       jclass clazz,
                       jobject flutterJNI,
                       jboolean is_background_view) {
  fml::jni::JavaObjectWeakGlobalRef java_object(env, flutterJNI);
  auto shell_holder = std::make_unique<AndroidShellHolder>(
      FlutterMain::Get().GetSettings(), java_object, is_background_view);
  if (shell_holder->IsValid()) {
    return reinterpret_cast<jlong>(shell_holder.release());
  } else {
    return 0;
  }
}
```

这里主要是初始化AndroidShellHolder对象。而这个对象内部又做了什么呢？

##### AndroidShellHolder 初始化

（flutter/shell/platform/android/android_shell_holder.cc）

```c++
AndroidShellHolder::AndroidShellHolder(
    flutter::Settings settings,
    fml::jni::JavaObjectWeakGlobalRef java_object,
    bool is_background_view)
    : settings_(std::move(settings)), java_object_(java_object) {
  static size_t shell_count = 1;
  auto thread_label = std::to_string(shell_count++);

  FML_CHECK(pthread_key_create(&thread_destruct_key_, ThreadDestructCallback) ==
            0);

  if (is_background_view) {
    ...
  } else {
    // 创建UI，GPU，IO 线程
    thread_host_ = {thread_label, ThreadHost::Type::UI | ThreadHost::Type::GPU |
                                      ThreadHost::Type::IO};
  }

  // 当ui ，gpu 线程停止时候分离jni
  auto jni_exit_task([key = thread_destruct_key_]() {
    FML_CHECK(pthread_setspecific(key, reinterpret_cast<void*>(1)) == 0);
  });
  thread_host_.ui_thread->GetTaskRunner()->PostTask(jni_exit_task);
  if (!is_background_view) {
    thread_host_.gpu_thread->GetTaskRunner()->PostTask(jni_exit_task);
  }

  fml::WeakPtr<PlatformViewAndroid> weak_platform_view;
  // 创建PlatformView    
  Shell::CreateCallback<PlatformView> on_create_platform_view =
      [is_background_view, java_object, &weak_platform_view](Shell& shell) {
        std::unique_ptr<PlatformViewAndroid> platform_view_android;
        if (is_background_view) {
         ...
        } else {
          // 创建 PlatformViewAndroid
          platform_view_android = std::make_unique<PlatformViewAndroid>(
              shell,                   // delegate
              shell.GetTaskRunners(),  // task runners
              java_object,             // java object handle for JNI interop
              shell.GetSettings()
                  .enable_software_rendering  // use software rendering
          );
        }
        weak_platform_view = platform_view_android->GetWeakPtr();
        return platform_view_android;
      };

  Shell::CreateCallback<Rasterizer> on_create_rasterizer = [](Shell& shell) {
    return std::make_unique<Rasterizer>(shell, shell.GetTaskRunners());
  };

  // 当前线程作为platform 线程，初始化MessageLoop    
  fml::MessageLoop::EnsureInitializedForCurrentThread();
  fml::RefPtr<fml::TaskRunner> gpu_runner;
  fml::RefPtr<fml::TaskRunner> ui_runner;
  fml::RefPtr<fml::TaskRunner> io_runner;
  fml::RefPtr<fml::TaskRunner> platform_runner =
      fml::MessageLoop::GetCurrent().GetTaskRunner();
  if (is_background_view) {
    ...
  } else {
    gpu_runner = thread_host_.gpu_thread->GetTaskRunner();
    ui_runner = thread_host_.ui_thread->GetTaskRunner();
    io_runner = thread_host_.io_thread->GetTaskRunner();
  }
  // 创建 task_runners     
  flutter::TaskRunners task_runners(thread_label,     // label
                                    platform_runner,  // platform
                                    gpu_runner,       // gpu
                                    ui_runner,        // ui
                                    io_runner         // io
  );
  // 创建 shell 
  shell_ =
      Shell::Create(task_runners,             // task runners
                    settings_,                // settings
                    on_create_platform_view,  // platform view create callback
                    on_create_rasterizer      // rasterizer create callback
      );

  platform_view_ = weak_platform_view;
  FML_DCHECK(platform_view_);

  is_valid_ = shell_ != nullptr;

  if (is_valid_) {
    //提升GPU 线程优先级
    task_runners.GetGPUTaskRunner()->PostTask([]() {
  
      if (::setpriority(PRIO_PROCESS, gettid(), -5) != 0) {
        if (::setpriority(PRIO_PROCESS, gettid(), -2) != 0) {
          FML_LOG(ERROR) << "Failed to set GPU task runner priority";
        }
      }
    });
    // 提升UI 线程优先级
    task_runners.GetUITaskRunner()->PostTask([]() {
      if (::setpriority(PRIO_PROCESS, gettid(), -1) != 0) {
        FML_LOG(ERROR) << "Failed to set UI task runner priority";
      }
    });
  }
}
```



这个方法很长，梳理下，主要是：

- 初始化ui、gpu、io 线程
- 创建PlatformView
- 创建Rasterizer
- **创建Shell**

###### ThreadHost创建

（flutter/shell/common/thread_host.cc）

```java
ThreadHost::ThreadHost(std::string name_prefix, uint64_t mask) {
  if (mask & ThreadHost::Type::Platform) {
    platform_thread = std::make_unique<fml::Thread>(name_prefix + ".platform");
  }

  if (mask & ThreadHost::Type::UI) {
    ui_thread = std::make_unique<fml::Thread>(name_prefix + ".ui");
  }

  if (mask & ThreadHost::Type::GPU) {
    gpu_thread = std::make_unique<fml::Thread>(name_prefix + ".gpu");
  }

  if (mask & ThreadHost::Type::IO) {
    io_thread = std::make_unique<fml::Thread>(name_prefix + ".io");
  }
}
```

在flutter 的native 层有4个线程，分别是platform，ui，gpu，io 。而在上面的AndroidShellHolder初始化中，创建了ui，gpu，ui 线程，而把当前线程作为platform线程。

###### PlatformView 创建

(flutter/shell/platform/android/platform_view_android.cc)

```c++
PlatformViewAndroid::PlatformViewAndroid(
    PlatformView::Delegate& delegate,
    flutter::TaskRunners task_runners,
    fml::jni::JavaObjectWeakGlobalRef java_object,
    bool use_software_rendering)
    : PlatformView(delegate, std::move(task_runners)),
      java_object_(java_object),
      android_surface_(AndroidSurface::Create(use_software_rendering)) {
  FML_CHECK(android_surface_)
      << "Could not create an OpenGL, Vulkan or Software surface to setup "
         "rendering.";
}
```



看一看见platformViewAndroid 继承自PlatformView。

todo

## Shell 创建

在AndroidShellHolder 初始化中，创建了shell 对象，这里看下里面做了什么。

(flutter/shell/common/shell.cc)

```c++
std::unique_ptr<Shell> Shell::Create(
    TaskRunners task_runners,
    Settings settings,
    const Shell::CreateCallback<PlatformView>& on_create_platform_view,
    const Shell::CreateCallback<Rasterizer>& on_create_rasterizer) {
  PerformInitializationTasks(settings);
  PersistentCache::SetCacheSkSL(settings.cache_sksl);

  TRACE_EVENT0("flutter", "Shell::Create");
  // 创建DartVM 
  auto vm = DartVMRef::Create(settings);
  FML_CHECK(vm) << "Must be able to initialize the VM.";

  auto vm_data = vm->GetVMData();
  // 
  return Shell::Create(std::move(task_runners),        //
                       std::move(settings),            //
                       vm_data->GetIsolateSnapshot(),  // isolate snapshot
                       on_create_platform_view,        //
                       on_create_rasterizer,           //
                       std::move(vm)                   //
  );
}
```

这里并没有直接创建Shell ，而是先创建了Dart虚拟机，然后调用Shell::Create 方法。

### Dart 虚拟机创建

(flutter/runtime/dart_vm_lifecycle.cc)

```c++
DartVMRef DartVMRef::Create(Settings settings,
                            fml::RefPtr<DartSnapshot> vm_snapshot,
                            fml::RefPtr<DartSnapshot> isolate_snapshot) {
  std::scoped_lock lifecycle_lock(gVMMutex);

  if (!settings.leak_vm) {
    FML_CHECK(!gVMLeak)
        << "Launch settings indicated that the VM should shut down in the "
           "process when done but a previous launch asked the VM to leak in "
           "the same process. For proper VM shutdown, all VM launches must "
           "indicate that they should shut down when done.";
  }

  // If there is already a running VM in the process, grab a strong reference to
  // it.
  if (auto vm = gVM.lock()) {
    FML_DLOG(WARNING) << "Attempted to create a VM in a process where one was "
                         "already running. Ignoring arguments for current VM "
                         "create call and reusing the old VM.";
    // There was already a running VM in the process,
    return DartVMRef{std::move(vm)};
  }

  std::scoped_lock dependents_lock(gVMDependentsMutex);

  gVMData.reset();
  gVMServiceProtocol.reset();
  gVMIsolateNameServer.reset();
  gVM.reset();

  // If there is no VM in the process. Initialize one, hold the weak reference
  // and pass a strong reference to the caller.
  auto isolate_name_server = std::make_shared<IsolateNameServer>();
  auto vm = DartVM::Create(std::move(settings),          //
                           std::move(vm_snapshot),       //
                           std::move(isolate_snapshot),  //
                           isolate_name_server           //
  );

  if (!vm) {
    FML_LOG(ERROR) << "Could not create Dart VM instance.";
    return {nullptr};
  }

  gVMData = vm->GetVMData();
  gVMServiceProtocol = vm->GetServiceProtocol();
  gVMIsolateNameServer = isolate_name_server;
  gVM = vm;

  if (settings.leak_vm) {
    gVMLeak = new std::shared_ptr<DartVM>(vm);
  }

  return DartVMRef{std::move(vm)};
}
```

这里是创建Dart 虚拟机，当进程存在时候复用。

Todo

### Shell::Create

（flutter/shell/common/shell.cc）

```c++
std::unique_ptr<Shell> Shell::Create(
    TaskRunners task_runners,
    Settings settings,
    fml::RefPtr<const DartSnapshot> isolate_snapshot,
    const Shell::CreateCallback<PlatformView>& on_create_platform_view,
    const Shell::CreateCallback<Rasterizer>& on_create_rasterizer,
    DartVMRef vm) {
  ....
  fml::TaskRunner::RunNowOrPostTask(
      task_runners.GetPlatformTaskRunner(),
      fml::MakeCopyable([&latch,                                          //
                         vm = std::move(vm),                              //
                         &shell,                                          //
                         task_runners = std::move(task_runners),          //
                         settings,                                        //
                         isolate_snapshot = std::move(isolate_snapshot),  //
                         on_create_platform_view,                         //
                         on_create_rasterizer                             //
  ]() mutable {
        // 真正shell 创建
        shell = CreateShellOnPlatformThread(std::move(vm),
                                            std::move(task_runners),      //
                                            settings,                     //
                                            std::move(isolate_snapshot),  //
                                            on_create_platform_view,      //
                                            on_create_rasterizer          //
        );
        
        latch.Signal();
      }));
  // 等待shell 创建完成
  latch.Wait();
  return shell;
}
```



这个重载方法里面也没有真正创建shell，实际创建交给CreateShellOnPlatformThread完成.

### CreateShellOnPlatformThread

（flutter/shell/common/shell.cc）

```c++
std::unique_ptr<Shell> Shell::CreateShellOnPlatformThread(
    DartVMRef vm,
    TaskRunners task_runners,
    Settings settings,
    fml::RefPtr<const DartSnapshot> isolate_snapshot,
    const Shell::CreateCallback<PlatformView>& on_create_platform_view,
    const Shell::CreateCallback<Rasterizer>& on_create_rasterizer) {
  ...
  // 创建shell 对象  （终于创建了！！！！）
  auto shell =
      std::unique_ptr<Shell>(new Shell(std::move(vm), task_runners, settings));

  // 在GPU 线程创建 rasterizer
  std::promise<std::unique_ptr<Rasterizer>> rasterizer_promise;
  auto rasterizer_future = rasterizer_promise.get_future();
  std::promise<fml::WeakPtr<SnapshotDelegate>> snapshot_delegate_promise;
  auto snapshot_delegate_future = snapshot_delegate_promise.get_future();
  fml::TaskRunner::RunNowOrPostTask(
      task_runners.GetGPUTaskRunner(), [&rasterizer_promise,  //
                                        &snapshot_delegate_promise,
                                        on_create_rasterizer,  //
                                        shell = shell.get()    //
  ]() {
        TRACE_EVENT0("flutter", "ShellSetupGPUSubsystem");
        std::unique_ptr<Rasterizer> rasterizer(on_create_rasterizer(*shell));
        snapshot_delegate_promise.set_value(rasterizer->GetSnapshotDelegate());
        rasterizer_promise.set_value(std::move(rasterizer));
      });

  // 在platform 线程（当前线程）创建 platformView，其实在创建AndroidShellHolder时候创建
  auto platform_view = on_create_platform_view(*shell.get());
  if (!platform_view || !platform_view->GetWeakPtr()) {
    return nullptr;
  }

  // 创建vsync_waiter.
  auto vsync_waiter = platform_view->CreateVSyncWaiter();
  if (!vsync_waiter) {
    return nullptr;
  }

  // 在IO 线程创建 ShellIOManager 对象
  std::promise<std::unique_ptr<ShellIOManager>> io_manager_promise;
  auto io_manager_future = io_manager_promise.get_future();
  std::promise<fml::WeakPtr<ShellIOManager>> weak_io_manager_promise;
  auto weak_io_manager_future = weak_io_manager_promise.get_future();
  std::promise<fml::RefPtr<SkiaUnrefQueue>> unref_queue_promise;
  auto unref_queue_future = unref_queue_promise.get_future();
  auto io_task_runner = shell->GetTaskRunners().GetIOTaskRunner();

  fml::TaskRunner::RunNowOrPostTask(
      io_task_runner,
      [&io_manager_promise,                                               //
       &weak_io_manager_promise,                                          //
       &unref_queue_promise,                                              //
       platform_view = platform_view->GetWeakPtr(),                       //
       io_task_runner,                                                    //
       is_backgrounded_sync_switch = shell->GetIsGpuDisabledSyncSwitch()  //
  ]() {
        TRACE_EVENT0("flutter", "ShellSetupIOSubsystem");
        auto io_manager = std::make_unique<ShellIOManager>(
            platform_view.getUnsafe()->CreateResourceContext(),
            is_backgrounded_sync_switch, io_task_runner);
        weak_io_manager_promise.set_value(io_manager->GetWeakPtr());
        unref_queue_promise.set_value(io_manager->GetSkiaUnrefQueue());
        io_manager_promise.set_value(std::move(io_manager));
      });
  
  auto dispatcher_maker = platform_view->GetDispatcherMaker();

  // UI 线程创建 Engine 对象
  std::promise<std::unique_ptr<Engine>> engine_promise;
  auto engine_future = engine_promise.get_future();
  fml::TaskRunner::RunNowOrPostTask(
      shell->GetTaskRunners().GetUITaskRunner(),
      fml::MakeCopyable([&engine_promise,                                 //
                         shell = shell.get(),                             //
                         &dispatcher_maker,                               //
                         isolate_snapshot = std::move(isolate_snapshot),  //
                         vsync_waiter = std::move(vsync_waiter),          //
                         &weak_io_manager_future,                         //
                         &snapshot_delegate_future,                       //
                         &unref_queue_future                              //
  ]() mutable {
        TRACE_EVENT0("flutter", "ShellSetupUISubsystem");
        const auto& task_runners = shell->GetTaskRunners();

        //  UI 线程创建 animator 对象
        auto animator = std::make_unique<Animator>(*shell, task_runners,
                                                   std::move(vsync_waiter));

        engine_promise.set_value(std::make_unique<Engine>(
            *shell,                         //
            dispatcher_maker,               //
            *shell->GetDartVM(),            //
            std::move(isolate_snapshot),    //
            task_runners,                   //
            shell->GetSettings(),           //
            std::move(animator),            //
            weak_io_manager_future.get(),   //
            unref_queue_future.get(),       //
            snapshot_delegate_future.get()  //
            ));
      }));

  if (!shell->Setup(std::move(platform_view),  //
                    engine_future.get(),       //
                    rasterizer_future.get(),   //
                    io_manager_future.get())   //
  ) {
    return nullptr;
  }

  return shell;
}
```



这一段代码较多，但里面基本上是对象的创建。

- **创建shell 对象**
- GPU 线程中创建Rasterizer
- 获取之前创建的platformView
- 创建vsync_waiter
- IO线程中创建ShellIOManager
- **UI 线程创建 Engine 对象**
- UI 线程中创建Animator

#### Shell 构建

(flutter/shell/common/shell.cc)

```c++
Shell::Shell(DartVMRef vm, TaskRunners task_runners, Settings settings)
    : task_runners_(std::move(task_runners)),
      settings_(std::move(settings)),
      vm_(std::move(vm)),
      is_gpu_disabled_sync_switch_(new fml::SyncSwitch()),
      weak_factory_(this),
      weak_factory_gpu_(nullptr) {
  FML_CHECK(vm_) << "Must have access to VM to create a shell.";
  FML_DCHECK(task_runners_.IsValid());
  FML_DCHECK(task_runners_.GetPlatformTaskRunner()->RunsTasksOnCurrentThread());

  fml::TaskRunner::RunNowOrPostTask(
      task_runners_.GetGPUTaskRunner(), fml::MakeCopyable([this]() mutable {
        this->weak_factory_gpu_ =
            std::make_unique<fml::WeakPtrFactory<Shell>>(this);
      }));

  service_protocol_handlers_[ServiceProtocol::kScreenshotExtensionName] = {
      task_runners_.GetGPUTaskRunner(),
      std::bind(&Shell::OnServiceProtocolScreenshot, this,
                std::placeholders::_1, std::placeholders::_2)};
  service_protocol_handlers_[ServiceProtocol::kScreenshotSkpExtensionName] = {
      task_runners_.GetGPUTaskRunner(),
      std::bind(&Shell::OnServiceProtocolScreenshotSKP, this,
                std::placeholders::_1, std::placeholders::_2)};
  service_protocol_handlers_[ServiceProtocol::kRunInViewExtensionName] = {
      task_runners_.GetUITaskRunner(),
      std::bind(&Shell::OnServiceProtocolRunInView, this, std::placeholders::_1,
                std::placeholders::_2)};
  service_protocol_handlers_
      [ServiceProtocol::kFlushUIThreadTasksExtensionName] = {
          task_runners_.GetUITaskRunner(),
          std::bind(&Shell::OnServiceProtocolFlushUIThreadTasks, this,
                    std::placeholders::_1, std::placeholders::_2)};
  service_protocol_handlers_
      [ServiceProtocol::kSetAssetBundlePathExtensionName] = {
          task_runners_.GetUITaskRunner(),
          std::bind(&Shell::OnServiceProtocolSetAssetBundlePath, this,
                    std::placeholders::_1, std::placeholders::_2)};
  service_protocol_handlers_
      [ServiceProtocol::kGetDisplayRefreshRateExtensionName] = {
          task_runners_.GetUITaskRunner(),
          std::bind(&Shell::OnServiceProtocolGetDisplayRefreshRate, this,
                    std::placeholders::_1, std::placeholders::_2)};
}
```



实际创建Shell 对象，关联之前创建的task_runners，settings，vm。

#### Engine 创建

（flutter/shell/common/engine.cc）

```c++
Engine::Engine(Delegate& delegate,
               const PointerDataDispatcherMaker& dispatcher_maker,
               DartVM& vm,
               fml::RefPtr<const DartSnapshot> isolate_snapshot,
               TaskRunners task_runners,
               Settings settings,
               std::unique_ptr<Animator> animator,
               fml::WeakPtr<IOManager> io_manager,
               fml::RefPtr<SkiaUnrefQueue> unref_queue,
               fml::WeakPtr<SnapshotDelegate> snapshot_delegate)
    : delegate_(delegate),
      settings_(std::move(settings)),
      animator_(std::move(animator)),
      activity_running_(true),
      have_surface_(false),
      image_decoder_(task_runners,
                     vm.GetConcurrentWorkerTaskRunner(),
                     io_manager),
      task_runners_(std::move(task_runners)),
      weak_factory_(this) {
  // 创建RuntimeController 对象
  runtime_controller_ = std::make_unique<RuntimeController>(
      *this,                        // runtime delegate
      &vm,                          // VM
      std::move(isolate_snapshot),  // isolate snapshot
      task_runners_,                // task runners
      std::move(snapshot_delegate),
      std::move(io_manager),                 // io manager
      std::move(unref_queue),                // Skia unref queue
      image_decoder_.GetWeakPtr(),           // image decoder
      settings_.advisory_script_uri,         // advisory script uri
      settings_.advisory_script_entrypoint,  // advisory script entrypoint
      settings_.idle_notification_callback,  // idle notification callback
      settings_.isolate_create_callback,     // isolate create callback
      settings_.isolate_shutdown_callback,   // isolate shutdown callback
      settings_.persistent_isolate_data      // persistent isolate data
  );

  pointer_data_dispatcher_ = dispatcher_maker(*this);
}
```

这里创建了一个RuntimeController 对象。

##### RuntimeController 创建

（flutter/runtime/runtime_controller.cc）

```c++
RuntimeController::RuntimeController(
    RuntimeDelegate& p_client,
    DartVM* p_vm,
    fml::RefPtr<const DartSnapshot> p_isolate_snapshot,
    TaskRunners p_task_runners,
    fml::WeakPtr<SnapshotDelegate> p_snapshot_delegate,
    fml::WeakPtr<IOManager> p_io_manager,
    fml::RefPtr<SkiaUnrefQueue> p_unref_queue,
    fml::WeakPtr<ImageDecoder> p_image_decoder,
    std::string p_advisory_script_uri,
    std::string p_advisory_script_entrypoint,
    const std::function<void(int64_t)>& idle_notification_callback,
    WindowData p_window_data,
    const fml::closure& p_isolate_create_callback,
    const fml::closure& p_isolate_shutdown_callback,
    std::shared_ptr<const fml::Mapping> p_persistent_isolate_data)
    : client_(p_client),
      vm_(p_vm),
      isolate_snapshot_(std::move(p_isolate_snapshot)),
      task_runners_(p_task_runners),
      snapshot_delegate_(p_snapshot_delegate),
      io_manager_(p_io_manager),
      unref_queue_(p_unref_queue),
      image_decoder_(p_image_decoder),
      advisory_script_uri_(p_advisory_script_uri),
      advisory_script_entrypoint_(p_advisory_script_entrypoint),
      idle_notification_callback_(idle_notification_callback),
      window_data_(std::move(p_window_data)),
      isolate_create_callback_(p_isolate_create_callback),
      isolate_shutdown_callback_(p_isolate_shutdown_callback),
      persistent_isolate_data_(std::move(p_persistent_isolate_data)) {
  //创建 root isolate
  auto strong_root_isolate =
      DartIsolate::CreateRootIsolate(vm_->GetVMData()->GetSettings(),  //
                                     isolate_snapshot_,                //
                                     task_runners_,                    //
                                     std::make_unique<Window>(this),   // 创建windows
                                     snapshot_delegate_,               //
                                     io_manager_,                      //
                                     unref_queue_,                     //
                                     image_decoder_,                   //
                                     p_advisory_script_uri,            //
                                     p_advisory_script_entrypoint,     //
                                     nullptr,                          //
                                     isolate_create_callback_,         //
                                     isolate_shutdown_callback_        //
                                     )
          .lock();

  FML_CHECK(strong_root_isolate) << "Could not create root isolate.";

  // The root isolate ivar is weak.
  root_isolate_ = strong_root_isolate;

  strong_root_isolate->SetReturnCodeCallback([this](uint32_t code) {
    root_isolate_return_code_ = {true, code};
  });

  if (auto* window = GetWindowIfAvailable()) {
    tonic::DartState::Scope scope(strong_root_isolate);
    window->DidCreateIsolate();
    if (!FlushRuntimeStateToIsolate()) {
      FML_DLOG(ERROR) << "Could not setup initial isolate state.";
    }
  } else {
    FML_DCHECK(false) << "RuntimeController created without window binding.";
  }

  FML_DCHECK(Dart_CurrentIsolate() == nullptr);
}
```

###### root isolate 创建

（flutter/runtime/dart_isolate.cc）

```c++
std::weak_ptr<DartIsolate> DartIsolate::CreateRootIsolate(
    const Settings& settings,
    fml::RefPtr<const DartSnapshot> isolate_snapshot,
    TaskRunners task_runners,
    std::unique_ptr<Window> window,
    fml::WeakPtr<SnapshotDelegate> snapshot_delegate,
    fml::WeakPtr<IOManager> io_manager,
    fml::RefPtr<SkiaUnrefQueue> unref_queue,
    fml::WeakPtr<ImageDecoder> image_decoder,
    std::string advisory_script_uri,
    std::string advisory_script_entrypoint,
    Dart_IsolateFlags* flags,
    const fml::closure& isolate_create_callback,
    const fml::closure& isolate_shutdown_callback) {
  TRACE_EVENT0("flutter", "DartIsolate::CreateRootIsolate");
  Dart_Isolate vm_isolate = nullptr;
  std::weak_ptr<DartIsolate> embedder_isolate;

  char* error = nullptr;

  // 创建DartIsolate 
  auto root_embedder_data = std::make_unique<std::shared_ptr<DartIsolate>>(
      std::shared_ptr<DartIsolate>(new DartIsolate(
          settings,                      // settings
          std::move(isolate_snapshot),   // isolate snapshot
          task_runners,                  // task runners
          std::move(snapshot_delegate),  // snapshot delegate
          std::move(io_manager),         // IO manager
          std::move(unref_queue),        // Skia unref queue
          std::move(image_decoder),      // Image Decoder
          advisory_script_uri,           // advisory URI
          advisory_script_entrypoint,    // advisory entrypoint
          nullptr,                       // child isolate preparer
          isolate_create_callback,       // isolate create callback
          isolate_shutdown_callback,     // isolate shutdown callback
          true,                          // is_root_isolate
          true                           // is_group_root_isolate
          )));
  // 创建 isolate
  std::tie(vm_isolate, embedder_isolate) = CreateDartVMAndEmbedderObjectPair(
      advisory_script_uri.c_str(),         // advisory script URI
      advisory_script_entrypoint.c_str(),  // advisory script entrypoint
      nullptr,                             // package root
      nullptr,                             // package config
      flags,                               // flags
      root_embedder_data.get(),            // parent embedder data
      true,                                // is root isolate
      &error                               // error (out)
  );

  if (error != nullptr) {
    free(error);
  }

  if (vm_isolate == nullptr) {
    return {};
  }

  std::shared_ptr<DartIsolate> shared_embedder_isolate =
      embedder_isolate.lock();
  if (shared_embedder_isolate) {
    // Only root isolates can interact with windows.
    shared_embedder_isolate->SetWindow(std::move(window));
  }

  root_embedder_data.release();

  return embedder_isolate;
}
```

这里创建了Isolate 。

Todo

###### windows 创建

（flutter/lib/ui/window/window.cc）

```c++
Window::Window(WindowClient* client) : client_(client) {}
```

Todo



## 创建FlutterView

### FlutterActivity.createFlutterView

在FlutterActivity 中通过调用createFlutterView 方法来设置contentView

```java
private View createFlutterView() {
  return delegate.onCreateView(
      null /* inflater */,
      null /* container */,
      null /* savedInstanceState */);
}
```

主要把创建工作交给代理类FlutterActivityAndFragmentDelegate

```java
View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
  Log.v(TAG, "Creating FlutterView.");
  ensureAlive();
  //创建flutter View
  flutterView = new FlutterView(host.getActivity(), host.getRenderMode(), host.getTransparencyMode());
  // 添加显示监听
  flutterView.addOnFirstFrameRenderedListener(flutterUiDisplayListener);
  //创建flutterSplashView
  flutterSplashView = new FlutterSplashView(host.getContext());
  // 设置id
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
    flutterSplashView.setId(View.generateViewId());
  } else {

    flutterSplashView.setId(486947586);
  }
  // 关联flutter view
  flutterSplashView.displayFlutterViewWithSplash(flutterView, host.provideSplashScreen());

  return flutterSplashView;
}
```

这里主要做了以下工作：

- 创建flutter view
- 创建flutterSplashView
- 调用FlutterSplashView 的 displayFlutterViewWithSplash 方法关联flutterView
- 返回flutterSplashView

所以设置到Activity 的view 其实是flutterSplashView。而FlutterSplashView 主要是控制FlutterView 未显示时候添加SplashView。

### FlutterView 初始化

构造内部调用init 方法。

```java
private void init() {
  Log.v(TAG, "Initializing FlutterView");

  switch (renderMode) {
    case surface:
      Log.v(TAG, "Internally using a FlutterSurfaceView.");
      FlutterSurfaceView flutterSurfaceView = new FlutterSurfaceView(getContext(), transparencyMode == TransparencyMode.transparent);
      renderSurface = flutterSurfaceView;
      addView(flutterSurfaceView);
      break;
    case texture:
      Log.v(TAG, "Internally using a FlutterTextureView.");
      FlutterTextureView flutterTextureView = new FlutterTextureView(getContext());
      renderSurface = flutterTextureView;
      addView(flutterTextureView);
      break;
  }

  // FlutterView needs to be focusable so that the InputMethodManager can interact with it.
  setFocusable(true);
  setFocusableInTouchMode(true);
}
```

根据渲染模式创建FlutterSurfaceView 或者FlutterTextureView，默认是surface 模式。

### FlutterSurfaceView 初始化

FlutterSurface 构造内也调用了init 方法：

```java
private void init() {
  // If transparency is desired then we'll enable a transparent pixel format and place
  // our Window above everything else to get transparent background rendering.
  if (renderTransparently) {
    getHolder().setFormat(PixelFormat.TRANSPARENT);
    setZOrderOnTop(true);
  }

  // 添加surface callback
  getHolder().addCallback(surfaceCallback);

  // 避免黑屏先设置透明
  setAlpha(0.0f);
}
```

而surfaceCallback 里面 调用了connectSurfaceToRenderer 方法，而最终会调用flutterJNI 的 onSurfaceCreated 方法：

```java
@UiThread
public void onSurfaceCreated(@NonNull Surface surface) {
  ensureRunningOnMainThread();
  ensureAttachedToNative();
  nativeSurfaceCreated(nativePlatformViewId, surface);
}
```

而nativeSurfaceCreated  方法注册：

(flutter/shell/platform/android/platform_view_android_jni.cc RegisterApi 方法内)

```c++
{
    .name = "nativeSurfaceCreated",
    .signature = "(JLandroid/view/Surface;)V",
    .fnPtr = reinterpret_cast<void*>(&SurfaceCreated),
},
```

 最终映射到SurfaceCreated 方法内。

```c++
static void SurfaceCreated(JNIEnv* env,
                           jobject jcaller,
                           jlong shell_holder,
                           jobject jsurface) {
  fml::jni::ScopedJavaLocalFrame scoped_local_reference_frame(env);
  // 创建AndroidNativeWindow 对象
  auto window = fml::MakeRefCounted<AndroidNativeWindow>(
      ANativeWindow_fromSurface(env, jsurface));
  ANDROID_SHELL_HOLDER->GetPlatformView()->NotifyCreated(std::move(window));
}
```

# FlutterActivity.onStart

前面可以看见在FlutterActivity 的onCreate 方法内主要进行一些初始化工作，还未加载dart 入口函数部分。

接下来看下FlutterActivity 的onStart 方法：

```java
  protected void onStart() {
    super.onStart();
    lifecycle.handleLifecycleEvent(Lifecycle.Event.ON_START);
    delegate.onStart();
  }
```

里面主要调用的是代理类的onStart方法：

```java
  void onStart() {
    Log.v(TAG, "onStart()");
    ensureAlive();
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

## attachToFlutterEngine

flutterview 与 Engine 关联

 ```java
  public void attachToFlutterEngine(
      @NonNull FlutterEngine flutterEngine
  ) {
    
    ....
    this.flutterEngine = flutterEngine;

    // Instruct our FlutterRenderer that we are now its designated RenderSurface.
    FlutterRenderer flutterRenderer = this.flutterEngine.getRenderer();
    isFlutterUiDisplayed = flutterRenderer.isDisplayingFlutterUi();
    renderSurface.attachToRenderer(flutterRenderer);
    flutterRenderer.addIsDisplayingFlutterUiListener(flutterUiDisplayListener);

    // 关联原生输入，触摸等
    .....
    
    flutterEngine.getPlatformViewsController().attachToView(this);

    // Notify engine attachment listeners of the attachment.
    for (FlutterEngineAttachmentListener listener : flutterEngineAttachmentListeners) {
      listener.onFlutterEngineAttachedToFlutterView(flutterEngine);
    }

    // 如果FlutterUi第一帧显示，通知监听器
    if (isFlutterUiDisplayed) {
      flutterUiDisplayListener.onFlutterUiDisplayed();
    }
  }
 ```

这里主要是关联一些原生的输入法输入，触摸等事件，以及处理一些监听回调。

## doInitialFlutterViewRun

(FlutterActivityAndFragmentDelegate)

```java
private void doInitialFlutterViewRun() {
  // 如果使用了缓存的Engine ，return 不再启动Engine
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

  // 如果配置了初始路由，通过natigationChannel 发送初始路由信息
  if (host.getInitialRoute() != null) {
    flutterEngine.getNavigationChannel().setInitialRoute(host.getInitialRoute());
  }

  // 配置dart 入口函数
  // flutter_assets , 
  DartExecutor.DartEntrypoint entrypoint = new DartExecutor.DartEntrypoint(
      host.getAppBundlePath(),
      host.getDartEntrypointFunctionName()
  );
  // 执行入口函数
  flutterEngine.getDartExecutor().executeDartEntrypoint(entrypoint);
}
```

这段代码的逻辑较为简单，如果使用cache Engine 则不执行启动engine，如果配置了初始路由，通过method channel 发送。这也是通过cache engine 启动 flutterActivity 时候没有提供初始路由的选项。

然后后面是构建DartEntrypoint，通过flutterengine 启动。

### DartEntrypoint构建

```java
DartExecutor.DartEntrypoint entrypoint = new DartExecutor.DartEntrypoint(
    host.getAppBundlePath(),
    host.getDartEntrypointFunctionName()
);
```

其中host.getAppBundlePath() 默认为 **flutter_assets**，而host.getDartEntrypointFunctionName() 默认为**“main”**

### executeDartEntrypoint

（DartExecutor.java）

```java
public void executeDartEntrypoint(@NonNull DartEntrypoint dartEntrypoint) {
  if (isApplicationRunning) {
    Log.w(TAG, "Attempted to run a DartExecutor that is already running.");
    return;
  }

  Log.v(TAG, "Executing Dart entrypoint: " + dartEntrypoint);

  flutterJNI.runBundleAndSnapshotFromLibrary(
      dartEntrypoint.pathToBundle,
      dartEntrypoint.dartEntrypointFunctionName,
      null,
      assetManager
  );

  isApplicationRunning = true;
}
```

### runBundleAndSnapshotFromLibrary

```java
public void runBundleAndSnapshotFromLibrary(
    @NonNull String bundlePath,
    @Nullable String entrypointFunctionName,
    @Nullable String pathToEntrypointFunction,
    @NonNull AssetManager assetManager
) {
  ensureRunningOnMainThread();
  ensureAttachedToNative();
  nativeRunBundleAndSnapshotFromLibrary(
      nativePlatformViewId,
      bundlePath,
      entrypointFunctionName,
      pathToEntrypointFunction,
      assetManager
  );
}
```

#### RunBundleAndSnapshotFromLibrary

在native中nativeRunBundleAndSnapshotFromLibrary 映射成RunBundleAndSnapshotFromLibrary方法。

(flutter/shell/platform/android/platform_view_android_jni.cc)

```c++
static void RunBundleAndSnapshotFromLibrary(JNIEnv* env,
                                            jobject jcaller,
                                            jlong shell_holder,
                                            jstring jBundlePath,
                                            jstring jEntrypoint,
                                            jstring jLibraryUrl,
                                            jobject jAssetManager) {
  //创建AssetManager对象
  auto asset_manager = std::make_shared<flutter::AssetManager>();

  asset_manager->PushBack(std::make_unique<flutter::APKAssetProvider>(
      env,                                             // jni environment
      jAssetManager,                                   // asset manager
      fml::jni::JavaStringToString(env, jBundlePath))  // apk asset dir
  );
  
  std::unique_ptr<IsolateConfiguration> isolate_configuration;
  if (flutter::DartVM::IsRunningPrecompiledCode()) {
    isolate_configuration = IsolateConfiguration::CreateForAppSnapshot();
  } else {
    std::unique_ptr<fml::Mapping> kernel_blob =
        fml::FileMapping::CreateReadOnly(
            ANDROID_SHELL_HOLDER->GetSettings().application_kernel_asset);
    if (!kernel_blob) {
      FML_DLOG(ERROR) << "Unable to load the kernel blob asset.";
      return;
    }
    isolate_configuration =
        IsolateConfiguration::CreateForKernel(std::move(kernel_blob));
  }

  RunConfiguration config(std::move(isolate_configuration),
                          std::move(asset_manager));

  {
    // 获取入口函数以及库地址
    auto entrypoint = fml::jni::JavaStringToString(env, jEntrypoint);
    auto libraryUrl = fml::jni::JavaStringToString(env, jLibraryUrl);

    if ((entrypoint.size() > 0) && (libraryUrl.size() > 0)) {
      config.SetEntrypointAndLibrary(std::move(entrypoint),
                                     std::move(libraryUrl));
    } else if (entrypoint.size() > 0) {
      config.SetEntrypoint(std::move(entrypoint));
    }
  }
  // 启动
  ANDROID_SHELL_HOLDER->Launch(std::move(config));
}
```

这里主要是获取java 传入的入口函数名称以及获取库的地址，然后调用AndroidShellHolder 的Launch 函数。

#### Launch

（flutter/shell/platform/android/android_shell_holder.cc）

```c++
void AndroidShellHolder::Launch(RunConfiguration config) {
  if (!IsValid()) {
    return;
  }

  shell_->RunEngine(std::move(config));
}
```

这里调用RunEngine 函数。

#### RunEngine

（flutter/shell/common/shell.cc）

```c++
void Shell::RunEngine(RunConfiguration run_configuration) {
  RunEngine(std::move(run_configuration), nullptr);
}
void Shell::RunEngine(
    RunConfiguration run_configuration,
    const std::function<void(Engine::RunStatus)>& result_callback) {
  auto result = [platform_runner = task_runners_.GetPlatformTaskRunner(),
                 result_callback](Engine::RunStatus run_result) {
    if (!result_callback) {
      return;
    }
    platform_runner->PostTask(
        [result_callback, run_result]() { result_callback(run_result); });
  };
  FML_DCHECK(is_setup_);
  FML_DCHECK(task_runners_.GetPlatformTaskRunner()->RunsTasksOnCurrentThread());

  fml::TaskRunner::RunNowOrPostTask(
      task_runners_.GetUITaskRunner(),
      fml::MakeCopyable(
          [run_configuration = std::move(run_configuration),
           weak_engine = weak_engine_, result]() mutable {
            if (!weak_engine) {
              FML_LOG(ERROR)
                  << "Could not launch engine with configuration - no engine.";
              result(Engine::RunStatus::Failure);
              return;
            }
            // 调用engine 的 run 方法
            auto run_result = weak_engine->Run(std::move(run_configuration));
            if (run_result == flutter::Engine::RunStatus::Failure) {
              FML_LOG(ERROR) << "Could not launch engine with configuration.";
            }
            result(run_result);
          }));
}
```

这里主要是调用Engine 的run 方法



#### Engine Run 运行

（flutter/shell/common/engine.cc）

```c++
Engine::RunStatus Engine::Run(RunConfiguration configuration) {
  ...
  // 获取入口函数以及library 路径  
  last_entry_point_ = configuration.GetEntrypoint();
  last_entry_point_library_ = configuration.GetEntrypointLibrary();
  // 运行isolate
  auto isolate_launch_status =
      PrepareAndLaunchIsolate(std::move(configuration));
  if (isolate_launch_status == Engine::RunStatus::Failure) {
    FML_LOG(ERROR) << "Engine not prepare and launch isolate.";
    return isolate_launch_status;
  } else if (isolate_launch_status ==
             Engine::RunStatus::FailureAlreadyRunning) {
    return isolate_launch_status;
  }
  .....

  return isolate_running ? Engine::RunStatus::Success
                         : Engine::RunStatus::Failure;
}
```

#### PrepareAndLaunchIsolate

（flutter/shell/common/engine.cc）

```c++
Engine::RunStatus Engine::PrepareAndLaunchIsolate(
    RunConfiguration configuration) {
  TRACE_EVENT0("flutter", "Engine::PrepareAndLaunchIsolate");
  // 更新 asset 
  UpdateAssetManager(configuration.GetAssetManager());
  // 获取isolate 配置                           
  auto isolate_configuration = configuration.TakeIsolateConfiguration();
  // 获取root isolate
  std::shared_ptr<DartIsolate> isolate =
      runtime_controller_->GetRootIsolate().lock();

  ..........  
  if (configuration.GetEntrypointLibrary().empty()) {
    if (!isolate->Run(configuration.GetEntrypoint(),
                      settings_.dart_entrypoint_args)) {
      FML_LOG(ERROR) << "Could not run the isolate.";
      return RunStatus::Failure;
    }
  } else {
    // 执行
    if (!isolate->RunFromLibrary(configuration.GetEntrypointLibrary(),
                                 configuration.GetEntrypoint(),
                                 settings_.dart_entrypoint_args)) {
      FML_LOG(ERROR) << "Could not run the isolate.";
      return RunStatus::Failure;
    }
  }

  return RunStatus::Success;
}
```

#### RunFromLibrary

（flutter/runtime/dart_isolate.cc）

```c++
bool DartIsolate::RunFromLibrary(const std::string& library_name,
                                 const std::string& entrypoint_name,
                                 const std::vector<std::string>& args,
                                 const fml::closure& on_run) {
  TRACE_EVENT0("flutter", "DartIsolate::RunFromLibrary");
  if (phase_ != Phase::Ready) {
    return false;
  }

  tonic::DartState::Scope scope(this);
  // 获取入口函数
  auto user_entrypoint_function =
      Dart_GetField(Dart_LookupLibrary(tonic::ToDart(library_name.c_str())),
                    tonic::ToDart(entrypoint_name.c_str()));
  // 获取入口函数参数
  auto entrypoint_args = tonic::ToDart(args);
  // 运行入口函数
  if (!InvokeMainEntrypoint(user_entrypoint_function, entrypoint_args)) {
    return false;
  }

  phase_ = Phase::Running;
  FML_DLOG(INFO) << "New isolate is in the running state.";

  if (on_run) {
    on_run();
  }
  return true;
}
```

#### InvokeMainEntrypoint 运行入口函数

（flutter/runtime/dart_isolate.cc）

```c++
static bool InvokeMainEntrypoint(Dart_Handle user_entrypoint_function,
                                 Dart_Handle args) {
  if (tonic::LogIfError(user_entrypoint_function)) {
    FML_LOG(ERROR) << "Could not resolve main entrypoint function.";
    return false;
  }

  Dart_Handle start_main_isolate_function =
      tonic::DartInvokeField(Dart_LookupLibrary(tonic::ToDart("dart:isolate")),
                             "_getStartMainIsolateFunction", {});

  if (tonic::LogIfError(start_main_isolate_function)) {
    FML_LOG(ERROR) << "Could not resolve main entrypoint trampoline.";
    return false;
  }

  if (tonic::LogIfError(tonic::DartInvokeField(
          Dart_LookupLibrary(tonic::ToDart("dart:ui")), "_runMainZoned",
          {start_main_isolate_function, user_entrypoint_function, args}))) {
    FML_LOG(ERROR) << "Could not invoke the main entrypoint.";
    return false;
  }

  return true;
}
```

调用Dart 层_runMainZoned方法 ，传入start_main_isolate_function ，用户入口函数user_entrypoint_function 以及相关参数。

##### _getStartMainIsolateFunction

而_getStartMainIsolateFunction定义在dart 的SDK 中。

（dart/sdk/lib/_internal/vm/lib/isolate_patch.dart）

```dart
@pragma("vm:entry-point", "call")
Function _getStartMainIsolateFunction() {
  return _startMainIsolate;
}

@pragma("vm:entry-point", "call")
void _startMainIsolate(Function entryPoint, List<String> args) {
  _startIsolate(
      null, // no parent port
      entryPoint,
      args,
      null, // no message
      true, // isSpawnUri
      null, // no control port
      null); // no capabilities
}
```

_getStartMainIsolateFunction函数其实是 _startMainIsolate。而内部是调用 _startIsolate。

#### _runMainZoned 函数

（flutter/lib/ui/hooks.dart）

```dart
@pragma('vm:entry-point')
void _runMainZoned(Function startMainIsolateFunction,
                   Function userMainFunction,
                   List<String> args) {
  startMainIsolateFunction((){
    runZoned<void>(() {
      if (userMainFunction is _BinaryFunction) {
        (userMainFunction as dynamic)(args, '');
      } else if (userMainFunction is _UnaryFunction) {
        (userMainFunction as dynamic)(args);
      } else {
        // main 方法
        userMainFunction();
      }
    }, onError: (Object error, StackTrace stackTrace) {
      _reportUnhandledException(error.toString(), stackTrace.toString());
    });
  }, null);
}
```

这里给startMainIsolateFunction 传入闭包函数以及null。而startMainIsolateFunction就是 上面的 _startMainIsolate。而内部是调用 _startIsolate 。

#### _startIsolate函数

（dart/sdk/lib/_internal/vm/lib/isolate_patch.dart）

```dart
@pragma("vm:entry-point", "call")
void _startIsolate(
    SendPort parentPort,
    Function entryPoint,
    List<String> args,
    var message,
    bool isSpawnUri,
    RawReceivePort controlPort,
    List capabilities) {
  // 控制端口不处理消息
  if (controlPort != null) {
    controlPort.handler = (_) {}; // Nobody home on the control port.
  }

  if (parentPort != null) {
    var readyMessage = new List(2);
    readyMessage[0] = controlPort.sendPort;
    readyMessage[1] = capabilities;
    capabilities = null;
    parentPort.send(readyMessage);
  }
  assert(capabilities == null);
  // 延迟所有用户代码，下次消息循环时候处理。
  RawReceivePort port = new RawReceivePort();
  port.handler = (_) {
    port.close();

    if (isSpawnUri) {
      if (entryPoint is _BinaryFunction) {
        (entryPoint as dynamic)(args, message);
      } else if (entryPoint is _UnaryFunction) {
        (entryPoint as dynamic)(args);
      } else {
        // 执行入口函数
        entryPoint();
      }
    } else {
      entryPoint(message);
    }
  };
  // 触发消息handler
  port.sendPort.send(null);
}
```

其中 entryPoint 是 _runMainZoned 中的闭包。也就是runZone 这段代码。 

而 runZone 定义是：

```dart
R runZoned<R>(R body(), {
    Map zoneValues, 
    ZoneSpecification zoneSpecification,
    Function onError,
})
```

等同于沙箱执行body()代码，其中zoneValue 是沙箱私有数据，zoneSpecification 为拦截器，onError 为错误异常回调。

最终会执行**userMainFunction**,也就是我们定义的main 方法。

```dart
void main() => runApp(MyApp());
```

# 总结

整个Flutter 引擎加载分为两个阶段，分别是准备阶段和启动阶段。准备阶段主要是指FlutterActivity的onCreate方法内，这里主要是进行资源加载以及初始化工作，在java 层进行了静态资源的整合，so库的加载，FlutterEngine等类的创建。在native层，主要进行了相关java 方法的注册关联，而最重要的是创建了AndroidShellHolder对象，而内部也创建了4大线程，以及Shell ，Engine ，DartVM 等重要对象。在准备阶段结束后，执行Activity的onStart 方法，进入启动阶段。这里java 把入口函数等信息封装成DartEntrypoint 对象。

