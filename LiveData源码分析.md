# LiveData源码分析

分析源码前，首先了解一下`MutableLiveData`和`LiveData`的区别。

MutableLiveData类
* 继承LiveData；
* 重写了postValue和setValue；
```
public class MutableLiveData<T> extends LiveData<T> {

    public MutableLiveData(T value) {
        super(value);
    }

    public MutableLiveData() {
        super();
    }

    @Override
    public void postValue(T value) {
        super.postValue(value);
    }

    @Override
    public void setValue(T value) {
        super.setValue(value);
    }
}
```
LiveData类
```
public abstract class LiveData<T> {
    ···
    省略代码
    
    //有参构造
    public LiveData(T value) {
        mData = value;
        mVersion = START_VERSION + 1;
    }

    //无参构造
    public LiveData() {
        //NOT_SET是一个Object
        mData = NOT_SET;
        //初始化LiveData数据的版本号，START_VERSION = -1
        mVersion = START_VERSION;
    }
    
    protected void postValue(T value) {
        ···
    }
    
    @MainThread
    protected void setValue(T value) {
        ···
    }
    
    ···
}
```

> LiveData是一个抽象类，只能通过它的子类MutableLiveData进行实例，MutableLiveData重写了LiveData的postValue和setValue方法，拥有了设置能力。

## postValue

先查看LiveData的postValue方法，因为post方法最后也调用setValue。
```
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        //mPendingData是一个临时数据，默认值是NOT_SET（Object）
        //如果mPendingData等于默认值，表示LiveData当前没有向主线程post数据，接下来会调用postToMainThread去更新数据；
        //相反，当前value数据可能被丢弃；
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    //postTask为false时会跳出，不会执行postToMainThread
    if (!postTask) {
        return;
    }
    //postToMainThread中获取主线程的handler，调用handler的post方法回到主线程中
    //ArchTaskExecutor类委托DefaultTaskExecutor实现细节
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}
```
接下来看看mPostValueRunnable做了哪些操作
```
private final Runnable mPostValueRunnable = new Runnable() {
    @SuppressWarnings("unchecked")
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            //把之前在postValue中的临时数据交给newValue并恢复默认值NOT_SET
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        //最后还是调用setValue进行设值
        setValue((T) newValue);
    }
};
```

## setValue

setValue方法需要在主线程调用，使用了`@MainThread`注解。
```
@MainThread
protected void setValue(T value) {
    //检查是否在主线程，否则会抛出错误
    assertMainThread("setValue");
    //版本号进行递增
    mVersion++;
    //真正设置数据的地方
    mData = value;
    //通知观察者数据已改变
    dispatchingValue(null);
}
```

## 观察者

添加观察者有两种方式
* observe：在具有生命周期的所有者里添加观察者，将观察者添加到观察者列表中。该方法只能在主线程上调度。如果LiveData已经有数据，则它将把数据发送给观察者。
只有当所有者处于Lifecycle.State.STARTED或Lifecycle.State.RESUMED状态（活动）时，观察者才会接收事件。
如果所有者移动到Lifecycle.State.DESTROYED状态，则观察者将自动删除。
当所有者不活动时数据发生更改时，它将不会收到任何更新。如果它再次激活，它将自动接收最后可用的数据。
只要给定的LifecycleOwner没有被破坏，LiveData就会保留对观察者和所有者的强引用。当它被销毁时，LiveData会删除对观察者和所有者的引用。
如果给定的所有者已经处于Lifecycle.State.DESTROYED状态，LiveData将忽略该调用。
如果给定的所有者、观察者元组已经在列表中，则该调用将被忽略。如果观察者已经和另一个所有者在列表中，LiveData会抛出一个IllegalArgumentException。

* observeForever：将给定的观察者添加到观察者列表中。此调用类似于始终处于活动状态的LifecycleOwner的observe（LifecycleOwner，Observer）。
这意味着给定的观察者将接收所有事件，并且永远不会被自动删除。
应该手动调用removeObserver（Observer）来停止观察此LiveData。虽然LiveData有一个这样的观察者，但它将被视为活动的。
如果观察者已经添加了该LiveData的所有者，那么LiveData将抛出一个IllegalArgumentException。

---

### observeForever

先看observeForever方法，需要在主线程调度。
```
@MainThread
public void observeForever(@NonNull Observer<? super T> observer) {
    //检查是否在主线程
    assertMainThread("observeForever");
    //AlwaysActiveObserver继承ObserverWrapper类，持有传入的observer
    AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
    //把观察者添加到观察者列表中，
    //如果已存在该观察者是LifecycleBoundObserver的子类则抛出错误
    //同一个观察者不能同时添加到一个所有者里
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing instanceof LiveData.LifecycleBoundObserver) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    //如果existing==null，表示添加成功
    //通知状态变更
    wrapper.activeStateChanged(true);
}
```
接下来看看LiveData#dispatchingValue
```
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            //重点，通知观察者
            considerNotify(initiator);
            initiator = null;
        } else {
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}

private void considerNotify(ObserverWrapper observer) {
    //如果观察者不活动，则退出
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    //AlwaysActiveObserver重写了shouldBeActive，返回true
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    //只有LiveData的版本号大于观察者的版本号时才通知观察者
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    //更新观察者的版本号
    observer.mLastVersion = mVersion;
    //调用观察者的onChanged
    observer.mObserver.onChanged((T) mData);
}
```

### observe

observe也需要在主线程调用
```
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    //检查是否在主线程
    assertMainThread("observe");
    //如果生命周期已到达DESTROYED就退出
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    //LifecycleBoundObserver继承ObserverWrapper，实现了LifecycleEventObserver接口
    //重点查看LifecycleBoundObserver如何通知数据变更
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    //isAttachedTo判断传入生命周期拥有者与原来保存的是否相等
    //不允许观察者添加到不同的生命周期拥有者中
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    //如果existing==null，表示添加成功
    //添加观察者到Lifecycle中
    owner.getLifecycle().addObserver(wrapper);
}

class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
    @NonNull
    final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        mOwner = owner;
    }

    @Override
    boolean shouldBeActive() {
        //当前生命周期状态大于等于STARTED，返回true
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
        //生命周期到达销毁时会移除观察者
        if (currentState == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        Lifecycle.State prevState = null;
        while (prevState != currentState) {
            prevState = currentState;
            //通知数据变更
            activeStateChanged(shouldBeActive());
            currentState = mOwner.getLifecycle().getCurrentState();
        }
    }

    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }

    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this);
    }
}
```

移除观察者removeObserver
```
@MainThread
public void removeObserver(@NonNull final Observer<? super T> observer) {
    assertMainThread("removeObserver");
    ObserverWrapper removed = mObservers.remove(observer);
    if (removed == null) {
        return;
    }
    removed.detachObserver();
    removed.activeStateChanged(false);
}
```

获取数据getValue
```
@Nullable
public T getValue() {
    Object data = mData;
    if (data != NOT_SET) {
        return (T) data;
    }
    return null;
}
```



