# 架构
总体来讲，项目是一种**类组件化**的结构。但也不能直接认定它是**模块化**，因为模块之间并没有直接的耦合。说它是类组件化，是因为它不具备**组件化**的很多优点，比如模块的责任/权限划分、模块形态的切换用以提高编译速度等。
项目分了很多模块，包括base模块、service模块和其他的业务、功能模块，这些模块在service模块中注册了服务，搭配`ARouter`，这为跨模块调用解耦提供了基础。
划分业务、功能模块的本意是方便多项目复用模块，但有不少模块维护的并不到位，导致不同的业务线拥有自己的分支，复用的意义已然不大；当然，这可能也跟不同业务线之间的产品需求、UI样式差距增大有关，但归根到底，肯定还是当时在应对*差距*的处理方式上偷懒了，没有在软件设计层面及时处理这些*差距*，导致积水成渊，后期在想重构一没精力，二也难以下手了。

## 组件化的好处
组件化带来的好处 就显而易见了：

- 加快编译速度：每个业务功能都是一个单独的工程，**可独立编译运行**，拆分后代码量较少，编译自然变快。
- 提高协作效率：**解耦** 使得组件之间 彼此互不打扰，组件内部代码相关性极高。 *团队中每个人有自己的责任组件，不会影响其他组件；降低团队成员熟悉项目的成本，只需熟悉责任组件即可；对测试来说，只需重点测试改动的组件，而不是全盘回归测试。*
- 功能重用：组件 类似我们引用的第三方库，只需维护好每个组件，一建引用集成即可。业务组件可上可下，灵活多变；而基础组件，为新业务随时集成提供了基础，减少重复开发和维护工作量。

## 组件化的组成
一般自下而上包括：基础组件、common组件、业务基础组件(功能组件)、业务组件、壳工程；一般都是上层依赖下层，而不能倒置；业务组件之间不能有依赖关系；最后由壳工程统一集成所有的业务组件、提供Application唯一实现、gradle、manifest配置，整合成完备的App。

## 去耦合
核心问题是 业务组件去耦合。那么存在哪些耦合的情况呢？前面有提到过，页面跳转、方法调用、事件通知。 而基础组件、业务基础组件，不存在耦合的问题，所以只需要抽离封装成库即可。 所以针对业务组件有以下问题：

- 业务组件，如何实现单独运行调试？
- 业务组件间 没有依赖，如何实现页面的跳转？
- 业务组件间 没有依赖，如何实现组件间通信/方法调用？
- 业务组件间 没有依赖，如何获取fragment实例？
- 业务组件不能反向依赖壳工程，如何获取Application实例、如何获取Application onCreate()回调（用于任务初始化）？

## 单独运行调试
我们的项目中，`module`并不能单独的调试运行，这一直都是开发体验中极差的部分，编译运行时间太长。后来，部分项目使用了单项目策略，module可以打包为aar，但并没有彻底去做这件事情，当时的目的是为了集成环境中测试打包方便，但其实已经有了module独立运行的基础。

- 使用单项目策略：
    核心就是在`gradle.properties`中定义`isModule`变量，对于每个`module`，根据这个变量，在`build.gradle`中：
    - 决定当前`module`是`application`还是`library`，`application`是独立运行的基础
    - 当是独立运行模式时，需要指定`applicationId`
    - `sourceSet.manifest`文件的位置，除了默认生成的`AndroidManifest.xml`文件外，对于独立运行的情景，需要新建`AndroidManifest.xml`，指定`launch Activity`
- 使用多项目策略:
    创建一个`app`工程，`module`作为`library`被依赖到`app`模块中，在`module`的`build.gradle`中，需要依赖`maven`或者`maven-publish`插件，目的是为了将module打包为`aar`，上传到私有仓库，这样对于壳工程来说，只需要依赖`aar`即可。`maven`和`maven-publish`的语法稍有不同，这也是`gradle`升级带来的经验。
    对于`maven`来说，打包上传`aar`的`task`如下：
    ```groovy
    android{
        //...
        uploadArchives {
            repositories {
                mavenDeployer {
                    repository(url: "http://artifactory.sc.local/artifactory/cxpt-releases") {
                        authentication(userName: 'developer', password: 'xxx')
                    }
                    //引用规则: 'groupId:artifactId:version'
                    pom.project {
                        version '1.0.0'
                        artifactId 'common'
                        groupId 'com.scmobility.module'
                        packaging 'arr'
                        description 'common lib'
                    }
                }
            }
        }
    }
    ```
    ```groovy
    android{
        //...
            afterEvaluate {
                publishing {
                    repositories {
                        maven {
                            allowInsecureProtocol(true)
                            url = "http://artifactory.sc.local/artifactory/cxpt-releases"
                            credentials {
                                username = 'developer'
                                password = 'xxx'
                            }
                        }
                    }

                    publications {
                        release(MavenPublication) {
                            //引用规则: 'groupId:artifactId:version'
                            groupId = 'com.scmobility.module'
                            artifactId = 'common'
                            version = '1.0.0'
                            description = 'common lib'
                        }
                    }
                }
        }
    }
    ```
