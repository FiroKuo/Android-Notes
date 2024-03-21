# 充电详情
充电详情即扫码支付后，准备开始充电到订单完成的状态变化过程。

## riverpod注意点
该模块使用了状态管理框架`riverpod`，它将数据的变化和UI的表达分开，适用于数据依赖关系较复杂的场景。模块中大量使用了`read`、`watch`、`listen`等获取数据的方式，理解它们的区别至关重要，`read`是主动读取一次`provider`中的数据；`watch`可以用于`widget`的`build`方法中，监听数据的改变，触发UI刷新，也可以用于其他`provider`中，用于对`provider`中的数据变化做出响应；`listen`也是用于监听数据变化的，与`watch`不一样，它不会读取当前`provider`的状态，只会监听后续的变化。有的时候我们需要梳理**数据监听链**是怎样的，比如，`widget`的`build`方法中，如果有监听多个`provider`，这些`provider`之间又有监听关系，那我们可以重新考虑一下这些`provider`之间是否可以使用`listen`而不是`watch`，或者我们是否可以在`build`中使用`listen`代替`watch`，这取决于具体需求。尤其需要注意使用`stateNotifierProvider`的时候，如果在构造函数中`watch`了其他的`provider`，当状态变化时，可能会导致构造函数重新调用，这样又会触发*初始状态*，如果想避免，则考虑初始状态设置的是否合理或者使用`listen`+`read`代替`watch`。

`riverpod`是有作用域一说的，一般，如果是整个`flutter`项目引入这个框架，仅需在`flutter app`处声明`ProviderScope`即可，这样全局都能共享`Provider`的状态；但这里仅用于详情模块，我们需要在详情模块共享这些`Provider`，则需声明在需要共享的页面的`ProviderScope`中:
```dart
@override
  Widget build(BuildContext context) {
    //虽说有依赖关系，并且同时使用了watch，但这里并不会导致widget多次构建：这三个provider都使用了listen。
    final status = ref.watch(orderStatusProvider);
    final page = ref.watch(chargePageProvider);
    final navigator = ref.watch(chargeNavigatorProvider);
    _onStatusChanged(status);
    return Scaffold(
      appBar: _buildAppBar(status, context),
      body: Container(
        decoration: ChargeThemeUtil.contentDecorationFor(status),
        child: Column(
          children: [
            Expanded(
              flex: 1,
              child: ProviderScope(
                observers: [ProviderLog()],
                overrides: [
                  pageActionProvider.overrideWithValue(this),
                ],
                parent: detailProviderContainer,
                child: page ?? Container(),
              ),
            ),
            ProviderScope(
              observers: [ProviderLog()],
              overrides: [
                navigatorPanelActionProvider
                    .overrideWithValue(() => _onPanelAction(context, status)),
              ],
              parent: detailProviderContainer,
              child: navigator ?? Container(),
            ),
            Container(
              height:
                  Platform.isIOS ? MediaQuery.of(context).padding.bottom : 0,
              decoration: ChargeThemeUtil.unSafeAreaDecorationFor(status),
            ),
          ],
        ),
      ),
    );
}
```
然后，在退出模块的时候需要手动清理`Provider`的状态，否则下次进入详情模块，`Provider`仍会保存上一次的状态，因为这些`Provider`一般是声明在顶层的，只有杀死进程后，状态才会重置：
```dart
 ///清理providers
void disposeProviders() {
    _chargeMessage?.unsubscribe();
    _chargeMessage = null;
    ref.invalidate(receiveMessageProvider);
    ref.invalidate(orderDetailProvider);
    ref.invalidate(orderStatusProvider);
    ref.invalidate(chargingStopStateProvider);
    ref.invalidate(payBtnStatusProvider);
    ref.invalidate(chargePaymentInfoProvider);
    ref.invalidate(couponSelectedProvider);
}
```

`Provider`是可以覆盖的，也就是说，我可以在一个子`ProviderScope`中覆盖全局的`Provider`，相当于这个子`Scope`有自己的状态管理逻辑，我们这里使用`overrides`把所有底部面板的动作委托到最顶层统一处理：
```dart
ProviderScope(
    observers: [ProviderLog()],
    overrides: [
        navigatorPanelActionProvider
            .overrideWithValue(() => _onPanelAction(context, status)),
    ],
    parent: detailProviderContainer,
    child: navigator ?? Container(),
)
```

## EventCode
充电状态包括启动中、充电中、停止中、账单生成中、待结算、订单完成，`eventCode`对应关系如下：
```dart
static const none = "0000";
static const unusual = "-1000";
static const prepare = "2000";
static const charging = "3000";
static const stopping = "4000";
static const generating = "5000";
static const pay = "7000";
static const finishing = "8000";
static const cancelled = "9000";
```

## 订单状态消息
App中一共有**两个**长连接：一是负责通知推送和订单状态推送的，一是负责`IM`消息的，它们都是应用启动后创建的(时序上后者稍早，后者在`main.dart`的初始化`flutter submodule`过程中，前者需要等`main.dart`执行到某种程度，通过`channel`向原生发送“flutter初始化成功”的消息)，使用同样的长连接框架`websocket`，但是创建了不同的`connection`，我们这里的充电消息推送使用的就是前者。`trippush`模块向`websocket`模块注册了消息监听，当收到来自`websocket`模块的消息时，`trippush`模块中除了通过`channel`通知原生消息到达，还会通过`listeners`通知`flutter`，`charge_util`向`trippush`模块注册了`listener`，这样就接收到了消息。

充电详情使用**长连接`websocket`**和`getLastEvent`主动轮询确保订单状态变化的及时响应，`charge_util`中订阅了长连接的消息，判断了时间戳和充电相关的`eventCode`，并且实现了`getLastEvent`轮询（默认10s，长连接中断时变为2s），对外统一封装成了同一个回调，统一了数据格式，外部只需订阅该回调即可接收充电状态的消息。

