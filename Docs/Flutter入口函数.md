# 入口函数
入口函数在flutter_center_dart/main.dart的main()函数，首先配置了一下全局异常捕获/上报，然后执行runApp：
```dart
_start(Function() runApp) async {
    FlutterError.onError = (FlutterErrorDetails details) async {
      Zone.current.handleUncaughtError(details.exception, details.stack ?? StackTrace.current);
    };

    runZonedGuarded(runApp, (error, stack) async {
      await _reportError(error, stack);
    });
  }
```
runApp的根组件是FlutterApp，它继承自StatefulWidget，

## initState
```dart
void initState() {
    _containers.add(_createContainer(PageInfo(pageName: widget.initialRoute), null, false));
    _pageManager = PageManager(this, widget.interceptNativeEvent);
    FlutterRouter.initRouter(_pageManager!);
    _iFlutterRouteApi = FlutterRouteApi(this, _pageManager as PageManager);
    _nativeRouteApi = NativeRouteApi();
    super.initState();
    if (null != widget.initStateCallback) {
        widget.initStateCallback!(_pageManager!);
    }
    WidgetsBinding.instance?.addPostFrameCallback((_) {
        refresh();
    });
}
```
1. 首先，创建了一个`Container`,`Container`是一个自定义数据结构，他是页面数据结构的抽象，包含了_navKey，key，pageInfo，firstWidget，backgroundTransparent等属性。
2. 根据`Container`可以构造`ContainerWidget`，它是一个`StateFullWidget`，向外部暴漏了`refresh`方法，实际上是调用`setState`方法刷新页面：
    ```dart
    @override
    Widget build(BuildContext context) {
        return HeroControllerScope(
        controller: HeroController(),
        child: Navigator(
            key: widget.container?._navKey,
            onGenerateRoute: (RouteSettings routeSettings) {
                return MaterialPageRoute<dynamic>(
                    settings: RouteSettings(
                        name: widget.container?.pageInfo?.pageName ??
                            routeSettings.name,
                        arguments: routeSettings.arguments),
                    builder: (context) {
                    return widget.container?.firstWidget ??
                        Container(
                            color: Colors.white,
                        );
                    });
            }),
        );
    }
    ```
    ```dart
    class _ContainerOverlayEntry extends OverlayEntry {
        _ContainerOverlayEntry(Container container)
            : containerUniqueId = container.pageInfo?.uniqueId,
                super(
                builder: (ctx) => ContainerWidget(container: container),

                ///Why the "opaque" is false and "maintainState" is true ? ?
                ///reason video link:  https://www.youtube.com/watch?v=Ya3k828Brt4
                opaque: container.backgroundTransparent,
                maintainState: true,
                );

        ///This overlay's id,which is the same as the it's related container
        final String? containerUniqueId;

        @override
        String toString() {
            return '_ContainerOverlayEntry: containerId:$containerUniqueId';
        }
    }
    ```
    由于`_createContainer`的时候传入的`firstWidget`是`null`，所以实际上，`initState`中默认添加的`ContainerWidget`是一个空白的`Container`。
3. `PageManager`是Flutter端`Router Api`的真正实现，`IRouter`定义了一套路由方法，包括 `registerPage`、`registerEvent`、`removeEvent`、`openNativePage`、`sendEventToNative`、`sendEventToFlutter`等。`FlutterRouter`是一个装饰类，调用`initRouter`将`PageManager`注入到`FlutterRouter`中。
4. `FlutterRouteApi`注册了几个接收Native方法调用的`channel`，比如生命周期的几个方法: `onForeground、onBackground、onContainerShow、onContainerHide`，以及打开返回页面：`pushFlutterPage、removeFlutterRoute`等，其中，`pushFlutterPage的实现如下`:
    ```dart
    @override
    void pushFlutterPage(RouteParam? param) {
        String? pageName = param?.pageName;
        String? uniqueId = param?.uniqueId;
        bool backgroundTransparent = param?.backgroundTransparent ?? false;
        Map<String, dynamic>? arguments = param?.arguments;
        Log.d(_TAG,
            "pushFlutterPage  pageName = $pageName  uniqueId = $uniqueId   arguments = $arguments  backgroundTransparent = $backgroundTransparent");
        if (_pageManager.registerMapPages.containsKey(pageName)) {
        Widget Function(dynamic info)? _function = _pageManager.registerMapPages[pageName];
        if (null != _function) {
            Widget _widget = _function(FlutterUtil.instance.wrapParams(params: arguments));
            _appState.push(
            pageName!,
            arguments: arguments,
            withContainer: true,
            uniqueId: uniqueId,
            first: _widget,
            backgroundTransparent: backgroundTransparent,
            );
        }
        }
    }
    ```
    flutter业务模块通过`FlutterRouter.registerFlutterPage`向外部提供页面，`FlutterRouter`将当前`route path`作为key，将这个函数作为value保存到一个`registerMapPages`中，打开flutter页面的时候，通过调用保存的function获取Widget，最后调用_appState.push打开页面。
