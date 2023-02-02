协程
来源： https://juejin.cn/post/6950616789390721037

在Android平台上，协程主要用来解决两个问题：
* 处理耗时任务
* 保证主线程安全

创建协程的两种方式：
* CoroutineScope.launch()  
创建新协程，不带返回值


* CoroutineScope.async()  
 创建新协程，允许使用await挂起函数返回结果，
调用await来获取结果或者异常，所以默认情况下不会抛出异常；
不使用await，则静默地丢弃异常。

- launch和async的最大差异在对异常的处理方式不同

取消协程问题
如果使用lifecycleScope(或者viewModelScope)则不需要手动取消，
否则。需要手动取消协程，防止协程泄漏 CoroutineScope#cancel()

详细分析async
* 获取返回值
开启一个新协程，并返回一个Deferred对象，Deferred可以获取返回值或者异常，
代码执行会往下走，需要配合await()挂起函数配合使用。
* 并发
使用async函数创建多个Deferred，使用await函数执行获取结果，实现并发场景。


launch函数参数分别是：
* CoroutineContext协程上下文
* CoroutineStart协程启动模式
* (suspend CoroutineScope.() -> Unit)协程体
返回值是协程实例Job
其中CoroutineContext又包括了Job、CoroutineDispatcher、CoroutineName。


CoroutineContext-协程上下文
* Job: 控制协程的生命周期；
* CoroutineDispatcher: 向合适的线程分发任务；
* CoroutineName: 协程的名称，调试的时候很有用；
* CoroutineExceptionHandler: 处理未被捕捉的异常。

CoroutineContext 有两个非常重要的元素 — Job 和 Dispatcher
Job 是当前的 Coroutine 实例，而 Dispatcher 决定了当前 Coroutine 执行的线程，还可以添加CoroutineName，用于调试，添加 CoroutineExceptionHandler 用于捕获异常，它们都实现了Element接口。

  在任务层级中，每个协程都会有一个父级对象，要么是 CoroutineScope 或者另外一个 coroutine。然而，实际上协程的父级 CoroutineContext 和父级协程的 CoroutineContext 是不一样的，因为有如下的公式:
父级上下文 = 默认值 + 继承的 CoroutineContext + 参数
其中:
* 一些元素包含默认值: Dispatchers.Default 是默认的 CoroutineDispatcher，以及 "coroutine" 作为默认的 CoroutineName；
* 继承的 CoroutineContext 是 CoroutineScope 或者其父协程的 CoroutineContext；
* 传入协程 builder 的参数的优先级高于继承的上下文参数，因此会覆盖对应的参数值。

Job
Job 用于处理协程。对于每一个所创建的协程 (通过 launch 或者 async)，它会返回一个 Job实例，该实例是协程的唯一标识，并且负责管理协程的生命周期
  CoroutineScope.launch 函数返回的是一个 Job 对象，代表一个异步的任务。Job 具有生命周期并且可以取消。 Job 还可以有层级关系，一个Job可以包含多个子Job，当父Job被取消后，所有的子Job也会被自动取消；当子Job被取消或者出现异常后父Job也会被取消。
  除了通过 CoroutineScope.launch 来创建Job对象之外，还可以通过 Job() 工厂方法来创建该对象。默认情况下，子Job的失败将会导致父Job被取消，这种默认的行为可以通过 SupervisorJob 来修改。
  具有多个子 Job 的父Job 会等待所有子Job完成(或者取消)后，自己才会执行完成。


job常用函数
* fun start(): Boolean 调用该函数来启动这个 Coroutine，如果当前 Coroutine 还没有执行调用该函数返回 true，如果当前 Coroutine 已经执行或者已经执行完毕，则调用该函数返回 false
* fun cancel(cause: CancellationException? = null) 通过可选的取消原因取消此作业。 原因可以用于指定错误消息或提供有关取消原因的其他详细信息，以进行调试。
* fun invokeOnCompletion(handler: CompletionHandler): DisposableHandle 通过这个函数可以给 Job 设置一个完成通知，当 Job 执行完成的时候会同步执行这个通知函数。 回调的通知对象类型为：typealias CompletionHandler = (cause: Throwable?) -> Unit. CompletionHandler 参数代表了 Job 是如何执行完成的。 cause 有下面三种情况：
    * 如果 Job 是正常执行完成的，则 cause 参数为 null
    * 如果 Job 是正常取消的，则 cause 参数为 CancellationException 对象。这种情况不应该当做错误处理，这是任务正常取消的情形。所以一般不需要在错误日志中记录这种情况。
    * 其他情况表示 Job 执行失败了。
		这个函数的返回值为 DisposableHandle 对象，如果不再需要监控 Job 的完成情况了， 则可以调用 			DisposableHandle.dispose 函数来取消监听。如果 Job 已经执行完了， 则无需调用 dispose 函数了，会自动取	消监听。
