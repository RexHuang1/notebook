#### Request/Response
Request是发送请求封装类，内部有url，header， method，body等常见的参数，Response是请求的结果，包含code，message，header，body；这两个类的定义是完全符合Http协议所定义的请求内容和响应内容。

---

#### OkHttpClient
Call和WebSocket实例对象的一个工厂类，用于发送HTTP请求和读取响应。

##### OkHttpClient.Builder
内部类Builder用于构建OkHttpClient实例。其无参构造方法设置OkHttpClient的默认参数值,其方法设置OkHttpClient的特定参数值。
```java
// Builer默认构造函数,OkHttpClient的默认参数都在这里设置
public Builder() {
    dispatcher = new Dispatcher();
    protocols = DEFAULT_PROTOCOLS;
    connectionSpecs = DEFAULT_CONNECTION_SPECS;
    eventListenerFactory = EventListener.factory(EventListener.NONE);
    proxySelector = ProxySelector.getDefault();
    if (proxySelector == null) {
        proxySelector = new NullProxySelector();
    }
    cookieJar = CookieJar.NO_COOKIES;
    socketFactory = SocketFactory.getDefault();
    hostnameVerifier = OkHostnameVerifier.INSTANCE;
    certificatePinner = CertificatePinner.DEFAULT;
    proxyAuthenticator = Authenticator.NONE;
    authenticator = Authenticator.NONE;
    connectionPool = new ConnectionPool();
    dns = Dns.SYSTEM;
    followSslRedirects = true;
    followRedirects = true;
    retryOnConnectionFailure = true;
    callTimeout = 0;
    connectTimeout = 10_000;
    readTimeout = 10_000;
    writeTimeout = 10_000;
    pingInterval = 0;
}
```

##### 重要的方法
```java
// 构造Call对象
@Override 
public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
}

// 构造WebSocket对象
@Override 
public WebSocket newWebSocket(Request request, WebSocketListener listener) {
    RealWebSocket webSocket = new RealWebSocket(request, listener, new Random(), pingInterval);
    webSocket.connect(this);
    return webSocket;
}
```

---

#### RealCall
真正的Call的实现类。负责请求的调度（同步和异步，同步即是走当前线程发送请求，异步则使用OkHttp内部的线程池进行）；负责构造内部逻辑责任链，并执行责任链相关逻辑，知道获取结果。

##### 重要的方法
```java
// 真正构造RealCall对象的方法
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    // 构造了Transimitter对象
    call.transmitter = new Transmitter(client, call);
    return call;
}
```
```java
// 执行同步请求的方法
@Override 
public Response execute() throws IOException {
    synchronized (this) {
        if (executed) throw new IllegalStateException("Already Executed");
        executed = true;
    }
    transmitter.timeoutEnter();
    transmitter.callStart();
    try {
        // 将同步的Call对象加入Dispatcher的runningSyncCalls队列中
        client.dispatcher().executed(this);
        // 重点在此处,OkHttpClient的责任链模式
        return getResponseWithInterceptorChain();
    } finally {
        // 将同步的Call对象从Dispatcher的runningSyncCalls队列中删除// ,并寻找可以加入到Dispatcher的runningAsyncCalls队列中的
        // AsyncCall对象,将其加入到线程池中等待执行。最后走OkHttpCl// ient的责任链模式.
        client.dispatcher().finished(this);
    }
}
```
```java
// 执行异步请求的方法
@Override 
public void enqueue(Callback responseCallback) {
    synchronized (this) {
        if (executed) throw new IllegalStateException("Already Executed");
        executed = true;
    }
    transmitter.callStart();
    // 构造异步的AsyncCall对象,将AsyncCall对象加入Dispatcher的
    // readyAsyncCalls队列中
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```
```java
// 重点，走责任链模式
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    // 构造Interceptor的集合
    List<Interceptor> interceptors = new ArrayList<>();
    // 自定义的interceptors,被称为application interceptors
    interceptors.addAll(client.interceptors());
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
        // 自定义的interceptors,被称为network interceptors
        interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));
    
    // 用上面的集合构造责任链对象,并传递index进去,index决定了责任链的哪一个interceptors
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
        originalRequest, this, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());
    
    boolean calledNoMoreExchanges = false;
    try {
        // 责任链的proceed方法,开始责任链
        Response response = chain.proceed(originalRequest);
        if (transmitter.isCanceled()) {
            closeQuietly(response);
            throw new IOException("Canceled");
        }
        return response;
    } catch (IOException e) {
        calledNoMoreExchanges = true;
        throw transmitter.noMoreExchanges(e);
    } finally {
        if (!calledNoMoreExchanges) {
            transmitter.noMoreExchanges(null);
        }
    }
}

```

