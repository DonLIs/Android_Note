# OkHttp3源码分析

## OkHttp的创建方式
使用无参的构造函数创建
```
val okHttpClient = OkHttpClient()
```

```
//无参构造函数会调用自身的有参构造，传入创建的Builder对象
constructor() : this(Builder())

open class OkHttpClient internal constructor(
  builder: Builder
) : Cloneable, Call.Factory, WebSocket.Factory {
    ···
}

//Builder对象包含了一些默认值，可以使用Builder.build()构建者模式自定义设置属性。
class Builder constructor() {
    //调度Call执行同步/异步操作
    internal var dispatcher: Dispatcher = Dispatcher()
    //连接池
    internal var connectionPool: ConnectionPool = ConnectionPool()
    //拦截器列表
    internal val interceptors: MutableList<Interceptor> = mutableListOf()
    //网路拦截器列表
    internal val networkInterceptors: MutableList<Interceptor> = mutableListOf()
    //事件监听器工厂
    internal var eventListenerFactory: EventListener.Factory = EventListener.NONE.asFactory()
    //
    internal var retryOnConnectionFailure = true
    internal var authenticator: Authenticator = Authenticator.NONE
    internal var followRedirects = true
    internal var followSslRedirects = true
    //cookie
    internal var cookieJar: CookieJar = CookieJar.NO_COOKIES
    //缓存
    internal var cache: Cache? = null
    //dns解析
    internal var dns: Dns = Dns.SYSTEM
    internal var proxy: Proxy? = null
    //代理选择器
    internal var proxySelector: ProxySelector? = null
    internal var proxyAuthenticator: Authenticator = Authenticator.NONE
    //socket工厂
    internal var socketFactory: SocketFactory = SocketFactory.getDefault()
    internal var sslSocketFactoryOrNull: SSLSocketFactory? = null
    internal var x509TrustManagerOrNull: X509TrustManager? = null
    //连接配置
    internal var connectionSpecs: List<ConnectionSpec> = DEFAULT_CONNECTION_SPECS
    //http协议版本
    internal var protocols: List<Protocol> = DEFAULT_PROTOCOLS
    internal var hostnameVerifier: HostnameVerifier = OkHostnameVerifier
    internal var certificatePinner: CertificatePinner = CertificatePinner.DEFAULT
    internal var certificateChainCleaner: CertificateChainCleaner? = null
    internal var callTimeout = 0
    //连接超时
    internal var connectTimeout = 10_000
    //读取超时
    internal var readTimeout = 10_000
    //写入超时
    internal var writeTimeout = 10_000
    //webSocket的ping间隔
    internal var pingInterval = 0
    internal var minWebSocketMessageToCompress = RealWebSocket.DEFAULT_MINIMUM_DEFLATE_SIZE
    internal var routeDatabase: RouteDatabase? = null
    
    ···
    
    fun build(): OkHttpClient = OkHttpClient(this)
}
```

使用OkHttpClient.Builder()构建者模式设置每个属性来构建。
```
val okHttpClient = OkHttpClient.Builder()
            .connectTimeout(10, TimeUnit.SECONDS)
            .readTimeout(10, TimeUnit.SECONDS)
            .build()
```

使用newBuilder复制OkHttpClient实例的默认实现。
```
val okHttpClient = OkHttpClient().newBuilder()
            .connectTimeout(10, TimeUnit.SECONDS)
            .readTimeout(10, TimeUnit.SECONDS)
            .build()
```


## 同步请求

同步请求是线程阻塞。需要在子线程中执行
```
thread {
    //创建Request
    val request = Request.Builder()
        .url("")
        .build()
    //使用newCall创建Call实例
    val call = okHttpClient.newCall(request)
    //执行请求，并获取响应结果，此处阻塞线程
    val response = call.execute()
}
```

使用Request.Builder创建Request对象
```
open class Builder {
    //请求地址
    internal var url: HttpUrl? = null
    //请求方式
    internal var method: String
    //请求头
    internal var headers: Headers.Builder
    //post的请求体
    internal var body: RequestBody? = null

    /** A mutable map of tags, or an immutable empty map if we don't have any. */
    internal var tags: MutableMap<Class<*>, Any> = mutableMapOf()

    constructor() {
      //默认是GET请求
      this.method = "GET"
      this.headers = Headers.Builder()
    }
    
    open fun build(): Request {
        return Request(
            //检查url不能为null
            checkNotNull(url) { "url == null" },
            method,
            headers.build(),
            body,
            tags.toImmutableMap()
        )
    }
```

