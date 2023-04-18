# 协程源码分析

简单的使用例子
```
val mainScope = MainScope()

mainScope.launch {
    val handle = request("url")

    println(handle)
}

private suspend fun request(param: String): String {
    delay(1000)
    return param
}
```

## CoroutineScope

CoroutineScope是一个接口，定义协程的作用域，通过继承CoroutineScope可以自定义协程的作用域。 

CoroutineScope接口
```
public interface CoroutineScope {
    public val coroutineContext: CoroutineContext
}
```

### MainScope

MainScope的实现类是ContextScope，ContextScope中的CoroutineContext是由SupervisorJob() + Dispatchers.Main组合而成。
```
public fun MainScope(): CoroutineScope = ContextScope(SupervisorJob() + Dispatchers.Main)
```

ContextScope构造函数
```
internal class ContextScope(context: CoroutineContext) : CoroutineScope {
    override val coroutineContext: CoroutineContext = context
    override fun toString(): String = "CoroutineScope(coroutineContext=$coroutineContext)"
}
```

### lifecycleScope

lifecycleScope是Lifecycle的扩展，携带生命周期管理的协程作用域。
```
lifecycleScope.launch {
    val handle = request("url")

    println(handle)
}
```

lifecycleScope的实现类是LifecycleCoroutineScopeImpl，是Lifecycle中的扩展变量，通过get()获取。
```
public val LifecycleOwner.lifecycleScope: LifecycleCoroutineScope
    get() = lifecycle.coroutineScope
    
public val Lifecycle.coroutineScope: LifecycleCoroutineScope
    get() {
        while (true) {
            //通过原子类AtomicReference<Object>保存LifecycleCoroutineScope
            //存在则返回
            val existing = mInternalScopeRef.get() as LifecycleCoroutineScopeImpl?
            if (existing != null) {
                return existing
            }
            //没有缓存，则实例化LifecycleCoroutineScopeImpl，传入Lifecycle和CoroutineContext
            val newScope = LifecycleCoroutineScopeImpl(
                this,
                SupervisorJob() + Dispatchers.Main.immediate
            )
            //保存newScope
            if (mInternalScopeRef.compareAndSet(null, newScope)) {
                //注册生命周期监听
                newScope.register()
                return newScope
            }
        }
    }
```

LifecycleCoroutineScopeImpl类
```
internal class LifecycleCoroutineScopeImpl(
    override val lifecycle: Lifecycle,
    override val coroutineContext: CoroutineContext
) : LifecycleCoroutineScope(), LifecycleEventObserver {
    init {
        //如果生命周期已经是DESTROYED状态，则取消协程
        if (lifecycle.currentState == Lifecycle.State.DESTROYED) {
            coroutineContext.cancel()
        }
    }

    //注册生命周期监听
    fun register() {
        //启动主线程协程，在初始化INITIALIZED状态或之后添加生命周期监听，否则取消协程
        launch(Dispatchers.Main.immediate) {
            if (lifecycle.currentState >= Lifecycle.State.INITIALIZED) {
                lifecycle.addObserver(this@LifecycleCoroutineScopeImpl)
            } else {
                coroutineContext.cancel()
            }
        }
    }

    override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
        //生命周期状态回调
        //DESTROYED状态时，移除监听和取消协程
        if (lifecycle.currentState <= Lifecycle.State.DESTROYED) {
            lifecycle.removeObserver(this)
            coroutineContext.cancel()
        }
    }
}
```

### viewModelScope

viewModelScope是ViewModel的扩展
```
viewModelScope.launch {
    val handle = request("url")

    println(handle)
}
```

viewModelScope的实现类是CloseableCoroutineScope
```
public val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope: CoroutineScope? = this.getTag(JOB_KEY)
        if (scope != null) {
            return scope
        }
        return setTagIfAbsent(
            JOB_KEY,
            CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
        )
    }
```

CloseableCoroutineScope除了实现了CoroutineScope外，还实现了Closeable接口。
```
internal class CloseableCoroutineScope(context: CoroutineContext) : Closeable, CoroutineScope {
    override val coroutineContext: CoroutineContext = context

    override fun close() {
        coroutineContext.cancel()
    }
}
```