我们使用`receiveMessageProvider`来提供订单状态数据，它本身实现了`charge_util`中的`IChargeMessageListener`，`Provider`对外提供`subscribe`和`unsubscribe`方法来订阅/取消订阅`charge_util`中的长连接/`lastEvent`:
```dart
class ReceiveMessageNotifier extends StateNotifier<ReceiveMessage?>
    implements IChargeMessageListener {
  ReceiveMessageNotifier() : super(null);

  String _subscribeId = "";

  ///订阅
  void subscribe(String orderId) {
    _subscribeId = orderId;
    ChargeDetailLogger.info("开始订阅长连接: $orderId");
    ChargeMessageReceiver().subscribe(orderId, this);
  }

  ///取消订阅
  void unsubscribe() {
    _subscribeId = "";
    ChargeDetailLogger.info("取消订阅长连接");
    ChargeMessageReceiver().unsubscribe(this);
  }

  ///收到长连接推送消息
  @override
  void onArrived(ReceiveMessage message) {
    //仅关注当前订阅的message
    if (_subscribeId.isNotEmpty && _subscribeId == message.orderNo) {
      ChargeDetailLogger.info("收到新的推送消息/lastEvent: ${message.toMap()}");
      state = message;
    }
  }

  @override
  void dispose() {
    unsubscribe();
    super.dispose();
  }
}

///推送/轮询消息提供者
final receiveMessageProvider =
    StateNotifierProvider<ReceiveMessageNotifier, ReceiveMessage?>(
        (ref) => ReceiveMessageNotifier());
```
然后，在进入订单详情时订阅消息，退出订单详情/订单完成/异常时取消订阅消息:
```dart
@override
void initState() {
    super.initState();
    _preLoad();
    WidgetsBinding.instance.addPostFrameCallback((timeStamp) {
        final initRes = ChargeDetailInit.instance.initResponse();
        if (initRes != null) {
        ref.read(orderDetailProvider.notifier).init(initRes);
        } else {
        ref.read(orderDetailProvider.notifier).request(widget.orderId);
        }
        //订阅订单消息
        ref.read(receiveMessageProvider.notifier).subscribe(widget.orderId);
    });
}
```
```dart
void _onStatusChanged(status) {
    //停止充电中展示停止中弹框
    if (OrderStatus.isPrepareOrder(status.state)) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        if (OrderStatus.isPrepareOrder(status.state)) {
          _stoppingDialog(context);
        }
      });
    }
    //已完单/异常不需要订单消息
    if (OrderStatus.isUnusual(status.state) ||
        OrderStatus.isFinished(status.state)) {
      ref.read(receiveMessageProvider.notifier).unsubscribe();
    }
}
```
## 订单状态变化自动感知
收到订单消息后，刷新订单详情、订单状态改变这些都是自动的，这依赖于`Provider`之间的观察，`orderDetailProvider`listen了`receiveMessageProvider`，`orderStatusProvider`listen了`orderDetailProvider`，这三者共同构成了消息接收、状态变化、页面切换、数据刷新的基础：
```dart
class OrderDetailNotifier extends StateNotifier<AsyncValue<OrderDetailRes>> {
  OrderDetailNotifier(ref) : super(const AsyncValue.loading()) {
    //用listen的目的是不重新设置loading状态，因为这个时候监听这个provider的provider会拿到null
    //那么页面就会表现为 有数据->无数据->有数据 的变化
    ref.listen(receiveMessageProvider, (previous, next) {
      final orderId = next?.orderNo;
      if (StringUtils.isNotNullOrEmpty(orderId)) {
        debugPrint("OrderDetailNotifier: 收到新的通知, 刷新订单详情: $orderId $next");
        request(orderId!);
      }
    });
  }
  //...
}
```
```dart
class OrderStatusNotifier extends StateNotifier<OrderStatus> {
  OrderStatusNotifier(ref) : super(const OrderStatus(OrderStatus.none)) {
    //watch和listen是不同的，此处要保持状态，避免调用构造函数
    ref.listen(orderDetailProvider, (previous, next) {
      final data = next.value;
      final eventCode = data?.orderInfo?.eventCode ?? "";
      if (StringUtils.isNotNullOrEmpty(eventCode)) {
        debugPrint("OrderStatusNotifier: 订单详情发生变化: $eventCode");
        _change(eventCode);
      }
    });
  }

  ///订单状态的改变不能主动调用 依赖orderDetail的eventCode
  ///判断一下当前state 不重复设置值 避免无意义的UI刷新
  void _change(String status) {
    if (OrderStatus.isValid(status) && state.state != status) {
      ChargeDetailLogger.info("更新订单状态 previous: ${state.state}, next: $status");
      state = OrderStatus(status);
    }
  }
}
```
当其他地方需要监听订单状态变化时，仅需关注`orderStatusProvider`，当有其他页面或者`Provider`需要监听订单详情中的部分数据变化时，仅需关注`orderDetailProvider`，分工明确。