查看OkHttpClient的newCall方法
```
//newCall返回RealCall对象，forWebSocket：是否是websocket
override fun newCall(request: Request): Call = RealCall(this, request, forWebSocket = false)
```

Call#execute()，执行同步操作
```
override fun execute(): Response {
    //检查是否已经执行
    check(executed.compareAndSet(false, true)) { "Already Executed" }
    
    timeout.enter()
    //回调事件监听的callStart
    callStart()
    try {
      //把RealCall添加到Dispatcher的runningSyncCalls中
      //runningSyncCalls：ArrayDeque<RealCall>
      client.dispatcher.executed(this)
      //执行拦截器并返回响应体
      return getResponseWithInterceptorChain()
    } finally {
      //runningSyncCalls移除RealCall
      client.dispatcher.finished(this)
    }
}

//Dispatcher#executed
@Synchronized internal fun executed(call: RealCall) {
    //runningSyncCalls：缓存同步任务
    runningSyncCalls.add(call)
}

//Dispatcher#finished
internal fun finished(call: RealCall) {
    //传入runningSyncCalls
    finished(runningSyncCalls, call)
}

private fun <T> finished(calls: Deque<T>, call: T) {
    val idleCallback: Runnable?
    synchronized(this) {
      //移除RealCall
      if (!calls.remove(call)) throw AssertionError("Call wasn't in-flight!")
      idleCallback = this.idleCallback
    }
    
    //查看异步请求中对promoteAndExecute的分析
    val isRunning = promoteAndExecute()
    
    if (!isRunning && idleCallback != null) {
      idleCallback.run()
    }
}
```

## 异步请求

异步请求不需要创建线程，只需要添加接口回调，回调的方法运行在子线程。
```
val request = Request.Builder()
    .url("")
    .build()
val call = okHttpClient.newCall(request)
call.enqueue(object : Callback {
    override fun onFailure(call: Call, e: IOException) {
        //回调：请求失败
        e.message
    }

    override fun onResponse(call: Call, response: Response) {
        //回调：请求成功
        response.body?.string()
    }

})
```

查看enqueue方法
```
override fun enqueue(responseCallback: Callback) {
    //检查是否已经执行了
    check(executed.compareAndSet(false, true)) { "Already Executed" }

    callStart()
    //创建AsyncCall来包装Callback，AsyncCall是Runnable接口的实现，是RealCall的内部类
    client.dispatcher.enqueue(AsyncCall(responseCallback))
}

internal fun enqueue(call: AsyncCall) {
    synchronized(this) {
      //readyAsyncCalls：ArrayDeque<AsyncCall>
      //把AsyncCall放入待执行的队列中
      readyAsyncCalls.add(call)

      // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
      // the same host.
      if (!call.call.forWebSocket) {
        //找到同一个host并且存活的AsyncCall
        val existingCall = findExistingCallWithHost(call.host)
        //
        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
      }
    }
    //把readyAsyncCalls队列中的Call添加到runningAsyncCalls队列中
    //并添加到线程池中执行
    promoteAndExecute()
}
```