## CoroutineContext

CoroutineContext是一个特殊的集合，element之间通过`+`号组合，Element有如下几种类型，都继承了CoroutineContext.Element。 

CoroutineContext = Job + CoroutineDispatcher + CoroutineName + CoroutineExceptionHandler 

* Job: 控制协程的生命周期；
* CoroutineDispatcher: 向合适的线程分发任务(IO、Default、Main、Unconfined)；
* CoroutineName: 协程的名称，调试的时候很有用；
* CoroutineExceptionHandler: 处理未被捕捉的异常。

CoroutineContext类
```
public interface CoroutineContext {
    /**
     * Returns the element with the given [key] from this context or `null`.
     */
    public operator fun <E : Element> get(key: Key<E>): E?

    /**
     * Accumulates entries of this context starting with [initial] value and applying [operation]
     * from left to right to current accumulator value and each element of this context.
     */
    public fun <R> fold(initial: R, operation: (R, Element) -> R): R

    /**
     * Returns a context containing elements from this context and elements from  other [context].
     * The elements from this context with the same key as in the other one are dropped.
     */
    public operator fun plus(context: CoroutineContext): CoroutineContext =
        if (context === EmptyCoroutineContext) this else // fast path -- avoid lambda creation
            context.fold(this) { acc, element ->
                val removed = acc.minusKey(element.key)
                if (removed === EmptyCoroutineContext) element else {
                    // make sure interceptor is always last in the context (and thus is fast to get when present)
                    val interceptor = removed[ContinuationInterceptor]
                    if (interceptor == null) CombinedContext(removed, element) else {
                        val left = removed.minusKey(ContinuationInterceptor)
                        if (left === EmptyCoroutineContext) CombinedContext(element, interceptor) else
                            CombinedContext(CombinedContext(left, element), interceptor)
                    }
                }
            }

    /**
     * Returns a context containing elements from this context, but without an element with
     * the specified [key].
     */
    public fun minusKey(key: Key<*>): CoroutineContext

    ···
}
```

Element继承CoroutineContext接口
```
/**
 * Key for the elements of [CoroutineContext]. [E] is a type of element with this key.
 */
public interface Key<E : Element>

/**
 * An element of the [CoroutineContext]. An element of the coroutine context is a singleton context by itself.
 */
public interface Element : CoroutineContext {
    /**
     * A key of this coroutine context element.
     */
    public val key: Key<*>

    public override operator fun <E : Element> get(key: Key<E>): E? =
        @Suppress("UNCHECKED_CAST")
        if (this.key == key) this as E else null

    public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
        operation(initial, this)

    public override fun minusKey(key: Key<*>): CoroutineContext =
        if (this.key == key) EmptyCoroutineContext else this
}
```

### Job

#### Job
```
public fun Job(parent: Job? = null): CompletableJob = JobImpl(parent)

public interface Job : CoroutineContext.Element {
    ···
}
```

Job的实现类JobImpl
```
internal open class JobImpl(parent: Job?) : JobSupport(true), CompletableJob {
    init { initParentJob(parent) }
    override val onCancelComplete get() = true

    override val handlesException: Boolean = handlesException()
    override fun complete() = makeCompleting(Unit)
    override fun completeExceptionally(exception: Throwable): Boolean =
        makeCompleting(CompletedExceptionally(exception))

    @JsName("handlesExceptionF")
    private fun handlesException(): Boolean {
        var parentJob = (parentHandle as? ChildHandleNode)?.job ?: return false
        while (true) {
            if (parentJob.handlesException) return true
            parentJob = (parentJob.parentHandle as? ChildHandleNode)?.job ?: return false
        }
    }
}
```

#### SupervisorJob
```
public fun SupervisorJob(parent: Job? = null) : CompletableJob = SupervisorJobImpl(parent)
```

