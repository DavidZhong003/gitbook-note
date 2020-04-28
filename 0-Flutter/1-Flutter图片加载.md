
# Flutter 图片加载

记录分析Flutter 加载图片过程。



# 使用

在Flutter 中加载图片可Image 控件。除了直接调用构造，Image 提供了4种方式加载，分别是：

- Image.asset  从AssetBundle 中加载图片，即是在pubspec.yaml中注册的图片。

  > 在pubspec.yaml中注册的图片在打包时候会按照K-V 形式写入 asset/flutter_assets/AssetManifest.json 文件中。

- Image.file 从文件中加载。
- Image.memory 内存中加载。
- Image.network 网络中加载。

# 分析

以Image.network为例:

```dart
  Image.network(
    String src, {
    Key key,
    double scale = 1.0,
    ....
  }) : image = ResizeImage.resizeIfNeeded(cacheWidth, cacheHeight, NetworkImage(src, scale: scale, headers: headers)),
       assert(alignment != null),
       assert(repeat != null),
       assert(matchTextDirection != null),
       assert(cacheWidth == null || cacheWidth > 0),
       assert(cacheHeight == null || cacheHeight > 0),
       super(key: key);
```

这里image为ImageProvider对象。ImageProvider 是个抽象类，不同的加载方式内部也是创建不同的ImageProvider。

## Image

Image 是一个StatefulWidget。其中State 为_ImageState。

```dart
class _ImageState extends State<Image> with WidgetsBindingObserver {
  ImageStream _imageStream;
  ImageInfo _imageInfo;
  ImageChunkEvent _loadingProgress;
  bool _isListeningToStream = false;
  bool _invertColors;
  int _frameNumber;
  bool _wasSynchronouslyLoaded;

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }

  @override
  void dispose() {
    assert(_imageStream != null);
    WidgetsBinding.instance.removeObserver(this);
    _stopListeningToStream();
    super.dispose();
  }

  @override
  void didChangeDependencies() {
    _updateInvertColors();
    _resolveImage();

    if (TickerMode.of(context))
      _listenToStream();
    else
      _stopListeningToStream();

    super.didChangeDependencies();
  }
  ....
  void _resolveImage() {
    final ImageStream newStream =
      widget.image.resolve(createLocalImageConfiguration(
        context,
        size: widget.width != null && widget.height != null ? Size(widget.width, widget.height) : null,
      ));
    assert(newStream != null);
    _updateSourceStream(newStream);
  }

  @override
  Widget build(BuildContext context) {
    ...
    return result;
  }
  ....
}
```

可以看见在didChangeDependencies 方法中会调用_resolveImage方法，这里会进行图片加载操作。而**widget.image** 就是上面提到的**ImageProvider**对象。这里是先调用createLocalImageConfiguration方法创建图片的配置信息ImageConfiguration，然后调用ImageProvider的让resolve方法。

## ImageProvider.resolve

```dart
  ImageStream resolve(ImageConfiguration configuration) {
    assert(configuration != null);
    final ImageStream stream = ImageStream();
    T obtainedKey;
    bool didError = false;
    Future<void> handleError(dynamic exception, StackTrace stack) async {
      if (didError) {
        return;
      }
      didError = true;
      await null; // wait an event turn in case a listener has been added to the image stream.
      final _ErrorImageCompleter imageCompleter = _ErrorImageCompleter();
      stream.setCompleter(imageCompleter);
      imageCompleter.setError(
        exception: exception,
        stack: stack,
        context: ErrorDescription('while resolving an image'),
        silent: true, // could be a network error or whatnot
        informationCollector: () sync* {
          // 生成迭代器
          yield DiagnosticsProperty<ImageProvider>('Image provider', this);
          yield DiagnosticsProperty<ImageConfiguration>('Image configuration', configuration);
          yield DiagnosticsProperty<T>('Image key', obtainedKey, defaultValue: null);
        },
      );
    }
		// fork 出一个Zone，添加上面的错误处理
    final Zone dangerZone = Zone.current.fork(
      specification: ZoneSpecification(
        handleUncaughtError: (Zone zone, ZoneDelegate delegate, Zone parent, Object error, StackTrace stackTrace) {
          handleError(error, stackTrace);
        }
      )
    );
    dangerZone.runGuarded(() {
      Future<T> key;
      try {
        // 图片配置信息生成Key值，网络图片通过url和缩放比scale 作为key值
        key = obtainKey(configuration);
      } catch (error, stackTrace) {
        handleError(error, stackTrace);
        return;
      }
      key.then<void>((T key) {
        obtainedKey = key;
        // 加载
        final ImageStreamCompleter completer = PaintingBinding.instance.imageCache.putIfAbsent(
          key,
          () => load(key, PaintingBinding.instance.instantiateImageCodec),
          onError: handleError,
        );
        if (completer != null) {
          stream.setCompleter(completer);
        }
      }).catchError(handleError);
    });
    return stream;
  }
```

