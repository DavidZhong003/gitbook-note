# Flutter 引擎启动（1.9）
---

# 前言

接触过Flutter也有大半年了，一直很好奇它是怎么在Android中启动执行的。经过一个礼拜的学习，渐渐摸清其主要分3个步骤。

1. FlutterApplication 启动。
2. FlutterActivity启动。
3. Flutter Engine 启动。

（本文分析基于Flutter 1.9 源码）

# FlutterApplication 启动

进行过Android 开发的同学都知道，App启动会经过Application 以及 Activity 的初始化。这里我们先看下FlutterApplication  做了什么事情。

##1.  FlutterApplication.onCreate

```java
    @Override
    @CallSuper
    public void onCreate() {
        super.onCreate();
        FlutterMain.startInitialization(this);
    }
```

可以看到这个类其实也没有做很多事，就是执行了FlutterMain.startInitialization方法。

## 2. FlutterMain.startInitialization

```java
public static void startInitialization(@NonNull Context applicationContext) {
    // Do nothing if we're running this in a Robolectric test.
    if (isRunningInRobolectricTest) {
        return;
    }
    startInitialization(applicationContext, new Settings());
}
```

这里创建了一个Setting对象，然后调用重载方法startInitialization。

```java
public static void startInitialization(@NonNull Context applicationContext, @NonNull Settings settings) {
    ...
		// 确保运行在主线程
    if (Looper.myLooper() != Looper.getMainLooper()) {
      throw new IllegalStateException("startInitialization must be called on the main thread");
    }
    // Do not run startInitialization more than once.
    if (sSettings != null) {
      return;
    }

    sSettings = settings;
		// 记录启动时长
    long initStartTimestampMillis = SystemClock.uptimeMillis();
    // 执行3 initConfig方法
    initConfig(applicationContext);
  	// 初始化资源
    initResources(applicationContext);
		// 加载flutter so库
    System.loadLibrary("flutter");
		...
    long initTimeMillis = SystemClock.uptimeMillis() - initStartTimestampMillis;
    //记录启动时长
    FlutterJNI.nativeRecordStartTimestamp(initTimeMillis);
}
```

可以看见这个方法其实主要做了4件事：

1. 执行initConfig 方法，进行配置初始化。
2. 初始化资源。
3. 加载flutter so库。
4. 记录启动时长。

## 3. FlutterMain.initConfig

```java
private static void initConfig(@NonNull Context applicationContext) {
    Bundle metadata = getApplicationInfo(applicationContext).metaData;

    // There isn't a `<meta-data>` tag as a direct child of `<application>` in
    // `AndroidManifest.xml`.
    if (metadata == null) {
        return;
    }
		//libapp.so
    sAotSharedLibraryName = metadata.getString(PUBLIC_AOT_SHARED_LIBRARY_NAME, DEFAULT_AOT_SHARED_LIBRARY_NAME);
  	//flutter_assets
    sFlutterAssetsDir = metadata.getString(PUBLIC_FLUTTER_ASSETS_DIR_KEY, DEFAULT_FLUTTER_ASSETS_DIR);
		//vm_snapshot_data
    sVmSnapshotData = metadata.getString(PUBLIC_VM_SNAPSHOT_DATA_KEY, DEFAULT_VM_SNAPSHOT_DATA);
  	//isolate_snapshot_data
    sIsolateSnapshotData = metadata.getString(PUBLIC_ISOLATE_SNAPSHOT_DATA_KEY, DEFAULT_ISOLATE_SNAPSHOT_DATA);
}
```

这个方法主要是读取清单文件中的一些配置。如果没有配置则赋予默认值。

| 属性名                | 默认名                |
| --------------------- | --------------------- |
| sAotSharedLibraryName | libapp.so             |
| sFlutterAssetsDir     | flutter_assets        |
| sVmSnapshotData       | vm_snapshot_data      |
| sIsolateSnapshotData  | isolate_snapshot_data |

## 4. FlutterMain.initResources

