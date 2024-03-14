# 嵌入流程

项目中有不少使用`Native View`的业务场景，比如地图、视频播放组件，扫码组件等，这其实都属于在`Flutter`布局中嵌入Native组件。

简单来说，Flutter里面想要嵌入原生组件，有固定的步骤：
1. 创建一个`View`，实现`PlatformView`接口，在`View getView()`方法中提供`PlatformView`;
2. 创建一个`Factory`，继承`PlatformViewFactory`，在`create(context: Context?, viewId: Int, params: Any?)`方法中进行`PlatformView`的初始化;
3. 创建一个`Plugin`，实现`FlutterPlugin`，在`onAttachedToEngine`中注册`Factory`: `registerViewFactory`;
4. 这样，在`Flutter`端，通过`Platform`比如`AndroidView(viewTypeId:)`就能引用到`PlatformView`;

两端通信的方式有两种：
- 可以在创建的时候传参初始化
- 可以在使用的时候通过MethodChannel进行方法调用

# 地图
## MapView
首先看一下MapView的声明，它实现了`PlatformView`、`MethodCallHandler`以及`Lifecycle`等接口：
```kotlin
class SCMapView(context: Context?, lifecycleProvider: LifecycleProvider) : PlatformView, MethodCallHandler,
    ActivityPluginBinding.OnSaveInstanceStateListener, DefaultLifecycleObserve
```
在 `init`中，创建了`mapView`的实例，配置了基本的高德`mapview?.map?`事件监听，并且设置了`methodCallHandler`的回调。
`PlatformView`的`getView`方法中，返回了创建的mapView:
```kotlin
override fun getView(): View? {
        return mapview
}
```
`onMethodCall`方法中，注册了大量的操作地图的方法，便于flutter侧对原生组件的控制:
```kotlin
override fun onMethodCall(call: MethodCall, result: MethodChannel.Result) {
        val method = call.method
        val wrapper = MethodResultWrapper(result)
        val args = call.arguments as Map<String, Any?>
        SCLog.d(TAG, method + "    " + JsonUtil.gson.toJson(args));
        when (method) {
            "sc/MapUserLocation:showUserLocation" -> {
                MapUtil.getInstance().showUserLocation(args, wrapper)
            }
            "sc/MapUserLocation:setUserTrackingMode" -> {
                MapUtil.getInstance().setUserTrackingMode(args, wrapper)
            }
            "sc/MapUserLocation:getUserLocation" -> {
                MapUtil.getInstance().getUserLocation(args, wrapper)
            }
            "sc/MapView:setMapType" -> {
                MapUtil.getInstance().setMapType(args, wrapper)
            }
            "sc/MapView:getZoomLevel" -> {
                MapUtil.getInstance().getZoomLevel(args, wrapper)
            }
            "sc/MapView:reLocation" -> {
                // 归位
                MapUtil.getInstance().reLocation(args, wrapper)
            }
            //....
        }
}
```

以`reLocation`为例，原生端的实现如下：
```kotlin
// 归位
public void reLocation(Map<String, Object> args, MethodResultWrapper methodResult) {
    int refId = (int) ((Map<String, Object>) args).get("refId");
    double zoomLevel = 14;
    try {
        if (args.containsKey("zoomLevel")) {
            zoomLevel = (double) args.get("zoomLevel");
        }
    } catch (Exception e) {
        zoomLevel = 14;
    }
    TextureMapView mapView = (TextureMapView) getHEAP().get(refId);
    AMap aMap = mapView.getMap();
    aMap.setMyLocationEnabled(true);
    if (aMap.getMyLocation() == null || aMap.getMyLocation().getLatitude() < 0.01) {
        methodResult.success(true);
    } else {
        LatLng latLng = new LatLng(aMap.getMyLocation().getLatitude(), aMap.getMyLocation().getLongitude());
        aMap.moveCamera(CameraUpdateFactory.changeLatLng(latLng));
//            aMap.moveCamera(CameraUpdateFactory.zoomTo((float) zoomLevel));
        methodResult.success(true);
    }
}
```
首先，flutter传过来`method`和`args`，代表将要调用的方法和传过来的参数，`refId`是flutter端传过来的指代`PlatformView`的唯一标志，`HEAP`其实就是一个`map`，是用来保存原生组件的；这样，拿到原生组件后，调用原生/组件相关的方法，来最终完成flutter端功能的调用，这里调用重新定位的方法，最终调用的是原生组件`MapView`中关联的`AMap`的`moveCamera`方法，将视野移动到用户当前所在的位置。

