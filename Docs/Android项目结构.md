# 结构
上一篇讨论了架构的问题，也给出了一个组件化的标准以及实现方式。这一片要讨论一下项目的构成，以及各`module`的作用。
我大致数了一下，包括壳工程和`flutter`混合工程，项目大致包含**34**个`module`，如果再算上`flutter`的`plugin module`，那工程中的`module`数量大概有**50**个左右，是一个相当庞大的项目。

## 目录
- shell
- ad
- alert-kit
- base
- city
- common
- common-style
- face-recognition
- future-travel
- hitchhike
- login
- multi-appoint
- near-car
- order-event-receiver
- passenger-common
- flutter-center
- flutter-impl
- passenger-main
- order-status
- passenger-style
- passenger-trip
- passenger-trip-cancel-order
- pay
- place-order
- placeorder-ent
- risk-control
- select-order-address
- service
- substitute
- substitute-cancel
- substitute-passenger-trip
- substitute-select-address
- user-info
- usual-address
- webview

其实，有一些`module`完全是可以合并的，有点边界划分不清晰，比如，`substitue`、`substitute-cancel`、`substitute-passenger-trip`、`substitute-select-address`这些都是属于代驾的业务，这些模块不可能单独拿出去在多个项目中分享，完全可以合在一起；再比如，`common`、`common-style`、`passenger-common`，事实上对于不同的业务线，这些模块也难以复用了，现状是不同的业务线，它们分别在不同的分支，那么，随着时间的发展，其实这些难以复用的模块没有再维护，不好重构的话，是否可以合并呢？`base`其实就是想做这件事情，它们同时出现在这里的原因只有一个，没有进行完毕，这也能看出来，想要重构项目在实际开发中并非易事，一般是没有额外工时的，这种吃力不讨好的事情，需要大家有较强的执行力。

## shell
`shell module`是项目的壳工程，它不包含任何业务逻辑，对于组件化来讲，它依赖所有的业务组件的成品aar，将组件整合到一起，构成整个应用
- build.gradle：这是工程的主要配置文件，包括`plugins`的应用与配置，`signingConfigs`、`buildTypes`的配置，`ndk abiFilters`的配置以及部分自定义任务等
- Application：整个项目的自定义Application，由于壳工程依赖所有的业务组件，所以业务组件的初始化是统一放在`onCreate`中的，比如，当前环境/域名的配置、`ARouter`初始化、配置第三方`appId`、配置数据合规、Bugly、数据上报、网络/DNS、Flutter引擎预热、踢出登录监听、初始化长连接、IM、推送、分享，预请求ads、新设备、一些实验值等
- Androidmanifest.xml: 除了`application`节点本身的配置外，基本上不能存放任何内容，`application`节点包括声明自定义`Application`、`icon/roundIcon/label`、`networkSecurityConfig`；这里还声明了包可见性；
    其中，有一个属性是`allowNativeHeapPointerTagging`：
    > 在 Android 10（API 级别 29）及更高版本中，操作系统引入了对原生代码的一项安全增强功能，即指针标记。指针标记是一种利用处理器架构特性来增强内存安全的技术。它通过在指针中添加额外的信息（标记）来帮助检测和防止内存损坏漏洞，比如缓冲区溢出或使用后释放（use-after-free）漏洞。
    
    升级到Android高版本后(targetSDKVersion>= 30)，发现AMap在vivo手机上有崩溃的现象，这个值要置为false。
    `networkSecurityConfig`中要配置一下允许`http`以及抓包的域名，基本是测试环境的域名。

#### 环境切换
#### easy bug
#### mmkv
#### webview
#### video
#### jacoco
#### 合规
#### bugly
#### reporter
#### location
#### http header
#### network
#### dns
#### 长连接
#### IM
#### push
#### share
#### 预请求

