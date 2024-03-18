# Jetpack
项目中使用了部分jetpack组件，但也没有严格按照google给出的最佳实践去做，主要是`ViewBinding/DataBinding`+`ViewModel`+`Repository`+`RequestFactory`。
项目中大量使用了`DataBinding`，无论是列表还是某组件的信息展示，列表主要依赖开源库(https://github.com/evant/binding-collection-adapter)，普通的组件就是`ObservableField`/`ObservableBoolean`..或者`LiveData`/`MutableLiveData`。

## DataBinding和ViewModel的初始化
base类声明如下：
```kotlin
abstract class BaseBindFragment<V : ViewDataBinding, M : BaseBindViewModel>() : BaseFragment(){

    //...
    constructor(brId: Int, vmcls: Class<M>) : this() {
        this.brId = brId
        this.vmcls = vmcls
    }
}
```
在`fragment`的`onCreateView`方法里面初始化：
```kotlin
override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        binding = DataBindingUtil.inflate(inflater, getLayoutId(), container, false)
        binding.lifecycleOwner = viewLifecycleOwner
        vmcls?.let { clazz ->
            // viewModel共享需绑定Activity的生命周期，如果不共享viewModel,则绑定Fragment自身生命周期
            viewModel = if (isShareViewModel()) {
                VMProviderUtils.getViewModel(mActivity, clazz)
            } else {
                VMProviderUtils.getViewModel(this, clazz)
            }
            // 绑定Event事件
            bindEvent(viewModel)
            // 绑定viewModel到布局文件
            binding.setVariable(brId, viewModel)
        }
        Log.d(TAG, "初始化binding" + if (::binding.isInitialized) "成功" else "失败")
        return binding.root
}
```
`VMProviderUtil`其实就是使用`ViewModelProvider`实例化`ViewModel`:
```kotlin
fun <T : ViewModel> getViewModel(@NonNull owner: ViewModelStoreOwner, cls: Class<T>, instance: (() -> T)? = null): T {
        return if (instance == null) {
            ViewModelProvider(owner).get(cls)
        } else {
            ViewModelProvider(owner, object : ViewModelProvider.Factory {
                override fun <T : ViewModel> create(modelClass: Class<T>): T {
                    return instance() as T
                }
            }).get(cls)
        }
}
```
这样一来，就初始化了`binding`和`viewModel`对象，并且将`brId`绑定到xml中的`viewModel`中。

## ViewModel
我们创建的`viewModel`可以选择继承自`BaseBindViewModel`或者`BaseBindRepositoryViewModel<T : BaseRepository>`，两者的区别就是需不需要创建一个`Repository`，后者其实也是前者的子类。

### BaseBindViewModel
`BaseBindViewModel`中主要做的事情就是创建了三个`LiveData`，这个算是一个优化，把与UI的交互统一封装起来：
```kotlin
// 页面状态
private val status = EventLiveData<PageStatus>()

// 页面事件
private val event = EventLiveData<PageEvent>()

// 页面跳转
private val intent = EventLiveData<PageIntent>()


fun uiPageEvent(eventName: String, eventData: Any? = null, callback: ((data: Any?) -> Unit)? = null) {
    event.postValue(PageEvent(eventName, eventData, callback))
}
```
比如，当需要在`viewModel`中改变UI状态时:
```kotlin
uiPageEvent(EVENT_SHOW_TIP)
```
`EventLiveData`是`MutableLiveData`的子类：
```kotlin
class EventLiveData<T> : MutableLiveData<T>() {

    override fun observe(owner: LifecycleOwner, observer: Observer<in T>) {
        LiveEventObserver.bind(this, owner, observer)
    }

    override fun postValue(value: T) {
        // 兼容未注册事件观察者前，发送事件，则接收不到
        // Activity可以在super.onCreate(savedInstanceState)之后，或initData()方法中发送事件
        // Fragment可以在super.onCreateView(inflater, container, savedInstanceState)之后，或onViewCreated、initData()方法中发送事件
        if (hasObservers()) { // 当前LiveDate有订阅者才发送postValue
            LiveDataUtils.postValue(this, value)
        } else { // 没有订阅者，清除value
            clearValue()
            SCLog.e("EventLiveData", ExceptionUtil.getTrace(Exception("请在绑定bindEvent后发送Event事件")))
        }
    }

    /**
     * 清除当前event事件
     */
    fun clearValue() {
        if (isMainThread()) {
            value = null
        } else {
            super.postValue(null)
        }
    }
}
```
它重写的`postValue`方法里面判断了有没有观察者，没有则清理数据，相当于某种程度上避免了**粘性**，这可能是考量到`Event`的实时性；`LiveDataUtils.postValue`其实就是判断一下线程，来决定调用`setValue`还是`postValue`；`observe`方法相当于把监听、发送数据托管给了`LiveEventObserver.bind`:
```kotlin
fun <T> bind(@NonNull liveData: LiveData<T>,
                    @NonNull owner: LifecycleOwner,
                    @NonNull observer: Observer<in T>) {
    if (owner.lifecycle.currentState == Lifecycle.State.DESTROYED) {
        return
    }
    LiveEventObserver(liveData, owner, observer)
}
```
这里创建了`LiveEventObserver`，然后把我们的`EventLiveData`、`LifecycleOwner`以及`Observer`传入：

```kotlin
init {
    mOwner?.lifecycle?.addObserver(this)
    mLiveData?.observeForever(this)
}
```
`LiveEventObserver`实现了`LifecycleObserver`和`Observer`，这说明它能监听生命周期、监听`LiveData`的数据变化，正如上边的代码的调用，`LiveEventObserver`相当于一个处于上游`LiveData`和下游`Observer`之间的**中介者**。当监听到`EventLiveData`中的数据流时会判断UI组件的生命周期是否处于`onStart`和`onPause`之间(`Lifecycle.State.STARTED`)，这个状态即为活跃状态：当是活跃状态时，中介收到上游的数据会直接发送给下游；当是非活跃状态时，会先把数据存储到`mPendingData`列表中，等监听到生命周期发生变化，即变成活跃状态后，再把消息逐个发送到下游；它使用`mPendingData`避免了`LiveData`默认的行为：只能**粘性**一条数据。它解决了**数据丢失**的问题。代码如下:
```kotlin
override fun onChanged(t: T?) {
    try {
        // 是否是激活状态，即 onStart 之后到 onPause 之前
        val isActive = mOwner?.lifecycle?.currentState?.isAtLeast(Lifecycle.State.STARTED)
                ?: false
        Log.d(TAG, "isActive = $isActive")
        if (isActive) {
            // 如果是激活状态，就直接更新
            mObserver?.onChanged(t)
        } else {
            // 非激活状态先把数据存起来
            mPendingData.add(t)
        }
    } catch (e: Exception) {
        // 上报异常堆栈日志
        Log.d(TAG, "事件接收异常:" + ExceptionUtil.getTrace(e))
    }
}
```
```kotlin
/**
 * onStart 之后就是激活状态了，如果之前存的有数据，就发送出去
 */
@OnLifecycleEvent(Lifecycle.Event.ON_ANY)
fun onEvent(owner: LifecycleOwner, event: Lifecycle.Event) {
    Log.d(TAG, "onEvent")
    if (owner != mOwner) {
        return
    }
    if (event == Lifecycle.Event.ON_START || event == Lifecycle.Event.ON_RESUME) {
        for (i in mPendingData.indices) {
            mObserver?.onChanged(mPendingData[i])
        }
        mPendingData.clear()
    }
}
```
当监听到组件销毁时，会进行资源释放，包括之前对`LiveData`使用的`observeForever`方法，需要手动注销监听:
```kotlin
 /**
  * onDestroy 时解除各方的观察和绑定，并清空数据
  */
@OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
fun onDestroy() {
    Log.d(TAG, "onDestroy")
    mLiveData?.removeObserver(this)
    mLiveData = null

    mOwner?.lifecycle?.removeObserver(this)
    mOwner = null

    mPendingData.clear()

    mObserver = null
}
```

### BaseBindRepositoryViewModel
这是一个泛型类，类的声明为：`open class BaseBindRepositoryViewModel<T : BaseRepository> : BaseBindViewModel()`，`T`即为当前`viewModel`需要绑定的`Repository`类:

```kotlin
val mRepository: T? by lazy { DUtil.getNewInstance<T>(this, 0) }
```
`mRepository`是个懒加载对象，它是通过泛型类反射创建出来的实例。这么做的优点是自动注入，缺点是反射调用的构造函数是无参的默认构造函数。按照`jetpack`的最佳实践，`Repository`的数据来源是创建对象时注入进来的，可能包括网络、数据库等，`Repository`仅是一个提供数据的抽象层而已，不在构造函数中注入，那就默认数据来源层是写死的单例对象了，这其实不利于`Repository`的测试、复用。

### 不合理的地方
其实`BaseBindViewModel`我认为有一点不合理的地方，它声明了`getActivity`的方法：
```kotlin
fun getActivity(): BaseActivity? = activityCallback?.invoke()
```
按照`jetpack`的最佳实践来讲，UI层是单向依赖`viewModel`层的，现在如果在`viewModel`中暴露了`activity`，这是有安全隐患的，`activity`有泄漏的风险，这就好像又回到了`MVP`的时代。

## Repository
上边已经提到过`Repository`的一些缺点，还有一点，项目中在设计`ViewModel`、`Repository`的时候，复用很少；一方面确实，如果业务差异较大，不好复用，那反过来想，是不是也存在职责划分不明确，不单一的情况存在呢？`ViewModel`在`Repository`的上层，与UI关联更紧密，难以复用是可以理解的，但`Repository`属于数据抽象层，项目中不应该存在大量的`Repository`，否则就会使项目变得臃肿，开发体验较差。

由于网络框架的特性，基于RxJava，会返回`Disposable`，所以`BaseRepository`存在一个`CompositeDisposable`对象管理这些网络请求；上边提到，`BaseBindRepositoryViewModel`默认会创建`mRepository`对象，在`onCleared`方法中，会把`BaseRepository`的`CompositeDisposable`清理掉:

```kotlin
override fun onCleared() {
    super.onCleared()
    if (mRepository != null) {
        mRepository?.unSubscribe()
    }
}
```
```java
 public void unSubscribe() {
    if (compositeDisposable != null) {
        compositeDisposable.clear();
    }
}
```

正如上边提到的，`Repository`的数据提供者并不是构造方法中注入的，而是作为一个单例，直接在`Repository`中使用的。这其实让`Repository`这一层的存在必要性变得单薄，这可能也与数据来源单一性有关。一个使用示例如下:
```kotlin
fun checkOriginAddress(
    originPoint: AddressPoint,
    carUseType: Int,
    appIdBody: String?,
    httpCallback: SCHttpCallback<SCCheckLocationResponse>
) {
    val request = SCCheckLocationRequest().apply {
        this.originPoint = SCCheckLocationRequest.GtwAddressPointObject().apply {
            lat = originPoint.lat
            lon = originPoint.lon
            cityCode = originPoint.cityCode
            address = originPoint.address
            addressDesc = originPoint.addressDesc
            poiId = originPoint.poiId
            poiType = originPoint.poiType
        }
        this.poiType = this.originPoint.poiType ?: ""
        // 用车方式：0-普通、1-接机、2-送机、3-包车
        this.carUseType = carUseType
        //1：校验起点，2：校验终点，3：校验多点
        this.checkType = 1
        // 企业单校验地址appId
        this.appIdBody = appIdBody
        this.protocol = ProtocolConstants.MATCH_PROTOCOL
    }
    SCLog.d(TAG, "校验上车点参数 = ${JsonUtil.toJson(request)}")
    addSubscribe(RequestFactory.INSTANCE.checkLocation(request, this, httpCallback))
}
```
事实上我还是觉得有点别扭，按理来说，`Repository`更关注不同数据来源之间的关系，而不应该是大段代码都在构造`Request`，如何构造请求是网络数据源关心的事情。

## RequestFactory
`jetpack`里应该没有这个组件，它是网络请求，属于数据来源`DataSource`的一种，名字叫`Factory`其实和工厂没有半毛钱关系，项目中它往往是一个单例，它所做的事情是构造并发起网络请求：
```java
 public Disposable checkLocation(SCCheckLocationRequest checkLocationRequest, SCRequestAccessory requestAccessory, SCHttpCallback<SCCheckLocationResponse> httpCallback) {
    String fullPath = SCEnvironmentManager.sharedInstance().getGtwUrl("integrate") + "/integrate/v1/p/order/checklocation";
    SCRequestWrapper requestWrapper = new SCRequestWrapper(fullPath, SCRequestMethod.POST);
    requestWrapper.setRequestEntity(checkLocationRequest);
    SCHttpRequest request = new SCHttpRequest();
    request.addAccessory(requestAccessory);
    return request.sendRequest(requestWrapper, httpCallback)
            .subscribe();
}
```
由于是封装的网络框架，这里使用了`RxJava`和`callback`，与`jetpack`+`kotlin`显得有点出戏。当然，如果使用`kotlin`协程，完全可以把callback改造成顺序调用代码的形式。一般一个业务模块仅包含一个`RequestFactory`，提高了`RequestFactory`的复用性。

# 生命周期

## ViewModel和UI层是单向持有关系
由上边提到过的，在`ViewModel`中获取了`Context(Activity)`会有导致内存泄漏的风险，因为`ViewModel`的生命周期是比`Activity`的生命周期长的，当旋转屏幕的时候，`Activity`会销毁重建，这个时候`ViewModel`仍然持有旧的`Activity`的生命周期，导致内存泄漏。所以，`ViewModel`中是绝对不能持有UI层的引用的，它只能是一种单向的关系，由UI层持有`ViewModel`。
如果确实需要`Context`访问数据，比如访问`SharePrefrence`中的数据，一般有以下几种方案：
- 将动作委托到UI层。
- 使用`AndroidViewModel`:`AndroidViewModel`提供了一种安全获取应用级`Context`的方法。这个`Context`的生命周期绑定到应用本身，因此不会因为`Activity`的销毁而导致内存泄漏。
- 自行获取`ApplicationContext`：其实核心问题在于，长生命周期的对象不能持有短生命周期的对象，所以使用`app`的`Context`完全没有问题。
- 使用`Repository`：即使`ViewModel`本身不直接使用`ApplicationContext`，你也可以在数据层（如`Repository`）中使用`ApplicationContext`，例如访问`SharedPreferences`或数据库。`ViewModel`通过与`Repository`交互来完成这些操作，而无需直接访问`Context`。

## ObservableFiled vs LiveData
`ObservableXxx`一般用于`databinding xml`布局中，当设置数据时，自动刷新UI，而不关心当前页面的生命周期；它也可以设置属性改变监听器，但是它不能监听生命周期自动注销监听器，这个需要注意。
`LiveData`也同样能用于xml，它和`ObservableXxx`最大的区别就是`LiveData`能感知生命周期的变化，只有当生命周期处于**活跃(STARTED)**状态时，才会更新UI。`LiveData`也常用于UI层需要监听数据变化，做出一些动作，而UI层的`Observer`并不需要关系什么时候注销监听器。
`LiveData`具有**粘性**的特质，但它只能保存最新的一条数据，有的时候不需要订阅的时候就收到之前发的粘性数据，有的时候需要确保在非活跃状态时发送的数据序列能保存下来，这两种场景上边自定义`LiveData`都有处理。
当这两者绑定到xml中时，并不需要关注生命周期的问题，这属于`databinding`自身的生命周期管理问题，在`binding`初始化的时候，会为其绑定生命周期:`binding.lifecycleOwner = viewLifecycleOwner`。

# 自定义Observable
创建数据实体，继承`BaseObservable`，实现`get/set`方法，然后编译。会通过`APT`生成一个`BR`类。然后在`get`方法上用`@Bindable`标注，在`set`方法内调用`notifyPropertyChanged()`，`notifyPropertyChanged()`方法就是用来通知属性数据变化的:
```java
public class Book extends BaseObservable {
    
    private String name;
    private String author;
    
    public Book(String name, String author) {
        this.name = name;
        this.author = author;
    }
    
    @Bindable
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
        notifyPropertyChanged(BR.name);
    }
    
    @Bindable
    public String getAuthor() {
        return author;
    }
    
    public void setAuthor(String author) {
        this.author = author;
        notifyPropertyChanged(BR.author);
    }
}
```

# Databinding

## 使用
对于普通xml布局，AS可以快捷方便的生成`databinding layout`源码，xml中的<data></data>部分使用`variable`或者`import`引入绑定对象。在布局中使用`@{}`或者`@={}`进行单向绑定或者双向绑定。对于不支持的属性或者需要大段代码计算的逻辑需要自定义`BindingAdapter`。

## BindingAdapter
- 自动选择方法：比如android:text="@{user.name}"为例，库会去自动的查找setText方法，并且setText方法的参数，是user.name的类型的参数：
    ```xml
    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@{user.name}" />
    ```
- @BindingMethods: 用于当 View 中的某个属性与其对应的 setter 方法名称不对应时进行映射:
    ```kotlin
    @BindingMethods(value = [
        BindingMethod(type = androidx.appcompat.widget.AppCompatTextView::class,attribute = "android:custom_text",method = "showCustomToast")
    ])
    class CustomTextView :AppCompatTextView{

        constructor(context: Context):this(context,null,0)
        constructor(context: Context, attrs: AttributeSet?):this(context,attrs,0)
        constructor(context: Context, attrs: AttributeSet?, defStyleAttr:Int=0):super(context,attrs,0)

        fun showCustomToast(text:String){
            Toast.makeText(context,text,Toast.LENGTH_SHORT).show()
        }
    }
    ```
- @BindingAdapter: BindingAdapter一般有两种场景，一种是对已有的属性需要做一些自定义的逻辑处理。一种是完全自定义的属性，自定义的逻辑:
    ```kotlin
    @JvmStatic
    @BindingAdapter(value=["imageurl","errorD","placeholderD"],requireAll = false)
    fun loadImage(imageView: ImageView, imageUrl: String,errorDrawable:Drawable,placeholderDrawble:Drawable) {
        Glide.with(imageView.context).load(imageUrl).apply(RequestOptions().error(errorDrawable).placeholder(placeholderDrawble)).into(imageView)
    }
    ```
    这里绑定了多个属性，`requireAll=false`表明这些属性是可选的。

- BindingConversion： 可以对数据，或者类型进行转换：
    ```kotlin
    @BindingConversion
    @JvmStatic
    fun conversionToDrawable(text: String):Drawable{
        return when(text){
            "红色"->ColorDrawable(Color.RED)
            else->ColorDrawable(Color.WHITE)
        }
    }
    ```
    ```xml
     <TextView
        android:id="@+id/tv_teacher_name"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        android:text="CTest"
        android:background="@{`红色`}"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"/>
    ```
    `android:background="@{红色}"`本来这种代码肯定是会报错的，因为`background`不能接收`String`类型，使用了`BindingConversion`可以将`String`类型转换为`Drawable`。

## 原理
`DataBinding`使用`APT`工具生成代码，帮助开发者在数据源发生变化时，自动调用更新数据的方法，最终执行数据更新的地方是`APT`生成的`XxxBindingImpl`中的`executeBindings`方法中。
1. 首先，databinding向绑定的可观察的属性注册`propertyChangedCallback`
2. 当数据源发生变化时，会调用`notifyPropertyChanged`方法，通知`callbacks`数据已经发生变化
3. 经过一系列判断与调用，最终会调用`XxxBindingImpl`的`executeBindings`方法
4. 这个方法中最终执行属性更新的逻辑，比如`TextView`的`setText`方法

```java
protected void executeBindings() {
    long dirtyFlags = 0L;
    synchronized(this) {
        dirtyFlags = this.mDirtyFlags;
        this.mDirtyFlags = 0L;
    }

    HistoryFlightBean item = this.mItem;
    OnItemClickListener<?> listener = this.mListener;
    String ItemFlightNumber1 = null;
    String itemFlightAddress = null;
    if ((dirtyFlags & 5L) != 0L && item != null) {
        ItemFlightNumber1 = item.getFlightNumber();
        itemFlightAddress = item.getFlightAddress();
    }

    if ((dirtyFlags & 5L) != 0L) {
        TextViewBindingAdapter.setText(this.itemFlightNumber, ItemFlightNumber1);
        TextViewBindingAdapter.setText(this.itemName, itemFlightAddress);
    }

    if ((dirtyFlags & 4L) != 0L) {
        this.layoutFront.setOnClickListener(this.mCallback1);
    }

}
```

## 双向绑定为何不会死循环
根本原因是，更新数据时，判断了`oldValue`与`data`是否相等，如果相等，则不更新。

## 序列化
有的时候，我们可能需要对绑定的对象进行序列化，注意使用多个`LiveData`的场景，可能会报错，这个时候可以添加`@Transient`注解忽略字段。

## BindingCollectionAdapter
这是个开源库：https://github.com/evant/binding-collection-adapter 。项目中有大量用于`RecyclerView`的数据绑定，就体验来说还是非常方便的，不过你要弄明白它的规则。

#### 基本使用
使用的时候一般我们在ViewModel中声明`itemBinding`绑定类和`items`数据集合：
```kotlin
val itemData = ObservableArrayList<BaseItem>()
```
数据集合是一个`ObservableArrayList`，这意味着它的内容的改变会自动触发UI更新；
不过有时候依赖这一点可能不不是我么想要的效果，我们有点时候期望自己调用`adapter`的非全量更新方法。

```kotlin
val onItemBind: OnItemBindClass<BaseItem> = OnItemBindClass<BaseItem>()
    .map(PoiActivityLooperItemBean::class.java, BR.poiActivity, R.layout.soa_item_poi_activity)
    .map(HomeWorkBean::class.java){binding,position,_->
        binding.clearExtras().set(BR.homework, R.layout.soa_item_home_work)
    }
    .map(HistoryItem::class.java){binding,position,_->
        binding.clearExtras().set(BR.hisitem, R.layout.soa_item_history)
            .bindExtra(BR.itemWidth,itemWidth)
    }
    .map(AddressItem::class.java) { itemBinding, position, _ ->
        itemBinding.clearExtras().set(BR.poiitem, R.layout.soa_item_address)
            .bindExtra(BR.position, position)
            .bindExtra(BR.mapPointsListener, switchMapWindowListener)
            .bindExtra(BR.itemDecoration, subItemDecoration)
    }
    .map(AirportItem::class.java, BR.airitem, R.layout.soa_item_airport)
    .map(TipItem::class.java, BR.tipitem, R.layout.soa_item_tip)
    .map(DefaultItem::class.java, BR.defaultItem, R.layout.soa_item_default)
```
这里其实是`RecyclerView`中不同的`itemType`做了`bean`和`item_layout`的绑定，`OnItemBindClass.map`可用于多布局，如果是单布局也可使用`ItemBinding.of`:

```kotlin
val items = ObservableArrayList<SCPoiItem>()
val itemBinding = ItemBinding.of<SCPoiItem>(BR.subitem, R.layout.soa_item_address_sub)
    .bindExtra(BR.listener, listener)
    .bindExtra(BR.expend, showSubPoiTag)
    .bindExtra(BR.subPoiTagListener, subPoiTagListener)
```
也可以声明`adapter`:`val adapter = BindingRecyclerViewAdapter<BaseItem>()`，xml中使用：

```xml
<!--地址列表-->
<androidx.recyclerview.widget.RecyclerView
    android:id="@+id/rv_poi"
    app:adapter="@{viewModel.adapter}"
    app:itemBinding="@{viewModel.onItemBind}"
    app:items="@{viewModel.itemData}"
    android:layout_width="match_parent"
    android:layout_height="0dp"
    android:layout_weight="1"
    android:overScrollMode="never"
    android:scrollbars="none"
    app:layoutManager="androidx.recyclerview.widget.LinearLayoutManager"
    tools:itemCount="3"
    tools:listitem="@layout/soa_item_address" />
```

### 源码分析
首先，`app:adapter&app:itemBinding&app:items`肯定都是库提供的自定义绑定属性，查看`BindingCollectionAdapters`:
```java
@BindingAdapter(value = {"itemBinding", "items", "adapter", "itemIds", "viewHolder", "diffConfig"}, requireAll = false)
public static <T> void setAdapter(RecyclerView recyclerView,
                                    ItemBinding<? super T> itemBinding,
                                    List<T> items,
                                    BindingRecyclerViewAdapter<T> adapter,
                                    BindingRecyclerViewAdapter.ItemIds<? super T> itemIds,
                                    BindingRecyclerViewAdapter.ViewHolderFactory viewHolderFactory,
                                    AsyncDifferConfig<T> diffConfig) {
    if (itemBinding != null) {
        BindingRecyclerViewAdapter oldAdapter = (BindingRecyclerViewAdapter) recyclerView.getAdapter();
        if (adapter == null) {
            if (oldAdapter == null) {
                adapter = new BindingRecyclerViewAdapter<>();
            } else {
                adapter = oldAdapter;
            }
        }
        adapter.setItemBinding(itemBinding);

        if (diffConfig != null && items != null) {
            AsyncDiffObservableList<T> list = (AsyncDiffObservableList<T>) recyclerView.getTag(R.id.bindingcollectiondapter_list_id);
            if (list == null) {
                list = new AsyncDiffObservableList<>(diffConfig);
                recyclerView.setTag(R.id.bindingcollectiondapter_list_id, list);
                adapter.setItems(list);
            }
            list.update(items);
        } else {
            adapter.setItems(items);
        }

        adapter.setItemIds(itemIds);
        adapter.setViewHolderFactory(viewHolderFactory);

        if (oldAdapter != adapter) {
            recyclerView.setAdapter(adapter);
        }
    } else {
        recyclerView.setAdapter(null);
    }
}

@BindingConversion
public static <T> AsyncDifferConfig<T> toAsyncDifferConfig(DiffUtil.ItemCallback<T> callback) {
    return new AsyncDifferConfig.Builder<>(callback).build();
}
```
除了可以绑定`RecyclerView`，还提供了`AdapterView`、`ViewPager`、`ViewPager2`以及`paging`等组件的绑定方法，原理基本都大同小异。
我们可以看到，这里声明了多个属性绑定，但都是可选的，但`itemBinding`属性是使用这个库的必要属性，当没有声明`adapter`属性时，会自动创建`BindingRecyclerViewAdapter`；`AsyncDifferConfig`可以搭配`AsyncListDiffer`使用，用于高效地处理列表数据的差异计算和更新；如果设置了`diffConfig`，则会使用`AsyncDiffObservableList`的`update`方法刷新数据，否则调用`adapter`的`setItems`方法；最后，为`recyclerView`设置`adapter`。
`BindingRecyclerViewAdapter`实现了`BindingCollectionAdapter`接口：
```java
public interface BindingCollectionAdapter<T> {
    void setItemBinding(@NonNull ItemBinding<? super T> itemBinding);

    @NonNull
    ItemBinding<? super T> getItemBinding();

    void setItems(@Nullable List<T> items);

    T getAdapterItem(int position);

    @NonNull
    ViewDataBinding onCreateBinding(@NonNull LayoutInflater inflater, @LayoutRes int layoutRes, @NonNull ViewGroup viewGroup);

    void onBindBinding(@NonNull ViewDataBinding binding, int variableId, @LayoutRes int layoutRes, int position, T item);
}
```
`setItemBinding`即设置`item`和`layout`的绑定关系，`setItems`即设置数据集合，如果数据集是`ObservableList`，则会自动触发UI的更新，如果是普通的`List`，则需要手动调用`adapter`的更新方法；`onCreateBinding`其实就是将layout inflate出来，生成绑定视图；`onBindBinding`其实就是为item绑定数据。
`setItems`方法如下：
```java
@Override
public void setItems(@Nullable List<T> items) {
    if (this.items == items) {
        return;
    }
    // If a recyclerview is listening, set up listeners. Otherwise wait until one is attached.
    // No need to make a sound if nobody is listening right?
    if (recyclerView != null) {
        if (this.items instanceof ObservableList) {
            ((ObservableList<T>) this.items).removeOnListChangedCallback(callback);
            callback = null;
        }
        if (items instanceof ObservableList) {
            callback = new WeakReferenceOnListChangedCallback<>(this, (ObservableList<T>) items);
            ((ObservableList<T>) items).addOnListChangedCallback(callback);
        }
    }
    this.items = items;
    notifyDataSetChanged();
}
```
如果设置的list是一个`ObservableList`，则会为其增加监听，`WeakReferenceOnListChangedCallback`继承了`ObservableList.OnListChangedCallback`，持有了`adapter`和`items`的弱引用，当`ObservableList`的内容发生变化时，会触发回调，比如：
```java
@Override
public void onItemRangeChanged(ObservableList sender, final int positionStart, final int itemCount) {
    BindingRecyclerViewAdapter<T> adapter = adapterRef.get();
    if (adapter == null) {
        return;
    }
    Utils.ensureChangeOnMainThread();
    adapter.notifyItemRangeChanged(positionStart, itemCount);
}
```
可以看到，这里根据`ObservableList`变化类型调用`adapter`对应的更新方法，而不止是简单的`notifyDataSetChanged`。

我们看一下`adapter`的`onCreateViewHolder`方法：
```java
@NonNull
@Override
public final ViewHolder onCreateViewHolder(@NonNull ViewGroup viewGroup, int layoutId) {
    if (inflater == null) {
        inflater = LayoutInflater.from(viewGroup.getContext());
    }
    ViewDataBinding binding = onCreateBinding(inflater, layoutId, viewGroup);
    final ViewHolder holder = onCreateViewHolder(binding);
    binding.addOnRebindCallback(new OnRebindCallback() {
        @Override
        public boolean onPreBind(ViewDataBinding binding) {
            return recyclerView != null && recyclerView.isComputingLayout();
        }

        @Override
        public void onCanceled(ViewDataBinding binding) {
            if (recyclerView == null || recyclerView.isComputingLayout()) {
                return;
            }
            int position = holder.getAdapterPosition();
            if (position != RecyclerView.NO_POSITION) {
                try {
                    notifyItemChanged(position, DATA_INVALIDATION);
                } catch (IllegalStateException e) {
                    // noop - this shouldn't be happening
                }
            }
        }
    });
    return holder;
}
```
这里首先创建了`ItemView`的`binding`以及`ViewHolder`，创建binding其实就是调用了`DataBindingUtil.inflate(inflater, layoutId, viewGroup, false);`，创建`ViewHolder`时如果不提供`viewHolderFactory`，则默认创建一个`BindingViewHolder`:
```java
private static class BindingViewHolder extends ViewHolder {
    BindingViewHolder(ViewDataBinding binding) {
        super(binding.getRoot());
    }
}
```
接着，为`bingding`增加了`addOnRebindCallback`，`onPreBind`表示是否需要重新绑定，这里只有在`recyclerView`正在重新布局的时候才需要重新绑定数据。`onCanceled`应该是取消绑定时回调，这个时候会调用`notifyItemChanged`刷新item布局。

我们看一下`adapter`的`onBindViewHolder`:
```java
@Override
@CallSuper
public void onBindViewHolder(@NonNull ViewHolder holder, int position, @NonNull List<Object> payloads) {
    ViewDataBinding binding = DataBindingUtil.getBinding(holder.itemView);
    if (isForDataBinding(payloads)) {
        binding.executePendingBindings();
    } else {
        T item = items.get(position);
        onBindBinding(binding, itemBinding.variableId(), itemBinding.layoutRes(), position, item);
    }
}
```

`isForDataBinding`其实判断的是`onCreateViewHolder`中对binding的监听`addOnRebindCallback`发出来的`notifyItemChanged(position, DATA_INVALIDATION);`消息，也就是说，非布局情况下的binding被取消后，`onBindViewHolder`会手动执行`binding.executePendingBindings()`。正常情况下，当更新`itemView`时，会触发`onBindBinding`，进行`itemView`的绑定:
```java
@Override
public void onBindBinding(@NonNull ViewDataBinding binding, int variableId, @LayoutRes int layoutRes, int position, T item) {
    tryGetLifecycleOwner();
    if (itemBinding.bind(binding, item)) {
        binding.executePendingBindings();
        if (lifecycleOwner != null) {
            binding.setLifecycleOwner(lifecycleOwner);
        }
    }
}
```
首先，获取`Lifecycle`，然后执行`itemBinding.bind`，`ItemBinding`这个类其实保存了一些绑定信息:
```java
@Nullable
private final OnItemBind<T> onItemBind;
private int variableId;
@LayoutRes
private int layoutRes;
private SparseArray<Object> extraBindings;
```
这些其实就是我们使用的时候创建的绑定关系，比如
```kotlin
val itemBinding = ItemBinding.of<SCPoiItem>(BR.subitem, R.layout.soa_item_address_sub)
    .bindExtra(BR.listener, listener)
    .bindExtra(BR.expend, showSubPoiTag)
    .bindExtra(BR.subPoiTagListener, subPoiTagListener)
```
`bind`方法中，为`itemView`的`binding`填充了这些变量:
```java
public boolean bind(@NonNull ViewDataBinding binding, T item) {
    if (variableId == VAR_NONE) {
        return false;
    }
    boolean result = binding.setVariable(variableId, item);
    if (!result) {
        Utils.throwMissingVariable(binding, variableId, layoutRes);
    }
    if (extraBindings != null) {
        for (int i = 0, size = extraBindings.size(); i < size; i++) {
            int variableId = extraBindings.keyAt(i);
            Object value = extraBindings.valueAt(i);
            if (variableId != VAR_NONE) {
                binding.setVariable(variableId, value);
            }
        }
    }
    return true;
}
```
也就是说，在`adapter`的`onBindViewHolder`方法中，为`itemView`的`binding`绑定了`id`和`data`的对应关系。接下来会执行`binding.executePendingBindings();`，这行代码最终的效果其实是触发xml中比如`android:text={@item.name}`，最终执行的其实是`TextView.setText(item.name)`，最后为`itemViewBinding`绑定生命周期，值得一提的是，这个`lifecycleOwner`一般是我们为页面布局文件的`binding`设置的`viewLifecycleOwner`，`findLifecycleOwner`传入的是`RecyclerView`：
```java
@Nullable
@MainThread
static LifecycleOwner findLifecycleOwner(View view) {
    ViewDataBinding binding = DataBindingUtil.findBinding(view);
    LifecycleOwner lifecycleOwner = null;
    if (binding != null) {
        lifecycleOwner = binding.getLifecycleOwner();
    }
    Context ctx = view.getContext();
    if (lifecycleOwner == null && ctx instanceof LifecycleOwner) {
        lifecycleOwner = (LifecycleOwner) ctx;
    }
    return lifecycleOwner;
}
```

我们看一下`getItemViewType`：
```java
@Override
public int getItemViewType(int position) {
    itemBinding.onItemBind(position, items.get(position));
    return itemBinding.layoutRes();
}
```
这个方法返回了`layoutRes`作为`item`的唯一标识。其实这个方法很关键，它区分了两种常见的声明`itemBinding`的方式，即上边展示的:`ItemBinding.of`和`OnItemBindClass`，使用`ItemBinding.set`的例子已经基本了解，往往不是多`itemType`的情况，这里`getItemViewType`直接返回`layoutRes`是没问题的。对于`OnItemBindClass`，当我们不断调用`map`方法往对象中添加绑定关系的时候，实际上会默认创建`OnItemBind`对象：
```java
public OnItemBindClass<T> map(@NonNull Class<? extends T> itemClass, final int variableId, @LayoutRes final int layoutRes) {
    int index = itemBindingClassList.indexOf(itemClass);
    if (index >= 0) {
        itemBindingList.set(index, itemBind(variableId, layoutRes));
    } else {
        itemBindingClassList.add(itemClass);
        itemBindingList.add(itemBind(variableId, layoutRes));
    }
    return this;
}
```
```java
private final List<Class<? extends T>> itemBindingClassList;
private final List<OnItemBind<? extends T>> itemBindingList;
```
`itemBindingList`和`itemBindingClassList`记录了绑定的数据类型和绑定的回调信息，即使你不调用回调入参的函数，`itemBind`方法也会默认创建回调：
```java
@NonNull
private OnItemBind<T> itemBind(final int variableId, @LayoutRes final int layoutRes) {
    return new OnItemBind<T>() {
        @Override
        public void onItemBind(@NonNull ItemBinding itemBinding, int position, T item) {
            itemBinding.set(variableId, layoutRes);
        }
    };
}
```
到这里问题来了，`class OnItemBindClass<T> implements OnItemBind<T> `，`OnItemBindClass`并没有继承`ItemBinding`类，为什么能直接在xml中写：
`app:itemBinding="@{viewModel.onItemBindClass}"`？其实，这就用到了`@BindingConversion`:
```java
@BindingConversion
public static <T> ItemBinding<T> toItemBinding(OnItemBind<T> onItemBind) {
    return ItemBinding.of(onItemBind);
}
```
也就是说，你设置的`OnItemBindClass`会被转换为`ItemBinding`。对于`OnItemBindClass`的场景，他是怎么返回`itemType`的？
`getItemViewType中`调用`itemBinding.onItemBind(position, items.get(position));`，其实会调用`OnItemBindClass`的`onItemBind`:

```java
public void onItemBind(int position, T item) {
    if (onItemBind != null) {
        variableId = VAR_INVALID;
        layoutRes = LAYOUT_NONE;
        onItemBind.onItemBind(this, position, item);
        if (variableId == VAR_INVALID) {
            throw new IllegalStateException("variableId not set in onItemBind()");
        }
        if (layoutRes == LAYOUT_NONE) {
            throw new IllegalStateException("layoutRes not set in onItemBind()");
        }
    }
}
```

```java
@Override
@SuppressWarnings("unchecked")
public void onItemBind(@NonNull ItemBinding itemBinding, int position, T item) {
    for (int i = 0; i < itemBindingClassList.size(); i++) {
        Class<? extends T> key = itemBindingClassList.get(i);
        if (key.isInstance(item)) {
            OnItemBind itemBind = itemBindingList.get(i);
            itemBind.onItemBind(itemBinding, position, item);
            return;
        }
    }
    throw new IllegalArgumentException("Missing class for item " + item);
}
```
这里从保存的类型和回调中匹配到当前`position`的那一个进行绑定，其实就是调用我们自己的回调函数:
```kotlin
.map(HomeWorkBean::class.java){binding,position,_->
        binding.clearExtras().set(BR.homework, R.layout.soa_item_home_work)
    }
```
这里的`binding`其实就是最外层的`ItemBinding`，调用`set`方法后，会覆盖为当前position的`variableId`和`layoutRes`，那么，在不断调用`OnBindViewHolder`绑定数据的时候，绑定的就是当前position的`ItemViewType`的数据了。

#### 刷新数据问题
前面提到，如果想要使用`adapter`中非全量刷新的方法，可以让数据集是`ObservableList`类型的，这样它就可以被观察，`adapter`会调用对应的`notifyXxx`方法。如果想要完全自定义刷新逻辑，那可以考虑数据集直接使用`List`，自己调用`adapter`的`notifyXxx`方法。

之前其实还遇到过一个问题，有时候我们想无感刷新列表（不想有一个先消失在出现的感知），但列表数量有时候可能发生改变，如果先`list.clear`再`list.addAll`，不能只调用一次`adapter.notifyItemRangeChanged`，这个方法其实是`items`的改变，并不涉及到数量的改变，如果手动先调用`notifyItemRangeRemoved`再调用`notifyItemRangeInsert`可能会有明显的视觉效果和闪动，这不是我们想要的。

更好的办法是`DiffUtil`，我们需要实现`DiffUtil.Callback`：
```java
public class MyDiffUtilCallback extends DiffUtil.Callback {

    List<String> oldList;
    List<String> newList;

    public MyDiffUtilCallback(List<String> oldList, List<String> newList) {
        this.oldList = oldList;
        this.newList = newList;
    }

    @Override
    public int getOldListSize() {
        return oldList.size();
    }

    @Override
    public int getNewListSize() {
        return newList.size();
    }

    @Override
    public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {
        // 在这里实现你的逻辑来判断两个对象是否代表同一个Item
        // 例如，如果你的items有唯一的id字段，可以在这里比较它们的id
        return oldList.get(oldItemPosition).equals(newList.get(newItemPosition));
    }

    @Override
    public boolean areContentsTheSame(int oldItemPosition, int newItemPosition) {
        // 在这里判断两个item是否含有相同的数据
        // 你可以比较他们的详细内容，以确定它们是否完全一样
        return oldList.get(oldItemPosition).equals(newList.get(newItemPosition));
    }
}
```
然后，在更新数据的时候使用`DiffUtil.calculateDiff`计算`DiffUtil.DiffResult`，并调用它的`dispatchUpdatesTo`:
```java
List<String> oldList = adapter.getDataList(); // 获取旧数据列表
List<String> newList = ... // 这是你新的数据列表

DiffUtil.DiffResult diffResult = DiffUtil.calculateDiff(new MyDiffUtilCallback(oldList, newList));

// 更新数据源
adapter.setDataList(newList);

// 将差异结果应用到Adapter上
diffResult.dispatchUpdatesTo(adapter);
```

对于我们这里的`databinding`，也提供了`diffConfig`属性:
```java
if (diffConfig != null && items != null) {
    AsyncDiffObservableList<T> list = (AsyncDiffObservableList<T>) recyclerView.getTag(R.id.bindingcollectiondapter_list_id);
    if (list == null) {
        list = new AsyncDiffObservableList<>(diffConfig);
        recyclerView.setTag(R.id.bindingcollectiondapter_list_id, list);
        adapter.setItems(list);
    }
    list.update(items);
} else {
    adapter.setItems(items);
}
```
如果指定了`diffConfig`属性，则会创建`AsyncDiffObservableList`，上面说到`setItems`会为`ObservableList`设置监听，调用`adapter`对应的`notifyXxx`方法，这种情况是直接修改源列表的情况；如果想使用`Diff`刷新数据，则源数据可以使用`ObservableFiled<List>`，然后每次更新数据的时候都创建`new List`, 这样最终会调用`list.update`:
```java
public AsyncDiffObservableList(@NonNull AsyncDifferConfig<T> config) {
    differ = new AsyncListDiffer<>(new ObservableListUpdateCallback(), config);
}

public void update(@Nullable List<T> newItems) {
    differ.submitList(newItems);
}
```
这样就触发了`Diff`的更新。