重点查看getResponseWithInterceptorChain方法
```
@Throws(IOException::class)
internal fun getResponseWithInterceptorChain(): Response {
    // Build a full stack of interceptors.
    val interceptors = mutableListOf<Interceptor>()
    //添加自定义的拦截器
    interceptors += client.interceptors
    //添加重试跟踪拦截器
    interceptors += RetryAndFollowUpInterceptor(client)
    //添加桥拦截器
    interceptors += BridgeInterceptor(client.cookieJar)
    //添加缓存拦截器
    interceptors += CacheInterceptor(client.cache)
    //添加连接拦截器
    interceptors += ConnectInterceptor
    //如果不是websocket，添加网络拦截器
    if (!forWebSocket) {
      interceptors += client.networkInterceptors
    }
    //请求拦截器，主要是报文的封装和报文的解析
    interceptors += CallServerInterceptor(forWebSocket)
    
    //实例一个RealInterceptorChain对象
    val chain = RealInterceptorChain(
        call = this,
        interceptors = interceptors,
        index = 0,
        exchange = null,
        request = originalRequest,
        connectTimeoutMillis = client.connectTimeoutMillis,
        readTimeoutMillis = client.readTimeoutMillis,
        writeTimeoutMillis = client.writeTimeoutMillis
    )
    
    var calledNoMoreExchanges = false
    try {
      //利用责任链模式，调度每一个拦截器，最后返回响应体
      val response = chain.proceed(originalRequest)
      //如果取消了，关闭响应体
      if (isCanceled()) {
        response.closeQuietly()
        throw IOException("Canceled")
      }
      return response
    } catch (e: IOException) {
      calledNoMoreExchanges = true
      throw noMoreExchanges(e) as Throwable
    } finally {
      if (!calledNoMoreExchanges) {
        noMoreExchanges(null)
      }
    }
}

//RealInterceptorChain实现了Interceptor.Chain接口
//核心方法
@Throws(IOException::class)
override fun proceed(request: Request): Response {
    //检查索引
    check(index < interceptors.size)
    
    calls++
    
    if (exchange != null) {
      check(exchange.finder.sameHostAndPort(request.url)) {
        "network interceptor ${interceptors[index - 1]} must retain the same host and port"
      }
      check(calls == 1) {
        "network interceptor ${interceptors[index - 1]} must call proceed() exactly once"
      }
    }
    
    // Call the next interceptor in the chain.
    //获取下一个拦截器索引的RealInterceptorChain实例
    val next = copy(index = index + 1, request = request)
    //当前拦截器
    val interceptor = interceptors[index]
    
    @Suppress("USELESS_ELVIS")
    //传入RealInterceptorChain实例
    //执行拦截器的intercept方法，intercept中继续调用RealInterceptorChain的proceed方法'realChain.proceed(request)'
    //索引递增，获取下一个拦截器
    //如此下去，执行全部拦截器并依次返回Response响应体
    val response = interceptor.intercept(next) ?: throw NullPointerException(
        "interceptor $interceptor returned null")
    
    if (exchange != null) {
      check(index + 1 >= interceptors.size || next.calls == 1) {
        "network interceptor $interceptor must call proceed() exactly once"
      }
    }
    
    check(response.body != null) { "interceptor $interceptor returned a response with no body" }
    
    return response
}

```

## 查看OkHttpClient一些默认添加的拦截器

拦截器需要实现Interceptor接口

---

RetryAndFollowUpInterceptor，重试与重定向拦截器
```
class RetryAndFollowUpInterceptor(private val client: OkHttpClient) : Interceptor {

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    //RealInterceptorChain
    val realChain = chain as RealInterceptorChain
    //Request
    var request = chain.request
    val call = realChain.call
    var followUpCount = 0
    var priorResponse: Response? = null
    var newExchangeFinder = true
    var recoveredFailures = listOf<IOException>()
    while (true) {
      call.enterNetworkInterceptorExchange(request, newExchangeFinder)

      var response: Response
      var closeActiveExchange = true
      try {
        //已取消则抛出异常
        if (call.isCanceled()) {
          throw IOException("Canceled")
        }

        try {
          response = realChain.proceed(request)
          newExchangeFinder = true
        } catch (e: RouteException) {
          // The attempt to connect via a route failed. The request will not have been sent.
          //尝试通过路由进行连接失败。该请求将不会被发送。
          if (!recover(e.lastConnectException, call, request, requestSendStarted = false)) {
            throw e.firstConnectException.withSuppressed(recoveredFailures)
          } else {
            recoveredFailures += e.firstConnectException
          }
          newExchangeFinder = false
          //继续重试
          continue
        } catch (e: IOException) {
          // An attempt to communicate with a server failed. The request may have been sent.
          //尝试与服务器通信失败。请求可能已发送。
          if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
            throw e.withSuppressed(recoveredFailures)
          } else {
            recoveredFailures += e
          }
          newExchangeFinder = false
          continue
        }

        // Attach the prior response if it exists. Such responses never have a body.
        //如果之前的响应体存在，则添加
        //响应体没有body
        if (priorResponse != null) {
          response = response.newBuilder()
              .priorResponse(priorResponse.newBuilder() //复制一个Response
                  .body(null)  //设置body为null
                  .build())
              .build()
        }

        val exchange = call.interceptorScopedExchange
        val followUp = followUpRequest(response, exchange)

        if (followUp == null) {
          if (exchange != null && exchange.isDuplex) {
            call.timeoutEarlyExit()
          }
          closeActiveExchange = false
          return response
        }

        val followUpBody = followUp.body
        if (followUpBody != null && followUpBody.isOneShot()) {
          closeActiveExchange = false
          return response
        }

        response.body?.closeQuietly()

        if (++followUpCount > MAX_FOLLOW_UPS) {
          throw ProtocolException("Too many follow-up requests: $followUpCount")
        }

        request = followUp
        priorResponse = response
      } finally {
        call.exitNetworkInterceptorExchange(closeActiveExchange)
      }
    }
  }
 
 ··· 
}
```