注意，在`orderDetailProvider`中如果某次接口请求失败，我们不直接返回`null`数据，而是复用上次的数据，否则页面就会展示*默认状态*，这个是要避免的：
```dart
///请求订单详情
///这里使用`copyWithPrevious`，出错的时候复用之前的数据
Future<void> request(String orderId) async {
  if (orderId.isEmpty) return;
  // state = const AsyncValue<OrderDetailRes>.loading().copyWithPrevious(state);
  try {
    final response = await ChargeDetailApi.getOrderDetail(orderId);
    final isSuccess = response.item1 == 0;
    final detail = response.item2;
    final message = response.item3;
    if (isSuccess && detail != null) {
      state = AsyncValue.data(detail);
    } else {
      state = AsyncValue<OrderDetailRes>.error(message, StackTrace.empty)
          .copyWithPrevious(state);
      ChargeDetailLogger.error("订单详情获取失败: $message");
    }
  } catch (e) {
    state = AsyncValue<OrderDetailRes>.error(e.toString(), StackTrace.current)
        .copyWithPrevious(state);
    ChargeDetailLogger.error("订单详情获取失败: ${e.toString()}");
  }
}
```
`AsyncValue<OrderDetailRes>.error(message, StackTrace.empty).copyWithPrevious`本身是一个`error`，但它也包含`data`，也就是说我们通过`valueOrNull`是可以取出来数据的。

## 页面切换
充电状态的切换意味着页面的切换，页面可以分为上下两部分：页面内容和底部操作区域，`flutter`中并没有`fragment`的概念，为了让页面自动响应状态，这里创建了`chargePageProvider`和`chargeNavigatorProvider`来提供上下两部分的`widget`，如果像是*启动充电中*这种状态，底部不需要要操作区域，`chargeNavigatorProvider`直接返回`Container`即可，这两个`Provider`监听了`orderStatusProvider`，当状态发生变化时，自动切换页面:
```dart
///订单状态发生变化时，自动切换页面
class ChargePageNotifier extends StateNotifier<Widget?> {
  ChargePageNotifier(ref) : super(null) {
    ref.listen(orderStatusProvider, (previous, next) {
      _switchPage(next);
      _log(previous, next);
    });
  }

  void _log(previous, next) {
    debugPrint("p: $previous, n: $next");
    SLog.onBehavior(
      'charge-076',
      '订单详情页面展示',
      page: '订单详情',
      type: BehaviorManager.TYPE_PAGE,
      data: {"orderStatus": next.state},
    );
  }

  void _switchPage(OrderStatus status) {
    switch (status.state) {
      case OrderStatus.prepare:
        if (state is! ChargePreparePage) {
          state = const ChargePreparePage();
        }
        break;
      case OrderStatus.charging:
      case OrderStatus.stopping:
      case OrderStatus.generating:
        if (state is! ChargeChargingPage) {
          state = const ChargeChargingPage();
        }
        break;
      case OrderStatus.unusual:
        if (state is! ChargeUnusualPage) {
          state = const ChargeUnusualPage();
        }
        break;
      case OrderStatus.pay:
      case OrderStatus.finishing:
        if (state is! ChargeOrderPage) {
          state = const ChargeOrderPage();
        }
        break;
      default:
        state = null;
        break;
    }
  }
}

///充电内容页提供者
final chargePageProvider =
    StateNotifierProvider<ChargePageNotifier, Widget?>((ref) {
  return ChargePageNotifier(ref);
});
```
注意，`dart`中，`is!`代表的才是非，而不是`!is`；只有在当前`state`不是目标页面类型时，才会切换页面，这是考虑到了复用，比如`charging`、`stopping`以及`generating`都会复用`ChargeChargingPage`，`pay`和`finishing`都会复用`ChargeOrderPage`，变的只是弹窗或者部分UI效果。

## 充电进度
充电中充电进度的变化是依靠轮询`realTime`实现的，这算是一个轻量级的接口，第一次进入充电中状态，当前的充电进度信息取自`orderDetail`，后续每30s轮询一次`realTimeInfo`，刷新页面的充电进度:
```dart
//不使用watch是避免状态更新的时候赋予初始值'null'
//搭配read&listen解决这个问题
RealTimeInfoNotifier(ref) : super(null) {
    //init data from orderDetail
    final init = ref.read(orderDetailProvider).value?.realTimeInfo;
    if (init != null) {
        state = _convert(init);
    }
    final current = ref.read(orderStatusProvider);
    _hintStatus(ref, current);
    ref.listen(orderStatusProvider, (previous, next) {
        _hintStatus(ref, next);
    });
}
```
可以看到初始数据取自`orderDetailProvider`，这里为了避免使用`watch`导致构造方法重新调用，最终导致页面充电进度展示为默认状态，使用了`read`+`listen`的方式获取`orderStatusProvider`的状态，只有在充电中才会轮询`realTimeInfo`，其他状态会取消轮询:
```dart
//状态检查
void _hintStatus(ref, status) {
    if (OrderStatus.isCharging(status.state)) {
        //watch read均可 方法结束后就结束了
        final orderId = ref.watch(chargeOrderInfoProvider)?.orderNo ?? "";
        if (orderId.isEmpty) return;
        //do loop
        _subscribe(ref, orderId);
    } else {
        _unsubscribe();
    }
}
```
轮询的过程就是启动一个`Timer`，定时查询接口，然后通过`Stream`更新状态，注意`mounted`的判断:
```dart
//开始轮询
void _subscribe(ref, orderId) {
    _unsubscribe();
    ChargeDetailLogger.info("准备轮询实时充电进度：$orderId");
    _stream = StreamController();
    _stream?.stream.listen((event) {
        ChargeDetailLogger.info("收到实时充电进度：$event");
        if (mounted) {
            state = event;
        }
    });
    _timer = Timer.periodic(_duration, (timer) async {
        debugPrint("loop realtime: currentTime: ${DateTime.now()}");
        _queryRealTimeInfo(ref, orderId);
    });
}

//查询充电实时进度
void _queryRealTimeInfo(StateNotifierProviderRef ref, orderId) async {
    final future = ref.read(queryRealTimeInfoFutureProvider(orderId).future);
    try {
        final response = await future;
        final info = response.item2?.info;
        if (info != null) {
        _stream?.add(_convert(info));
        }
    } catch (e) {
        ChargeDetailLogger.error(e.toString());
    }
}
```
因为`queryRealTimeInfoFutureProvider`是一个`FutureProvider`，`FutureProvider`的特性是第一次会触发接口请求，当已经获取到数据后，下次读取会直接返回数据。我们使用`ref.read(queryRealTimeInfoFutureProvider(orderId).future)`拿到`future`，然后主动触发接口调用。

