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