```java
private static void initResources(@NonNull Context applicationContext) {
   	// 资源清理工作
    new ResourceCleaner(applicationContext).start();

    if (BuildConfig.DEBUG || BuildConfig.JIT_RELEASE) {
        final String dataDirPath = PathUtils.getDataDirectory(applicationContext);
        final String packageName = applicationContext.getPackageName();
        final PackageManager packageManager = applicationContext.getPackageManager();
        final AssetManager assetManager = applicationContext.getResources().getAssets();
        sResourceExtractor = new ResourceExtractor(dataDirPath, packageName, packageManager, assetManager);

        // debug或者jit 模式下加载资源到内存供DartVM使用
        sResourceExtractor
            .addResource(fromFlutterAssets(sVmSnapshotData))
            .addResource(fromFlutterAssets(sIsolateSnapshotData))
            .addResource(fromFlutterAssets(DEFAULT_KERNEL_BLOB));

        sResourceExtractor.start();
    }
}
```

这个方法主要做了两件事：

1. 启动清理资源工作。
2. debug 或者jit 模式下加载相关资源。



## 5. System.loadLibrary

在上面我们提到，FlutterMain 会加载flutter.so 库工作。而最终会调用到库中的JNI_OnLoad 方法。

```c++
// This is called by the VM when the shared library is first loaded.
JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
  // Initialize the Java VM.
  fml::jni::InitJavaVM(vm);
	// 关联当前线程与jvm
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

（flutter/shell/platform/android/library_loader.cc）

这里主要做了4件事：

1. 初始化JVM。
2. 注册FlutterMain。
3. 注册PlatformView。
4. 注册VSyncWaiter。

## 5.1 InitJavaVM

```c++
void InitJavaVM(JavaVM* vm) {
  FML_DCHECK(g_jvm == nullptr);
  g_jvm = vm;
}
```

（flutter/fml/platform/android/jni_util.cc）

这里主要是把当前进程创建的JVM 保存到静态变量中。

## 5.2 FlutterMain::Register

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

（flutter/shell/platform/android/flutter_main.cc）

这里主要注册了FlutterJNI的nativeInit 以及 nativeRecordStartTimestamp 方法。

## 5.3 PlatformViewAndroid::Register

```c++
bool PlatformViewAndroid::Register(JNIEnv* env) {
  ...

  g_flutter_callback_info_class = new fml::jni::ScopedJavaGlobalRef<jclass>(
      env, env->FindClass("io/flutter/view/FlutterCallbackInformation"));
  ...

  g_flutter_callback_info_constructor = env->GetMethodID(
      g_flutter_callback_info_class->obj(), "<init>",
      "(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)V");
	...
  g_flutter_jni_class = new fml::jni::ScopedJavaGlobalRef<jclass>(
      env, env->FindClass("io/flutter/embedding/engine/FlutterJNI"));
	...

  g_surface_texture_class = new fml::jni::ScopedJavaGlobalRef<jclass>(
      env, env->FindClass("android/graphics/SurfaceTexture"));
	...

  g_attach_to_gl_context_method = env->GetMethodID(
      g_surface_texture_class->obj(), "attachToGLContext", "(I)V");

  ...

  g_update_tex_image_method =
      env->GetMethodID(g_surface_texture_class->obj(), "updateTexImage", "()V");

	...

  g_get_transform_matrix_method = env->GetMethodID(
      g_surface_texture_class->obj(), "getTransformMatrix", "([F)V");

	...
  g_detach_from_gl_context_method = env->GetMethodID(
      g_surface_texture_class->obj(), "detachFromGLContext", "()V");

	...
  return RegisterApi(env);
}
```

（flutter/shell/platform/android/platform_view_android_jni.cc）

这里以及后续调用的RegisterApi方法中都是进行java native 方法的注册。

其中主要是FlutterJNI。



## 5.4 VsyncWaiterAndroid::Register

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

这里也是注册了java 与c++ 互调的方法。

## 6. FlutterJNI.nativeRecordStartTimestamp