## 充电动画
充电动画使用的并不是`lottie`，UI的意思是`lottie`不能实现蒙版透明的效果，所以这里其实是一段循环播放的视频：
```dart
///充电视频
Widget _buildVideoWidget() => ClipRect(
    child: SizedBox(
        width: 327,
        height: 160,
        child: FittedBox(
            fit: BoxFit.contain,
            child: Transform.scale(
                alignment: Alignment.center,
                scaleX: 1.1,
                scaleY: 1.1,
                child: ClipRect(
                    child: SizedBox(
                        width: _controller.value.size.width,
                        height: _controller.value.size.height,
                        child: VideoPlayer(_controller),
                    ),
                ),
            ),
        ),
    ),
);
```
注意，`flutter`渲染原生组件的视频组件是有问题的，会出现"绿边"的现象，所以这里使用了`ClipRect`和`FittedBox`组件搭配，缩放然后把"绿边"裁掉，后续调整视频大小的时候要注意这一点。

## 停止中/账单生成中
这两个状态比较特殊，他们都是不可取消的弹框，页面是直接复用`ChargeChargingPage`，我们在最顶层页面的`build`方法中监听了状态变化，如果是停止中/账单生成中，会展示弹框，包括首次进入的时候也能正确展示弹框：
```dart
void _onStatusChanged(status) {
    //停止充电中/账单生成中展示弹框
    if (OrderStatus.isPrepareOrder(status.state)) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        if (OrderStatus.isPrepareOrder(status.state)) {
          _stoppingDialog(context);
        }
      });
    }
    //...
}
```
注意这里的`addPostFrameCallback`，因为这个方法是在`build`中调用的，所以不能直接弹框，要等到下一帧。
```dart
///停止充电弹框
Future<void> _stoppingDialog(context) async {
    if (_isStoppingDialogShowing) return;
    _isStoppingDialogShowing = true;
    final message = await DialogUtils.show(
        context,
        ProviderScope(
        observers: [ProviderLog()],
        parent: detailProviderContainer,
        child: const StopChargingDialog(),
        ),
    );
    if (StringUtils.isNotNullOrEmpty(message)) {
        SCToastUtils.text(context, message);
    }
    _isStoppingDialogShowing = false;
}
```
我们使用`_isStoppingDialogShowing`避免重复弹出`dialog`，`StopChargingDialog`中，我们为了健壮性考虑，仍然需要在`build`方法中监听`status`的状态，自动销毁弹框:
```dart
final status = ref.watch(orderStatusProvider);
if (!OrderStatus.isPrepareOrder(status.state)) {
    WidgetsBinding.instance.addPostFrameCallback((timeStamp) {
        if (!OrderStatus.isPrepareOrder(status.state) &&
            ModalRoute.of(context)?.isCurrent == true) {
            Navigator.pop(context);
        }
    });
}
```
此外，还要注意一下接口调用，我们使用`stopChargeProvider`这个`FutureProvider`发起停止请求，如果接口调用成功，会延迟1.5s获取订单详情，然后把停止结果返回：
```dart
///停止充电
final stopChargeProvider = FutureProvider.autoDispose
    .family<Tuple3<int, ChargeEndRes?, String>, String>((ref, orderNo) async {
  final state = ref.watch(chargingStopStateProvider);
  //当自动停止的时候，按钮状态为StopBtnStatus.normal，stopChargeProvider不需要主动发起endCharge
  if (state == StopBtnStatus.normal) throw Exception("状态异常");
  if (orderNo.isEmpty) throw Exception("参数异常!");
  final result = await ChargeDetailApi.endCharge(orderNo);
  ChargeDetailLogger.info("停止充电结果：${result.toString()}");
  if (result.item1 == 0 && result.item2 != null) {
    ///刷新订单详情
    await Future.delayed(const Duration(milliseconds: 1500), () {
      ChargeDetailLogger.info("停止充电1.5s后主动刷新订单详情.");
      return ref.read(orderDetailProvider.notifier).request(orderNo);
    });
  }
  return result;
});
```
`StopDChargingDialog`中还监听了这种立即停止的情况，详情请求成功后，关闭弹框:
```dart
ref.listen(
    stopChargeProvider(ref.read(chargeOrderInfoProvider)?.orderNo ?? ""),
    (previous, next) {
        next.when(
            data: (info) {
                final success = info.item1 == 0;
                if (!success && ModalRoute.of(context)?.isCurrent == true) {
                  Navigator.pop(context, info.item3);
                }
    },
    error: (error, stack) {
        // debugPrint("停止充电异常：$error");
        //因为stopChargeProvider是一个futureProvider，所以第一次listen的时候会执行它的逻辑。那就不能写`if (state == StopBtnStatus.normal) throw Exception("状态异常");`那自动停止就会调用`endCharge`方法。
        //反之，如果这里pop掉，那就没办法处理订单自己停止的场景(自动停止按钮状态不会变化，仍为 StopBtnStatus.normal) 弹框弹不出来
        // if (ModalRoute.of(context)?.isCurrent == true) {
        //   Navigator.pop(context, error.toString());
        // }
    },
    loading: () {},
    );
});
```
此外，注意一下*自动停止*，如果接口返回停止中的状态，这个时候不需要我们手动调用`endCharge`，在`stopChargeProvider`中我们判断了按钮的状态:
```dart
final state = ref.watch(chargingStopStateProvider);
//当自动停止的时候，按钮状态为StopBtnStatus.normal，stopChargeProvider不需要主动发起endCharge
if (state == StopBtnStatus.normal) throw Exception("状态异常");
```
还有一点，就是对`stopChargeProvider`所抛出的异常的处理，这里我们不需要处理任何异常分支，上边的注释解释了这一情况。我们依赖订单状态改变销毁弹出，而不是`endCharge`调用异常。

