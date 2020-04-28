# Redux 介绍

**状态管理框架**，源自JS ，React ，[中文文档](https://www.redux.org.cn/)

### 为什么要用redux

1. 程序复杂带来的问题
   - 状态多
   - 数据源多
   - 变化难控

2. Flutter 的状态管理

   StatefulWidget

3. 其他第三方状态管理框架

   ScopedModel，BLoC，Redux，MobX

### Redux 核心概念

#### 三大原则

- 单一数据源。
- state 只可读。
- reducer 为纯函数。

#### 重要角色概念

![概念图](https://hackernoon.com/hn-images/0*cntBtPADjE2ykLSP.png)



##### action

应用（View）层发起的更改意图，只描述事情发生动作，可以为任何对象。

- type
- payload
- 尽量减少在 action 中传递的数据

##### reducer

响应action，处理数据并保存新数据到store中。

- 纯函数
- 可拆分
- (action , state) => state;

**根据它们的 key 来筛选出 state 中的一部分数据并处理**

##### store

单一的数据存储器。

- 维护应用state

- 提过getstate，获取state

- 提供dispatch ，更新state

- 添加监听器，subscribe(listener)

  

##### middleware

中间件，拦截器。

- 函数
- （stroe）=>dynamic



![xxxx](https://pic2.zhimg.com/v2-404460ceece985d433e1ed1f36cd4215_b.webp)



### FlutterRedux 中重要的类

#### StoreProvider

```dart
  const StoreProvider({
    Key key,
    @required Store<S> store,
    @required Widget child,
  })  : assert(store != null),
        assert(child != null),
        _store = store,
        super(key: key, child: child);
```



#### StoreConnector

```dart
const StoreConnector({
  Key key,
  @required this.builder,
  @required this.converter,
  this.distinct = false,
  this.onInit,
  this.onDispose,
  this.rebuildOnChange = true,
  this.ignoreChange,
  this.onWillChange,
  this.onDidChange,
  this.onInitialBuild,
})  : assert(builder != null),
      assert(converter != null),
      super(key: key);
```

- buidler

  ```dart
  typedef ViewModelBuilder<ViewModel> = Widget Function(
    BuildContext context,
    ViewModel vm,
  );
  ```

- converter

  ```dart
  typedef StoreConverter<S, ViewModel> = ViewModel Function(
    Store<S> store,
  );
  ```

# Redux 使用

## 1. 添加依赖

- [redux](https://pub.dev/packages/redux#-installing-tab-)
- [flutter_redux](https://pub.dev/packages/flutter_redux#-readme-tab-)

## 2. 创建Action

```dart
enum DemoAction { increment }	
```



## 3.  创建State

```dart
///创建一个State
@immutable
class DemoState {
  final int _count;

  get count => _count;

  DemoState(this._count);

  DemoState.initState() {
    _count = 0;
  }
}
```



## 4. 创建Reducer

```dart
///创建状态生成器
///构造一个Store 存储器时候需要传入一个生成器
///生成器的定义:
///typedef State Reducer<State>(State state, dynamic action);
///本质是一个方法
DemoState reducer(DemoState state, action){
  if(action == DemoAction.increment){
    return DemoState(state.count+1);
  }
  return state;
}
```



## 5. 创建Store

```dart
final store = Store<DemoState>(reducer, initialState: DemoState.initState());

```



## 6. 顶层StoreProvider

```dart
class MyApp extends StatelessWidget {

  @override
  Widget build(BuildContext context) {
    return StoreProvider<DemoState>(
      store: store,
      child: MaterialApp(
        title: 'Flutter Demo',
        theme: ThemeData(
          primarySwatch: Colors.blue,
        ),
        home: MyHomePage(title: 'Flutter Demo Home Page'),
      ),
    );
  }
}
```



## 7. 发起Action

```dart
        floatingActionButton: StoreConnector<DemoState, VoidCallback>(
          converter: (store) {
            return () => store.dispatch(DemoAction.increment);
          },
          builder: (context, callback) {
            return FloatingActionButton(
              onPressed: callback,
              tooltip: 'Increment',
              child: Icon(Icons.add),
            );
          },
        )
```



## 8. 响应

```dart
              StoreConnector<DemoState, int>(
                converter: (store) => store.state.count,
                builder: (context, count) {
                  return Text(
                    '$count',
                    style: Theme
                        .of(context)
                        .textTheme
                        .display1,
                  );
                },
              )
```

# 实际使用

需求：

1. 计数器改造成Redux实现。

2. 一个页面输入信息，另外一个页面获取信息。

# 原理

## StoreProvider

```dart
class StoreProvider<S> extends InheritedWidget {
  final Store<S> _store;

  const StoreProvider({
    Key key,
    @required Store<S> store,
    @required Widget child,
  })  : assert(store != null),
        assert(child != null),
        _store = store,
        super(key: key, child: child);
  ...

  @override
  bool updateShouldNotify(StoreProvider<S> oldWidget) =>
      _store != oldWidget._store;
}
```

[InheritedWidget](https://api.flutter-io.cn/flutter/widgets/InheritedWidget-class.html)

原生数据共享Widget，数据从上到下传递。

[中文文档](https://book.flutterchina.club/chapter7/inherited_widget.html)

携带共享数据

**updateShouldNotify** 确认子树是否需要重建。



### 子树获取Store

```dart
  static Store<S> of<S>(BuildContext context, {bool listen = true}) {
    final type = _typeOf<StoreProvider<S>>();
    final provider = (listen
        ? context.inheritFromWidgetOfExactType(type)
        : context
            .ancestorInheritedElementForWidgetOfExactType(type)
            ?.widget) as StoreProvider<S>;

    if (provider == null) throw StoreProviderError(type);

    return provider._store;
  }
```



- inheritFromWidgetOfExactType 子树重构
- ancestorInheritedElementForWidgetOfExactType 子树不变化

## Store.dispath

```dart
List<NextDispatcher> _dispatchers;  

dynamic dispatch(dynamic action) {
    return _dispatchers[0](action);
}

```



分发器处理动作：

```dart
class Store<State> {

  Reducer<State> reducer;

  final StreamController<State> _changeController;
  State _state;
  List<NextDispatcher> _dispatchers;
  
  Store(
    this.reducer, {
    State initialState,
    List<Middleware<State>> middleware = const [],
    bool syncStream = false,
    bool distinct = false,
  }) : _changeController = StreamController.broadcast(sync: syncStream) {
    _state = initialState;
    _dispatchers = _createDispatchers(
      middleware,
      _createReduceAndNotify(distinct),
    );
  }


  NextDispatcher _createReduceAndNotify(bool distinct) {
    return (dynamic action) {
      //reducer 处理action
      final state = reducer(_state, action);
			// 如果distance 为true 而且新的state 与旧的相同 return 处理
      if (distinct && state == _state) return;

      // 更新state
      _state = state;
      // 添加到Stream 中
      _changeController.add(state);
    };
  }

  List<NextDispatcher> _createDispatchers(
    List<Middleware<State>> middleware,
    NextDispatcher reduceAndNotify,
  ) {
    final dispatchers = <NextDispatcher>[]..add(reduceAndNotify);

    // 转换中间件Middleware为分发器
    for (var nextMiddleware in middleware.reversed) {
      final next = dispatchers.last;

      dispatchers.add(
        (dynamic action) => nextMiddleware(this, action, next),
      );
    }

    return dispatchers.reversed.toList();
  }

	...

}

```

在创建store 时候:

- 调用_createReduceAndNotify方法创建一个NextDispatcher分发器，
- 调用_createDispatchers方法创建一个分发器链。

这里主要把中间件以及reducer 包装成NextDispatcher存储，经过两次reversed后，执行顺序是先中间件list ，再reducer。

所以当执行store.dispath操作后，最终执行_createReduceAndNotify方法内生成的分发器，

- action 经过reducer 处理获取新的state。
- 判断state 是否变化（可选）。
- 更新state。
- 添加进StreamController 中通知更新。

## StoreConnector

```dart
  @override
  Widget build(BuildContext context) {
    return _StoreStreamListener<S, ViewModel>(
      store: StoreProvider.of<S>(context),
      builder: builder,
      converter: converter,
      distinct: distinct,
      onInit: onInit,
      onDispose: onDispose,
      rebuildOnChange: rebuildOnChange,
      ignoreChange: ignoreChange,
      onWillChange: onWillChange,
      onDidChange: onDidChange,
      onInitialBuild: onInitialBuild,
    );
  }
```

本质是一个_StoreStreamListener，其中Store 是StoreProvider 全局Store。

### _StoreStreamListener

```dart
class _StoreStreamListenerState<S, ViewModel>
    extends State<_StoreStreamListener<S, ViewModel>> {
  Stream<ViewModel> stream;
  ViewModel latestValue;

  @override
  void initState() {
    ///执行onInit
    if (widget.onInit != null) {
      widget.onInit(widget.store);
    }
		/// 执行 onInitialBuild
    if (widget.onInitialBuild != null) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        widget.onInitialBuild(latestValue);
      });
    }
		/// state 转换
    latestValue = widget.converter(widget.store);
    /// 初始化流
    _createStream();

    super.initState();
  }

 ...

  @override
  Widget build(BuildContext context) {
   /// rebuildOnChange 默认为true,返回一个StreamBuilder对象
    return widget.rebuildOnChange
        ? StreamBuilder<ViewModel>(
            stream: stream,
            builder: (context, snapshot) => widget.builder(
              context,
              latestValue,
            ),
          )
        : widget.builder(context, latestValue);
  }

  ...
	/// 创建Stream，在initState 和didUpdateWidget 中调用
  void _createStream() {
    stream = widget.store.onChange
        .where(_ignoreChange)
        .map(_mapConverter)
        .where(_whereDistinct)
        .transform(StreamTransformer.fromHandlers(handleData: _handleChange));
  }
	/// 更新viewModel
  void _handleChange(ViewModel vm, EventSink<ViewModel> sink) {
    if (widget.onWillChange != null) {
      widget.onWillChange(latestValue, vm);
    }

    latestValue = vm;

    if (widget.onDidChange != null) {
      ///安全执行onDidChange
      WidgetsBinding.instance.addPostFrameCallback((_) {
        widget.onDidChange(latestValue);
      });
    }

    sink.add(vm);
  }
}

```

主要流程是：

- initState 方法中，执行相关回调（onInit，onInitialBuild），创建Stream
- build 方法中，如果**rebuildOnChange** true ，返回一个StreamBuilder，否则直接执行widget 的build 方法。

而Stream 的创建是：

- 获取store 中的StreamControl
- 执行_ignoreChange 方法，如果设置了ignoreChange 回调则执行
- 执行_mapConverter 方法，执行设置的converter 方法，把顶层state 转为为相关viewmodel
- 执行_whereDistinct 方法，如果设置distinct(默认为false),比较新旧viewmodel
- 执行_handleChange方法，如果设置了onWillChange ，onDidChange 回调则执行

## 流程图

![img](https://tva1.sinaimg.cn/large/006tNbRwgy1gar8zx5eu2j31020u0n8d.jpg)