---

#### Interceptor（OkHttp的核心）
##### RetryAndFollowUpInterceptor（失败和重定向拦截器）
###### 重要的方法
```java
@Override 
public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Transmitter transmitter = realChain.transmitter();
    
    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
        // 为发送数据做准备,会创建ExchangeFinder对象,为后面获取exchange对象做准备
        transmitter.prepareToConnect(request);
    
        if (transmitter.isCanceled()) {
            throw new IOException("Canceled");
        }
        
        Response response;
        boolean success = false;
        try {
            // 下一责任链
            response = realChain.proceed(request, transmitter, null);
            success = true;
        } catch (RouteException e) {
            // The attempt to connect via a route failed. The request will not have been sent.
            // 是否发生Route类型的异常,如果有,判断是否满足重试条件,满足则continue重试,重试的逻辑在recover方法中
            if (!recover(e.getLastConnectException(), transmitter, false, request)) {
                throw e.getFirstConnectException();
            }
            continue;
        } catch (IOException e) {
            // An attempt to communicate with a server failed. The request may have been sent.
            // 是否发生IO类型的异常,如果有,判断是否满足重试条件,满足则continue重试
            boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
            if (!recover(e, transmitter, requestSendStarted, request))     throw e;
            continue;
        } finally {
            // The network call threw an exception. Release any resources.
            if (!success) {
                transmitter.exchangeDoneDueToException();
            }
        }
        
        // Attach the prior response if it exists. Such responses never have a body.
        if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
        }
        
        Exchange exchange = Internal.instance.exchange(response);
        Route route = exchange != null ? exchange.connection().route() : null;
        // 这里处理是否重定向的逻辑并开始构造重定向请求,具体情况看源码分析
        Request followUp = followUpRequest(response, route);
        
        if (followUp == null) {
            if (exchange != null && exchange.isDuplex()) {
                transmitter.timeoutEarlyExit();
            }
            return response;
        }
        
        RequestBody followUpBody = followUp.body();
        if (followUpBody != null && followUpBody.isOneShot()) {
            return response;
        }
        
        closeQuietly(response.body());
        if (transmitter.hasExchange()) {
            exchange.detachWithViolence();
        }
        
        if (++followUpCount > MAX_FOLLOW_UPS) {
            throw new ProtocolException("Too many follow-up requests: " + followUpCount);
        }
        
        request = followUp;
        priorResponse = response;
    }
}
```

---

##### Interceptors和NetworkInterceptors的区别
在 OkHttpClient.Builder 中，使用者可以通过 addInterceptor 和 addNetworkdInterceptor 添加自定义的拦截器，分析完 RetryAndFollowUpInterceptor 就可以知道这两种自动拦截器的区别了。
从添加拦截器的顺序可以知道 Interceptors 和 networkInterceptors 刚好一个在 RetryAndFollowUpInterceptor 的前面，一个在后面。
可以分析出来，假如一个请求在 RetryAndFollowUpInterceptor 这个拦截器内部重试或者重定向了 N 次，那么其内部嵌套的所有拦截器也会被调用N次，同样 networkInterceptors 自定义的拦截器也会被调用 N 次。而相对的 Interceptors 则一个请求只会调用一次，所以在OkHttp的内部也将其称之为 Application Interceptor。

---