## putIfAbsent

上调用了PaintingBinding.instance.imageCache的putIfAbsent方法，imageCache 是啥？putIfAbsent 方法又做了什么事情。

### ImageCache

ImageCache 是Framework 层提供的图片缓存存储类，最多缓存1000张图片，最大缓存量为100M。LRU 缓存机制。内部有两个Map，分别存储待处理的图片以及缓存的图片。

### putIfAbsent

```dart
  ImageStreamCompleter putIfAbsent(Object key, ImageStreamCompleter loader(), { ImageErrorListener onError }) {
    assert(key != null);
    assert(loader != null);
    ImageStreamCompleter result = _pendingImages[key]?.completer;
    // 如果图片是待处理中，return
    if (result != null)
      return result;
    // 如果缓存中有，取出，放到前面，返回缓存图片
    final _CachedImage image = _cache.remove(key);
    if (image != null) {
      _cache[key] = image;
      return image.completer;
    }
    try {
      // 执行loader 函数，获取结果
      result = loader();
    } catch (error, stackTrace) {
      if (onError != null) {
        onError(error, stackTrace);
        return null;
      } else {
        // 继续传播异常
        rethrow;
      }
    }
    void listener(ImageInfo info, bool syncCall) {
      // Images that fail to load don't contribute to cache size.
      final int imageSize = info?.image == null ? 0 : info.image.height * info.image.width * 4;
      final _CachedImage image = _CachedImage(result, imageSize);
      // 调整最大size
      if (maximumSizeBytes > 0 && imageSize > maximumSizeBytes) {
        _maximumSizeBytes = imageSize + 1000;
      }
      _currentSizeBytes += imageSize;
      final _PendingImage pendingImage = _pendingImages.remove(key);
      if (pendingImage != null) {
        pendingImage.removeListener();
      }

      _cache[key] = image;
      _checkCacheSize();
    }
    // 回调监听
    if (maximumSize > 0 && maximumSizeBytes > 0) {
      final ImageStreamListener streamListener = ImageStreamListener(listener);
      _pendingImages[key] = _PendingImage(result, streamListener);
      // Listener is removed in [_PendingImage.removeListener].
      result.addListener(streamListener);
    }
    return result;
  }
```

## loader

```dart
() => load(key, PaintingBinding.instance.instantiateImageCodec)
```

load方法是抽象方法，不同ImageProvider 有不同的实现，这里以NetImage为例：

```dart
  ImageStreamCompleter load(image_provider.NetworkImage key, image_provider.DecoderCallback decode) {
    // Ownership of this controller is handed off to [_loadAsync]; it is that
    // method's responsibility to close the controller's stream when the image
    // has been loaded or an error is thrown.
    final StreamController<ImageChunkEvent> chunkEvents = StreamController<ImageChunkEvent>();

    return MultiFrameImageStreamCompleter(
      codec: _loadAsync(key, chunkEvents, decode),
      chunkEvents: chunkEvents.stream,
      scale: key.scale,
      informationCollector: () {
        return <DiagnosticsNode>[
          DiagnosticsProperty<image_provider.ImageProvider>('Image provider', this),
          DiagnosticsProperty<image_provider.NetworkImage>('Image key', key),
        ];
      },
    );
  }
```

这里返回的是一个ImageStreamCompleter的子类MultiFrameImageStreamCompleter对象，而**ImageStreamCompleter**是个抽象类，里面是一些图片加载过程的接口，可用来监听图片加载的状态。

```dart
  MultiFrameImageStreamCompleter({
    @required Future<ui.Codec> codec,
    @required double scale,
    Stream<ImageChunkEvent> chunkEvents,
    InformationCollector informationCollector,
  }) : assert(codec != null),
       _informationCollector = informationCollector,
       _scale = scale {
    codec.then<void>(_handleCodecReady, onError: (dynamic error, StackTrace stack) {
      reportError(
        context: ErrorDescription('resolving an image codec'),
        exception: error,
        stack: stack,
        informationCollector: informationCollector,
        silent: true,
      );
    });
		....
  }
```

创建MultiFrameImageStreamCompleter中会执行codec 函数，而codec 是个Future函数，即是前面的

__loadAsync_

## _loadAsync