### 停止中和账单生成中状态的切换
关于如何从停止中切换到账单生成中，`StopChargingDialog`是**复用**的，因为两者的弹框数据都是本地的，我们在弹框中监听`status`的改变即可，当`status`改变时，弹框的数据会刷新，达到切换状态的目的。

## 账单刷新
待支付需要拉取收银台的信息`querySettlePaymentInfo`，刷新的时机有以下两种：
- 当订单详情发生变化时
- 当选择优惠券时

我们使用`chargePaymentInfoProvider`来监听这两种场景，自动触发收银台的数据刷新：
```dart
void _listen(StateNotifierProviderRef ref) {
    ref.listen(chargeOrderInfoProvider, (previous, next) {
        final status = ref.read(orderStatusProvider);
        final orderNo = next?.orderNo ?? "";
        final coupon = ref.read(couponSelectedProvider);
        _prepare(ref, status, orderNo, coupon);
    });

    ref.listen(couponSelectedProvider, (previous, next) {
        //获取预付信息后，会给selectProvider赋值，避免刷新
        if (next?.isDefault == true) return;
        final status = ref.read(orderStatusProvider);
        final orderNo = ref.read(chargeOrderInfoProvider)?.orderNo ?? "";
        final coupon = next;
        _prepare(ref, status, orderNo, coupon);
    });
}

void _prepare(StateNotifierProviderRef ref, OrderStatus status,String orderNo, CouponSelected? coupon) {
    if ((OrderStatus.isPay(status.state) || OrderStatus.isFinished(status.state)) && orderNo.isNotEmpty) {
        ChargeDetailLogger.info(
            "订单详情刷新/选择优惠券的时候，刷新账单信息： orderNo:$orderNo,coupon: ${coupon.toString()}");
        _request(ref, orderNo, coupon);
    }
}
```
`chargeOrderInfoProvider`包含了部分`orderDetail`中的数据，是对`orderDetailProvider`更精简的提炼，使用它获取`orderNo`，并监听订单详情的更新；`couponSelectedProvider`是优惠券选择状态的提供者，包含当前选择的优惠券信息。当监听到这两者任意一个发生变化，会触发接口请求，刷新收银台:
```dart
///发起请求
Future<void> _request(StateNotifierProviderRef ref, String orderId, CouponSelected? coupon) async {
    if (orderId.isEmpty) return;
    try {
        final paymentRequest = QuerySettlePaymentInfoReq(orderId,
                selectedChargeCoupon: coupon?.coupon,
                selectedCoupon: coupon?.selectedCoupon ?? 0);
        final response =
            await ChargePayApi.querySettlePaymentInfo(paymentRequest);
        final isSuccess = response.item1 == 0;
        final payment = response.item2;
        final message = response.item3;
        if (mounted) {
            if (isSuccess && payment != null) {
                state = AsyncValue.data(payment);
                ref
                    .read(couponSelectedProvider.notifier)
                    .defaultCoupon(payment.couponDetailList);
            } else {
                state = AsyncValue.error(message, StackTrace.empty);
                ref.read(couponSelectedProvider.notifier).defaultCoupon(null);
            }
        }
    } catch (e) {
        if (mounted) {
            state = AsyncValue.error(e.toString(), StackTrace.current);
            ref.read(couponSelectedProvider.notifier).defaultCoupon(null);
        }
    }
}
```

## 选择优惠券
收银台的优惠券数据是`queryPaymentInfo`一起返回的，默认勾选的优惠券/限时优惠存放在`couponDetailList`，用户充电券列表存放在`chargeCouponList`中，可以用前者匹配后者的默认选中:
```dart
if (isSuccess && payment != null) {
      state = AsyncValue.data(payment);
      ref
          .read(couponSelectedProvider.notifier)
          .defaultCoupon(payment.couponDetailList);
    } else {
      state = AsyncValue.error(message, StackTrace.empty);
      ref.read(couponSelectedProvider.notifier).defaultCoupon(null);
    }
```
我们使用`couponSelectedProvider`来管理优惠券的选中状态，值得注意的是，上边我们有提到，收银台刷新的条件之一就是优惠券选中状态发生变化，**为了避免循环触发优惠券选中-刷新收银台**我们在`CouponSelected`对象中使用`isDefault`标志是否是收银台数据请求后的默认选中，然后，在触发收银台刷新的位置判断:
```dart
 ref.listen(couponSelectedProvider, (previous, next) {
    //获取预付信息后，会给selectProvider赋值，避免刷新
    if (next?.isDefault == true) return;
    final status = ref.read(orderStatusProvider);
    final orderNo = ref.read(chargeOrderInfoProvider)?.orderNo ?? "";
    final coupon = next;
    _prepare(ref, status, orderNo, coupon);
  });
```
选择优惠券的时候，是否可以点击是根据充电券列表中的`status`字段判断状态的，`any status == 1` 则说明有可选的优惠券，把`chargeCouponList` map 为个人优惠券列表模块的数据格式，然后由个人优惠券列表渲染，选择完毕后返回`batchId`和`couponId`，从`chargeCouponList`中匹配选中的对象，调用`couponSelectedProvider`的`select`方法设置优惠券，触发收银台的数据刷新:
```dart
///选中优惠券
void select(int couponId, int batchId) {
  final chargeCoupons =
      _ref?.read(paymentChargeCouponsProvider) ?? List.empty();
  ChargeCoupon? selected;
  for (int i = 0; i < chargeCoupons.length; i++) {
    final coupon = chargeCoupons[i];
    if (coupon.couponId == couponId && coupon.batchId == batchId) {
      selected = coupon;
      ChargeDetailLogger.info("选中优惠券: $coupon");
      break;
    }
  }
  if (mounted) {
    state = CouponSelected(
      selected,
      selectedCoupon: selected == null ? 2 : 1,
      isDefault: false,
    );
  }
}
```