SupervisorJob的实现类SupervisorJobImpl
```
private class SupervisorJobImpl(parent: Job?) : JobImpl(parent) {
    //重写了childCancelled，直接返回false，子job抛出异常时不会取消父job和其他兄弟job
    override fun childCancelled(cause: Throwable): Boolean = false
}
```

### CoroutineDispatcher

CoroutineDispatcher的4种实现
```
public actual object Dispatchers {

    @JvmStatic
    public actual val Default: CoroutineDispatcher = createDefaultDispatcher()

    @JvmStatic
    public actual val Main: MainCoroutineDispatcher get() = MainDispatcherLoader.dispatcher

    @JvmStatic
    public actual val Unconfined: CoroutineDispatcher = kotlinx.coroutines.Unconfined

    @JvmStatic
    public val IO: CoroutineDispatcher = DefaultScheduler.IO
}
```

CoroutineDispatcher实现了ContinuationInterceptor接口，ContinuationInterceptor也是继承CoroutineContext.Element，负责执行挂起函数前的拦截。
```
public abstract class CoroutineDispatcher :
    AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {

    /** @suppress */
    @ExperimentalStdlibApi
    public companion object Key : AbstractCoroutineContextKey<ContinuationInterceptor, CoroutineDispatcher>(
        ContinuationInterceptor,
        { it as? CoroutineDispatcher })
    
    ···  
}
```
#### Dispatchers.Default
```
@JvmStatic
public actual val Default: CoroutineDispatcher = createDefaultDispatcher()
```

useCoroutinesScheduler：true使用DefaultScheduler(kotlin的线程池)，false使用CommonPool(java的线程池)
```
internal actual fun createDefaultDispatcher(): CoroutineDispatcher =
    if (useCoroutinesScheduler) DefaultScheduler else CommonPool
```

DefaultScheduler
```
internal object DefaultScheduler : ExperimentalCoroutineDispatcher() {
    val IO: CoroutineDispatcher = LimitingDispatcher(
        this,
        systemProp(IO_PARALLELISM_PROPERTY_NAME, 64.coerceAtLeast(AVAILABLE_PROCESSORS)),
        "Dispatchers.IO",
        TASK_PROBABLY_BLOCKING
    )

    override fun close() {
        throw UnsupportedOperationException("$DEFAULT_DISPATCHER_NAME cannot be closed")
    }

    ···
}

@InternalCoroutinesApi
public open class ExperimentalCoroutineDispatcher(
    private val corePoolSize: Int,
    private val maxPoolSize: Int,
    private val idleWorkerKeepAliveNs: Long,
    private val schedulerName: String = "CoroutineScheduler"
) : ExecutorCoroutineDispatcher() {

    ···
}
```

CommonPool
```
internal object CommonPool : ExecutorCoroutineDispatcher() {

    ···
}
```

> DefaultScheduler和CommonPool使用相同的父类ExecutorCoroutineDispatcher

ExecutorCoroutineDispatcher
```
public abstract class ExecutorCoroutineDispatcher: CoroutineDispatcher(), Closeable {
    /** @suppress */
    @ExperimentalStdlibApi
    public companion object Key : AbstractCoroutineContextKey<CoroutineDispatcher, ExecutorCoroutineDispatcher>(
        CoroutineDispatcher,
        { it as? ExecutorCoroutineDispatcher })

    public abstract val executor: Executor

    public abstract override fun close()
}
```

#### Dispatchers.Main
```
@JvmStatic
public actual val Main: MainCoroutineDispatcher get() = MainDispatcherLoader.dispatcher

internal object MainDispatcherLoader {

    private val FAST_SERVICE_LOADER_ENABLED = systemProp(FAST_SERVICE_LOADER_PROPERTY_NAME, true)

    @JvmField
    val dispatcher: MainCoroutineDispatcher = loadMainDispatcher()

    private fun loadMainDispatcher(): MainCoroutineDispatcher {
        return try {
            val factories = if (FAST_SERVICE_LOADER_ENABLED) {
                FastServiceLoader.loadMainDispatcherFactory()
            } else {
                // We are explicitly using the
                // `ServiceLoader.load(MyClass::class.java, MyClass::class.java.classLoader).iterator()`
                // form of the ServiceLoader call to enable R8 optimization when compiled on Android.
                ServiceLoader.load(
                        MainDispatcherFactory::class.java,
                        MainDispatcherFactory::class.java.classLoader
                ).iterator().asSequence().toList()
            }
            @Suppress("ConstantConditionIf")
            factories.maxByOrNull { it.loadPriority }?.tryCreateDispatcher(factories)
                ?: createMissingDispatcher()
        } catch (e: Throwable) {
            // Service loader can throw an exception as well
            createMissingDispatcher(e)
        }
    }
}
```