5. `NativeRouteApi`封装了几个flutter调用原生页面的`channel`，如：`pushNativeRoute、popRoute`等方法，`pushNativeRoute`是可以接收来自Native的回调的：
    ```dart
    final Map<String, dynamic>? encoded = arg?.toMap();
    const BasicMessageChannel<dynamic> channel =
        BasicMessageChannel<dynamic>('xx.NativeRouterApi.pushNativeRoute', StandardMessageCodec());
    final dynamic replyMap = await channel.send(encoded);
    ```
6. `initStateCallback`中进行了基础库的初始化，其实调用的是main.dart中的`_initBaseLibs`，包括：
    - 网络库配置、长链接配置、日志上报库配置等；
    - flutter业务模块的注册(registerPage/registerEvent)；
    - 注册登录监听，并向flutter开放登录事件的监听。登录态的变化最终是通过Native发送到flutter的，flutter通过`_interceptNativeEvent`拦截登录Event，并分发到`_loginCallback`;
    - 上述操作做完后，向native发送flutter初始化成功的Event；
    - 加载样式文件style_config.json，便于不同的App定制自己的主题风格；
7. `addPostFrameCallback`中有一个refresh方法，当flutter关联到FlutterView后，会把所有的containers插入到下面build方法中的Overlay组件中，展示页面:
    ```dart
    ///Refresh all of overlayEntries
    void refreshAllOverlayEntries(List<Container> containers) {
        final overlayState = overlayKey.currentState;
        if (overlayState == null) {
            return;
        }
        if (_lastEntries.isNotEmpty) {
            for (var entry in _lastEntries) {
            entry.remove();
            }
        }
        _lastEntries =
            containers.map<_ContainerOverlayEntry>((container) => _ContainerOverlayEntry(container)).toList(growable: true);
        final hasScheduledFrame = SchedulerBinding.instance.hasScheduledFrame;
        final framesEnabled = SchedulerBinding.instance.framesEnabled;

        overlayState.insertAll(_lastEntries);

        // https://github.com/alibaba/flutter_boost/issues/1056
        // Ensure this frame is refreshed after schedule frame，
        // otherwise the PageState.dispose may not be called
        if (hasScheduledFrame! || !framesEnabled!) {
            SchedulerBinding.instance?.scheduleWarmUpFrame();
        }
    }
    ```
## build
```dart
MaterialApp(
    home: WillPopScope(
        onWillPop: () async {
            final canPop = topContainer.navigator?.canPop();
            if (null != canPop && canPop) {
                topContainer.navigator?.pop();
                return true;
            }
            return false;
        },
        child: Overlay(
            key: overlayKey,
            initialEntries: const <OverlayEntry>[],
        ),
    ),
    theme: ThemeData(
        primaryColor: Colors.white,
        primarySwatch: Colors.grey,
        pageTransitionsTheme: PageTransitionsTheme(
            builders: _pageTransition,
        ),
    ),
    builder: (context, child) {
        return MediaQuery(
            child: child!,
            // 文字大小不随着系统设置改变而改变
            data: MediaQuery.of(context).copyWith(textScaleFactor: 1.0),
        );
    },
    localizationsDelegates: [
        const CupertinoLocalizationsDelegate(),
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
    ],
    supportedLocales: [
        const Locale('zh', 'CN'),
        const Locale('en', 'US'),
    ],
);
```
build方法很简单，其实就是构造了一个MaterialApp:
- 配置了App的主题、页面转换动画、国际化
- 设置了字体不随系统设置而改变
- `topContainer.navigator`指向的是`_navKey`，它其实用于`ContainerWidget`中的`Navigator`组件的key:`GlobalKey<NavigatorState>`，通过这个GlobalKey可以拿到`Navigator`组件
- 页面内容是一个`Overlay`，它是一个浮动组件，目前它没有内容，当把一个Flutter页面关联到`FlutterView`后，会把所有的`ContainerOverlay`插入到当前组件。

