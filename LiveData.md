# LiveData


LiveData 是基于 Lifecycle 框架实现的生命周期感知型数据容器，能够让数据观察者更加安全地应对宿主（Activity / Fragment 等）生命周期变化，核心概括为 2 点：
* 1、自动取消订阅： 当宿主生命周期进入消亡（DESTROYED）状态时，LiveData 会自动移除观察者，避免内存泄漏；
* 2、安全地回调数据： 在宿主生命周期状态低于活跃状态（STAETED）时，LiveData 不会回调数据，避免产生空指针异常或不必要的性能损耗；当宿主生命周期不低于活跃状态（STAETED）时，LiveData 会重新尝试回调数据，确保观察者接收到最新的数据。

## 注册观察者
LiveData 支持两种注册观察者的方式：
* LiveData#observe(LifecycleOwner, Observer) 带生命周期感知的注册： 更常用的注册方式，这种方式能够获得 LiveData 自动取消订阅和安全地回调数据的特性；
* LiveData#observeForever(Observer) 永久注册： LiveData 会一直持有观察者的引用，只要数据更新就会回调，因此这种方式必须在合适的时机手动移除观察者。

```
//Observer观察者接口
public interface Observer<T> {
    void onChanged(T t);
}
```

设置数据： LiveData 设置数据需要利用子类 MutableLiveData 提供的接口：
* setValue() 为同步设置数据；
* postValue() 为异步设置数据，内部将 post 到主线程再修改数据。


## LiveData 存在的局限
LiveData 是 Android 生态中一个的简单的生命周期感知型容器。简单即是它的优势，也是它的局限，当然这些局限性不应该算 LiveData 的缺点，因为 LiveData 的设计初衷就是一个简单的数据容器，需要具体问题具体分析。对于简单的数据流场景，使用 LiveData 完全没有问题。
* 1、LiveData 只能在主线程更新数据： 只能在主线程 setValue，即使 postValue 内部也是切换到主线程执行；
* 2、LiveData 数据重放问题： 注册新的订阅者，会重新收到 LiveData 存储的数据，这在有些情况下不符合预期；
* 3、LiveData 不防抖问题： 重复 setValue 相同的值，订阅者会收到多次 onChanged() 回调（可以使用 distinctUntilChanged() 优化）；
* 4、LiveData 丢失数据问题： 在数据生产速度 > 数据消费速度时，LiveData 无法观察者能够接收到全部数据。比如在子线程大量 postValue 数据但主线程消费跟不上时，中间就会有一部分数据被忽略。


## LiveData 的替代者
* 1、RxJava： RxJava 是第三方组织 ReactiveX 开发的组件，Rx 是一个包括 Java、Go 等语言在内的多语言数据流框架。功能强大是它的优势，支持大量丰富的操作符，也支持线程切换和背压。然而 Rx 的学习门槛过高，对开发反而是一种新的负担，也会带来误用的风险。
* 2、Kotlin Flow： Kotlin Flow 是基于 Kotlin 协程基础能力搭建的一套数据流框架，从功能复杂性上看是介于 LiveData 和 RxJava 之间的解决方案。Kotlin Flow 拥有比 LiveData 更丰富的能力，但裁剪了 RxJava 大量复杂的操作符，做得更加精简。并且在 Kotlin 协程的加持下，Kotlin Flow 目前是 Google 主推的数据流框架。


## LiveData 实现原理分析

注册观察者的执行过程

```
// 注册方式 1：带生命周期感知的注册方式 
@MainThread 
public void observe(LifecycleOwner owner, Observer<? super T> observer) { 
	// 1.1 主线程检查 
	assertMainThread("observe"); 
	// 1.2 宿主生命周期状态是 DESTROY，则跳过 
	if (owner.getLifecycle().getCurrentState() == DESTROYED) { 
		return; 
	} 
	// 1.3 将 Observer 包装为 LifecycleBoundObserver 
	LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer); 
	ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper); 
	// 1.4 禁止将 Observer 绑定到不同的宿主上 
	if (existing != null && !existing.isAttachedTo(owner)) { 
		throw new IllegalArgumentException("Cannot add the same observer with different lifecycles"); 
	} 
	if (existing != null) { 
		return; 
	} 
	// 1.5 将包装类注册到宿主声明周期上 
	owner.getLifecycle().addObserver(wrapper); 
}
```

```
// 注册方式 2：永久注册的方式 
@MainThread 
public void observeForever(Observer<? super T> observer) { 
	// 2.1 主线程检查 
	assertMainThread("observeForever"); 
	// 2.2 将 Observer 包装为 AlwaysActiveObserver 
	AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer); 
	// 2.3 禁止将 Observer 注册到生命周期宿主后又进行永久注册 
	ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper); 
	if (existing instanceof LiveData.LifecycleBoundObserver) { 
		throw new IllegalArgumentException("Cannot add the same observer with different lifecycles"); 
	} 
	if (existing != null) { 
		return; 
	} 
	// 2.4 分发最新数据 
	wrapper.activeStateChanged(true); 
}
```

```
// 注销观察者 
@MainThread 
public void removeObserver(@NonNull final Observer<? super T> observer) { 
	// 主线程检查 
	assertMainThread("removeObserver"); 
	// 移除 
	ObserverWrapper removed = mObservers.remove(observer); 
	if (removed == null) { 
		return; 
	} 
	// removed.detachObserver() 方法： 
	// LifecycleBoundObserver 最终会调用 Lifecycle#removeObserver() 
	// AlwaysActiveObserver 为空实现 
	removed.detachObserver(); 
	removed.activeStateChanged(false); 
}
```