> Android就需要引入kotlinx-coroutines-android库，它里面有Android对应的Dispatchers.Main实现，其实就是把任务通过Handler运行在Android的主线程。

#### Dispatchers.Unconfined
```
@JvmStatic
public actual val Unconfined: CoroutineDispatcher = kotlinx.coroutines.Unconfined

internal object Unconfined : CoroutineDispatcher() {
    //Unconfined = false；Default、IO = true
    override fun isDispatchNeeded(context: CoroutineContext): Boolean = false

    override fun dispatch(context: CoroutineContext, block: Runnable) {
        // It can only be called by the "yield" function. See also code of "yield" function.
        //yield() 表示当前协程让出自己所在的线程给其他协程运行
        val yieldContext = context[YieldContext]
        if (yieldContext != null) {
            // report to "yield" that it is an unconfined dispatcher and don't call "block.run()"
            yieldContext.dispatcherWasUnconfined = true
            return
        }
        throw UnsupportedOperationException("Dispatchers.Unconfined.dispatch function can only be used by the yield function. " +
            "If you wrap Unconfined dispatcher in your code, make sure you properly delegate " +
            "isDispatchNeeded and dispatch calls.")
    }
    
    override fun toString(): String = "Dispatchers.Unconfined"
}
```

#### Dispatchers.IO
```
@JvmStatic
public val IO: CoroutineDispatcher = DefaultScheduler.IO

internal object DefaultScheduler : ExperimentalCoroutineDispatcher() {
    val IO: CoroutineDispatcher = LimitingDispatcher(
            this,
            systemProp(IO_PARALLELISM_PROPERTY_NAME, 64.coerceAtLeast(AVAILABLE_PROCESSORS)),
            "Dispatchers.IO",
            TASK_PROBABLY_BLOCKING
        )
    
    ···
}
```

LimitingDispatcher
```
private class LimitingDispatcher(
    private val dispatcher: ExperimentalCoroutineDispatcher,
    private val parallelism: Int,
    private val name: String?,
    override val taskMode: Int
) : ExecutorCoroutineDispatcher(), TaskContext, Executor {

    private val queue = ConcurrentLinkedQueue<Runnable>()
    private val inFlightTasks = atomic(0)

    override val executor: Executor
        get() = this

    override fun execute(command: Runnable) = dispatch(command, false)

    override fun close(): Unit = error("Close cannot be invoked on LimitingBlockingDispatcher")

    override fun dispatch(context: CoroutineContext, block: Runnable) = dispatch(block, false)

    private fun dispatch(block: Runnable, tailDispatch: Boolean) {
        var taskToSchedule = block
        while (true) {
            // Commit in-flight tasks slot
            val inFlight = inFlightTasks.incrementAndGet()

            // Fast path, if parallelism limit is not reached, dispatch task and return
            if (inFlight <= parallelism) {
                dispatcher.dispatchWithContext(taskToSchedule, this, tailDispatch)
                return
            }

            // Parallelism limit is reached, add task to the queue
            queue.add(taskToSchedule)

            if (inFlightTasks.decrementAndGet() >= parallelism) {
                return
            }

            taskToSchedule = queue.poll() ?: return
        }
    }

    override fun dispatchYield(context: CoroutineContext, block: Runnable) {
        dispatch(block, tailDispatch = true)
    }

    override fun toString(): String {
        return name ?: "${super.toString()}[dispatcher = $dispatcher]"
    }

    override fun afterTask() {
        var next = queue.poll()
        // If we have pending tasks in current blocking context, dispatch first
        if (next != null) {
            dispatcher.dispatchWithContext(next, this, true)
            return
        }
        inFlightTasks.decrementAndGet()

        next = queue.poll() ?: return
        dispatch(next, true)
    }
}
```