## pushNativePage
打开原生页面，是通过`NativeRouteApi.pushNativeRoute`实现的，上面已经介绍过NativeRouteApi，它其实就是通过Channel向原生发送了消息，并且等待原生的结果。
它的原生实现部分上面也提到过，其实是在moudle-flutter-center模块中，在预热引擎时，创建了`FlutterToNativeRouteApi`，它实现了`IFlutterToNativeRouteApi`接口，接口中注册了channel:
```dart
    val channel = BasicMessageChannel<Any>(bm, "xx.NativeRouterApi.pushNativeRoute", StandardMessageCodec())
    channel.setMessageHandler(object : BasicMessageChannel.MessageHandler<Any> {
        override fun onMessage(message: Any?, reply: BasicMessageChannel.Reply<Any>) {
            val map = mutableMapOf<String, Any?>()
            try {
                val input = RouteParam().fromMap(message as Map<String, Any?>?)

                val hasFunction = input?.hasFunction
                //api 就是FlutterToNativeRouteApi对象
                if (hasFunction!!) {
                    api.pushNativeRoute(input, object : Replay<Any?> {
                        override fun replay(replay: Any?) {
                            reply.reply(replay)
                        }
                    })
                } else {
                    api.pushNativeRoute(input, null)
                    map["result"] = null
                    reply.reply(map)
                }

            } catch (exception: Exception) {
                map["error"] = wrapError(exception)
                reply.reply(map)
            }
        }
    })
```

## push
- Native调用原生端的`openFlutterPage`，最终会通过channel调用到这里
- Flutter通过Router调用`openFlutterPage`，最终的实现就是这里
```dart
 void push(
    String pageName, {
    String? uniqueId,
    Map<String, dynamic>? arguments,
    bool withContainer = true,
    Widget? first,
    bool backgroundTransparent = false,
    bool transparentRoute = false,
    bool fullscreenDialog = false,
  }) {
    final existed = _findContainerByUniqueId(uniqueId);

    if (null != existed) {
      if (topContainer.pageInfo?.uniqueId != uniqueId) {
        containers.remove(existed);
        containers.add(existed);

        refreshOnMoveToTop(existed);
      }
    } else {
      final pageInfo = PageInfo(
          pageName: pageName,
          uniqueId: uniqueId ?? _createUniqueId(pageName),
          arguments: arguments,
          withContainer: withContainer);

      if (withContainer) {
        final container = _createContainer(pageInfo, first, backgroundTransparent);

        containers.add(container);

        refreshOnPush(container);
      } else {
        if (!transparentRoute) {
          topContainer.navigator?.push(
            MaterialPageRoute(
                builder: (context) {
                  return first!;
                },
                settings: RouteSettings(name: pageName),
                fullscreenDialog: fullscreenDialog,),
          );
        } else {
          topContainer.navigator?.push(TransparentRoute(
              builder: (context) {
                return first!;
              },
              settings: RouteSettings(name: pageName),));
        }
      }
    }
  }
```

```dart
String _createUniqueId(String? pageName) {
    return '${DateTime.now().millisecondsSinceEpoch}_$pageName';
}
```