## 结算
订单存在自动结算的情况，比方说预付的钱花完了，退款为0的情况，这种情况下可能不会进入待结算状态。反之如果有退款，则可能需要用户手动点击结算按钮，这个时候会发起结算，我们使用`payBtnStatusProvider`这个`StateProvider`来标志按钮点击状态:
```dart
 ///发起结算
void _settlement(context) {
  ref.read(payBtnStatusProvider.notifier).state = PayBtnStatus.paying;
}
```
当多次点击按钮的时候，因为状态已经是`PayBtnStatus.paying`，并不会多次触发状态更新。我们在`Widget`中监听状态的变化，展示按钮的文案以及`loading`，在`makeChargePayProvider`中监听状态变化，触发真正的
结算接口调用：
```dart
MakeChargePayNotifier(StateNotifierProviderRef ref)
    : super(const AsyncValue.loading()) {
    //监听按钮的状态
    //使用listen是因为支付成功后不会将PayBtnStatus改为PayBtnStatus.normal(避免再次点击)
    //而支付成功后，会刷新详情，进而触发页面build，如果用watch的话这里的构造函数会刷新，再次执行_makePay
    ref.listen(payBtnStatusProvider, (previous, next) {
        if (PayBtnStatus.paying == next) {
          ChargeDetailLogger.info("监听到支付按钮点击，发起支付: $next");
          _makePay(ref);
        }
      });
    //...
}
```
当发起支付成功后，要轮询查询支付状态，轮询时间间隔为1.5s:
```dart
/// 轮询
void _periodic(StateNotifierProviderRef ref, String payOrderId) {
  _timer?.cancel();
  _stream?.close();
  _stream = StreamController();
  _stream?.stream.listen((event) {
    if (mounted) {
      state = event;
    }
  });
  _timer = Timer.periodic(_duration, (timer) async {
    final status = ref.read(orderStatusProvider);
    //状态已经发生变化
    if (!OrderStatus.isPay(status.state)) {
      ChargeDetailLogger.info("订单状态已经发生变化,取消支付状态查询的轮询");
      _timer?.cancel();
      _stream?.close();
      if (mounted) {
        state = const AsyncValue.data(null);
      }
      return;
    }
    if (!(await _queryPayStatus(ref, payOrderId))) {
      _timer?.cancel();
      _stream?.close();
    }
  });
}
```
注意，轮询过程中*要多次判断当前订单状态*`final status = ref.read(orderStatusProvider);`，不止是轮询开始，包括接口请求结束都需要判断最新订单状态，如果不是待支付，则取消轮询。还有一点，在使用`FutureProvider`的时候，`AsyncValue`的状态包括`AsyncValue.loading`、`AsyncValue.error`、`AsyncValue.data`都是框架判断返回的，你不能自己`return AsyncValue.error()`，这是不生效的，想要触发`AsyncValue.error`只能`throw exception`，这中特点是`FutureProvider`特有的。

## 充电结束/充电异常
充电结束直接复用的是结算页面`ChargeChargingPage`，只不过这个状态底部没有操作区，其实是`chargeNavigatorProvider`直接返回`null`，这样会渲染为`Container`，底部就不会展示内容了。充电结束收银台信息仍然是调用`queryPaymentInfo`获取账单信息。账单内容展示有些条目是有差异的，这个都是根据`status`判断的；充电结束不能选择优惠券，充电结束不会订阅订单消息。

充电异常是复用的充电中的页面`ChargeChargingPage`，是指`eventCode`是-1000的情况；9000是取消状态，是个预留状态，目前并没有对应的页面展示这种状态。异常和充电中这两种页面布局十分相似，大多数都是UI的差异，我们使用`ThemeUtil`依据`status`处理这些差异情况。异常状态底部按钮点击会触发联系客服，调用原生进行拨号。异常状态不会订阅订单消息。

