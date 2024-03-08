# Flutter混合工程
首先，创建一个文件目录module-flutter-center，它并不是flutter工程，而是包含Android/Ios的一个工程，在这个目录下创建Flutter工程：flutter_center_dart。
这里的原生工程所做的事情，主要处理了Engine的预热、定义了一套原生与Flutter交互规范。比如路由，openFlutterPage、pushNativePage、sendEventToFlutter、sendEventToNative等。这里就是原生的实现部分，也就是说，最终这一部分会作为一个module包含在项目中，对于Android来讲，settings.gradle中include了这部分原生代码：
```groovy
include ':module-flutter-center'
project(':module-flutter-center').projectDir = new File('module-flutter-center/android/app')
```
flutter-center_dart是我们创建的Flutter工程，需要在settings.gradle中执行`include_flutter.groovy`脚本，这个脚本的主要作用是自动化地处理一些必要的配置步骤，以便将Flutter模块正确地嵌入到Android项目中：
```groovy
setBinding(new Binding([gradle: this]))//为了include_flutter.groovy脚本能访问到当前gradle环境变量
evaluate(new File(
        rootProject.projectDir,
        './module-flutter-center/flutter_center_dart/.android/include_flutter.groovy'
))
```

# 编译原理
## Android编译产物
Android侧Flutter编译后的产物主要有：
- libflutter.so：flutter引擎的C++代码编译产物。
- libapp.so：flutter业务与框架的Dart代码编译产物，内部由四部分组成。（flutter 1.7.8之后不再产生isolate_snapshot_data、isolate_snapshot_instr、vm_snapshot_data、vm_snapshot_instr四部分，转而将其打包成一个so。）
- flutter.jar：flutter引擎的Java代码编译产物。
- flutter_assets：包括图片、字体、LISENSE等静态资源。

## 打包流程
前面我们提到需要在settings.gradle中执行`include_flutter.groovy`脚本，我们说一说flutter集成过程中都干了什么事情。
1. Android工程集成Flutter
    - 在.android/local.properties中增加了flutter sdk的路径
        sdk.dir=/Users/Library/Android/sdk
        flutter.sdk=/Users/.flutter_sdk
    - .android/Flutter/build.gradle的功能主要是
        - 从local.properties中读取flutter sdk path
        - 将flutter.gradle脚本导入当前build.gradle中
        - 配置Flutter工程路径
        ```groovy
        //获取flutter sdk路径、versionCode、VersionName
        def flutterRoot = localProperties.getProperty('flutter.sdk')
        def flutterVersionCode = localProperties.getProperty('flutter.versionCode')
        def flutterVersionName = localProperties.getProperty('flutter.versionName')

        //导入flutter.gradle，脚本位于flutter_tools目录下
        apply from: "$flutterRoot/packages/flutter_tools/gradle/flutter.gradle"

        //配置Flutter工程路径，为parent: Flutter目录
        flutter {
            source '../..'
        }
        ```
2. 加入Flutter依赖
flutter.gradle的作用主要是Flutter工程编译和将flutter工程依赖的jar和so，打包到APK中。flutter.gradle有两大核心部分：FlutterPlugin和FlutterTask。
```groovy
apply plugin: FlutterPlugin

class FlutterPlugin implements Plugin<Project> {

@Override
  void apply(Project project) {}
}

abstract class BaseFlutterTask extends DefaultTask {}

class FlutterTask extends BaseFlutterTask {}

```
  - FlutterPlugin
    FlutterPlugin是一个Gradle插件，负责将Flutter集成到现有的Android项目中：
    - 项目配置：自动配置项目以支持Flutter构建。这包括设置正确的依赖项、配置构建路径以及确保Flutter SDK的路径被正确设置。
    - 任务注入：向项目的构建流程中注入特定的构建任务（即FlutterTask实例）。这些任务负责执行具体的构建步骤，如Dart代码的编译、资源的打包等。
    - 环境准备：确保所有的构建前提条件都满足，包括检查Flutter环境是否就绪，以及是否所有需要的工具和依赖都已经安装。
    - 平台特定的配置：对于不同的平台（Android/iOS），FlutterPlugin会进行特定的配置，以确保Flutter代码可以无缝地在这些平台上运行和构建。
  - FlutterTask
    FlutterTask是定义在flutter.gradle脚本中的一个或多个Gradle任务，它们是具体执行构建步骤的实体。每一个FlutterTask都是为了完成特定的构建活动而设计的，例如：
    - 编译Dart代码：将Dart代码编译成原生平台代码的任务。
    - 打包资源：将图片、字体等资源打包到应用程序中的任务。
    - 生成桥接代码：生成允许Flutter与原生代码交互的桥接代码的任务。
    - 构建APK或IPA：在Android或iOS项目中分别构建APK或IPA文件的任务。