```dart
void refreshSpecificOverlayEntries(Container container, SpecificEntryRefreshMode mode) {
  //Get OverlayState from global key
  final overlayState = overlayKey.currentState;
  if (overlayState == null) {
    return;
  }

  //deal with different situation
  switch (mode) {
    case SpecificEntryRefreshMode.add:
      final entry = _ContainerOverlayEntry(container);
      _lastEntries.add(entry);
      overlayState.insert(entry);
      break;
    case SpecificEntryRefreshMode.remove:
      if (_lastEntries.isNotEmpty) {
        //Find the entry matching the container
        final entryToRemove = _lastEntries.singleWhere((element) {
          return element.containerUniqueId == container.pageInfo?.uniqueId;
        });

        //remove from the list and overlay
        _lastEntries.remove(entryToRemove);
        entryToRemove.remove();
        // https://github.com/alibaba/flutter_boost/issues/1056
        // Ensure this frame is refreshed after schedule frame,
        // otherwise the PageState.dispose may not be called
        SchedulerBinding.instance?.scheduleWarmUpFrame();
      }
      break;
    case SpecificEntryRefreshMode.moveToTop:
      final existingEntry = _lastEntries.singleWhere((element) {
        return element.containerUniqueId == container.pageInfo?.uniqueId;
      });
      //remove the entry from list and overlay
      //and insert it to list'top and overlay 's top
      _lastEntries.remove(existingEntry);
      _lastEntries.add(existingEntry);
      existingEntry.remove();
      overlayState.insert(existingEntry);
      break;
  }
}
```

`push`方法的参数如下：
- `pageName`: 页面的路由
- `uniqueId`: Continer的唯一标识，原生打开flutter页面时，这个值是原生容器的hashCode；flutter打开flutter页面时，一般这个值不会传
- `arguments`: 携带参数
- `withContainer`: 上面提到，Contianer---_ContainerOverlayEntry---ContainerWidget(包含一个Navigator)是一个对应关系；当原生打开flutter页面时，这个值为true；当flutter打开flutter页面时，这个值为false
- `first`: 其实就是当前要打开的页面widget，在需要创建container的情况下，这个值会作为栈中第一个展示的页面，如果为null，则会展示一个白色的Container
- `backgroundTransparent`: true对应_ContainerOverlayEntry的opaque为true
- `transparentRoute`: true则使用自定义`PageRoute`打开页面，false则使用`MaterialPageRoute`打开页面
- `fullscreenDialog`: 是否打开一个全屏Dialog
经过上面的分析，我们可以得知以下几点：
- flutter页面直接打开flutter页面，其实就是调用navigator的push方法，直接将当前页面加入到当前栈中。
有的时候，我们的页面可能跨模块直接打开，那么，如果有需要关闭当前模块的需求，需要使用popUntil搭配RouteSettings判断。这也是这里我们给settings设置pageName的原因，pageName其实就是路由。
还有一种情况，我们的flutter模块可能由flutter模块直接打开，也可能由原生打开，当我们有关闭模块的需求时，就需要根据不同的入口做不同的操作了，因为它可能涉及到关闭当前flutter容器的操作。
- native打开flutter页面，由于它的`uniqueId`就是`FlutterActivity`的`hashCode`，所以每一次打开新容器，都会创建一个`Container`，然后调用`refreshOnPush`，它实际上调用的是 `refreshSpecificOverlayEntries`，将当前`_ContainerOverlayEntry`放到`Overlay`顶部，此时待打开的页面`first`自然就展示到页面最上层了。

`Container`就是一个自带**栈管理**的根页面，它的集合是作为`_ContainerOverlayEntry`的集合插入到`MaterialApp`的home的`Overlay`中的。这样我们就理解了这个混合工程页面管理的逻辑，对于Flutter App来讲，不同的`FlutterActivity`容器所打开的页面，是一个个`Container`的概念，它本身自带了栈管理功能，并且使用`Overlay`展示了这些`_ContainerOverlayEntry`，Flutter页面栈和原生页面栈是不可能混在一起的，它只是从一个`FlutterView`关联到另一个`FlutterView`。