```dart
Future<ui.Codec> _loadAsync(
  NetworkImage key,
  StreamController<ImageChunkEvent> chunkEvents,
  image_provider.DecoderCallback decode,
) async {
  try {
    assert(key == this);
		// 解析图片地址
    final Uri resolved = Uri.base.resolve(key.url);
    // 构建请求
    final HttpClientRequest request = await _httpClient.getUrl(resolved);
    // 请求头不为空话添加请求头
    headers?.forEach((String name, String value) {
      request.headers.add(name, value);
    });
    //解析获取响应
    final HttpClientResponse response = await request.close();
    // 响应失败处理
    if (response.statusCode != HttpStatus.ok)
      throw image_provider.NetworkImageLoadException(statusCode: response.statusCode, uri: resolved);
		// 响应转为字节数组
    final Uint8List bytes = await consolidateHttpClientResponseBytes(
      response,
      onBytesReceived: (int cumulative, int total) {
        chunkEvents.add(ImageChunkEvent(
          cumulativeBytesLoaded: cumulative,
          expectedTotalBytes: total,
        ));
      },
    );
    if (bytes.lengthInBytes == 0)
      throw Exception('NetworkImage is an empty file: $resolved');
		// 解码图片返回
    return decode(bytes);
  } finally {
    chunkEvents.close();
  }
}
```



## decode

上面的decode 对象是`PaintingBinding.instance.instantiateImageCodec`（在ImageProvider 的resolve方法内赋值传入）,

```dart
  Future<ui.Codec> instantiateImageCodec(Uint8List bytes, {
    int cacheWidth,
    int cacheHeight,
  }) {
    assert(cacheWidth == null || cacheWidth > 0);
    assert(cacheHeight == null || cacheHeight > 0);
    return ui.instantiateImageCodec(
      bytes,
      targetWidth: cacheWidth,
      targetHeight: cacheHeight,
    );
  }
```

这里调用painting.dart 下的instantiateImageCodec函数

```dart
Future<Codec> instantiateImageCodec(Uint8List list, {
  int targetWidth,
  int targetHeight,
}) {
  return _futurize(
    (_Callback<Codec> callback) => _instantiateImageCodec(list, callback, null, targetWidth ?? _kDoNotResizeDimension, targetHeight ?? _kDoNotResizeDimension)
  );
}
```

### _futurize

_futurize这个函数式把一个回调函数包装成Future函数。

```dart
Future<T> _futurize<T>(_Callbacker<T> callbacker) {
  final Completer<T> completer = Completer<T>.sync();
  final String error = callbacker((T t) {
    if (t == null) {
      completer.completeError(Exception('operation failed'));
    } else {
      completer.complete(t);
    }
  });
  if (error != null)
    throw Exception(error);
  return completer.future;
}
```

这里是通过构建Complete对象来把回调函数包装成future，与直接创建Future不同是，Complete可手动控制结束Future。

### _instantiateImageCodec

```dart
String _instantiateImageCodec(Uint8List list, _Callback<Codec> callback, _ImageInfo imageInfo, int targetWidth, int targetHeight)
  native 'instantiateImageCodec';
```

所以解码的任务交给native 去实现。

## _handleCodecReady

在图片下载解码后进行渲染工作。这个方法定义在**MultiFrameImageStreamCompleter**，

```dart
    codec.then<void>(_handleCodecReady, onError: ...);
```

在解码图片完成后调用_handleCodecReady。

```dart
  void _handleCodecReady(ui.Codec codec) {
    _codec = codec;
    assert(_codec != null);

    if (hasListeners) {
      _decodeNextFrameAndSchedule();
    }
  }
```

而在Image Widget 的_updateSourceStream方法中会添加监听器，所以hasListeners 为true。

## _decodeNextFrameAndSchedule

```dart
Future<void> _decodeNextFrameAndSchedule() async {
  try {
    //获取下一帧
    _nextFrame = await _codec.getNextFrame();
  } catch (exception, stack) {
    reportError(
      context: ErrorDescription('resolving an image frame'),
      exception: exception,
      stack: stack,
      informationCollector: _informationCollector,
      silent: true,
    );
    return;
  }
  // 如果图片只有一帧（非动图）
  if (_codec.frameCount == 1) {
    // 非动图处理
    _emitFrame(ImageInfo(image: _nextFrame.image, scale: _scale));
    return;
  }
  // 动图处理
  _scheduleAppFrame();
}
```

这里是获取图片下一帧，进行动图或者非动图处理

### _emitFrame

```dart
void _emitFrame(ImageInfo imageInfo) {
  setImage(imageInfo);
  _framesEmitted += 1;
}

  void setImage(ImageInfo image) {
    _currentImage = image;
    if (_listeners.isEmpty)
      return;
    // 创建监听器集合副本，防止并发修改。
    final List<ImageStreamListener> localListeners =
        List<ImageStreamListener>.from(_listeners);
    for (ImageStreamListener listener in localListeners) {
      try {
        // 调用监听器onImage方法
        listener.onImage(image, false);
      } catch (exception, stack) {
        reportError(
          context: ErrorDescription('by an image listener'),
          exception: exception,
          stack: stack,
        );
      }
    }
  }
```