**总的来说，FlutterPlugin负责配置和准备项目的构建环境，并且向项目的构建流程中注入特定的构建任务。而FlutterTask则是执行实际构建步骤的工作单元，负责将Flutter代码和资源转换为最终的应用程序包。**

## 包大小优化
了解了Flutter产物之后，我们就可以分析下如何优化包的大小，包大小优化的核心思想就是三个：删减、压缩、动态下发。
1. 删减就是删除无用代码，无用资源。Flutter本身就会消除未被调用的函数（Flutter在构建应用程序时会尝试进行树摇（tree shaking），以移除未使用的代码），要想继续精简只能从业务方面入手。flutter本身**并不会检测未使用的资源**，这些资源最终会被打包到apk中，所以删减未使用的资源、重复的资源是很重要的优化点儿。flutter_assets中的LICENSE许可证文件在运行时用不到，可以删除，删除后可以节省包体积约60KB。删除过程统一可以通过自定义一个gradle plugin来完成。对于字体资源，可以统一使用google_fonts库来加载，删除本地的字体资源。
2. 压缩是把一些图片之类的资源压缩，在使用时再解压。图片资源本身也可以进一步进行压缩，比方说使用svg(flutter_svg)、webp或者直接压缩png，图片资源本身也可以考虑只使用一套比方说只使用2x的(虽然这可能导致在非常高密度的屏幕上图片看起来不够清晰，或者在低密度屏幕上资源占用更多内存，但它是一种有效减少应用包大小的方法。)。还可以考虑将资源换成网络资源，而不是直接打包到apk中。资源文件本身也可以进一步进行压缩上传到服务器，然后动态下发。
3. 动态下发，就是放到服务端，等到APK启动后再按需下载。libflutter.so、libapp.so和flutter_assets.zip都可以进行动态下发，需要自定义gradle task，hook到so以及assets的生成，然后上传到服务端，然后在应用启动的时候校验资源的版本，判断是否需要下载，并且要有flutter不可用的降级方案。对于Flutter产物的动态加载，大致有两种方案：
- 修改so加载路径并重新编译Flutter引擎
  - 通过Flutter引擎源码可以得知FlutterLoader中startInitialization方法就是加载libflutter.so的地方，这里Flutter采用System.loadLibrary加载的libflutter.so，无法修改so路径，所以我们反射ClassLoader的pathList和nativeLibraryDirectories，把自定义的libflutter.so路径添加进去，需要注意下不同版本ClassLoader源码不太一样，需要分多个版本处理反射。
  - 通过分析发现FlutterLoader中的ensureInitializationComplete方法就是加载libapp.so的地方，其中shellArgs存放的是Flutter的libapp.so的路径，我们在原有路径之前添加了自定义的路径，这样就可以加载到下发的libapp.so
  - 通过分析发现Flutter资源的加载是在platform_view_android_jni.cc的RunBundleAndSnapshotFromLibrary方法中，我们在方法队列头部新增一个DirectoryAssetBundle去加载自定义的Flutter资源路径
- 重写Flutter加载类，并反射Flutter初始化变量

关于动态下发so，可参考
- https://blog.csdn.net/Android_Programmer/article/details/135340532
- https://juejin.cn/post/6950641450916773902