## ad
ad是全局的广告/活动服务提供模块，可以把它看成业务功能组件，可以获取一个广告`getAd`，也可同时获取多个广告`getMultiAd`，广告区分不同的业务线、广告位置，最终调用的都是`/ad/requestAd`。但需要注意的是，获取多条广告的时候，有些组合不能同时搭配，这可能是不同的后台中台导致的。除了获取广告，该模块还提供`report`方法，上报广告的点击连接等。

## alert-kit
全局弹窗模块可以看作一个业务功能模块。
最开始弹窗逻辑是写在一个工具类中的，是一个递归，检测`index`的位置作为结束条件，使用`flag`标志当前是否正在处于弹窗中；后来发现这种方式不易维护，没办法自定义弹窗的类型和顺序，增加减少弹窗都需要修改弹窗结束条件，不符合开闭原则。

后来参考`okhttp`的拦截器的责任链模式重构了这一块的逻辑，但也有不同。`okhttp`的责任链设计为链表前序遍历处理`request`，后续遍历处理`response`，类似于：
```java
Response intercept(Chain chain){
    //request
    intercept(newChain)
    //response
}
```
这里涉及弹窗的时候只需要在前序遍历的时候处理弹窗的逻辑即可，当弹窗关闭后，看情况决定是否要继续流转，我们约定弹窗关闭后一定会调用`chain.process`或者`chain.interrupt`，便于记录状态，核心流转逻辑为：
```kotlin
override fun process() {
    if (index >= interceptors.size) {
        status.complete()
        processListener?.alterComplete()
        return
    }
    if (isInterrupted()) {
        return
    }
    if (index == 0) {
        processListener?.alterStart()
    }
    val next = copy(index + 1, interceptors)
    val pendingInterceptor = interceptors[index]
    status.process(next)
    processListener?.alterProcess(index)
    try {
        pendingInterceptor.intercept(next)
    } catch (e: Exception) {
        SCLog.i("AlterInterceptor", "chain exception: $e \n ${e.printStackTrace()}")
        //抛出异常 跳过当前节点
        next.process()
    }
}
```
正是由于在`interceptor`中调用`chain.process`，才能使下一个`interceptor`调用`intercept`，这样就会一个一个的经过`interceptor`的处理，其实也是类似于递归的思想，整个流转过程变化的是`index`变量，随着递归栈增加。`Chain`中的变量全部声明为不可变的，`copy`方法会在递归过程中创建新的`Chain`对象，并且将它的`index`增加。

