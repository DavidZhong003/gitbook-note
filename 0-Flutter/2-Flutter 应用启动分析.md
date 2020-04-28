# Flutter 应用启动分析

dart 虚拟机启动时候调用默认的入口函数main，而通常我们调用runApp来启动我们的flutter 应用。而里面又做了什么事情呢？怎么关联我们的Widget？



# 流程

## 1. runApp

```dart
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..scheduleAttachRootWidget(app)
    ..scheduleWarmUpFrame();
}
```

## 1.1 ensureInitialized

```dart
class WidgetsFlutterBinding extends BindingBase with GestureBinding, ServicesBinding, SchedulerBinding, PaintingBinding, SemanticsBinding, RendererBinding, WidgetsBinding {
  static WidgetsBinding ensureInitialized() {
    if (WidgetsBinding.instance == null)
      WidgetsFlutterBinding();
    return WidgetsBinding.instance;
  }
}
```

这里WidgetFlutterBinding类with 了很多类。

- BindingBase 抽象基类

- GestureBinding 手势绑定类

- ServicesBinding 平台服务绑定类

- SchedulerBinding 调度帧绘制绑定类

- PaintingBinding 绘制绑定类

- SemanticsBinding 语义绑定类

- RendererBinding 渲染绑定类

- WidgetsBinding widget组件绑定类

   

scheduleAttachRootWidget 调用：

---WidgetsBinding::scheduleAttachRootWidget

​	---WidgetsBinding::attachRootWidget

​		---RenderObjectToWidgetAdapter::attachToRenderTree

​			 ---RenderObjectToWidgetAdapter::createElement

​			 ---RenderObjectToWidgetElement::mout(null,null)

​				 ---RootRenderObjectElement::mount

​					 ---RenderObjectElement::mount 

​						 ---Element::mount

​			 --- RenderObjectToWidgetElement::_rebuild

​      			---Element::updateChild

​					  ---Element::inflateWidget 

​						   ---newWidget.createElement();     MyApp	

​						   ---Element::mout 





根Element   RenderObjectToWidgetElement直接用了自身持有的根WidgetRenderObjectToWidgetAdapter持有的child来关联了我们传入的MyApp作为子Widget

> 建立Element树最重要的操作就是Element.mount。
> 每一种具体类型的Element，实现了如何将当前Element挂接(mount)到父节点上的操作；这个挂接操作除了与父Element建立指向关系外，还规定了当前Element的一些其它属性的创建时机和操作。
>
> 创建一个Element最重要的操作就是Element.updateChild。
> 更具体的是Element.inflateWidget方法；通过创建子Widget方式的不同，区分了两大类Element和Widget: (StatelessElement, StatelessWidget)和(StatefulElement, StatefulWidget)
>
> 所谓的Element树更像是前向单链表网，单链表有共同的表头。
> 父类Element不持有Element子节点，而是通过Element.visitChildren把遍历操作交给具体的Element子类型来实现。
>
> 但是RenderObject却像普通的单链表，因为通过mixin RenderObjectWithChildMixin<RenderObject>提供的child, RenderObject能够直接遍历子节点。