### CoroutineName

协程的名字，方便调试时使用
```
public data class CoroutineName(
    val name: String
) : AbstractCoroutineContextElement(CoroutineName) {
    /**
     * Key for [CoroutineName] instance in the coroutine context.
     */
    public companion object Key : CoroutineContext.Key<CoroutineName>

    /**
     * Returns a string representation of the object.
     */
    override fun toString(): String = "CoroutineName($name)"
}
```

### CoroutineExceptionHandler

协程异常捕捉器
```
lifecycleScope.launch(Job() + Dispatchers.Main + CoroutineExceptionHandler { context, throwable ->
    //捕捉到的异常
}) {
    val handle = request("url")

    println(handle)
}
```

* 如果不使用SupervisorJob，在子协程产生的异常会传递给父协程的CoroutineExceptionHandler处理，子协程不会处理CoroutineExceptionHandler；
* 如果使用SupervisorJob，在产生异常时则不会把异常传递给其他协程造成协程取消，SupervisorJob下捕捉的异常会交给CoroutineExceptionHandler处理；
* CoroutineExceptionHandler只对launch有效，而async的异常会放在Deferred中，调用Deferred.await时需要try-catch；
* 子协程同时抛出多个异常时，CoroutineExceptionHandler只会捕捉第一个异常，后续的异常会存储在第一个异常的suppressed数组之中；
* 取消协程时会抛出CancellationException，但是所有的CoroutineExceptionHandler不会接收到，只能通过try-catch实现捕获；


## 源码分析

以这段代码为例
```
val mainScope = MainScope()

mainScope.launch {
    val handle = request("url")

    println(handle)
}

private suspend fun request(param: String): String {
    delay(1000)
    return param
}
```

## 协程启动
```
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    //根据上下文创建一个新的CoroutineContext
    //分析newCoroutineContext
    val newContext = newCoroutineContext(context)
    //根据协程启动模式，创建AbstractCoroutine的实现类
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    //开始协程
    //分析coroutine.start
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

分析newCoroutineContext
```
public actual fun CoroutineScope.newCoroutineContext(context: CoroutineContext): CoroutineContext {
    val combined = coroutineContext + context
    val debug = if (DEBUG) combined + CoroutineId(COROUTINE_ID.incrementAndGet()) else combined
    return if (combined !== Dispatchers.Default && combined[ContinuationInterceptor] == null)
        debug + Dispatchers.Default else debug
}
```

分析coroutine.start
```
//AbstractCoroutine
/**
 * Starts this coroutine with the given code [block] and [start] strategy.
 * This function shall be invoked at most once on this coroutine.
 * 
 * * [DEFAULT] uses [startCoroutineCancellable].
 * * [ATOMIC] uses [startCoroutine].
 * * [UNDISPATCHED] uses [startCoroutineUndispatched].
 * * [LAZY] does nothing.
 */