我们使用`StatusHolder`来记录当前弹窗的状态和`Chain`，比如`interrupt`方法：
```kotlin
fun interrupt() {
    status.interrupt(this)
}
```
```kotlin
fun interrupt(interrupt: AlterChain) {
    if (status == STATUS_COMPLETE) return
    interruptNode = interrupt
    processingNode = null
    status = STATUS_INTERRUPT
}
```
```kotlin
 fun isInterrupted() = status == STATUS_INTERRUPT
```
```kotlin
/**
 * 尝试获取中断的节点
 */
fun findInterruptNode(): AlterChain? = status.getInterruptNode()
```
当然，`StatusHolder`实例在`copy`中是共享的，也就是整个弹框逻辑共享一个。
我们把句柄定义在`Service`模块，仅对外暴露`process`和`interrupt`方法：
```kotlin
/**
 * chain句柄
 */
interface IChain {
    /**
     * @param maybeContinue true 从上次中断处继续 false 从头开始
     */
    fun process(maybeContinue: Boolean = false)

    /**
     * 中断当前流程
     * @param maybeDismiss true 如果当前有弹框展示 则关闭
     */
    fun interrupt(maybeDismiss: Boolean = true)
}
```
在模块真正的实现中，我们通过当前弹窗状态和入参来决定继续弹窗、重新开始，还有中断弹窗:
```kotlin
class IChainImpl(private var chain: AlterChain) : IModuleAlterKitService.IChain {

    override fun process(maybeContinue: Boolean) {
        if (chain.isProcessing()) {
            Log.i(AbsAlterInterceptor.TAG, "正在弹框中...")
            return
        }
        if (chain.isInterrupted()) {
            val node = chain.findInterruptNode()
            Log.i(AbsAlterInterceptor.TAG, "流程中断过 从中断处开始? $maybeContinue 中断节点: $node")
            chain = if (node == null || !maybeContinue) chain.reset() else chain.reStart()
        }
        if (chain.isComplete()) {
            Log.i(AbsAlterInterceptor.TAG, "弹框已完成,重置...")
            chain = chain.reset()
        }
        Log.i(AbsAlterInterceptor.TAG, "继续弹框,chain: $chain")
        chain.process()
    }

    override fun interrupt(maybeDismiss: Boolean) {
        if (!chain.isProcessing()) return
        val node = chain.findProcessingNode() ?: return
        if (!maybeDismiss) {
            node.interrupt()
            return
        }
        val showingIndex = node.index - 1
        if (showingIndex !in chain.interceptors.indices) return
        val interceptor = chain.interceptors[showingIndex]
        interceptor.interrupt(node)
    }
}
```
对于`reset`和`restart`的实现，我们仍然依赖`StateHolder`这个状态持有者：
```kotlin
/**
 * 重置
 */
fun reset(): AlterChain {
    return copy(0, interceptors, status = status.apply {
        none()
    })
}
```
```kotlin
/**
 * 从断点处重新创建
 * 目的是状态重置
 */
fun reStart(): AlterChain {
    val interrupt = findInterruptNode()
    return if (interrupt == null) reset() else copy(
        interrupt.index,
        interrupt.interceptors,
        status = interrupt.status.apply {
            none()
        })
}
```
`StatusHolder`中的`none`方法就是重置状态:
```kotlin
fun none() {
    interruptNode = null
    processingNode = null
    status = STATUS_INIT
}
```

实际的弹框尽量使用`FragmentDialog`，这样就不必监听声明周期，销毁弹窗了。

## common
`common`模块存放的就是基类，通用的组件。`base`模块脱胎于`common`模块，当时想的应该是把最基础的、几乎不变的内容单独放到一个模块中，把它打包为`aar`，这样它就基本不用变动了，这有益于分模块构建流水线打包时提高效率，因为对于流水线来说，基础模块发生变动，依赖它的模块都要重新打包`aar`。
但是这种场景总是难以避免的，我们只能说少去变动基础的模块，或者，在变动的时候尽量保证Api的兼容性；如果单纯的为了避免`common aar`版本号发生变化时修改其他业务模块，可以考虑使用`compileOnly`或者版本号使用latest，但不推荐；如果壳工程中使用新版本，而其他业务模块依赖旧版本，最终还是会使用新版本的`api`（优先就近原则，这个近是指壳工程/入口），如果能确保`api`的兼容性，其他业务模块暂时不立即更新也是没问题的。但最佳实践还是定期同步版本，减少版本不一致的风险。

其他诸如`passenger-common`、`common-style`本质是用来区分业务线的，但目前多业务线都有自己的分支，已经达不到共用的目的了，所以完全可以整合一下，减少模块的个数；当然，要考虑哪些类是适合放入基础模块中的，就算仅有两个模块用到一个组件，我觉得至少比维护多个相同的类更好一些。

通用模块基本包括
 - util类
 - base类
 - base bind 类
 - base dialog
 - 通用组件

## face-recognition
 这是人脸识别模块，并非支付宝实名认证(在另一个模块中，直接集成自第三方sdk)。这是一个自定义人脸识别页面，流程大致包括摄像头的获取、预览页面的准备(`Textsure`)、图片流的读取(`ImageReader`)、字节流的转换(convertYUV420ToARGB8888)、`bitmap`的创建。
 
 字节流的转换和`bitmap`的创建是在子线程`HanderThread`中完成的。
 
 当`bitmap`创建之后会进行"预检"，"预检"是在线程池的子线程中做的，调用我们封装的`xx-face-recognition aar`：`val faceDetect = enginer.faceDetect(b)`根据返回结果`faceDetect`判断"预检"结果，包括人脸位置的判断等，只有"预检"连续通过一定次数后，才算"预检"成功。
 
 "预检"成功后，会进行活体检测，摄像头仍然在不断捕获图片，活体检测仍运行于单线程的线程池中，调用`aar`的`var pass = enginer.silentAntiFaceSpoofing(b)`方法，仅需检测一次即认为活体检测成功。
 
 活检成功后会把预检的最后一张`bitmap`上传到服务器，最终是不是实名认证成功，仍需依赖接口的返回结果，也就是说`aar`只负责活体检测，实名和图片是否对应，依赖接口的判断，最后把实名结果返回到调用出，页面退出。