##### BridgeInterceptor（封装request和response拦截器）
###### 重要的方法
```java
@Override 
public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();
    
    RequestBody body = userRequest.body();
    if (body != null) {
        MediaType contentType = body.contentType();
        if (contentType != null) {
            requestBuilder.header("Content-Type", contentType.toString());
        }
        
        long contentLength = body.contentLength();
        if (contentLength != -1) {
            requestBuilder.header("Content-Length", Long.toString(contentLength));
            requestBuilder.removeHeader("Transfer-Encoding");
        } else {
            requestBuilder.header("Transfer-Encoding", "chunked");
            requestBuilder.removeHeader("Content-Length");
        }
    }
    
    if (userRequest.header("Host") == null) {
        requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }
    
    if (userRequest.header("Connection") == null) {
        requestBuilder.header("Connection", "Keep-Alive");
    }
    
    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
        transparentGzip = true;
        requestBuilder.header("Accept-Encoding", "gzip");
    }
    
    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
        requestBuilder.header("Cookie", cookieHeader(cookies));
    }
    
    if (userRequest.header("User-Agent") == null) {
        requestBuilder.header("User-Agent", Version.userAgent());
    }
    
    Response networkResponse = chain.proceed(requestBuilder.build());
    
    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());
    
    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);
    
    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
        GzipSource responseBody = new GzipSource(networkResponse.body().source());
        Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
        responseBuilder.headers(strippedHeaders);
        String contentType = networkResponse.header("Content-Type");
        responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }
    
    return responseBuilder.build();
}
```
这个拦截器比较简单,功能如下:
1. 负责把用户构造的请求转换为发送到服务器的请求 、把服务器返回的响应转换为用户友好的响应，是从应用程序代码到网络代码的桥梁
2. 设置内容长度，内容编码
3. 设置gzip压缩，并在接收到内容后进行解压。省去了应用层处理数据解压的麻烦
4. 添加cookie
5. 设置其他报头，如User-Agent,Host,Keep-alive等。其中Keep-Alive是实现连接复用的必要步骤

---

##### CacheInterceptor（缓存拦截器）
###### 重要的方法
```java
@Override 
public Response intercept(Chain chain) throws IOException {
    // 通过Request在Cache中拿缓存,前提是OkHttpClient中配置了缓存,默认不支持
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;
    
    long now = System.currentTimeMillis();
    // 根据response,time,request构造一个缓存策略，用于判断怎样使用缓存。
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    // 如果该请求没有使用网络就为null
    Request networkRequest = strategy.networkRequest;
    // 如果该请求没有使用缓存则为空
    Response cacheResponse = strategy.cacheResponse;
    
    if (cache != null) {
        cache.trackResponse(strategy);
    }
    
    if (cacheCandidate != null && cacheResponse == null) {
        closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }
    
    // If we're forbidden from using the network and the cache is insufficient, fail.
    // 如果缓存策略中设置禁止使用网络，并且缓存也为空，则构建一个Response直接返回，注意返回码=504
    if (networkRequest == null && cacheResponse == null) {
        return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }
    
    // If we don't need the network, we're done.
    // 如果不使用网络但有缓存,则返回缓存
    if (networkRequest == null) {
        return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }
    
    Response networkResponse = null;
    try {
        // 走后续拦截器流程
        networkResponse = chain.proceed(networkRequest);
    } finally {
        // If we're crashing on I/O or otherwise, don't leak the cache body.
        if (networkResponse == null && cacheCandidate != null) {
            closeQuietly(cacheCandidate.body());
        }
    }
    
    // If we have a cache response too, then we're doing a conditional get.
    // 缓存存在且网络返回的Response为304,则使用缓存的Response
    if (cacheResponse != null) {
        if (networkResponse.code() == HTTP_NOT_MODIFIED) {
            Response response = cacheResponse.newBuilder()
                .headers(combine(cacheResponse.headers(), networkResponse.headers()))
                .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
                .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
                .cacheResponse(stripBody(cacheResponse))
                .networkResponse(stripBody(networkResponse))
                .build();
            networkResponse.body().close();
            
            // Update the cache after combining headers but before stripping the
            // Content-Encoding header (as performed by initContentStream()).
            cache.trackConditionalCacheHit();
            // 更新缓存
            cache.update(cacheResponse, response);
            return response;
        } else {
            closeQuietly(cacheResponse.body());
        }
    }
    // 构建网络请求的Resposne
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();
    // OkHttpClient中配置了Cache的话,缓存Response
    if (cache != null) {
        if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
            // Offer this request to the cache.
            CacheRequest cacheRequest = cache.put(response);
            return cacheWritingResponse(cacheRequest, response);
        }
        
        if (HttpMethod.invalidatesCache(networkRequest.method())) {
            try {
                cache.remove(networkRequest);
            } catch (IOException ignored) {
                // The cache cannot be written.
            }
        }
    }
    
    return response;
}
```

