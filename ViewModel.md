# ViewModel


## ViewModel的作用
* 界面控制器维度
在最初的 MVC 模式中，Activity / Fragment 中承担的职责过重，因此，在后续的 UI 开发模式中，我们选择将 Activity / Fragment 中与视图无关的职责抽离出来，在 MVP 模式中叫作 Presenter，在 MVVM 模式中叫作 ViewModel。因此，我们使用 ViewModel 来承担界面控制器的职责，并且配合 LiveData / Flow 实现数据驱动。

* 数据维度
由于 Activity 存在因配置变更销毁重建的机制，会造成 Activity 中的所有瞬态数据丢失，例如网络请求得到的用户信息、视频播放信息或者异步任务都会丢失。而 ViewModel 能够应对 Activity 因配置变更而重建的场景，在重建的过程中恢复 ViewModel 数据，从而降低用户体验受损。


## ViewModel创建方式

* ViewModelProvider 是创建 ViewModel 的工具类

```
// 不带工厂的创建方式 
val vm = ViewModelProvider(this).get(MainViewModel::class.java) 

// 带工厂的创建方式 
val vmFactory = ViewModelProvider(this, MainViewModelFactory()).get(MainViewModel::class.java) 
// ViewModel 工厂 
class MainViewModelFactory( ) : ViewModelProvider.Factory { 
private val repository = MainRepository() 
	override fun <T : ViewModel> create(modelClass: Class<T>): T { 
		return MainViewModel(repository) as T 
	} 
}

```

* 使用 Kotlin by 委托属性，本质上是间接使用了 ViewModelProvider

```
// 在 Activity 中使用
//使用 Activity 的作用域
private val viewModel : MainViewModel by viewModels()

// 在 Fragment 中使用
// 使用 Activity 的作用域，与 MainActivity 使用同一个对象 
val activityViewModel : MainViewModel by activityViewModels() 
// 使用 Fragment 的作用域 
val viewModel : MainViewModel by viewModels()

```
activityViewModels和viewModels区别在于activityViewModels的viewModelStore是获取activity的，viewModels是获取fragment的。


* Hilt 提供了注入部分 Jetpack 架构组件的支持

```
@HiltAndroidApp 
class DemoApplication : Application() { ... } 

@HiltViewModel 
class MainViewModel @Inject constructor() : ViewModel() { ... } 

@AndroidEntryPoint 
class MainHiltActivity : AppCompatActivity(){ 
	val viewModel by viewModels<MainViewModel>() 
}

```

```
注意不要使用依赖注入的方式创建（
	例如：
	@Inject 
	lateinit var viewModel : MainViewModel
）
```
需要注意的是，虽然可以使用依赖注入普通对象的方式注入 ViewModel，但是这相当于绕过了 ViewModelProvider 来创建 ViewModel。这意味着 ViewModel 实例一定不会存放在 ViewModelStore 中，将失去 ViewModel 恢复界面数据的特性。


## ViewModel 的创建过程

ViewModelProvider 可以理解为创建 ViewModel 的工具类，它需要 2 个参数：
* 参数 1 ViewModelStoreOwner： 它对应于 Activity / Fragment 等持有 ViewModel 的宿主，它们内部通过 ViewModelStore 维持一个 ViewModel 的映射表，ViewModelStore 是实现 ViewModel 作用域和数据恢复的关键；
* 参数 2 Factory： 它对应于 ViewModel 的创建工厂，缺省时将使用默认的 NewInstanceFactory 工厂来反射创建 ViewModel 实例。


创建 ViewModelProvider 工具类后，你将通过 get() 方法来创建 ViewModel 的实例。get() 方法内部首先会通过 ViewModel 的全限定类名从映射表（ViewModelStore）中取缓存，未命中才会通过 ViewModel 工厂创建实例再缓存到映射表中。
正因为同一个 ViewModel 宿主使用的是同一个 ViewModelStore 映射表，因此在同一个宿主上重复调用 ViewModelProvider#get() 返回同一个 ViewModel 实例。


## ViewModelProvider
不带工厂的创建方式 ，默认使用NewInstanceFactory.getInstance()，反射创建 ViewModel

- ViewModelProvider#get(Class<T> modelClass)方法
从mViewModelStore中获取缓存，存在则返回，否则使用mFactory.create(modelClass)创建实例，
把viewModel实例存储到ViewModelStore中，并返回。

NewInstanceFactory
NewInstanceFactory.create()反射创建ViewModel

ViewModelStore
实质上是ViewModel的映射表，<String - ViewModel>哈希表

ViewModelStoreOwner
是一个接口，实现类是Activity / Fragment，提供ViewModelStore


## 分析by viewModels()实现

by 关键字是 Kotlin 的委托属性，内部也是通过 ViewModelProvider 来创建 ViewModel。

* ActivityViewModelLazy.kt
* ViewModelLazy.kt


## ViewModel 如何实现不同的作用域
ViewModel 内部会为不同的 ViewModel 宿主分配不同的 ViewModelStore 映射表，不同宿主是从不同的数据源来获取 ViewModel 的实例，因而得以区分作用域。

具体来说，在使用 ViewModelProvider 时，我们需要传入一个 ViewModelStoreOwner 宿主接口，它将在 getViewModelStore() 接口方法中返回一个 ViewModelStore 实例。

* 对于 Activity 来说，ViewModelStore 实例是直接存储在 Activity 的成员变量中的；
* 对于 Fragment 来说，ViewModelStore 实例是间接存储在 FragmentManagerViewModel 中的 <Fragment - ViewModelStore> 映射表中的。

这样就实现了不同的 Activity 或 Fragment 分别对应不同的 ViewModelStore 实例，进而区分不同作用域。