## future-travel
 未来出行，这个就是纯业务模块了。包括首页图选上车点、搜索下车点列表、预估、行程中、收银台结算、订单完成等主要流程。

 首页的`activity`包含了`mapFragment`以及`homeFragment`和`estimateInfoFragment`，`mapFragment`是`home`和`estimate`共用的，。

 首页地图样式更具科技感，扎针吸附逻辑是按站点吸附，才是有效的上车点，如果指定半径内没有可吸附的站点，则可以任意拖动，但底部下单入口会被关闭；当然，能否下单有很多限制条件，比如该城市是否开通服务、是否在运营时段等等。

 地址搜索是双选模式(可以选择上车点和下车点)，下车点有搜索模式(站点列表)和图选模式，未来出行的地址搜索略有差别，地址列表由多部分组成，比方说热门区域、附近区域、推荐区域等，没有搜索关键字的时候也会展示历史区域、推荐区域等，各有逻辑，列表使用`databinding`，`item`项分为普通站点和附近站点，附近站点有详细地址。

 点击搜索结果，会校验地址，比如是否满足下单半径，是否允许下单等。

 上下车点选择完毕，将地址信息设置到`createOrderParams`，然后切换到预估页面，预估根据勾选的车级，调用接口刷新价格，当然也有定时刷新的逻辑。预估页面从UI上来说，整个背景是一个地图，上部按照上下车点聚焦，上下左右需要留出适当的间隔，确保展示到车级列表之上，车级列表是可滑动的，有的业务线会根据列表的高度动态的缩放展示区域。

勾选车级，下单之前，仍需一番校验，下单成功后，进入`trip`模块。

## hitchhike
顺风车，接入的第三方的sdk，它初始化的时机是在`Application`的`onCreate`中，使用了`IdleHander`闲时初始化：
```java
//闲时初始化顺风车以降低冷启动耗时
Looper.myQueue().addIdleHandler(() -> {
    Log.d(TAG, "init carpool in idleHandler");
    initCarpool();
    return false;
});
```
当登录app后，请求后台对应的第三方的`authCode`，调用sdk的登录方法可登录第三方业务线。

第三方sdk提供了主页面`Fragment`，我们嵌套到自己的`Activity`中即可，同时提供了根据`url`打开其他页面的能力，这在推送中打开页面非常有用，当在订单列表中打开订单详情的时候，我们使用`url`拼接`params`打开顺风车订单详情页面，其他的比如邀请页面、乘客、司机以及钱包页面等等。

提供了同步城市的方法，监听了`city`模块的城市改变，同步到顺风车首页。

第三方sdk提供了支付宝实名认证的功能，可以在项目中共用。

## login
登录模块，分为手机验证码登录、第三方授权登录和一键登录；登录模块提供了注册登录监听的方法，app中有很多地方需要监听登录状态的改变，比如最基本的登录成功后需要缓存用户登录信息，退出登录后需要清除用户的登录信息，还有些业务逻辑必须登录后才能进行。