虽然这两种方案都是可行的，但明显从职责划分、权限控制上来说，多项目的策略更具优势。

## 页面跳转
我们项目中，这一部分倒没什么大问题，我们定义了一个`service module`，用作所有业务`module`的抽象层，业务`module`之间不直接相互依赖，页面跳转、方法调用均是通过`service module`进行的。
有两点问题值得探讨：
1. 如果使用`service`提供的接口进行页面跳转，方法中需要传递`Activity`，这个时候为`module`中的页面添加路由，意义已然不大，但是如果使用路由的话，方法入参中就不需要传递`Activity`了。
2. 把所有的接口都放在`service`模块中，其实不是最好的选择，这样所有的业务模块的功能都集中在这个`module`中，大家都可以修改，仍然会有可能导致混乱。更好的做法是，拆分`service module`，将某个`module`的抽象与`module`对应起来。

模块之间页面的跳转，一般是依赖路由框架，如果是使用隐式跳转的话，也能实现我们的目的，但是无疑需要其他`module`的负责人员预先在`Androidmanifest.xml`中设置`intent-filter`。使用路由的好处，不止能用于跳转页面，还能用于后续模块之间的方法调用。
引入`ARouter`，需要在`common`模块进行`api`的方式依赖，这样其他业务模块就能间接依赖到了。需要注意的是，不止`common`模块需要声明`kapt`/`annotationProcessor`，其他使用`ARouter`的模块都需要显式声明，这样才能让`ARouter`生成路由索引文件。除此之外，还需要指定`moduleName`:
```groovy
//业务组件的build.gradle
android {
    ...
    defaultConfig {
        ...
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [AROUTER_MODULE_NAME: project.getName()]
            }
        }
    }
...
}
```
然后，在`Application`的`onCreate`中完成初始化:
```java
private void initARouter() {
    if (BuildConfig.DEBUG) {
        ARouter.openDebug();
        ARouter.openLog();
    }
    ARouter.init(BaseApplication.sharedInstance());
}
```
这样，准备工作就完成了，现在想要跨模块进行页面跳转，我们只需要在目标页面上增加`path`：
```java
@Route(path = "/cart/cartActivity")
```
然后，在待跳转的地方使用这个`path`即可：
```java
ARouter.getInstance()
    .build("/cart/cartActivity")
    .withString("key1","value1")//携带参数1
    .withString("key2","value2")//携带参数2
    .navigation();
```
当然，这些路由最好不暴露出来，因为对于外部`module`来说，它并不关心细节，这就要进一步进行封装了。