# 启动流程
通过上述flutter的构建产物分析得知，engine的java代码在flutter.jar中。as的三方依赖库中，查看io.flutter:flutter_embedding_release:version的源代码。
FlutterApplication是flutter提供的一个Application，用于进行应用级的全局初始化：
```java
public class FlutterApplication extends Application {
  @Override
  @CallSuper
  public void onCreate() {
    super.onCreate();
    FlutterInjector.instance().flutterLoader().startInitialization(this);
  }

  private Activity mCurrentActivity = null;

  public Activity getCurrentActivity() {
    return mCurrentActivity;
  }

  public void setCurrentActivity(Activity mCurrentActivity) {
    this.mCurrentActivity = mCurrentActivity;
  }
}
```
FlutterInjector是一个注入器，可以方便设置FlutterJNI以及FlutterLoader，这两个组件是加载libflutter.so和libapp.so的核心，我们可以创建CustomFlutterJNI来修改加载这两个so的路径，然后注入到FlutterInjector中。`startInitialization`方法中加载了libflutter.so：
```java
public void startInitialization(@NonNull Context applicationContext, @NonNull Settings settings) {
    // Do not run startInitialization more than once.
    if (this.settings != null) {
      return;
    }
    if (Looper.myLooper() != Looper.getMainLooper()) {
      throw new IllegalStateException("startInitialization must be called on the main thread");
    }

    TraceSection.begin("FlutterLoader#startInitialization");
    try {
      // Ensure that the context is actually the application context.
      final Context appContext = applicationContext.getApplicationContext();

      this.settings = settings;

      initStartTimestampMillis = SystemClock.uptimeMillis();
      flutterApplicationInfo = ApplicationInfoLoader.load(appContext);

      VsyncWaiter waiter;
      if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1 /* 17 */) {
        final DisplayManager dm =
            (DisplayManager) appContext.getSystemService(Context.DISPLAY_SERVICE);
        waiter = VsyncWaiter.getInstance(dm, flutterJNI);
      } else {
        float fps =
            ((WindowManager) appContext.getSystemService(Context.WINDOW_SERVICE))
                .getDefaultDisplay()
                .getRefreshRate();
        waiter = VsyncWaiter.getInstance(fps, flutterJNI);
      }
      waiter.init();

      // Use a background thread for initialization tasks that require disk access.
      Callable<InitResult> initTask =
          new Callable<InitResult>() {
            @Override
            public InitResult call() {
              TraceSection.begin("FlutterLoader initTask");
              try {
                ResourceExtractor resourceExtractor = initResources(appContext);

                flutterJNI.loadLibrary();
                flutterJNI.updateRefreshRate();

                // Prefetch the default font manager as soon as possible on a background thread.
                // It helps to reduce time cost of engine setup that blocks the platform thread.
                executorService.execute(() -> flutterJNI.prefetchDefaultFontManager());

                if (resourceExtractor != null) {
                  resourceExtractor.waitForCompletion();
                }

                return new InitResult(
                    PathUtils.getFilesDir(appContext),
                    PathUtils.getCacheDirectory(appContext),
                    PathUtils.getDataDirectory(appContext));
              } finally {
                TraceSection.end();
              }
            }
          };
      initResultFuture = executorService.submit(initTask);
    } finally {
      TraceSection.end();
    }
}
```
源码中我们可以得知一些信息：
- 必须在主线程中调用`startInitialization`，且只会执行一次
- `ApplicationInfoLoader`解析了`AndroidManifest.xml`文件中的部分元数据，主要是一些flutter和VM的配置信息
- 初始化了`VsyncWaiter`，这是一个用于处理垂直同步的类，确保flutter的UI更新频率与设备屏幕的刷新率一致
- 启动一个后台任务，主要涉及资源的提取和准备工作，以确保Flutter引擎能够高效地运行，同时优化应用的启动时间。其中，`flutterJNI.loadLibrary()`就是加载libflutter.so的地方。