* suspend fun join()
	join 函数和前面三个函数不同，这是一个 suspend 函数。所以只能在 Coroutine 内调用。
	这个函数会暂停当前所处的 Coroutine，直到该子Coroutine(调用join的Coroutine)执行完成。所以 join 函数一般用来在一个Coroutine 中等待 job 执行完成后继续向下执行。当 Job 执行完成后， job.join 函数恢复，这个时候 job 这个任务已经处于完成状态了，而调用 job.join 的 Coroutine 还继续处于 activie 状态。
	请注意，只有在其所有子级都完成后，作业才能完成。
	该函数的挂起是可以被取消的，并且始终检查调用的Coroutine的Job是否取消。如果在调用此挂起函数或将其	挂起时，调用Coroutine的Job被取消或完成，则此函数将引发 CancellationException。


Deferred常用函数
	通过使用async创建协程可以得到一个有返回值Deferred，Deferred 接口继承自 Job 接口，额外提供了获取 Coroutine 返回结果的方法。由于 Deferred 继承自 Job 接口，所以 Job 相关的内容在 Deferred 上也是适用的。 Deferred 提供了额外三个函数来处理和Coroutine执行结果相关的操作。
* suspend fun await(): T 用来等待这个Coroutine执行完毕并返回结果。
* fun getCompleted(): T 用来获取Coroutine执行的结果。如果Coroutine还没有执行完成则会抛出 IllegalStateException ，如果任务被取消了也会抛出对应的异常。所以在执行这个函数之前，可以通过 isCompleted 来判断一下当前任务是否执行完毕了。
* fun getCompletionExceptionOrNull(): Throwable? 获取已完成状态的Coroutine异常信息，如果任务正常执行完成了，则不存在异常信息，返回null。如果还没有处于已完成状态，则调用该函数同样会抛出 IllegalStateException，可以通过 isCompleted 来判断一下当前任务是否执行完毕了。


CoroutineDispatcher-调度器
	CoroutineDispatcher 定义了 Coroutine 执行的线程。CoroutineDispatcher 可以限定 Coroutine 在某一个线程执行、也可以分配到一个线程池来执行、也可以不限制其执行的线程。
  CoroutineDispatcher 是一个抽象类，所有 dispatcher 都应该继承这个类来实现对应的功能。Dispatchers 是一个标准库中帮我们封装了切换线程的帮助类，可以简单理解为一个线程池。

* Dispatchers.Default 默认的调度器，适合处理后台计算，是一个CPU密集型任务调度器。如果创建 Coroutine 的时候没有指定 dispatcher，则一般默认使用这个作为默认值。Default dispatcher 使用一个共享的后台线程池来运行里面的任务。注意它和IO共享线程池，只不过限制了最大并发数不同。
* Dispatchers.IO 顾名思义这是用来执行阻塞 IO 操作的，是和Default共用一个共享的线程池来执行里面的任务。根据同时运行的任务数量，在需要的时候会创建额外的线程，当任务执行完毕后会释放不需要的线程。
* Dispatchers.Unconfined 由于Dispatchers.Unconfined未定义线程池，所以执行的时候默认在启动线程。遇到第一个挂起点，之后由调用resume的线程决定恢复协程的线程。
* Dispatchers.Main： 指定执行的线程是主线程，在Android上就是UI线程。

为什么子Coroutine 为指定Dispatcher时会继承父Coroutine的Dispatcher？
由于子Coroutine 会继承父Coroutine 的 context，所以为了方便使用，我们一般会在 父Coroutine 上设定一个 Dispatcher，然后所有 子Coroutine 自动使用这个 Dispatcher。


CoroutineStart-协程启动模式
* CoroutineStart.DEFAULT:
	协程创建后立即开始调度，在调度前如果协程被取消(cancel)，其将直接进入取消响应的状态
	虽然是立即调度，但也有可能在执行前被取消
* CoroutineStart.ATOMIC:
	协程创建后立即开始调度，协程执行到第一个挂起点(suspend)之前不响应取消
	虽然是立即调度，但其将调度和执行两个步骤合二为一了，就像它的名字一样，其保证调度和执行是原子操作，	因此协程也一定会执行
* CoroutineStart.LAZY:
	只要协程被需要时，包括主动调用该协程的start、join或者await等函数时才会开始调度，如果调度前就被取		消，协程将直接进入异常结束状态