## 体验优化
由于订单详情有两种背景，黑色(启动充电、充电中、充电异常)和白色(待结算、订单完成)，我们打开订单详情的时候，不太好使用一种默认的背景，因为你也不知道此时订单状态是什么，这样会造成页面闪动的问题，用户体验不太好。我们使用一个**透明的中间页面**来请求订单详情，比方说，在订单列表中点击，这个时候会先跳转到透明页面`ChargeDetailSpringboard`，请求订单详情数据，但这个页面不会立即展示`loading`，而是延迟2s，如果订单详情在此期间返回数据，就不会展示`loading`了；除此之外，订单详情接口本身请求也有时间限制，如果5s内没有获取到数据，网络不太顺畅，则会`Toast`网络异常并关闭透明页面:
```dart
///预加载数据
void _preload() {
  _loadingTask = _showLoading();
  _preloadTask = ChargeDetailApi.getOrderDetail(widget.orderId)
      .timeout(_max)
      .then((value) {
    final response = value.item2;
    ChargeDetailInit.instance.init(res: response);
    setState(() {
      _loading = false;
    });
    _navigator();
  }).catchError((error) {
    setState(() {
      _loading = false;
    });
    ChargeDetailInit.instance.clear();
    SCToastUtils.error(context, "网络异常");
    Navigator.pop(context);
  });
}

Future<void> _showLoading() => Future.delayed(_delay, () {
    setState(() {
      _loading = true;
    });
});
```
这个页面的结构很简单，它根据`_loading`变量展示隐藏`LoadingWidget`:
```dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    backgroundColor: Colors.transparent,
    body: Center(
      child: Visibility(
        visible: _loading,
        child: SCLoading.loadingWidget(),
      ),
    ),
  );
}
```
进入订单详情的来源大致可以分为两类，一类是正常下单流程中从始至终走进入详情，一类就是从其他入口打开订单详情了。对于前一种情况，要求把从首页到详情中间所有的页面出栈，我们使用`clearTop`标志这种情况:
```dart
//对于正常下单流程 详情返回需要pop到首页
if (widget.clearTop) {
  Navigator.pushAndRemoveUntil(
      context,
      MaterialPageRoute(
        settings: const RouteSettings(
          name: OrderDetailManager.detailRouteDirect,
        ),
        builder: (context) => ChargeDetailScope(widget.orderId),
      ), (route) {
    debugPrint("clear Top: ${route.settings.name}");
    if (route.settings.name == CR.chargeHome) {
      debugPrint("clear Top stop: ${route.settings.name}");
      return true;
    }
    return false;
  });
} else {
  Navigator.pushReplacement(
    context,
    MaterialPageRoute(
      settings: const RouteSettings(
        name: OrderDetailManager.detailRouteDirect,
      ),
      builder: (context) => ChargeDetailScope(widget.orderId),
    ),
  );
}
```
如果是`clearTop`，我们需要判断路由，是不是`CR.chargeHome`，不是则关闭页面，所以，为相应的页面设置路由很重要；如果一个页面需要判断页面路由，则需要为其设置路由，一种情况是我们使用框架中的`RouterApi`的`openFlutterPage`，这种情况我们已经在框架中统一添加了当前页面的路由，`saic_flutter_app.dart`中的`push`方法:
```dart
if (withContainer) {
    final container =
        _createContainer(pageInfo, first, backgroundTransparent);
    containers.add(container);
    // FocusManager.instance.primaryFocus?.unfocus();
    refreshOnPush(container);
  } else {
    if (!transparentRoute) {
      topContainer.navigator?.push(MaterialPageRoute(
          builder: (context) {
            return first!;
          },
          settings: RouteSettings(name: pageName),
          fullscreenDialog: fullscreenDialog));
    } else {
      topContainer.navigator?.push(TransparentRoute(
          builder: (context) {
            return first!;
          },
          settings: RouteSettings(name: pageName)));
    }
}
```
如果是直接从`flutter`中打开页面，则走`else`的逻辑，我们增加了`settings: RouteSettings(name: pageName)`，`pageName`即为`path`；如果是原生打开`flutter`页面，`witchContainer`是`true`，最终会创建`ContainerOverlayEntry`，它对应的`Widget`其实是`StateWidget`，我们在`build`方法中为其添加路由:
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
                arguments: routeSettings.arguments,
              ),
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
还有一种情况是项目中自己使用`Navigator.push`打开页面，如果这个页面会被判断路由，则要手动为其增加路由，这一点要注意。

# 充电弹窗
充电弹窗设计类似于原生，均参考`okhttp`拦截器的责任链设计，优点是可以自定义弹窗组合、弹窗顺序，提供继续弹窗和中断弹窗的接口，外部可以选择1次弹窗或者按场景触发弹窗而不会创建多个弹窗链。

`okhttp`的责任链设计为链表前序遍历的时机处理`request`，后续遍历的时机处理`response`，类似于：
```java
Response intercept(Chain chain){
    //request
    Response response = intercept(chain.next);
    //response
    return response;
}
```
整体上来看，他就是链表的递归遍历方式，`Chain`是作为链表节点存在的，`Chain`的`index`索引的是`interceptors`中`interceptor`的位置。`okhttp`约定在`Chain`中调用`interceptor`的`intercept`方法，在`interceptor`中调用`Chain`的`process`方法，结合来看，就是上边伪代码的链表递归遍历。

对于我们弹窗责任链的设计，是没有返回值的，我们使用一次遍历即可解决弹窗问题，这和`okhttp interceptor`的设计是思想上的不同，有返回值更类似于一种分解子问题的思想。