## 生命周期感知源码分析
LifecycleBoundObserver 是 LifecycleEventObserver 的实现类，当宿主生命周期变化时，会回调其中的 LifecycleEventObserve#onStateChanged() 方法：

LifecycleBoundObserver继承ObserverWrapper实现LifecycleEventObserver 

AlwaysActiveObserver继承ObserverWrapper


## 同步设置数据的执行过程

LiveData 使用 setValue() 方法进行同步设置数据（必须在主线程调用），需要注意的是，设置数据后并不一定会回调 Observer#onChanged() 分发数据，而是需要同时 2 个条件：
* 条件 1： 观察者绑定的生命周期处于活跃状态；
    * observeForever() 观察者：一直处于活跃状态；
    * observe() 观察者：owner 宿主生命周期处于活跃状态。
* 条件 2： 观察者的持有的版本号小于 LiveData 的版本号时。

总结一下回调 Observer#onChanged() 的情况：
* 1、注册观察者时，观察者绑定的生命处于活跃状态，并且 LiveData 存在已设置的旧数据；
* 2、调用 setValue() / postValue() 设置数据时，观察者绑定的生命周期处于活跃状态；
* 3、观察者绑定的生命周期由非活跃状态转为活跃状态，并且 LiveData 存在未分发到该观察者的数据（即观察者持有的版本号小于 LiveData 持有的版本号）；


## 异步设置数据的执行过程
LiveData 使用 postValue() 方法进行异步设置数据（允许在子线程调用），内部会通过一个临时变量 mPendingData 存储数据，再通过 Handler 将切换到主线程并调用 setValue(临时变量)。因此，当在子线程连续 postValue() 时，可能会出现中间的部分数据不会被观察者接收到。

总结一下 LiveData 可能丢失数据的场景，此时观察者可能不会接收到所有的数据：
* 情况 1（背压问题）： 使用 postValue() 异步设置数据，并且观察者的消费速度小于数据生产速度；
* 情况 2： 在观察者处理回调（Observer#obChanged()）的过程中重新设置新数据，此时会中断旧数据的分发，部分观察者将无法接收到旧数据；
* 情况 3： 观察者绑定的生命周期处于非活跃状态时，连续使用 setValue() / postValue() 设置数据时，观察将无法接收到中间的数据。

## LiveData 数据重放原因分析
LiveData 的数据重放问题也叫作数据倒灌、粘性事件，核心源码在 LiveData#considerNotify(Observer) 中：
* 首先，LiveData 和观察者各自会持有一个版本号 version，每次 LiveData#setValue 或 postValue 后，LiveData 持有的版本号会自增 1。在 LiveData#considerNotify(Observer) 尝试分发数据时，会判断观察者持有版本号是否小于 LiveData 的版本号（Observer#mLastVersion >= LiveData#mVersion 是否成立），如果不成立则说明这个观察者还没有消费最新的数据版本。
* 而观察者的持有的初始版本号是 -1，因此当注册新观察者并且正好宿主的生命周期是大于等于可见状态（STARTED）时，就会尝试分发数据，这就是数据重放。
为什么 Google 要把 LiveData 设计为粘性呢？LiveData 重放问题需要区分场景来看 —— 状态适合重放，而事件不适合重放：
* 当 LiveData 作为一个状态使用时，在注册新观察者时重放已有状态是合理的；
* 当 LiveData 作为一个事件使用时，在注册新观察者时重放已经分发过的事件就是不合理的。


## LiveData 数据重放问题的解决方案
这里我们总结一下业界提出处理 LiveData 数据重放问题的方案：
* Event 事件包装器（不推荐）
* SingleLiveData 事件包装器变型方案
* 反射修改观察者版本号
* UnPeekLiveData 反射方案优化
* Kotlin Flow

UnPeekLiveData方案设计思路：
* 1、继承LiveData类;
* 2、维护一个原子整形变量（AtomicInteger）标记数据版本号；
* 3、在setValue(T)时，版本号自增1；
* 4、创建一个自定义的观察者（Observer）,内部维护一个外部添加的观察者和观察者版本号
ObserverWrapper(@NonNull Observer<? super T> observer, int version)
* 5、调用LiveData#observe添加观察者时，获取LiveData的版本号同步到ObserverWrapper的版本号；
* 6、在Observer#onChanged(T)回调时，判断LiveData版本号 > Observer版本号时才更新外部添加的观察者；

```
转换LiveData数据类型，使用Transformations#map
val intLiveData = MutableLiveData(1)
val strLiveData:LiveData<String> = Transformations.map(intLiveData){
    "$it"
}
strLiveData.observe(this){
    println(it is String)//true
}
```

Transformations#switchMap

```
Transformations#distinctUntilChanged，处理重复数据
val boolLiveData = MutableLiveData<Boolean>()
val distinctLiveData = Transformations.distinctUntilChanged(boolLiveData)
Thread {
    for (index in 0..10) {
        SystemClock.sleep(100)
        boolLiveData.postValue((index < 9))
    }
}.start()
distinctLiveData.observe(this){
    println(it) //结果去重了  只有true 和 false  两个输出
}
```


参考资料

Jetpack 系列（2）—— 为什么 LiveData 会重放数据，怎么解决？</br>
https://juejin.cn/post/7121621553670225957