## 方法调用
对于方法调用，上边已经提到了，我们的关键点仍然是业务模块之间不直接相互依赖，那么就要求对需要提供外部访问的业务模块进行抽象，提供一个`module_export`模块，供外部模块依赖。当然，`module`也需要依赖`module_export`，实现具体的抽象方法。之所以每一个`module`都单独抽象出来`module_export`，我们上边也有提到过，是为了合理控制权限、避免导致混乱。我们仍然需要路由框架`ARouter`提供`Service`的访问：
`module_export`中声明的接口:
```java
/**
 * 购物车组件对外暴露的服务
 * 必须继承IProvider
 * @author hufeiyang
 */
public interface ICartService extends IProvider {

    /**
     * 获取购物车中商品数量
     * @return
     */
    CartInfo getProductCountInCart();
}
```
`module`中对接口的实现：
```java
/**
 * 购物车组件服务的实现
 * 需要@Route注解、指定CartRouterTable中定义的服务路由
 * @author hufeiyang
 */
@Route(path = CartRouterTable.PATH_SERVICE_CART)
public class CartServiceImpl implements ICartService {

    @Override
    public CartInfo getProductCountInCart() {
    	//这里实际项目中 应该是 请求接口 或查询数据库
        CartInfo cartInfo = new CartInfo();
        cartInfo.productCount = 666;
        return cartInfo;
    }

    @Override
    public void init(Context context) {
        //初始化工作，服务注入时会调用，可忽略
    }
}
```
我们用单独的一个类来管理路由表：
```java
public interface CartRouterTable {

    /**
     * 购物车页面
     */
    String PATH_PAGE_CART = "/cart/cartActivity";

    /**
     * 购物车服务
     */
    String PATH_SERVICE_CART = "/cart/service";

}
```
为了方便我们提供一个工具类，把路由的细节封装起来：
```java
/**
 * 购物车组件服务工具类
 * 其他组件直接使用此类即可：页面跳转、获取服务。
 * @author hufeiyang
 */
public class CartServiceUtil {

    /**
     * 跳转到购物车页面
     * @param param1
     * @param param2
     */
    public static void navigateCartPage(String param1, String param2){
        ARouter.getInstance()
                .build(CartRouterTable.PATH_PAGE_CART)
                .withString("key1",param1)
                .withString("key2",param2)
                .navigation();
    }

    /**
     * 获取服务
     * @return
     */
    public static ICartService getService(){
        //return ARouter.getInstance().navigation(ICartService.class);//如果只有一个实现，这种方式也可以
        return (ICartService) ARouter.getInstance().build(CartRouterTable.PATH_SERVICE_CART).navigation();
    }

    /**
     * 获取购物车中商品数量
     * @return
     */
    public static CartInfo getCartProductCount(){
        return getService().getProductCountInCart();
    }
}
```
`CartInfo`是一个需要放在`module_export`中对外暴露的类：
```java
/**
 * 购物车信息
 */
public class CartInfo {

    public int productCount;
}
```
其实，我们可以发现一些问题，比如，`module`单独运行测试的时候，由于`module`依赖的是其他业务组件的`export`模块，这个时候调用接口是没有真正的反馈的。有以下几种解决方案：
- 我们可以直接在多项目工程中的`app module`中依赖业务模块实现/mock，但要确保`liabrary module`仅依赖`export_module`。这样在不需要修改`library`的依赖情况下，就可以完成单独运行和发布`aar`集成运行，简洁方便。
- 也可以考虑使用依赖注入，配合`gradle`自定义变量，来决定注入的实现到底时`mock`还是真实的业务模块。
    ```groovy
    android {
        ...
        buildTypes {
            debug {
                buildConfigField "boolean", "USE_MOCK", "true"
            }
            release {
                buildConfigField "boolean", "USE_MOCK", "false"
            }
        }
    }

    if (BuildConfig.USE_MOCK) {
    // 使用Hilt或Dagger设置Mock模块
    } else {
    // 使用Hilt或Dagger设置App模块
    }
    ```
另外，还可以使用`EventBus`之类的事件总线框架在业务模块之间传递信息(注意Event实体类要定义在export_xxx中)，但我不是很喜欢`EventBus`框架，滥用会让它变成一种反设计模式的存在，最终你都分不清楚业务逻辑、调用流程是怎样的了。

## Fragment 的获取
跨模块获取`Fragment`，
- 一种方法是借助路由框架`ARouter`，需要对Fragment添加`path`注解：
```java
//添加注解@Route，指定路径
@Route(path = CartRouterTable.PATH_FRAGMENT_CART)
public class CartFragment extends Fragment {
}
```
使用`ARouter`获取实例:
```java
//使用ARouter获取Fragment实例 并添加
Fragment userFragment = (Fragment) ARouter.getInstance().build(CartRouterTable.PATH_FRAGMENT_CART).navigation();
```

- 还可以在`export`模块定义接口，比如：
```java
public interface IFragmentProvider {
    Fragment getFragment();
}
```
定义一个管理者:
```java
public class FragmentFactory {
    private static Map<String, IFragmentProvider> providers = new HashMap<>();

    public static void registerProvider(String key, IFragmentProvider provider) {
        providers.put(key, provider);
    }

    public static Fragment getFragment(String key) {
        IFragmentProvider provider = providers.get(key);
        if (provider != null) {
            return provider.getFragment();
        }
        return null;
    }
}
```
在`export`的实现`module`中实现并注册:
```java
public class NewsFragmentProvider implements IFragmentProvider {
    @Override
    public Fragment getFragment() {
        return new NewsFragment();
    }
}

// 注册
FragmentFactory.registerProvider("news", new NewsFragmentProvider());
```
其他模块获取`Fragment`:
```java
Fragment fragment = FragmentFactory.getFragment("news");
```
- 还可以使用依赖注入
在`export`模块中定义`Provider`接口:
```kotlin
// IFragmentProvider.kt
interface IFragmentProvider {
    fun getFragment(): Fragment
}
```
在业务实现模块中实现它:
```kotlin
// NewsFragmentProvider.kt in News Module
class NewsFragmentProvider : IFragmentProvider {
    override fun getFragment(): Fragment {
        return NewsFragment()
    }
}
```
在壳工程中，定义`Hilt`模块，用于提供IFragmentProvider的依赖：
```kotlin
// AppModule.kt
@Module
@InstallIn(SingletonComponent::class)
abstract class AppModule {

    @Binds
    @IntoMap
    @StringKey("news")
    abstract fun bindNewsFragmentProvider(newsFragmentProvider: NewsFragmentProvider): IFragmentProvider
}
```
在使用的地方，依赖注入：
```kotlin
@Inject
lateinit var fragmentProviders: Map<String, @JvmSuppressWildcards IFragmentProvider>

val newsFragment = fragmentProviders["news"]?.getFragment()
```
**Fragment方法的调用**
如果想要调用`Fragment`的方法，可以定义接口，然后在使用的地方向上转型，进行方法调用。实际上，外部模块不应该知道`Fragment`的细节的，如果有这样的需求，可能要考虑下设计是否合理了；但通过接口调用，已经尽可能的解耦了。