---

##### ConnectInterceptor（网络连接拦截器,负责和服务器建立连接,重点，未完成）
###### 重要的方法
```java
@Override 
public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    Transmitter transmitter = realChain.transmitter();
    
    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    // 注释1 从Transmitter中获取新的Exchange对象
    Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);
    
    return realChain.proceed(request, transmitter, exchange);
}
```
###### 源码逻辑跳转:

```java
// 源码位置: Transmitter.java
/** Returns a new exchange to carry a new request and response. */
Exchange newExchange(Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
  synchronized (connectionPool) {
    if (noMoreExchanges) {
      throw new IllegalStateException("released");
    }
    if (exchange != null) {
      throw new IllegalStateException("cannot make a new request because the previous response "
          + "is still open: please call response.close()");
    }
  }
  // 调用ExchangeFinder的find()获取ExchangeCodec
  ExchangeCodec codec = exchangeFinder.find(client, chain, doExtensiveHealthChecks);
  // 用上面获取的codec对象构建新的Exchange对象
  Exchange result = new Exchange(this, call, eventListener, exchangeFinder, codec);

  synchronized (connectionPool) {
    this.exchange = result;
    this.exchangeRequestDone = false;
    this.exchangeResponseDone = false;
    return result;
  }
}
```

```java
// 源码位置: ExchangFinder.java
public ExchangeCodec find(
    OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
  int connectTimeout = chain.connectTimeoutMillis();
  int readTimeout = chain.readTimeoutMillis();
  int writeTimeout = chain.writeTimeoutMillis();
  int pingIntervalMillis = client.pingIntervalMillis();
  boolean connectionRetryEnabled = client.retryOnConnectionFailure();

  try {
    // 调用自身findHealthyConnection方法获取RealConnection
    RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
        writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
    return resultConnection.newCodec(client, chain);
  } catch (RouteException e) {
    trackFailure();
    throw e;
  } catch (IOException e) {
    trackFailure();
    throw new RouteException(e);
  }
}
```

```java
// 源码位置: ExchangeFinder.java
/**
 * Finds a connection and returns it if it is healthy. If it is unhealthy the process is repeated
 * until a healthy connection is found.
 */
private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
    int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
    boolean doExtensiveHealthChecks) throws IOException {
  while (true) {
    // 在ExchangeFinder的findConnection方法循环获取可用的RealConnection
    RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
        pingIntervalMillis, connectionRetryEnabled);

    // If this is a brand new connection, we can skip the extensive health checks.
    synchronized (connectionPool) {
      // 判断获取的RealConnection是否可用,若可用返回,不可用继续寻找
      if (candidate.successCount == 0 && !candidate.isMultiplexed()) {
        return candidate;
      }
    }

    // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
    // isn't, take it out of the pool and start again.
    if (!candidate.isHealthy(doExtensiveHealthChecks)) {
      candidate.noNewExchanges();
      continue;
    }

    return candidate;
  }
}
```