```
//Activity
ComponentActivity实现了ViewModelStoreOwner接口，由成员变量mViewModelStore持有ViewModelStore实例。

//Fragment
Fragment实现了ViewModelStoreOwner接口，依次调用了
Fragment#getViewModelStore()->
FragmentManager#getViewModelStore(Fragment)->
FragmentManagerViewModel#getViewModelStore(Fragment)
```

mNonConfig的类型是FragmentManagerViewModel


## 为什么 Activity 在屏幕旋转重建后可以恢复 ViewModel？

Activity 因配置变更而重建时，可以将页面上的数据或状态可以定义为 2 类：
* 第 1 类 - 配置数据： 例如窗口大小、多语言字符、多主题资源等，当设备配置变更时，需要根据最新的配置重新读取新的数据，因此这部分数据在配置变更后便失去意义，自然也就没有存在的价值；
* 第 2 类 - 非配置数据： 例如用户信息、视频播放信息、异步任务等非配置相关数据，这些数据跟设备配置没有一点关系，如果在重建 Activity 的过程中丢失，不仅没有必要，而且会损失用户体验（无法快速恢复页面数据，或者丢失页面进度）。

## 在Activity中的调用过程

* 阶段1、系统在处理 Activity 因配置变更而重建时，会先调用 retainNonConfigurationInstances 获取旧 Activity 中的数据，其中包含 ViewModelStore 实例，而这一份数据会临时存储在当前 Activity 的 ActivityClientRecord；

* 阶段2、在新 Activity 重建后，系统通过在 Activity#onAttach(…) 中将这一份数据传递到新的 Activity 中；

* 阶段3、Activity 在构造 ViewModelStore 时，会优先从旧 Activity 传递过来的这份数据中获取，为空才会创建新的 ViewModelStore。

分析阶段1、 ComponentActivity#onRetainNonConfigurationInstance()方法中返回了实例NonConfigurationInstances，
而NonConfigurationInstances的viewModelStore存储了ViewModelStore实例；

分析阶段2、Activity#attach() 中传递旧 Activity 的数据，旧 Activity 的数据就传递到新 Activity 的成员变量 mLastNonConfigurationInstances 中；

分析阶段3、ComponentActivity#getViewModelStore()优先使用旧 Activity 传递过来的 ViewModelStore（通过getLastNonConfigurationInstance()方法获取mLastNonConfigurationInstances ），mLastNonConfigurationInstances存储旧Activity 的viewModelStore；

## 在ActivityThread中的调用过程
* 阶段1、在处理 Destroy 逻辑时，调用 Activity#retainNonConfigurationInstances() 方法获取旧 Activity 中的非配置数据，并临时保存在 ActivityClientRecord 中；

* 阶段2、在处理 Launch 逻辑时，调用 Activity#attach(…) 将 ActivityClientRecord 中临时保存的非配置数据传递到新 Activity 中。


## ViewModel 的数据在什么时候才会清除
* 第 1 种： 直接调用 Activity#finish() 或返回键等间接方式；
* 第 2 种： 异常退出 Activity，例如内存不足；
* 第 3 种： 强制退出应用。

ComponentActivity在实例时，向lifecycle添加生命周期监听，onStateChanged监听回调中，
event == Lifecycle.Event.ON_DESTROY时调用getViewModelStore().clear()


## ViewModel 的内存泄漏问题

ViewModel 的内存泄漏是指 Activity 已经销毁，但是 ViewModel 却被其他组件引用。这往往是因为数据层是通过回调监听器的方式返回数据，并且数据层是单例对象或者属于全局生命周期，所以导致 Activity 销毁了，但是数据层依然间接持有 ViewModel 的引用。
如果 ViewModel 是轻量级的或者可以保证数据层操作快速完成，这个泄漏影响不大可以忽略。但如果数据层操作并不能快速完成，或者 ViewModel 存储了重量级数据，就有必要采取措施。例如：

* 方法 1： 在 ViewModel#onCleared() 中通知数据层丢弃对 ViewModel 回调监听器的引用；
* 方法 2： 在数据层使用对 ViewModel 回调监听器的弱引用（这要求 ViewModel 必须持有回调监听器的强引用，而不能使用匿名内部类，这会带来编码复杂性）；
* 方法 3： 使用 EventBus 代替回调监听器（这会带来编码复杂性）；
* 方法 4： 使用 LiveData 的 Transformations.switchMap() API 包装数据层的请求方法，这相当于在 ViewModel 和数据层中间使用 LiveData 进行通信。


## ViewModel 和 onSaveInstanceState() 的对比

ViewModel 和 onSaveInstanceState() 都是对数据的恢复机制，但由于它们针对的场景不同，导致它们的实现原理不同，进而优缺点也不同。
* 1、ViewModel： 使用场景针对于配置变更重建中非配置数据的恢复，由于内存是可以满足这种存储需求的，因此可以选择内存存储。又由于内存空间相对较大，因此可以存储大数据，但会受到内存空间限制；
* 2、onSaveInstanceState() ：使用场景针对于应用被系统回收后重建时对数据的恢复，由于应用进程在这个过程中会消亡，因此不能选择内存存储而只能选择使用持久化存储。又由于这部分数据需要通过 Bundle 机制在应用进程和 AMS 服务之间传递，因此会受到 Binder 事务缓冲区大小限制，只可以存储小规模数据。


参考资料：

Jetpack 系列（3）—— 为什么 Activity 都重建了 ViewModel 还存在？</br>
https://juejin.cn/post/7121998366103306254
	
Kotlin | 委托机制 & 原理 & 应用</br>
https://juejin.cn/post/6958346113552220173