BridgeInterceptor，桥拦截器
```
class BridgeInterceptor(private val cookieJar: CookieJar) : Interceptor {

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val userRequest = chain.request()
    //复制一个新的请求
    val requestBuilder = userRequest.newBuilder()

    val body = userRequest.body
    //请求体
    if (body != null) {
      val contentType = body.contentType()
      //设置Content-Type
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString())
      }

      val contentLength = body.contentLength()
      //内容长度
      if (contentLength != -1L) {
        requestBuilder.header("Content-Length", contentLength.toString())
        requestBuilder.removeHeader("Transfer-Encoding")
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked")
        requestBuilder.removeHeader("Content-Length")
      }
    }

    //Host
    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", userRequest.url.toHostHeader())
    }

    //Connection
    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive")
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    var transparentGzip = false
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true
      requestBuilder.header("Accept-Encoding", "gzip")
    }

    //可以通过OkHttpClient的cookieJar设置Cookie
    val cookies = cookieJar.loadForRequest(userRequest.url)
    //设置Cookie
    if (cookies.isNotEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies))
    }

    //User-Agent
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", userAgent)
    }

    //执行下一个拦截器
    val networkResponse = chain.proceed(requestBuilder.build())

    //接收响应体的Cookie
    cookieJar.receiveHeaders(userRequest.url, networkResponse.headers)

    //复制一个响应体，记录原来的请求对象
    val responseBuilder = networkResponse.newBuilder()
        .request(userRequest)

    if (transparentGzip &&
        "gzip".equals(networkResponse.header("Content-Encoding"), ignoreCase = true) &&
        networkResponse.promisesBody()) {
      val responseBody = networkResponse.body
      if (responseBody != null) {
        val gzipSource = GzipSource(responseBody.source())
        val strippedHeaders = networkResponse.headers.newBuilder()
            .removeAll("Content-Encoding")
            .removeAll("Content-Length")
            .build()
        responseBuilder.headers(strippedHeaders)
        val contentType = networkResponse.header("Content-Type")
        responseBuilder.body(RealResponseBody(contentType, -1L, gzipSource.buffer()))
      }
    }

    return responseBuilder.build()
  }

  /** Returns a 'Cookie' HTTP request header with all cookies, like `a=b; c=d`. */
  private fun cookieHeader(cookies: List<Cookie>): String = buildString {
    cookies.forEachIndexed { index, cookie ->
      if (index > 0) append("; ")
      append(cookie.name).append('=').append(cookie.value)
    }
  }
}
```