我们同样约定，在`interceptor`中，要么调用`process`流转，要么调用`interrupt`中断。流转的核心代码如下:
```dart
 ///进行弹框流程
  void process() {
    if (index >= interceptors.length) {
      status.complete();
      debugPrint("弹框结束...");
      return;
    }
    if (isInterrupted()) {
      return;
    }
    if (alertConfig.isAssociatedPage) {
      final isTopRoute = ModalRoute.of(buildContext)?.isCurrent ?? false;
      if (!isTopRoute) {
        debugPrint("非初始弹窗页面,弹框中断...");
        status.interrupt(this);
        return;
      }
    }
    if (index == 0) {
      debugPrint("弹框开始...");
    }
    final next = _copy(index + 1, interceptors, buildContext, alertConfig,
        logicListener, status);
    final interceptor = interceptors[index];
    status.process(next);
    debugPrint("弹框进行...$index}");
    try {
      interceptor.intercept(next);
    } catch (e) {
      ChargeAlertLogger.error("chain exception: $e}");
      //跳过异常节点
      next.process();
    }
  }
```
我们使用`StateHolder`保持弹窗流程整体的状态，`StateHolder`是整个链共享的，比如中断状态:
```dart
void interrupt(AlertChain node) {
    if (_status == statusComplete) return;
    _interruptNode = node;
    _processingNode = null;
    _status = statusInterrupt;
}

bool isInterrupt() => _status == statusInterrupt && getInterruptNode() != null;

AlertChain? getInterruptNode() => _interruptNode;
```
我们对外暴露一个接口，仅包含`process`和`interrupt`方法：
```dart
abstract class IDialogChain {
  void process({bool maybeContinue = false});

  void interrupt({bool maybeDismiss = false});
}
```
我们通常在`flutter`页面可见时，处理弹窗逻辑，这个时候，调用`process`:
```dart
@override
void process({bool maybeContinue = false}) {
    if (_chain.isProcessing()) {
        debugPrint("${ChargeAlertLogger.logTag},正在弹框中...");
        return;
    }
    if (_chain.isInterrupted()) {
        final node = _chain.findInterruptNode();
        debugPrint(
            "${ChargeAlertLogger.logTag},流程中断过,从中断处继续? $maybeContinue 中断节点位置: ${node?.index}");
        if (node == null || !maybeContinue) {
            _chain = _chain.reset();
            } else {
            _chain = _chain.reboot();
        }
    }
    if (_chain.isCompleted()) {
        _chain = _chain.reset();
        debugPrint("${ChargeAlertLogger.logTag},弹框已完成,重置...");
    }
    debugPrint("${ChargeAlertLogger.logTag},继续弹框,chain: ${_chain.index}");
    _chain.process();
}
```
这些状态判断仍然依赖于`StateHolder`，如果判断为弹窗流程结束，需要重新弹窗，则调用`reset`:
```dart
///重置弹框流程
AlertChain reset() {
    status.none();
    return _copy(0, interceptors, buildContext, alertConfig, logicListener, status);
}
```
如果判断为弹窗中断，需要继续弹窗，则调用`reboot`:
```dart
///继续弹框流程
AlertChain reboot() {
    final interrupt = findInterruptNode();
    if (interrupt == null) {
        return reset();
    }
    final status = interrupt.status;
    status.none();
    return _copy(
        interrupt.index,
        interrupt.interceptors,
        interrupt.buildContext,
        interrupt.alertConfig,
        interrupt.logicListener,
        status);
}
```
对于`interceptor`，我们把流程的流转抽象出来，实现的时候仅需关注是否需要弹窗、提供弹窗、处理弹窗返回值几个函数即可:
```dart
abstract class IAlertInterceptor {
  //通常来说，你不需要重写此方法
  void intercept(AlertChain chain) {
    //try-catch不能捕获Future异常
    //如果重写的preconditions是async的，你要自己捕获异常
    try {
      preconditions(chain.alertConfig, chain.logicListener, (data) {
        //使用回调避免警告：Do not use BuildContexts across async gaps.
        //但仍需对异步后的BuildContext做健壮性判断，捕获异常
        if (data.item1 == true) {
          _showDialog(chain, data.item2);
        } else {
          chain.process();
        }
      });
    } catch (e) {
      if (onCatchError(e)) {
        chain.process();
      } else {
        chain.interrupt();
      }
    }
  }

  //展示弹框
  void _showDialog(AlertChain chain, dynamic data) {
    final dialog = provideDialog(data);
    if (dialog == null) {
      chain.process();
      return;
    }
    if (chain.alertConfig.isAssociatedPage) {
      final isTopRoute = ModalRoute.of(chain.buildContext)?.isCurrent ?? false;
      if (!isTopRoute) {
        debugPrint("非初始弹窗页面1,弹框中断...");
        chain.interrupt();
        return;
      }
    }
    DialogUtils.show(
      chain.buildContext,
      dialog,
    ).then(
      (value) async {
        bool isContinue;
        try {
          isContinue =
              await handleResult(value, chain.alertConfig, chain.logicListener);
        } catch (e) {
          isContinue = onCatchError(e);
        }
        if (isContinue) {
          chain.process();
        } else {
          chain.interrupt();
        }
      },
    ).catchError((e) {
      //捕获异常 确保流程继续
      if (onCatchError(e)) {
        chain.process();
      } else {
        chain.interrupt();
      }
    });
  }
  //...
}
```
`Dialog`中返回值的话，其实就是`Navigator.pop(context,value)`。

值得注意的一点是，我们可能需要仅在指定页面弹出弹窗，`config`中提供了是否关联页面属性`isAssociatedPage`，如果当前弹窗由于请求接口等原因还没有展示出来，流程中会判断当前栈顶是否是初始弹窗页面，这样的判断一共有两处，一是流程`process`方法中，一是真正弹窗前:
```dart
if (alertConfig.isAssociatedPage) {
      final isTopRoute = ModalRoute.of(buildContext)?.isCurrent ?? false;
      if (!isTopRoute) {
        debugPrint("非初始弹窗页面,弹框中断...");
        status.interrupt(this);
        return;
      }
}
```
```dart
if (chain.alertConfig.isAssociatedPage) {
      final isTopRoute = ModalRoute.of(chain.buildContext)?.isCurrent ?? false;
      if (!isTopRoute) {
        debugPrint("非初始弹窗页面1,弹框中断...");
        chain.interrupt();
        return;
      }
}
```
最终，弹窗使用时在页面初始化的时候创建弹窗链：
```dart
void _initAlertChain() {
    _chain = ChargeDialogProvider.getAlterChain(
      context,
      AlertConfig(businessTag: 121725),
      [DialogType.notice, DialogType.redEnvelop, DialogType.ads],
      this,
    );
}
```
在生命周期方法中调用`process`即可：
```dart
/// 首页弹出检测
/// 协议弹框必须第一时间弹出来
_homeAlert() async {
    debugPrint('首页弹窗检测');
    await ChargeManager.instance.tryChargeLoginIfNeed(timeout: 200);
    _chain.process();
}
```