看一下FlutterActivity的onCreate，这个方法创建了FlutterActivityAndFragmentDelegate，`onAttach`方法中，设置了flutterEngine：
```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
  switchLaunchThemeForNormalTheme();

  super.onCreate(savedInstanceState);

  delegate = new FlutterActivityAndFragmentDelegate(this);
  delegate.onAttach(this);
  delegate.onRestoreInstanceState(savedInstanceState);

  lifecycle.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);

  configureWindowForTransparency();

  setContentView(createFlutterView());

  configureStatusBarForFullscreenFlutterExperience();
}
```
```java
void onAttach(@NonNull Context context) {
    ensureAlive();

    if (flutterEngine == null) {
      setupFlutterEngine();
    }

    if (host.shouldAttachEngineToActivity()) {
      Log.v(TAG, "Attaching FlutterEngine to the Activity that owns this delegate.");
      flutterEngine.getActivityControlSurface().attachToActivity(this, host.getLifecycle());
    }

    platformPlugin = host.providePlatformPlugin(host.getActivity(), flutterEngine);

    host.configureFlutterEngine(flutterEngine);
    isAttached = true;
  }
```
`setupFlutterEngine`的逻辑是先从cache中获取engine，再从provideFlutterEngine方法中获取，最后创建一个engine对象，源代码如下：
```java
void setupFlutterEngine() {
    Log.v(TAG, "Setting up FlutterEngine.");

    // First, check if the host wants to use a cached FlutterEngine.
    String cachedEngineId = host.getCachedEngineId();
    if (cachedEngineId != null) {
      flutterEngine = FlutterEngineCache.getInstance().get(cachedEngineId);
      isFlutterEngineFromHost = true;
      if (flutterEngine == null) {
        throw new IllegalStateException(
            "The requested cached FlutterEngine did not exist in the FlutterEngineCache: '"
                + cachedEngineId
                + "'");
      }
      return;
    }

    // Second, defer to subclasses for a custom FlutterEngine.
    flutterEngine = host.provideFlutterEngine(host.getContext());
    if (flutterEngine != null) {
      isFlutterEngineFromHost = true;
      return;
    }

    // Our host did not provide a custom FlutterEngine. Create a FlutterEngine to back our
    // FlutterView.
    Log.v(
        TAG,
        "No preferred FlutterEngine was provided. Creating a new FlutterEngine for"
            + " this FlutterFragment.");
    flutterEngine =
        new FlutterEngine(
            host.getContext(),
            host.getFlutterShellArgs().toArray(),
            /*automaticallyRegisterPlugins=*/ false,
            /*willProvideRestorationData=*/ host.shouldRestoreAndSaveState());
    isFlutterEngineFromHost = false;
}
```
如果进行了engine预热，并且跳转FlutterActivity的时候传递了cacheEnginede的id，那么，此时就能取出预热的engin对象：
```kotlin
val intent = FlutterActivity.CachedEngineIntentBuilder(FlutterDialogActivity::class.java, FlutterContanst.KEY_ENGINE_CACHE)
              .backgroundMode(FlutterActivityLaunchConfigs.BackgroundMode.transparent)
              .destroyEngineWithActivity(false)
              .build(act)
```
```java
@NonNull
public Intent build(@NonNull Context context) {
  return new Intent(context, activityClass)
      .putExtra(EXTRA_CACHED_ENGINE_ID, cachedEngineId)
      .putExtra(EXTRA_DESTROY_ENGINE_WITH_ACTIVITY, destroyEngineWithActivity)
      .putExtra(EXTRA_BACKGROUND_MODE, backgroundMode);
}
```
```java
@Override
@Nullable
public String getCachedEngineId() {
  return getIntent().getStringExtra(EXTRA_CACHED_ENGINE_ID);
}
```
Engine对象初始化的时候，会确保libflutter.so和libapp.so被正确加载：
```java
if (!flutterJNI.isAttached()) {
    flutterLoader.startInitialization(context.getApplicationContext());
    flutterLoader.ensureInitializationComplete(context, dartVmArgs);
}
```
首次执行该方法，这个判断为false，上边说到，`startInitialization`只会被执行1次，所以会执行`ensureInitializationComplete`:
```java
String kernelPath = null;
if (BuildConfig.DEBUG || BuildConfig.JIT_RELEASE) {
  String snapshotAssetPath =
      result.dataDirPath + File.separator + flutterApplicationInfo.flutterAssetsDir;
  kernelPath = snapshotAssetPath + File.separator + DEFAULT_KERNEL_BLOB;
  shellArgs.add("--" + SNAPSHOT_ASSET_PATH_KEY + "=" + snapshotAssetPath);
  shellArgs.add("--" + VM_SNAPSHOT_DATA_KEY + "=" + flutterApplicationInfo.vmSnapshotData);
  shellArgs.add(
      "--" + ISOLATE_SNAPSHOT_DATA_KEY + "=" + flutterApplicationInfo.isolateSnapshotData);
} else {
  shellArgs.add(
      "--" + AOT_SHARED_LIBRARY_NAME + "=" + flutterApplicationInfo.aotSharedLibraryName);

  // Most devices can load the AOT shared library based on the library name
  // with no directory path.  Provide a fully qualified path to the library
  // as a workaround for devices where that fails.
  shellArgs.add(
      "--"
          + AOT_SHARED_LIBRARY_NAME
          + "="
          + flutterApplicationInfo.nativeLibraryDir
          + File.separator
          + flutterApplicationInfo.aotSharedLibraryName);

  // In profile mode, provide a separate library containing a snapshot for
  // launching the Dart VM service isolate.
  if (BuildConfig.PROFILE) {
    shellArgs.add(
        "--" + AOT_VMSERVICE_SHARED_LIBRARY_NAME + "=" + VMSERVICE_SNAPSHOT_LIBRARY);
  }
}
```
在这个方法中，设置了`libapp.so`的加载路径。如果我们需要自定义`libapp.so`的加载路径，可以重写`FlutterJNI`，在args中添加自定义路径。