`MethodResultWrapper`其实是对`MethodChannel.Result`的包装，它在**主线程**中把原生的结果回调到flutter。

## MapViewFactory
`MapViewFactory`的声明如下:

```kotlin
class SCMapViewFactory(createArgsCodec: MessageCodec<Any?>, private val lifecycleProvider: LifecycleProvider) :
    PlatformViewFactory(createArgsCodec)
```
它的`create`方法中创建了`PlatformView`，并根据初始化参数`params`初始化了`MapView`，比如设置了部分`MapView`的属性以及`AMap`的`style`、`listener`等，然后将原生组件保存到`HEAP`中，方便查找、统一管理资源，最后将`MapView`返回：
```kotlin
override fun create(context: Context?, viewId: Int, params: Any?): PlatformView {
    SCLog.d(TAG, " PlatformView create 传入的id: $viewId  , params = $params")

    val scMapView = SCMapView(context, lifecycleProvider)
    val mapView = scMapView.view as? TextureMapView

    val aMap = mapView?.map

    if (params is Map<*, *>) {
        val uiSettings = aMap?.uiSettings
        uiSettings?.setLogoBottomMargin(-80)

        params["mapType"]?.let {
            aMap?.mapType = it as Int
        }
        //...
    }

    initMarkerClickListeners(aMap!!)

    HEAP[Int.MAX_VALUE - viewId] = mapView!!
    Log.d(TAG, " android getHEAP().put 的id: " + (Int.MAX_VALUE - viewId))
    return scMapView
}
```

## MapPlugin
`MapPlugin`完成了原生组件的注册，它实现了`FlutterPlugin`：

```kotlin
public class SCMapPlugin implements FlutterPlugin,ActivityAware
```
在`onAttachedToEngine`方法中，完成了`Factory`的注册：
```kotlin
 @Override
public void onAttachedToEngine(FlutterPluginBinding binding) {
    Log.d(TAG, "onAttachedToEngine");
    MapChannel.setMapChannel(new MethodChannel(binding.getBinaryMessenger(), CHANNEL_MAPVIEW));

    scMapService = new SCMapService(binding.getApplicationContext(), binding.getBinaryMessenger());
    scMapSearch = new SCMapSearch(binding.getApplicationContext(), binding.getBinaryMessenger());
    scMapGeoFence = new SCMapGeoFence(binding.getApplicationContext(), binding.getBinaryMessenger());
    binding.getPlatformViewRegistry().registerViewFactory("sc.flutter/SCMapView", new SCMapViewFactory(new StandardMessageCodec(), () -> lifecycle));
}
```
在`onDetachedFromEngine`方法中，反注册了`channel`调用：
```kotlin
@Override
public void onDetachedFromEngine(@NonNull FlutterPluginBinding binding) {
    Log.d(TAG, "onDetachedFromEngine");
    scMapService.release();
    scMapService = null;
    scMapSearch.release();
    scMapSearch = null;
    scMapGeoFence.release();
    scMapGeoFence = null;
    if (null != MapChannel.getMapChannel()) {
        MapChannel.getMapChannel().setMethodCallHandler(null);
    }
}
```

`MapService`、`MapSearch`以及`MapGeoFence`实现了`MethodChannel.MethodCallHandler`，是分别处理地图、搜索以及围栏相关方法调用的，它们内部分别注册了`channel`:

```kotlin
init {
    mapServiceChannel = MethodChannel(messenger, "sc/map/service")
    mapServiceChannel?.setMethodCallHandler(this)
}

fun release() {
    mapServiceChannel?.setMethodCallHandler(null)
    mapServiceChannel = null
    context = null
}
```

## AndroidView
flutter端可以直接声明组件:
```dart
SCMapView(
    maskDelay: const Duration(milliseconds: 500),
    zoomLevel: 12.5,
    rotateGestureEnabled: false,
    showZoomControl: false,
    showUserLocation: false,
    useAndroidViewSurface: false,
    //surfaceView在单独的窗口渲染，会导致页面遮挡问题
    userLocationImage: SCImageWithConfig(
        AssetImage(R.home_point, package: R.module_name),
        createLocalImageConfiguration(context, size: const Size(32, 30)),
    ),
    centerCoordinate: ChargeCityManager.instance.location,
    customMapStyleDataFileName: '${ChargeManager.instance.mapData}.data',
    customMapStyleExtraDataFileName: '${ChargeManager.instance.mapData}_extra.data',
    onMapCreated: (controller) async {
        mapManager = ChargeMapManager(controller, context);
        ref.read(mapProvider.notifier).state = controller;
        if (Platform.isIOS) {
        controller.setZoomLevel(14, true,
            minZoomLevel: 1, maxZoomLevel: 19);
        }
        _initMap();
    },
),
```
`SCMapView`是一个`StatefulWidget`，事实上，它是对`PlatformView`的包装:

```dart
 @override
  Widget build(BuildContext context) {
    if (Platform.isAndroid) {
      return Stack(
        children: <Widget>[
          RepaintBoundary(key: _markerKey, child: _widgetLayer),
          com_amap_api_maps_MapView_Android(
            useAndroidViewSurface: widget.useAndroidViewSurface,
            creationParams: getCreateParams(),
            onViewCreated: (refId) async {
              _controller = SCMapController.create(refId);
              if (widget.onMapCreated != null) {
                await widget.onMapCreated!(_controller!);
              }
            },
          ),
          _mask,
        ],
      );
    } else if (Platform.isIOS) {
      return Stack(
        children: <Widget>[
          RepaintBoundary(key: _markerKey, child: _widgetLayer),
          SCMapView_iOS(
            onViewCreated: (refId) async {
              _controller = SCMapController.create(refId);
              await _initIOS();
              if (widget.onMapCreated != null) {
                await widget.onMapCreated!(_controller!);
              }
            },
          ),
          _mask,
        ],
      );
    } else {
      return Center(child: Text('未实现的平台'));
    }
  }
```
`com_amap_api_maps_MapView_Android`组件的`build`方法如下：

`static const viewType = 'sc.flutter/SCMapView';`

```dart
 @override
  Widget build(BuildContext context) {
    if (widget.useAndroidViewSurface == true) {
      return PlatformViewLink(
        viewType: viewType,
        surfaceFactory: (context, controller) {
          return AndroidViewSurface(
            controller: controller as AndroidViewController,
            gestureRecognizers: gestureRecognizers!,
            hitTestBehavior: PlatformViewHitTestBehavior.opaque,
          );
        },
        onCreatePlatformView: (PlatformViewCreationParams params) {
          return PlatformViewsService.initExpensiveAndroidView(
            id: params.id,
            viewType: viewType,
            layoutDirection: TextDirection.ltr,
            creationParams: widget.creationParams,
            creationParamsCodec: const StandardMessageCodec(),
            onFocus: () => params.onFocusChanged(true),
          )
            ..addOnPlatformViewCreatedListener(params.onPlatformViewCreated)
            ..addOnPlatformViewCreatedListener(_onViewCreated)
            ..create();
        },
      );
    }

    return AndroidView(
      viewType: viewType,
      gestureRecognizers: gestureRecognizers,
      onPlatformViewCreated: _onViewCreated,
      creationParamsCodec: const StandardMessageCodec(),
      creationParams: widget.creationParams,
    );
  }
```
可以看到，根据是否使用`surface`，使用的`Widget`是不一样的，一个是`AndroidViewSurface`，一个是`AndroidView`。`creationParams`是初始化参数，创建`PlatformView`的时候可以使用这些参数初始化。
当`PlatformView`创建完毕后，会调用`_onViewCreated`方法，如下：
```dart
void _onViewCreated(int id) {
    // 碰到一个对象返回的hashCode为0的情况, 造成和这个id冲突了, 这里用一个magic number避免一下
    int refId = 2147483647 - id;
    if (widget.onViewCreated != null) {
      widget.onViewCreated!(refId);
    }
}
```
他会进一步回调到上边的`SCMapView`的`build`方法中，在这个回调中创建了`_controller`:
`_controller = SCMapController.create(refId);`
这个`SCMapController`就是封装了flutter和native的相互调用，我们可以看到，flutter声明了与native互调用的channels:
```dart
final  MethodChannel mapChannel = MethodChannel("sc/MapView");

final  MethodChannel serviceChannel = MethodChannel("sc/map/service");

final  MethodChannel geoFenchChannel = MethodChannel("sc/map/GeoFench");
```

比方说，上边重新定位的方法，在`_controller`中进行了定义：

```dart
/// 归位
Future<bool> reLocation(double zoomLevel) async {
    return await mapChannel.invokeMethod("sc/MapView:reLocation", {"zoomLevel": zoomLevel, "refId": refId});
}
```

这样，通过channel调用，flutter就调用到了native，控制原生组件的行为方式。