```java
// 源码位置: ExchangeFinder.java
/**
 * Returns a connection to host a new stream. This prefers the existing connection if it exists,
 * then the pool, finally building a new connection.
 */
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
    int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
  boolean foundPooledConnection = false;
  RealConnection result = null;
  Route selectedRoute = null;
  RealConnection releasedConnection;
  Socket toClose;
  synchronized (connectionPool) {
    if (transmitter.isCanceled()) throw new IOException("Canceled");
    hasStreamFailure = false; // This is a fresh attempt.

    // Attempt to use an already-allocated connection. We need to be careful here because our
    // already-allocated connection may have been restricted from creating new exchanges.
    // 尝试使用已分配的连接,已经分配的连接可能已经被限制创建新的流
    releasedConnection = transmitter.connection;
    toClose = transmitter.connection != null && transmitter.connection.noNewExchanges
        ? transmitter.releaseConnectionNoEvents()
        : null;

    if (transmitter.connection != null) {
      // We had an already-allocated connection and it's good.
      // 已分配连接,并且该连接可用
      result = transmitter.connection;
      releasedConnection = null;
    }

    if (result == null) {
      // Attempt to get a connection from the pool.
      // 尝试从连接池中获取一个连接
      if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, null, false)) {
        foundPooledConnection = true;
        result = transmitter.connection;
      } else if (nextRouteToTry != null) {
        selectedRoute = nextRouteToTry;
        nextRouteToTry = null;
      } else if (retryCurrentRoute()) {
        selectedRoute = transmitter.connection.route();
      }
    }
  }
  // 关闭连接
  closeQuietly(toClose);

  if (releasedConnection != null) {
    eventListener.connectionReleased(call, releasedConnection);
  }
  if (foundPooledConnection) {
    eventListener.connectionAcquired(call, result);
  }
  if (result != null) {
    // If we found an already-allocated or pooled connection, we're done.
    // 如果已经从连接池中获取到了一个连接，就将其返回
    return result;
  }

  // If we need a route selection, make one. This is a blocking operation.
  boolean newRouteSelection = false;
  if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
    newRouteSelection = true;
    routeSelection = routeSelector.next();
  }

  List<Route> routes = null;
  synchronized (connectionPool) {
    if (transmitter.isCanceled()) throw new IOException("Canceled");

    if (newRouteSelection) {
      // Now that we have a set of IP addresses, make another attempt at getting a connection from
      // the pool. This could match due to connection coalescing.
      // 根据一系列的 IP 地址从连接池中获取一个链接
      routes = routeSelection.getAll();
      if (connectionPool.transmitterAcquirePooledConnection(
          address, transmitter, routes, false)) {
        foundPooledConnection = true;
        result = transmitter.connection;
      }
    }

    if (!foundPooledConnection) {
      if (selectedRoute == null) {
        selectedRoute = routeSelection.next();
      }

      // Create a connection and assign it to this allocation immediately. This makes it possible
      // for an asynchronous cancel() to interrupt the handshake we're about to do.
      // 创建一个新的连接，并将其分配，这样我们就可以在握手之前进行终端
      result = new RealConnection(connectionPool, selectedRoute);
      connectingConnection = result;
    }
  }

  // If we found a pooled connection on the 2nd time around, we're done.
  // 如果我们在第二次的时候发现了一个池连接，那么我们就将其返回
  if (foundPooledConnection) {
    eventListener.connectionAcquired(call, result);
    return result;
  }

  // Do TCP + TLS handshakes. This is a blocking operation.
  // 进行 TCP 和 TLS 握手
  result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
      connectionRetryEnabled, call, eventListener);
  connectionPool.routeDatabase.connected(result.route());

  Socket socket = null;
  synchronized (connectionPool) {
    connectingConnection = null;
    // Last attempt at connection coalescing, which only occurs if we attempted multiple
    // concurrent connections to the same host.
    if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, routes, true)) {
      // We lost the race! Close the connection we created and return the pooled connection.
      result.noNewExchanges = true;
      socket = result.socket();
      result = transmitter.connection;

      // It's possible for us to obtain a coalesced connection that is immediately unhealthy. In
      // that case we will retry the route we just successfully connected with.
      nextRouteToTry = selectedRoute;
    } else {
      connectionPool.put(result);
      transmitter.acquireConnectionNoEvents(result);
    }
  }
  closeQuietly(socket);

  eventListener.connectionAcquired(call, result);
  return result;
}
```

注释1处(跟源码)内部逻辑如下:

1. ConnectInterceptor调用transmitter.newExchange
2. Transmitter先调用ExchangeFinder的find()获得ExchangeCodec
3. ExchangeFinder调用自身的findHealthyConnection获得RealConnection
4. ExchangeFinder的findHealthyConnection方法调用自身的findConnection获得RealConnection
5. ExchangeFinder通过刚才获取的RealConnection的codec()方法获得ExchangeCodec
6. Transmitter获取到了ExchangeCodec，然后new了一个ExChange，将刚才的ExchangeCodec包含在内。

通过上面的逻辑,ConnectInterceptor可以获得一个Exchange类,这个类有两个实现,一个是Http1ExchangeCodec,一个是Http2ExchangeCodec,分别对应Http1和Http2协议。  

Exchange类里面包含了ExchangeCodec对象，而这个对象里面又包含了一个RealConnection对象，RealConnection的属性成员有socket、handlShake、protocol等，可见它应该是一个Socket连接的包装类，而ExchangeCode对象是对RealConnection操作（writeRequestHeader、readResposneHeader）的封装。

---

##### CallServerInterceptor（执行流操作拦截器,负责向服务器发送请求数据、从服务器读取响应数据 进行http请求报文的封装与请求报文的解析）
###### 重要的方法
```java
@Override 
public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Exchange exchange = realChain.exchange();
    Request request = realChain.request();
    
    long sentRequestMillis = System.currentTimeMillis();
    // 写入请求头
    exchange.writeRequestHeaders(request);
    
    boolean responseHeadersStarted = false;
    Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
        // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
        // Continue" response before transmitting the request body. If we don't get that, return
        // what we did get (such as a 4xx response) without ever transmitting the request body.
        if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
            exchange.flushRequest();
            responseHeadersStarted = true;
            exchange.responseHeadersStart();
            responseBuilder = exchange.readResponseHeaders(true);
        }
        // 写入请求体
        if (responseBuilder == null) {
            if (request.body().isDuplex()) {
                // Prepare a duplex body so that the application can send a request body later.
                exchange.flushRequest();
                BufferedSink bufferedRequestBody = Okio.buffer(
                    exchange.createRequestBody(request, true));
                request.body().writeTo(bufferedRequestBody);
            } else {
                // Write the request body if the "Expect: 100-continue" expectation was met.
                BufferedSink bufferedRequestBody = Okio.buffer(
                    exchange.createRequestBody(request, false));
                request.body().writeTo(bufferedRequestBody);
                bufferedRequestBody.close();
            }
        } else {
            exchange.noRequestBody();
            if (!exchange.connection().isMultiplexed()) {
                // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
                // from being reused. Otherwise we're still obligated to transmit the request body to
                // leave the connection in a consistent state.
                exchange.noNewExchangesOnConnection();
            }
        }
    } else {
        exchange.noRequestBody();
    }
    
    if (request.body() == null || !request.body().isDuplex()) {
        exchange.finishRequest();
    }
    
    if (!responseHeadersStarted) {
        exchange.responseHeadersStart();
    }
    
    if (responseBuilder == null) {
        // 读取响应头
        responseBuilder = exchange.readResponseHeaders(false);
    }
    
    Response response = responseBuilder
        .request(request)
        .handshake(exchange.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();
    
    int code = response.code();
    if (code == 100) {
        // server sent a 100-continue even though we did not request one.
        // try again to read the actual response
        response = exchange.readResponseHeaders(false)
          .request(request)
          .handshake(exchange.connection().handshake())
          .sentRequestAtMillis(sentRequestMillis)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    
        code = response.code();
    }
    
    exchange.responseHeadersEnd(response);
    
    if (forWebSocket && code == 101) {
        // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
        response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
        // 读取响应体
        response = response.newBuilder()
          .body(exchange.openResponseBody(response))
          .build();
    }
    
    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
        exchange.noNewExchangesOnConnection();
    }
    
    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
        throw new ProtocolException(
            "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }
    
    return response;
}
```
CallServerInterceptor由以下步骤组成:
1. 向服务器发送 request header
2. 如果有 request body，就向服务器发送
3. 读取 response header，先构造一个 Response 对象
4. 如果有 response body，就在 3 的基础上加上 body 构造一个新的 Response 对象