public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
    start(block, receiver, this)
}
```
> start方法追溯不到下一个方法，从注释中得知[DEFAULT] uses [startCoroutineCancellable]，默认CoroutineStart是DEFAULT

在Cancellable.kt中找到startCoroutineCancellable方法
```
@InternalCoroutinesApi
public fun <T> (suspend () -> T).startCoroutineCancellable(completion: Continuation<T>): Unit = runSafely(completion) {
    //1、createCoroutineUnintercepted
    //2、intercepted
    //3、resumeCancellableWith
    createCoroutineUnintercepted(completion).intercepted().resumeCancellableWith(Result.success(Unit))
}
```

1、createCoroutineUnintercepted
```
//IntrinsicsKt
public actual fun <T> (suspend () -> T).createCoroutineUnintercepted(
    completion: Continuation<T>
): Continuation<Unit> {
    //创建Continuation<*>
    val probeCompletion = probeCoroutineCreated(completion)
    return if (this is BaseContinuationImpl)
        //跟踪create
        create(probeCompletion)
    else
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function1<Continuation<T>, Any?>).invoke(it)
        }
}
```

跟踪create，到BaseContinuationImpl#create，没有具体实现
```
public open fun create(completion: Continuation<*>): Continuation<Unit> {
        throw UnsupportedOperationException("create(Continuation) has not been overridden")
}
```

查看BaseContinuationImpl的继承，发现ContinuationImpl是其子类，并且注释中`named suspend functions extend from this class`
```
// State machines for named suspend functions extend from this class
internal abstract class ContinuationImpl(
    completion: Continuation<Any?>?,
    private val _context: CoroutineContext?
) : BaseContinuationImpl(completion) {
    ···
    
    ···
}
```

ContinuationImpl的子类SuspendLambda，
```
// Suspension lambdas inherit from this class
internal abstract class SuspendLambda(
    public override val arity: Int,
    completion: Continuation<Any?>?
) : ContinuationImpl(completion), FunctionBase<Any?>, SuspendFunction {
    constructor(arity: Int) : this(arity, null)

    public override fun toString(): String =
        if (completion == null)
            Reflection.renderLambdaToString(this) // this is lambda
        else
            super.toString() // this is continuation
}
```

> 最终createCoroutineUnintercepted(completion)创建的是SuspendLambda

2、intercepted
```
//this是ContinuationImpl
public actual fun <T> Continuation<T>.intercepted(): Continuation<T> =
    (this as? ContinuationImpl)?.intercepted() ?: this
```

ContinuationImpl
```
internal abstract class ContinuationImpl(
    completion: Continuation<Any?>?,
    private val _context: CoroutineContext?
) : BaseContinuationImpl(completion) {
    constructor(completion: Continuation<Any?>?) : this(completion, completion?.context)

    public override val context: CoroutineContext
        get() = _context!!

    @Transient
    private var intercepted: Continuation<Any?>? = null

    //调用ContinuationInterceptor.interceptContinuation，CoroutineDispatcher是ContinuationInterceptor的实现
    public fun intercepted(): Continuation<Any?> =
        intercepted
            ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
                .also { intercepted = it }

    ···
}
```

CoroutineDispatcher.interceptContinuation
```
public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        DispatchedContinuation(this, continuation)
```

DispatchedContinuation
```
internal class DispatchedContinuation<in T>(
    @JvmField val dispatcher: CoroutineDispatcher,
    @JvmField val continuation: Continuation<T>
) : DispatchedTask<T>(MODE_UNINITIALIZED), CoroutineStackFrame, Continuation<T> by continuation {

    ···
}
```

> 最后createCoroutineUnintercepted(completion).intercepted()返回DispatchedContinuation

3、resumeCancellableWith
```
inline fun resumeCancellableWith(
    result: Result<T>,
    noinline onCancellation: ((cause: Throwable) -> Unit)?
) {
    val state = result.toState(onCancellation)
    if (dispatcher.isDispatchNeeded(context)) {
        _state = state
        resumeMode = MODE_CANCELLABLE
        //this是DispatchedTask，实现了runnable接口
        //dispatch方法交由子类实现，最终会调用runnable.run
        //分析dispatcher.dispatch
        dispatcher.dispatch(context, this)
    } else {
        executeUnconfined(state, MODE_CANCELLABLE) {
            if (!resumeCancelled(state)) {
                resumeUndispatchedWith(result)
            }
        }
    }
}
```

分析dispatcher.dispatch
```
//block是一个Runnable，传入的this是父类DispatchedTask
public abstract fun dispatch(context: CoroutineContext, block: Runnable)
```

查看DispatchedTask的run
```
public final override fun run() {
    assert { resumeMode != MODE_UNINITIALIZED } // should have been set before dispatching
    val taskContext = this.taskContext
    var fatalException: Throwable? = null
    try {
        val delegate = delegate as DispatchedContinuation<T>
        val continuation = delegate.continuation
        withContinuationContext(continuation, delegate.countOrElement) {
            val context = continuation.context
            val state = takeState() // NOTE: Must take state in any case, even if cancelled
            val exception = getExceptionalResult(state)
            /*
             * Check whether continuation was originally resumed with an exception.
             * If so, it dominates cancellation, otherwise the original exception
             * will be silently lost.
             */
            //检查当前job是否异常，如果存在异常则取消job
            val job = if (exception == null && resumeMode.isCancellableMode) context[Job] else null
            if (job != null && !job.isActive) {
                val cause = job.getCancellationException()
                cancelCompletedResult(state, cause)
                continuation.resumeWithStackTrace(cause)
            } else {
                //
                if (exception != null) {
                    //失败回调
                    continuation.resumeWithException(exception)
                } else {
                    //分析continuation.resume
                    continuation.resume(getSuccessfulResult(state))
                }
            }
        }
    } catch (e: Throwable) {
        // This instead of runCatching to have nicer stacktrace and debug experience
        fatalException = e
    } finally {
        val result = runCatching { taskContext.afterTask() }
        handleFatalException(fatalException, result.exceptionOrNull())
    }
}
```

continuation从delegate.continuation获取，而delegate是DispatchedContinuation<T> 
而DispatchedContinuation使用了类委托`by continuation`，实际上是调用了Continuation.resume
```
internal class DispatchedContinuation<in T>(
    @JvmField val dispatcher: CoroutineDispatcher,
    @JvmField val continuation: Continuation<T>
) : DispatchedTask<T>(MODE_UNINITIALIZED), CoroutineStackFrame, Continuation<T> by continuation {
    ···
}
```

Continuation.resume又调用了自己的resumeWith，即还是调用DispatchedContinuation.resumeWith
```
public inline fun <T> Continuation<T>.resume(value: T): Unit =
    resumeWith(Result.success(value))