CacheInterceptor，缓存拦截器
```
class CacheInterceptor(internal val cache: Cache?) : Interceptor {

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val call = chain.call()
    //备用缓存
    val cacheCandidate = cache?.get(chain.request())

    val now = System.currentTimeMillis()

    //创建缓存策略
    val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
    //networkRequest为null，则被禁止使用网络
    val networkRequest = strategy.networkRequest
    //cacheResponse为null，则不应用缓存
    val cacheResponse = strategy.cacheResponse

    cache?.trackResponse(strategy)
    val listener = (call as? RealCall)?.eventListener ?: EventListener.NONE

    if (cacheCandidate != null && cacheResponse == null) {
      // The cache candidate wasn't applicable. Close it.
      //不应用缓存，并关闭它
      cacheCandidate.body?.closeQuietly()
    }

    // If we're forbidden from using the network and the cache is insufficient, fail.
    //网络被禁用和不应用缓存，则返回一个504的响应
    if (networkRequest == null && cacheResponse == null) {
      return Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(HTTP_GATEWAY_TIMEOUT)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build().also {
            listener.satisfactionFailure(call, it)
          }
    }

    // If we don't need the network, we're done.
    //没有网络，但有缓存情况下，使用缓存响应
    if (networkRequest == null) {
      return cacheResponse!!.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build().also {
            listener.cacheHit(call, it)
          }
    }

    //一些事件回调
    if (cacheResponse != null) {
      listener.cacheConditionalHit(call, cacheResponse)
    } else if (cache != null) {
      listener.cacheMiss(call)
    }

    var networkResponse: Response? = null
    try {
      //继续下一个拦截器，返回请求响应
      networkResponse = chain.proceed(networkRequest)
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      //如果面临io崩溃，需要释放缓存资源
      if (networkResponse == null && cacheCandidate != null) {
        cacheCandidate.body?.closeQuietly()
      }
    }

    // If we have a cache response too, then we're doing a conditional get.
    //如果缓存不为null，则把请求响应和缓存合并并更新缓存
    if (cacheResponse != null) {
      if (networkResponse?.code == HTTP_NOT_MODIFIED) {
        val response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers, networkResponse.headers))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis)
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis)
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build()

        networkResponse.body!!.close()

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache!!.trackConditionalCacheHit()
        //更新缓存
        cache.update(cacheResponse, response)
        return response.also {
          listener.cacheHit(call, it)
        }
      } else {
        cacheResponse.body?.closeQuietly()
      }
    }

    val response = networkResponse!!.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build()

    if (cache != null) {
      //响应是否可以缓存，是则写入缓存
      if (response.promisesBody() && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        val cacheRequest = cache.put(response)
        return cacheWritingResponse(cacheRequest, response).also {
          if (cacheResponse != null) {
            // This will log a conditional cache miss only.
            listener.cacheMiss(call)
          }
        }
      }

      //如果缓存失效，则移除
      if (HttpMethod.invalidatesCache(networkRequest.method)) {
        try {
          cache.remove(networkRequest)
        } catch (_: IOException) {
          // The cache cannot be written.
        }
      }
    }

    return response
  }
  
  ···
}
```

ConnectInterceptor，连接拦截器
```
object ConnectInterceptor : Interceptor {
  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    //查找新连接或连接池，用来处理请求和响应
    val exchange = realChain.call.initExchange(chain)
    val connectedChain = realChain.copy(exchange = exchange)
    return connectedChain.proceed(realChain.request)
  }
}

//查看RealCall的initExchange方法，返回一个Exchange对象
internal fun initExchange(chain: RealInterceptorChain): Exchange {
    //检查一些状态参数
    synchronized(this) {
      check(expectMoreExchanges) { "released" }
      check(!responseBodyOpen)
      check(!requestBodyOpen)
    }
    
    //exchangeFinder在RetryAndFollowUpInterceptor中调用RealCall的enterNetworkInterceptorExchange初始化的
    val exchangeFinder = this.exchangeFinder!!
    //查找可用的ExchangeCodec
    val codec = exchangeFinder.find(client, chain)
    //组装Exchange对象
    val result = Exchange(this, eventListener, exchangeFinder, codec)
    this.interceptorScopedExchange = result
    this.exchange = result
    synchronized(this) {
      this.requestBodyOpen = true
      this.responseBodyOpen = true
    }
    
    if (canceled) throw IOException("Canceled")
    return result
}

//查看ExchangeFinder的find方法
fun find(
    client: OkHttpClient,
    chain: RealInterceptorChain
  ): ExchangeCodec {
    try {
      //查找可用的连接
      val resultConnection = findHealthyConnection(
          connectTimeout = chain.connectTimeoutMillis,
          readTimeout = chain.readTimeoutMillis,
          writeTimeout = chain.writeTimeoutMillis,
          pingIntervalMillis = client.pingIntervalMillis,
          connectionRetryEnabled = client.retryOnConnectionFailure,
          doExtensiveHealthChecks = chain.request.method != "GET"
      )
      return resultConnection.newCodec(client, chain)
    } catch (e: RouteException) {
      trackFailure(e.lastConnectException)
      throw e
    } catch (e: IOException) {
      trackFailure(e)
      throw RouteException(e)
    }
}
```