## pop
```dart
Future<void> pop({String? uniqueId, Map<String, dynamic>? arguments, bool forceFinish = false}) async {
    Container? container;
    if (null != uniqueId) {
      container = _findContainerByUniqueId(uniqueId);
      if (container == null) {
        return;
      }
      if (container != topContainer) {
        _removeContainer(container);
        return;
      }
    } else {
      container = topContainer;
    }

    final handled = await container.navigator?.maybePop();
    Log.d(TAG, "pop page  handled = $handled    forceFinish = $forceFinish");

    if (forceFinish) {
      final params = RouteParam()
        ..pageName = container.pageInfo?.pageName
        ..uniqueId = container.pageInfo?.uniqueId
        ..arguments = container.pageInfo?.arguments;
      _finishContainer(params);
    } else {
      if (handled != null && !handled) {
        final params = RouteParam()
          ..pageName = container.pageInfo?.pageName
          ..uniqueId = container.pageInfo?.uniqueId
          ..arguments = container.pageInfo?.arguments;
        _finishContainer(params);
      }
    }
}
```

```dart
void _removeContainer(Container page) {
    containers.remove(page);
    if (page.pageInfo!.withContainer) {
      final params = RouteParam()
        ..pageName = page.pageInfo!.pageName
        ..uniqueId = page.pageInfo!.uniqueId
        ..arguments = page.pageInfo!.arguments;
      _finishContainer(params);
    }
}
```

```dart
void _finishContainer(RouteParam param) {
    _nativeRouteApi?.popRoute(param);
}
```

参数如下：
- `uniqueId`： Container的唯一id，一般来说它为null
- `arguments`: 额外参数
- `forceFinish`: 是否关闭当前flutter容器
如果传递了`uniqueId`，并且`Container`不在顶部，则直接把该`Container`移除掉，如果该`Container`有对应的原生容器，会把该原生容器`finish`掉。
一般来讲，container对应的是`topContainer`，对它的navigator调用maybePop方法，如果`Container`的栈内还有，flutter页面，则`handled`会返回true，反之，则返回false，说明已经没有flutter页面，这个时候需要关闭flutter容器，即调用`_finishContainer`。如果`forceFinish`的取值是`true`，则会直接把原生容器`finish`掉。

我们还要探究一个问题，就是当一个`Container`关闭后，再次从原生页面回到Flutter页面，当前的FlutterApp是如何关联到`FlutterView`并正确展示的？
这要看一下原生对声明周期的处理了：
```dart
 override fun onResume() {
        if (flutterView == null) {
            findFlutterView(window.decorView)
        }
        super.onResume()
        //Build.VERSION_CODES.Q
        if (Build.VERSION.SDK_INT == 29) {
            if (!ActivityStackManager.getInstance().isAppFront && !ContainerManager.instance.isTopContainer(containerUniqueId())) {
                return
            }
        }
        observer?.onAppear(this)
        ActivityAndFragmentHelper.onResumeAttachToFlutterEngine(flutterView, flutterEngine!!)
        // 页面行为打点，统一onResumed()执行时上报，方便排查系统按页面对日志分组
        onAction("activity", BehaviorType.Page, "展示Flutter页面", null, BehaviorResult.None, pageName())
}
```

```dart
 override fun onAppear(container: IFlutterContainer) {
        Log.d(TAG, "onAppear")
        ContainerManager.instance.pushContainer(container.containerUniqueId(), container)
        val p = RouteParam()
        p.uniqueId = container.containerUniqueId()
        p.pageName = container.pageName()
        p.arguments = container.pageArguments()
        p.backgroundTransparent = container.backgroundTransparent()

        FlutterCenter.instance.native2FlutterApi?.pushFlutterPage(p, null)

        FlutterCenter.instance.native2FlutterApi?.onContainerShow(p, null)
}
```
可以看到，这里主要有两个关键逻辑：
- 调用flutter的`push`方法，此时push方法中existed不为null，确保topContainer就是待打开的目标页面，也就是说，Overlay的层级正确
- 调用`onResumeAttachToFlutterEngine`，其实是调用`flutterView?.attachToFlutterEngine(flutterEngine)`，将Flutter App关联到FlutterView，渲染出来

这样一来，从FlutterApp的角度来讲，其实就是将顶层的OverlayEntry移除，展示了下层的OverlayEntry，并重新关联到FlutterView上。