```

DispatchedContinuation.resumeWith
```
override fun resumeWith(result: Result<T>) {
    val context = continuation.context
    val state = result.toState()
    if (dispatcher.isDispatchNeeded(context)) {
        _state = state
        resumeMode = MODE_ATOMIC
        dispatcher.dispatch(context, this)
    } else {
        executeUnconfined(state, MODE_ATOMIC) {
            withCoroutineContext(this.context, countOrElement) {
                continuation.resumeWith(result)
            }
        }
    }
}
```

> 实际上是把Continuation包装成DispatchedContinuation，使用Dispatcher.dispatch运行DispatchedContinuation中的Continuation.resumeWith

继承关系：Continuation -> BaseContinuationImpl -> ContinuationImpl -> SuspendLambda 

BaseContinuationImpl实现了resumeWith方法
```
internal abstract class BaseContinuationImpl(
    public val completion: Continuation<Any?>?
) : Continuation<Any?>, CoroutineStackFrame, Serializable {
    // This implementation is final. This fact is used to unroll resumeWith recursion.
    public final override fun resumeWith(result: Result<Any?>) {
        // This loop unrolls recursion in current.resumeWith(param) to make saner and shorter stack traces on resume
        var current = this
        var param = result
        while (true) {
            // Invoke "resume" debug probe on every resumed continuation, so that a debugging library infrastructure
            // can precisely track what part of suspended callstack was already resumed
            probeCoroutineResumed(current)
            with(current) {
                val completion = completion!! // fail fast when trying to resume continuation without completion
                val outcome: Result<Any?> =
                    try {
                        //执行suspend代码块
                        val outcome = invokeSuspend(param)
                        //如果是suspend挂起函数，则直接返回
                        if (outcome === COROUTINE_SUSPENDED) return
                        Result.success(outcome)
                    } catch (exception: Throwable) {
                        Result.failure(exception)
                    }
                releaseIntercepted() // this state machine instance is terminating
                if (completion is BaseContinuationImpl) {
                    // unrolling recursion via loop
                    //继续循环完成子completion的代码
                    current = completion
                    param = outcome
                } else {
                    // top-level completion reached -- invoke and return
                    //resumeWith执行剩下的代码
                    completion.resumeWith(outcome)
                    return
                }
            }
        }
    }
    
    ···
}
```