## Application生命周期分发
组件化中，壳工程依赖业务模块，依赖是不能倒置的，那么，业务模块应该如何监听`Application`的`onCreate`呢？可以使用`Lifecycle`插件: https://github.com/hufeiyang/Android-AppLifecycleMgr
步骤如下：
- common组件依赖 applifecycle-api
- 业务组件依赖applifecycle-compiler、实现接口+注解
    ```java
    /**
     * 组件的AppLifecycle
    * 1、@AppLifecycle
    * 2、实现IApplicationLifecycleCallbacks
    */
    @AppLifecycle
    public class CartApplication implements IApplicationLifecycleCallbacks {

        public  Context context;

        /**
        * 用于设置优先级，即多个组件onCreate方法调用的优先顺序
        * @return
        */
        @Override
        public int getPriority() {
            return NORM_PRIORITY;
        }

        @Override
        public void onCreate(Context context) {
            //可在此处做初始化任务，相当于Application的onCreate方法
            this.context = context;

            Log.i("CartApplication", "onCreate");
        }

        @Override
        public void onTerminate() {
        }

        @Override
        public void onLowMemory() {
        }

        @Override
        public void onTrimMemory(int level) {
        }
    }
    ```
- 壳工程中引入`Plugin`，并且在`Application`中分发事件
   ```java
   //壳工程 MyApplication
    public class MyApplication extends Application {

        @Override
        public void onCreate() {
            super.onCreate();

            ...

            ApplicationLifecycleManager.init();
            ApplicationLifecycleManager.onCreate(this);
        }

        @Override
        public void onTerminate() {
            super.onTerminate();

            ApplicationLifecycleManager.onTerminate();
        }

        @Override
        public void onLowMemory() {
            super.onLowMemory();

            ApplicationLifecycleManager.onLowMemory();
        }

        @Override
        public void onTrimMemory(int level) {
            super.onTrimMemory(level);

            ApplicationLifecycleManager.onTrimMemory(level);
        }
    }
   ``` 
   如果使用`ARouter`的话，其实还可以考虑一个地方，那就是`ARouter`的ServiceImpl初始化的地方，其实是`IProvider`的`init`方法，仅供参考。

## 总结
综上，目前的项目虽说没有完全的实现组件化，但其实已经做了大部分的工作，包括业务组件、功能组件的划分，不同组件之间的路由、方法调用，这些都很好的符合和规范；项目欠缺的一是独立打包运行和集成测试，这其中其实包含了职责、权限的划分，建议使用多工程模式；一是对`Service module`的处理，目前是所有的业务模块的抽象接口都放到了`Service module`中，每个人都能修改这个模块，容易导致混乱以及职责划分不清的问题，其实也可以再次改造，为需要提供服务的`module`抽象出`export_module`以解决这个问题；整体进一步改造的难度不大。日常需求迭代中，要严格遵守组件化的思想和规范，多考虑一下设计和合理性，让组件化的架构可以持续。

# Jetpack
项目中使用了部分jetpack组件，但也没有严格按照google给出的最佳变成实践去做，主要是`ViewBinding/DataBinding`+`ViewModel`+`Repository`+`RequestFactory`。
项目中大量使用了`DataBinding`，无论是列表还是某组件的信息展示，列表主要依赖开源库(https://github.com/evant/binding-collection-adapter)，普通的组件就是`ObservableField`/`ObservableBoolean`..或者`LiveData`/`MutableLiveData`