CallServerInterceptor，请求和响应服务拦截器
```
class CallServerInterceptor(private val forWebSocket: Boolean) : Interceptor {

  @Throws(IOException::class)
  override fun intercept(chain: Interceptor.Chain): Response {
    val realChain = chain as RealInterceptorChain
    //在ConnectInterceptor中创建的Exchange对象
    val exchange = realChain.exchange!!
    val request = realChain.request
    val requestBody = request.body
    val sentRequestMillis = System.currentTimeMillis()

    //写入请求头信息
    exchange.writeRequestHeaders(request)
    
    //ExchangeCodec接口，实现类Http1ExchangeCodec，Http2ExchangeCodec
    //exchange.flushRequest()，发送请求

    var invokeStartEvent = true
    var responseBuilder: Response.Builder? = null
    if (HttpMethod.permitsRequestBody(request.method) && requestBody != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return
      // what we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equals(request.header("Expect"), ignoreCase = true)) {
        exchange.flushRequest()
        responseBuilder = exchange.readResponseHeaders(expectContinue = true)
        exchange.responseHeadersStart()
        invokeStartEvent = false
      }
      if (responseBuilder == null) {
        if (requestBody.isDuplex()) {
          // Prepare a duplex body so that the application can send a request body later.
          exchange.flushRequest()
          val bufferedRequestBody = exchange.createRequestBody(request, true).buffer()
          requestBody.writeTo(bufferedRequestBody)
        } else {
          // Write the request body if the "Expect: 100-continue" expectation was met.
          val bufferedRequestBody = exchange.createRequestBody(request, false).buffer()
          requestBody.writeTo(bufferedRequestBody)
          bufferedRequestBody.close()
        }
      } else {
        exchange.noRequestBody()
        if (!exchange.connection.isMultiplexed) {
          // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
          // from being reused. Otherwise we're still obligated to transmit the request body to
          // leave the connection in a consistent state.
          exchange.noNewExchangesOnConnection()
        }
      }
    } else {
      exchange.noRequestBody()
    }

    if (requestBody == null || !requestBody.isDuplex()) {
      exchange.finishRequest()
    }
    if (responseBuilder == null) {
      responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
      if (invokeStartEvent) {
        exchange.responseHeadersStart()
        invokeStartEvent = false
      }
    }
    var response = responseBuilder
        .request(request)
        .handshake(exchange.connection.handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build()
    var code = response.code
    if (code == 100) {
      // Server sent a 100-continue even though we did not request one. Try again to read the actual
      // response status.
      responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
      if (invokeStartEvent) {
        exchange.responseHeadersStart()
      }
      response = responseBuilder
          .request(request)
          .handshake(exchange.connection.handshake())
          .sentRequestAtMillis(sentRequestMillis)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build()
      code = response.code
    }

    exchange.responseHeadersEnd(response)

    response = if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response.newBuilder()
          .body(EMPTY_RESPONSE)
          .build()
    } else {
      response.newBuilder()
          .body(exchange.openResponseBody(response))
          .build()
    }
    if ("close".equals(response.request.header("Connection"), ignoreCase = true) ||
        "close".equals(response.header("Connection"), ignoreCase = true)) {
      exchange.noNewExchangesOnConnection()
    }
    if ((code == 204 || code == 205) && response.body?.contentLength() ?: -1L > 0L) {
      throw ProtocolException(
          "HTTP $code had non-zero Content-Length: ${response.body?.contentLength()}")
    }
    return response
  }
}
```

## WebSocket

```
val request = Request.Builder()
    .url("")
    .build()

val listener = object : WebSocketListener() {
    override fun onOpen(webSocket: WebSocket, response: Response) {
        super.onOpen(webSocket, response)
    }

    override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
        super.onFailure(webSocket, t, response)
    }

    override fun onClosed(webSocket: WebSocket, code: Int, reason: String) {
        super.onClosed(webSocket, code, reason)
    }
}
//创建webSocket实例
val webSocket = okHttpClient.newWebSocket(request, listener)
```