这里的逻辑是取出监听器，并且调用onImage方法。而监听器是：

```dart
ImageStreamListener _getListener([ImageLoadingBuilder loadingBuilder]) {
  loadingBuilder ??= widget.loadingBuilder;
  return ImageStreamListener(
    _handleImageFrame,
    onChunk: loadingBuilder == null ? null : _handleImageChunk,
  );
}
```

而onImage方法就是_handleImageFrame

```dart
void _handleImageFrame(ImageInfo imageInfo, bool synchronousCall) {
  setState(() {
    _imageInfo = imageInfo;
    _loadingProgress = null;
    _frameNumber = _frameNumber == null ? 0 : _frameNumber + 1;
    _wasSynchronouslyLoaded |= synchronousCall;
  });
}
```

这里是调用setState 来刷新Widget界面。

### _scheduleAppFrame

```dart
  void _scheduleAppFrame() {
    if (_frameCallbackScheduled) {
      return;
    }
    _frameCallbackScheduled = true;
    //注册下一帧的回调
    SchedulerBinding.instance.scheduleFrameCallback(_handleAppFrame);
  }
```

#### scheduleFrameCallback

（binding.dart）

```dart
int scheduleFrameCallback(FrameCallback callback, { bool rescheduling = false }) {
  scheduleFrame();
  _nextFrameCallbackId += 1;
  //callback 添加到_transientCallbacks中
  _transientCallbacks[_nextFrameCallbackId] = _FrameCallbackEntry(callback, rescheduling: rescheduling);
  return _nextFrameCallbackId;
}
```

这里调用了scheduleFrame，而scheduleFrame内调用window.scheduleFrame(); 来注册Vsync 信号回调。在Engine收到Vsync 信号后会调用onBeginFrame 以及onDrawFrame 方法，而在binding里会执行handleBeginFrame方法。

```dart
  void handleBeginFrame(Duration rawTimeStamp) {
    Timeline.startSync('Frame', arguments: timelineWhitelistArguments);
    _firstRawTimeStampInEpoch ??= rawTimeStamp;
    _currentFrameTimeStamp = _adjustForEpoch(rawTimeStamp ?? _lastRawTimeStamp);
    if (rawTimeStamp != null)
      _lastRawTimeStamp = rawTimeStamp;

		...
    _hasScheduledFrame = false;
    try {
      // TRANSIENT FRAME CALLBACKS
      Timeline.startSync('Animate', arguments: timelineWhitelistArguments);
      _schedulerPhase = SchedulerPhase.transientCallbacks;
      // 获取_transientCallbacks
      final Map<int, _FrameCallbackEntry> callbacks = _transientCallbacks;
      //_transientCallbacks置空
      _transientCallbacks = <int, _FrameCallbackEntry>{};
      callbacks.forEach((int id, _FrameCallbackEntry callbackEntry) {
        if (!_removedIds.contains(id))  
          //调用callback
          _invokeFrameCallback(callbackEntry.callback, _currentFrameTimeStamp, callbackEntry.debugStack);
      });
      _removedIds.clear();
    } finally {
      _schedulerPhase = SchedulerPhase.midFrameMicrotasks;
    }
  }
```

#### _handleAppFrame

上面分析了下一帧信号来的的时候会执行_handAppFrame回调。

（image_stream.dart/MultiFrameImageStreamCompleter）

```dart
  void _handleAppFrame(Duration timestamp) {
    _frameCallbackScheduled = false;
    // 没有监听return
    if (!hasListeners)
      return;
    // 如果是第一帧，或者超过某一帧维持时间
    if (_isFirstFrame() || _hasFrameDurationPassed(timestamp)) {
      // 插入一帧
      _emitFrame(ImageInfo(image: _nextFrame.image, scale: _scale));
      // 更新当前帧显示时间
      _shownTimestamp = timestamp;
      // 更新当前帧持续时间
      _frameDuration = _nextFrame.duration;
      _nextFrame = null;
      // 判断需要重复执行
      final int completedCycles = _framesEmitted ~/ _codec.frameCount;
      if (_codec.repetitionCount == -1 || completedCycles <= _codec.repetitionCount) {
        _decodeNextFrameAndSchedule();
      }
      return;
    }
    // 计算间隔时间
    final Duration delay = _frameDuration - (timestamp - _shownTimestamp);
    // 定时刷新
    _timer = Timer(delay * timeDilation, () {
      _scheduleAppFrame();
    });
  }
```

这里对于动图，核心逻辑是对于需要重绘的帧调用_emitFrame去刷新。如果是第一帧或者超过当前帧的维持时间，调用 _emitFrame 方法显示一帧图片，否则计算下一帧的间隔时间，执行定时任务再次调用 _scheduleAppFrame方法。所以动图本质也是一帧一帧去显示。