当前用户设备如果支持一键登录(网络环境)，sdk能成功拉起一键登录页面，识别掩码的手机号，会优先打开一键登录页面，否则会打开默认的手机号验证码登录。sdk中一键登录页面是允许某种程度上的定制的，比如阿里云的能按照sdk提供的api对页面元素、位置进行部分定制。由于判断一键登录以及获取协议列表等逻辑具有一定的相关性(先获取协议，设置到sdk中)，使用一个透明的`Activity`封装拉起登录（一键登录或者普通登录）的过程，当然，要严格控制透明`Activity`的声明周期，进入透明页面后，可以注册一个`ActivityLifecycleCallbacks`，严格同步sdk中目标`Activity`的声明周期，否则，容易造成卡死的假象，操作透明页面无反应。当用户点击一键登录后，会通过sdk获取`token`，然后调用后台的登录接口，后台拿`token`去运营商换取用户信息(比如明文的手机号)，接口成功后，将用户信息缓存到本地，然后通知`listeners`，一键登录页面关闭，登录逻辑完成。

一键登录页面点击掩码手机号旁边的`edit`图标，可以切换到验证码登录，输入手机号后，点击下一步本地会对手机号做简单的正则校验，正则来自配置接口，协议勾选并且校验通过后，会打开验证码输入页面，此时调用接口发送验证码，开启倒计时，这个倒计时是本地的，可能不太严谨，只有倒计时完成后才会展示重新获取的按钮；接口调用成功后可能会触发语音验证码，就是会有来电，而不是发短信，这时候会弹窗提示用户注意接听。接口如果调用失败，可能是触发了风控/极验(`geetest`)，需要调用`riskControl`模块，它们都是`Dialog`图形验证码的形式，只不过极验是调用`sdk`，风控是自己实现的，它们是异常验证方式的两种`type`。

第三方登录放在手机号登录/验证码/一键登录的下方，支持支付宝和微信，需要接入相关sdk。因为多处使用，所以把第三方登录的方式封装成一个组合自定义View，这里要注意以下，需要手动计算item的最大宽度，不能让文字展示省略的样式，`GridLayoutManager`的`spanCount`也要动态调整。以微信为例，发起第三方登录：
```kotlin
private fun requestWeChatAuthCode() {
    state = EncryptUtils.encryptMD5ToString(UUID.randomUUID().toString()).trim()
    Log.i(TAG, "state: $state")
    val req = SendAuth.Req()
    req.scope = "snsapi_userinfo"
    req.state = this.state
    wxApi.sendReq(req)
}
```
微信sdk需要实现`IWXAPIEventHandler`接口，在`onResp`方法中，再将`response`委托到发起处，如果成功获取到了`authCode`，则调用登录接口进行登录。

设置页面还可以绑定第三方账号，其实和登录类似，都与要获取第三方的`authCode`，只是最终调用的是`casBind`接口。

踢出登录的功能，指账号互踢的操作，踢出登录的功能是统一封装到网络框架的，将与后台约定好的业务异常`errorCode`注入到网络框架，访问接口时，如果已经被踢出登录，认证网关会返回相应的`errorCode`，这个时候，业务层对网络框架注册的`setKickOffDelegate`就会被调用，业务层往消息队列中增加`Runnable`，1s后执行踢出登录的逻辑，当然，注意如果已经打开了登录页面，这个时候不需要再执行踢出了，`kickOut`会先清除本地缓存的用户数据、网关token，然后重置内存中`HttpHeader`中相关的参数如:`useId`、`token`以及`janusToken`，做完清理操作后，将`logout`的回调通知到`listeners`列表中，已经打开的页面如果监听了退出登录，可能会有相关的逻辑处理。