* CoroutineStart.UNDISPATCHED:
	协程创建后立即在当前函数调用栈中执行，执行挂起点之前的代码（包括父Coroutine和该Coroutine挂起点之前的代码），直到遇到第一个真正的挂起点(suspend)，继续按顺序执行调用站外的代码，是否继续执行挂起点之后的代码取决于逻辑（调用job.join()、start()等）。
	是立即执行，因此协程一定会执行。


CoroutineScope-协程作用域

CoroutineScope是一个接口

CoroutineScope 只是定义了一个新 Coroutine 的执行 Scope。每个 coroutine builder 都是 CoroutineScope 的扩展函数，并且自动的继承了当前 Scope 的 coroutineContext 。

	官方框架在实现复合协程的过程中也提供了作用域，主要用以明确写成之间的父子关系，以及对于取消或者异常处理等方面的传播行为。该作用域包括以下三种：
* 顶级作用域 没有父协程的协程所在的作用域为顶级作用域。
* 协同作用域 协程中启动新的协程，新协程为所在协程的子协程，这种情况下，子协程所在的作用域默认为协同作用域。此时子协程抛出的未捕获异常，都将传递给父协程处理，父协程同时也会被取消。
* 主从作用域 与协同作用域在协程的父子关系上一致，区别在于，处于该作用域下的协程出现未捕获的异常时，不会将异常向上传递给父协程。
除了三种作用域中提到的行为以外，父子协程之间还存在以下规则：
* 父协程被取消，则所有子协程均被取消。由于协同作用域和主从作用域中都存在父子协程关系，因此此条规则都适用。
* 父协程需要等待子协程执行完毕之后才会最终进入完成状态，不管父协程自身的协程体是否已经执行完。
* 子协程会继承父协程的协程上下文中的元素，如果自身有相同key的成员，则覆盖对应的key，覆盖的效果仅限自身范围内有效。


GlobalScope - 不推荐使用
GlobalScope是一个单例实现，源码十分简单，上下文是EmptyCoroutineContext，是一个空的上下文，切不包含任何Job，该作用域常被拿来做示例代码，由于 GlobalScope 对象没有和应用生命周期组件相关联，需要自己管理 GlobalScope 所创建的 Coroutine，且GlobalScope的生命周期是 process 级别的，所以一般而言我们不推荐使用 GlobalScope 来创建 Coroutine。

runBlocking{} - 主要用于测试
这是一个顶层函数，从源码的注释中我们可以得到一些信息，运行一个新的协程并且阻塞当前可中断的线程直至协程执行完成，该函数不应从一个协程中使用，该函数被设计用于桥接普通阻塞代码到以挂起风格（suspending style）编写的库，以用于主函数与测试。该函数主要用于测试，不适用于日常开发，该协程会阻塞当前线程直到协程体执行完成。

MainScope() - 可用于开发
该函数是一个顶层函数，用于返回一个上下文是SupervisorJob() + Dispatchers.Main的作用域，该作用域常被使用在Activity/Fragment，并且在界面销毁时要调用fun CoroutineScope.cancel(cause: CancellationException? = null)对协程进行取消，这是官方库中可以在开发中使用的一个用于获取作用域的顶层函数，使用示例在官方库的代码注释中已经给出，上面的源码中也有，使用起来也是十分的方便。

LifecycleOwner.lifecycleScope - 推荐使用
该扩展属性是 Android 的Lifecycle Ktx库提供的具有生命周期感知的协程作用域，它与LifecycleOwner的Lifecycle绑定，Lifecycle被销毁时，此作用域将被取消。这是在Activity/Fragment中推荐使用的作用域，因为它会与当前的UI组件绑定生命周期，界面销毁时该协程作用域将被取消，不会造成协程泄漏。

ViewModel.viewModelScope - 推荐使用
它是ViewModel的扩展属性，也是来自Android 的Lifecycle Ktx库，它能够在此ViewModel销毁时自动取消，同样不会造成协程泄漏。

coroutineScope & supervisorScope（挂起函数）
首先这两个函数都是挂起函数，需要运行在协程内或挂起函数内。supervisorScope属于主从作用域，会继承父协程的上下文，它的特点就是子协程的异常不会影响父协程，它的设计应用场景多用于子协程为独立对等的任务实体的时候，比如一个下载器，每一个子协程都是一个下载任务，当一个下载任务异常时，它不应该影响其他的下载任务。coroutineScope和supervisorScope都会返回一个作用域，它俩的差别就是异常传播：coroutineScope 内部的异常会向上传播，子协程未捕获的异常会向上传递给父协程，任何一个子协程异常退出，会导致整体的退出；supervisorScope 内部的异常不会向上传播，一个子协程异常退出，不会影响父协程和兄弟协程的运行。