查看newWebSocket
```
override fun newWebSocket(request: Request, listener: WebSocketListener): WebSocket {
    //创建RealWebSocket对象，设置websocket需要的配置
    val webSocket = RealWebSocket(
        taskRunner = TaskRunner.INSTANCE,
        originalRequest = request,
        listener = listener,
        random = Random(),
        pingIntervalMillis = pingIntervalMillis.toLong(),
        extensions = null, // Always null for clients.
        minimumDeflateSize = minWebSocketMessageToCompress
    )
    //连接服务，重点查看connect方法
    webSocket.connect(this)
    return webSocket
}
```

```
fun connect(client: OkHttpClient) {
    if (originalRequest.header("Sec-WebSocket-Extensions") != null) {
      failWebSocket(ProtocolException(
          "Request header not permitted: 'Sec-WebSocket-Extensions'"), null)
      return
    }

    //复制一个新的
    val webSocketClient = client.newBuilder()
        .eventListener(EventListener.NONE)
        .protocols(ONLY_HTTP1) //http1.1
        .build()
        
    //复制一个新请求对象
    val request = originalRequest.newBuilder()
        .header("Upgrade", "websocket") //升级协议：升级为websocket
        .header("Connection", "Upgrade") //连接方式：升级
        .header("Sec-WebSocket-Key", key)  //随机码
        .header("Sec-WebSocket-Version", "13")  //WebSocket版本13
        .header("Sec-WebSocket-Extensions", "permessage-deflate")
        .build()
    //创建RealCall对象
    call = RealCall(webSocketClient, request, forWebSocket = true)
    //异步请求
    call!!.enqueue(object : Callback {
      override fun onResponse(call: Call, response: Response) {
        val exchange = response.exchange
        val streams: Streams
        try {
          //检查升级是否成功，成功时code为101
          //校验响应头的信息
          checkUpgradeSuccess(response, exchange)
          //
          streams = exchange!!.newWebSocketStreams()
        } catch (e: IOException) {
          exchange?.webSocketUpgradeFailed()
          failWebSocket(e, response)
          response.closeQuietly()
          return
        }

        // Apply the extensions. If they're unacceptable initiate a graceful shut down.
        // TODO(jwilson): Listeners should get onFailure() instead of onClosing() + onClosed(1010).
        val extensions = WebSocketExtensions.parse(response.headers)
        this@RealWebSocket.extensions = extensions
        if (!extensions.isValid()) {
          synchronized(this@RealWebSocket) {
            messageAndCloseQueue.clear() // Don't transmit any messages.
            close(1010, "unexpected Sec-WebSocket-Extensions in response header")
          }
        }

        // Process all web socket messages.
        try {
          val name = "$okHttpName WebSocket ${request.url.redact()}"
          //初始化WebSocket的写入对象和读取对象，用于对流的操作
          initReaderAndWriter(name, streams)
          //回调onOpen方法
          listener.onOpen(this@RealWebSocket, response)
          //轮询读取
          loopReader()
        } catch (e: Exception) {
          failWebSocket(e, null)
        }
      }

      override fun onFailure(call: Call, e: IOException) {
        //失败回调
        failWebSocket(e, null)
      }
    })
}

@Throws(IOException::class)
fun initReaderAndWriter(name: String, streams: Streams) {
    val extensions = this.extensions!!
    synchronized(this) {
      this.name = name
      this.streams = streams
      //初始化写入对象WebSocketWriter
      this.writer = WebSocketWriter(
          isClient = streams.client,
          sink = streams.sink,
          random = random,
          perMessageDeflate = extensions.perMessageDeflate,
          noContextTakeover = extensions.noContextTakeover(streams.client),
          minimumDeflateSize = minimumDeflateSize
      )
      this.writerTask = WriterTask()
      if (pingIntervalMillis != 0L) {
        val pingIntervalNanos = MILLISECONDS.toNanos(pingIntervalMillis)
        //时间间隔内发送Ping信号
        taskQueue.schedule("$name ping", pingIntervalNanos) {
          writePingFrame()
          return@schedule pingIntervalNanos
        }
      }
      if (messageAndCloseQueue.isNotEmpty()) {
        //运行WriterTask，发送一帧数据
        runWriter() // Send messages that were enqueued before we were connected.
      }
    }

    //初始化读取对象WebSocketReader
    reader = WebSocketReader(
        isClient = streams.client,
        source = streams.source,
        frameCallback = this,
        perMessageDeflate = extensions.perMessageDeflate,
        noContextTakeover = extensions.noContextTakeover(!streams.client)
    )
}

internal fun writePingFrame() {
    val writer: WebSocketWriter
    val failedPing: Int
    synchronized(this) {
      if (failed) return
      writer = this.writer ?: return
      //awaitingPong默认为false，所以failedPing = -1
      //awaitingPong会在onReadPong回调中设置为false
      failedPing = if (awaitingPong) sentPingCount else -1
      sentPingCount++
      awaitingPong = true
    }

    //awaitingPong为false时不进入判断
    if (failedPing != -1) {
      //如果在下一次轮询之前不能接收到pong信号，则会进入这里，关闭WebSocket
      failWebSocket(SocketTimeoutException("sent ping but didn't receive pong within " +
          "${pingIntervalMillis}ms (after ${failedPing - 1} successful ping/pongs)"), null)
      return
    }

    try {
      //发送一个ping信号
      writer.writePing(ByteString.EMPTY)
    } catch (e: IOException) {
      failWebSocket(e, null)
    }
}

@Throws(IOException::class)
  fun loopReader() {
    while (receivedCloseCode == -1) {
      // This method call results in one or more onRead* methods being called on this thread.
      reader!!.processNextFrame()
    }
}

@Throws(IOException::class)
  fun processNextFrame() {
    readHeader()
    if (isControlFrame) {
      //ping信号时会走这里
      readControlFrame()
    } else {
      readMessageFrame()
    }
}

@Throws(IOException::class)
  private fun readControlFrame() {
    if (frameLength > 0L) {
      source.readFully(controlFrameBuffer, frameLength)

      if (!isClient) {
        controlFrameBuffer.readAndWriteUnsafe(maskCursor!!)
        maskCursor.seek(0)
        toggleMask(maskCursor, maskKey!!)
        maskCursor.close()
      }
    }

    when (opcode) {
      OPCODE_CONTROL_PING -> {
        frameCallback.onReadPing(controlFrameBuffer.readByteString())
      }
      OPCODE_CONTROL_PONG -> {
        //pong信号的回调
        frameCallback.onReadPong(controlFrameBuffer.readByteString())
      }
      OPCODE_CONTROL_CLOSE -> {
        var code = CLOSE_NO_STATUS_CODE
        var reason = ""
        val bufferSize = controlFrameBuffer.size
        if (bufferSize == 1L) {
          throw ProtocolException("Malformed close payload length of 1.")
        } else if (bufferSize != 0L) {
          code = controlFrameBuffer.readShort().toInt()
          reason = controlFrameBuffer.readUtf8()
          val codeExceptionMessage = WebSocketProtocol.closeCodeExceptionMessage(code)
          if (codeExceptionMessage != null) throw ProtocolException(codeExceptionMessage)
        }
        frameCallback.onReadClose(code, reason)
        closed = true
      }
      else -> {
        throw ProtocolException("Unknown control opcode: " + opcode.toHexString())
      }
    }
}

@Synchronized override fun onReadPong(payload: ByteString) {
    // This API doesn't expose pings.
    receivedPongCount++
    awaitingPong = false
}
```

webSocket失败回调
```
fun failWebSocket(e: Exception, response: Response?) {
    val streamsToClose: Streams?
    val readerToClose: WebSocketReader?
    val writerToClose: WebSocketWriter?
    synchronized(this) {
      if (failed) return // Already failed.
      failed = true
      streamsToClose = this.streams
      this.streams = null
      readerToClose = this.reader
      this.reader = null
      writerToClose = this.writer
      this.writer = null
      taskQueue.shutdown()
    }

    try {
      listener.onFailure(this, e, response)
    } finally {
      streamsToClose?.closeQuietly()
      readerToClose?.closeQuietly()
      writerToClose?.closeQuietly()
    }
}
```