## multi-appoint
即多日接送业务线，下单时，选择上下车点，以及预约日期、时间、车型，是否往返，进行预估、下单，可以创建一个母订单和多个子预约单。
多日接送首页没有地图，上下车点可以通过搜索选址选择，选完上下车点后就会进入预估页面，弹窗选择日期，可选日期范围是配置的，日期选择组件使用开源组件(https://github.com/kizitonwose/Calendar)进行定制，要支持类似`IOS`的日历手势，可以拖动（横向、竖向、斜向）选择日期，也可以反选，选择的日期连接处和边缘有不同的样式。这个开源组件的优点就是将数据和表示分开，可以方便的自定义样式。
下单成功后即进入`trip`模块，大多数业务线的`trip`模块是通用的。

## near-car
这个模块就是查附近的车辆，有地图的业务线首页一般会根据扎针的经纬度查询附近车辆，然后把小车绘制到地图上。

最开始这一块逻辑并没有独立出来，后来把它独立出来，使用`LiveData`缓存接口查询(10s查询一次，无限循环)的数据，这样便于其他模块添加监听，而且`LiveData`是有声明周期感知能力的，当处于后台时，不会进行多余的车辆绘制操作。

当设置监听的时候，会传入`LifecycleOwner`、`MapControl`以及`callback`: `LifecycleOwner`用于绑定声明周期，`MapControl`用于绘制小车，并对它做动画，`callback`的作用是用来回调机场引导/气泡展示预估上车时间/异常文案，刷新扎针气泡。

根据`carService`获取不同业务线小车的图片，根据返回的数据获取小车最近的几个坐标以及唯一的`driverId`，调用`amap`的`api`绘制小车的`marker`以及动画。

## event-receiver
这个模块主要就是**长连接/getLastEvent**循环的业务调用层，长连接的真正实现是在`flutter`中，`flutter`和原生通过`channel`通信，原生的实现也封装成了一个`bridge`库，这个模块对长连接的创建和连接进一步进行了封装。

长连接的创建是在启动进程后，当然需要`flutter engine`初始化完毕后，长连接的实现其实不止`flutter websocket`一种，包括`mqtt`、`flutter socket`以及`java`层面的`Socket`，不过目前仅剩下`flutter websocket`以及`java socket`两种，后者作为一种降级方案存在，当`flutter engine`初始化失败后，会使用后者进行兜底。之所以之前很多方案都废弃了，可能是后台的要求或者所谓的稳定行不佳。确实，有些实现方案可能会在后台不易存活。

长连接是在进程启动后就创建的，不止用于行程中订单状态变更，对于`Android`来说，还是一种推送方案，比如某些手机厂商没有推送，则后台会有降级方案，在长连接连接后推送消息。

整个应用中其实有两个长连接存在，一个是应用启动时创建的，一个是IM消息长连接，并不是复用的，IM创建连接时是重新创建的`websocket connection`。IM长连接创建的时机是`flutter engine`初始化时，执行`main.dart`的过程中，进行im模块的初始化。

只有进入到`trip`模块才会开始订阅长连接，所谓的订阅其实就是判断消息的类型和消息的`orderId`是不是当前订单，除此之外还会判断`timestamp`，只有时间有变化，是较新的，才会通知到下游。

## flutter-center/flutter-impl
前者是flutter混合工程，包括flutter的入口函数、flutter sub modules的注册、http的header同步等。后者是原生调用flutter功能的入口，定义了一些flutter模块交互的方法，其实是封装了原生与flutter的交互过程，对外透明。

当App启动时，会预热flutter引擎，执行flutter-center中的`main.dart`，虽然无感知，但是futter是一直运行在后台的，当打开一个页面的时候，会把flutter和`FlutterView`绑定，这个时候才看到Flutter的内容。当再次打开一个FlutterActivity时，flutter视图会从第一个容器中脱离，与第二个容器的FlutterView绑定，由于我们的FlutterApp的特点，其实不同的Flutter页面是放在`Overlay`中的，类似一个FrameLayout，页面叠加在一起；当从第二个容器返回的时候，首先第二个Flutter页面出栈，然后会重新把当前Flutter视图关联到第一个容器的FlutterView中，页面重新展示出来。

原生与Flutter交互无非就是打开页面、调用方法，都是通过`channel`完成的，当某个Flutter业务模块向原生提供服务的时候，原生需要注册`channel`，实现`MethodCallHandler`。当然，这里是优化的，对所有的flutter与原生的交互，只需提供`native2FlutterChannel`和`flutter2NativeChannel`，具体的方法调用可以再分发。调用的时候会给到一个callback:`MethodChannel.Result`，可以返回给flutter结果。

## main
`main`模块里有几件重要的事情：设置页面、本地协议弹框/全屏协议弹框、对`Schame`的处理。
设置页面后来由原生改为flutter了，具体的设置选项是配置的，我们仍需知道配置的keys来处理不同的逻辑，并且仅解析当前版本已知的key，实现版本兼容。

本地协议弹框是应用首次打开，内容是本地的，因为要做合规，只有同意协议我们才会初始化大部分组件，包括网络框架。设置页面可以撤回协议，协议撤回后，要清理sp数据，中断原生和flutter的数据上报，然后清理任务栈，启动新的任务栈，打开全屏协议授权页面:
```kotlin
private fun restartTheWorld() {
    startActivity(
        Intent(
            this@PrivacySettingActivity,
            ProtocolActivity::class.java
        ).apply {
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        })
    finishAffinity()
}
```
全屏协议授权页面和本地授权协议弹窗类似，如果不同意，则直接退出App。
除此之外，首页每次`resume`的时候都会检查最新的协议弹窗，这个协议弹窗的内容加载的是接口返回的h5连接，底部协议列表也是后台返回的，如果不同意，则会退出App；对于其他业务线，也有对应的协议弹框，往往在业务线内部或者入口处弹出，不同意则退出业务线。

这其实就涉及到了主协议和第三方协议，这些都是可以在设置中手动撤销的，如果撤销主协议，就会走上述撤销逻辑。如果撤销第三方协议，则下次进入第三方业务线，会重新弹出协议。

解析`schame`需要使用`SchameActivity`，它在`intent-filter`中定义了`data android:scheme=`协议并声明了`action`：
```xml
<action android:name="android.intent.action.VIEW" />
<category android:name="android.intent.category.DEFAULT" />
<category android:name="android.intent.category.BROWSABLE" />
```
如果是第三方app拉起我们的应用，或者小部件点击，都会打开这个页面做中转。起初这个页面是透明的，后来为了适配Android12 splash，它的背景现在是开屏页的背景。这个页面首先判断主页面是否已经打开，没有的话会拉起主页面后，延迟到主页面处理`intent.data`这个`uri`，否则，直接处理这个`uri`，这其实是热启动和冷启动的区别。

使用`SchemeManager`处理`uri`，因为`webview`的`interceptUrl`、`push`的点击也会复用这个逻辑，基本也都是打开h5页面或者natvie页面。

## order-status
这个模块是`trip`与`event-reciever`中间层，其实封装了一系列的消息订阅过程。

## trip
trip模块即为行程，是项目中大多数业务线行程复用的模块，也是很重要的一个模块。行程中的状态大致分为：等待应答、等待接驾、司机到站、行程中、待支付、订单完成、订单取消。订单详情是一个`activity`，内部包含`fragments`，每一个`fragment`对应一种行程中的状态，当收到长连接或者轮询到的订单消息，会判断当前页面展示，切换页面；它还包含一个地图页面，用来绘制上下车点、途径点的`marker`、小车图标、以及司乘同显，`activity`提供`mapControl`的获取方法，这样，在各个状态切换的时候，`fragment`中能自主绘制内容。

等待应答有追加车级的功能，客户端会有计时，达到配置的时间就会请求可追加的车级，还有加价调度的功能，后台会推送附近可接单司机，这个是通过长连接来实现的。等待应答地图元素比较简单，只有起点`marker`和水波纹动画，但所有的页面都要注意一下聚焦问题，`zoomArray`可以指定聚焦内容，上下左右的留白，地图元素是不能被底部内容遮挡的。当然，等待应答页面还要处理超时没有任何司机响应的状况。

等待接驾页面需要聚焦司机和起点，并且绘制司乘同显，`sctx`类型需要查询接口，`sctx`本身需要配置不少东西，比如起终点、小车图片、订单号、callback等等，