当调用`Engine`预热时，执行了入口函数，如果此时不执行，在`FlutterActivity`的`onStart`方法中也会执行：
```java
void onStart() {
    ...
    doInitialFlutterViewRun();
    ...
}
```
```java
private void doInitialFlutterViewRun() {
    ...
    // Configure the Dart entrypoint and execute it.
    DartExecutor.DartEntrypoint entrypoint =
        libraryUri == null
            ? new DartExecutor.DartEntrypoint(
                appBundlePathOverride, host.getDartEntrypointFunctionName())
            : new DartExecutor.DartEntrypoint(
                appBundlePathOverride, libraryUri, host.getDartEntrypointFunctionName());
    flutterEngine.getDartExecutor().executeDartEntrypoint(entrypoint, host.getDartEntrypointArgs());
}
```
这个方法最终会从java中的`FlutterJNI`调用到native的`DartIsolate`：
```c++
[[nodiscard]] bool DartIsolate::Run(const std::string& entrypoint_name,
                                    const std::vector<std::string>& args,
                                    const fml::closure& on_run) {
  if (phase_ != Phase::Ready) {
    return false;
  }

  tonic::DartState::Scope scope(this);

  auto user_entrypoint_function =
      Dart_GetField(Dart_RootLibrary(), tonic::ToDart(entrypoint_name.c_str()));

  auto entrypoint_args = tonic::ToDart(args);

  if (!InvokeMainEntrypoint(user_entrypoint_function, entrypoint_args)) {
    return false;
  }

  phase_ = Phase::Running;

  if (on_run) {
    on_run();
  }
  return true;
}
```
这里首先将 `Isolate` 的状态设置为 `Running` 接着调用 `dart` 的 `main` 方法。这里 `InvokeMainEntrypoint` 是执行 `main` 方法的关键：

```c++
[[nodiscard]] static bool InvokeMainEntrypoint(
    Dart_Handle user_entrypoint_function,
    Dart_Handle args) {
  ...
  Dart_Handle start_main_isolate_function =
      tonic::DartInvokeField(Dart_LookupLibrary(tonic::ToDart("dart:isolate")),
                             "_getStartMainIsolateFunction", {});
  ...
  if (tonic::LogIfError(tonic::DartInvokeField(
          Dart_LookupLibrary(tonic::ToDart("dart:ui")), "_runMainZoned",
          {start_main_isolate_function, user_entrypoint_function, args}))) {
    return false;
  }
  return true;
}
```

参考：
- https://juejin.cn/post/7010655914025811975#heading-4
- https://www.kancloud.cn/petermx/android/2357006

# Engine预热
FlutterActivity默认会创建它自己的FlutterEngine，每个FlutterEngine会有一个明显的预热时间，这意味着加载一个标准的FlutterActivity时，会有一个短暂的延迟，想要最小化这个延迟时间，可以在启动FlutterActivity之前，初始化一个FlutterEngine，然后用这个已经预热好的FlutterEngine。
在Application中，会执行FlutterEngine的预热，代码位置是在上述的原生部分module-flutter-center中，`FlutterCenter#initFlutter`：
```kotlin
 fun initFlutter(context: Context, delegate: PushPageDelegate?, callback: Replay<Boolean>?) {
        var engine = FlutterEngineCache.getInstance().get(FlutterContanst.KEY_ENGINE_CACHE)
        if (null == engine) {
            // 初始化一个FlutterEngine.
            engine = FlutterEngine(context)
            // 配置初始路由
            engine.navigationChannel.setInitialRoute("/")
            // 执行dart入口函数，这里是main.dart
            engine.dartExecutor.executeDartEntrypoint(DartExecutor.DartEntrypoint.createDefault())
            // Cache the FlutterEngine to be used by FlutterActivity.
            FlutterEngineCache.getInstance().put(FlutterContanst.KEY_ENGINE_CACHE, engine)
        }

        //native<-->flutter交互的api
        native2FlutterApi = NativeToFlutterRouter(engine.dartExecutor.binaryMessenger, delegate)
        flutter2NativeApi = FlutterToNativeRouteApi(engine.dartExecutor.binaryMessenger)

        manager = FlutterChannelManager(engine.dartExecutor.binaryMessenger)
        // 当监听到自定义event：xxx://FlutterCenter/initFinished的时候，初始化完毕的回调会触发
        manager?.addFlutterEngineInitListener(IFlutterEngineInitListener { callback?.replay(true) })
    }
```

