---
layout: post
title: "Apache HttpClient Connection Management" 
author: Gunju Ko
tags: [http, httpClient]
---

## Apache HttpClient Connection Management

아파치 HttpClient는 지속 커넥션을 지원한다.

### HttpConnectionManager

Http 커넥션은 복잡하고, stateful하며 thread-safe하지 않다. 따라서 제대로 동작하게 하기 위해서는 제대로 관리해주어야 한다. Http 커넥션을 여러 스레드에서 동시에 사용하면 안된다. HttpClient는 Http 커넥션을 관리하기 위해 HttpClientConnectionManager를 사용한다. HttpClientConnectionManager의 역할은 다음과 같다.

- 새로운 Http 커넥션 생성
- 지속 커넥션 라이프사이클 관리
- 지속 커넥션에 대한 접근을 동기화하여 지속 커넥션을 여러개의 스레드에서 접근하지 못하도록 한다.

Http 커넥션 매니저는 내부적으로 ManagedHttpClientConnection 객체를 사용한다. ManagedHttpClientConnection는 커넥션 상태를 관리하고 I/O 작업의 실행을 제어하는 실제 커넥션에 대한 프록시 역할을 한다. 

만약에 관리되고 있는 커넥션이 release되거나 명시적으로 close된 경우에는 커넥션이 프록시로부터 분리가 되고 Http 커넥션 매니저에게 다시 반환된다. 클라이언트가 프록시에 대한 참조를 가지고 있더라도 더이상 I/O 작업을 하거나 실제 커넥션의 상태를 변경할 순 없다.

``` java
/**
 * Represents a managed connection whose state and life cycle is managed by
 * a connection manager. This interface extends {@link HttpClientConnection}
 * with methods to bind the connection to an arbitrary socket and
 * to obtain SSL session details.
 *
 * @since 4.3
 */
public interface ManagedHttpClientConnection extends HttpClientConnection, HttpInetConnection {

    /**
     * Returns connection ID which is expected to be unique
     * for the life span of the connection manager.
     */
    String getId();

    /**
     * Binds this connection to the given socket. The connection
     * is considered open if it is bound and the underlying socket
     * is connection to a remote host.
     *
     * @param socket the socket to bind the connection to.
     * @throws IOException
     */
    void bind(Socket socket) throws IOException;

    /**
     * Returns the underlying socket.
     */
    Socket getSocket();

    /**
     * Obtains the SSL session of the underlying connection, if any.
     * If this connection is open, and the underlying socket is an
     * {@link javax.net.ssl.SSLSocket SSLSocket}, the SSL session of
     * that socket is obtained. This is a potentially blocking operation.
     *
     * @return  the underlying SSL session if available,
     *          {@code null} otherwise
     */
    SSLSession getSSLSession();

}
```

아래 코드는 커넥션 매니저에서 커넥션을 가져오는 샘플 코드이다.

``` java
HttpClientContext context = HttpClientContext.create();
HttpClientConnectionManager connMrg = new BasicHttpClientConnectionManager();
HttpRoute route = new HttpRoute(new HttpHost("localhost", 80));
// Request new connection. This can be a long process
ConnectionRequest connRequest = connMrg.requestConnection(route, null);
// Wait for connection up to 10 sec
HttpClientConnection conn = connRequest.get(10, TimeUnit.SECONDS);
try {
    // If not open
    if (!conn.isOpen()) {
        // establish connection based on its route info
        connMrg.connect(conn, route, 1000, context);
        // and mark it as route complete
        connMrg.routeComplete(conn, route, context);
    }
    // Do useful things with the connection.
} finally {
    connMrg.releaseConnection(conn, null, 1, TimeUnit.MINUTES);
}
```

#### BasicHttpClientConnectionManager

BasicHttpClientConnectionManager는 한번에 하나의 커넥션만 유지하는 아주 간단한 커넥션 매니저이다. 이 클래스는 thread-safe하지만, 하나의 실행 스레드만 사용해야 한다. BasicHttpClientConnectionManager는 동일한 라우트에 대한 후속 요청을 처리하기 위해 커넥션을 재사용한다. 하지만 새로운 요청의 라우트가 기존 커넥션의 라우트와 동일하지 않는 경우, 기존 커넥션을 close하고 새로운 커넥션을 생성한다. 

#### PoolingHttpClientConnectionManager

PoolingHttpClientConnectionManager는 좀더 복잡한 구현체로 클라이언트 커넥션 풀을 관리하고, 여러 스레드에서 커넥션 요청을 처리할 수 있다. 커넥션은 라우트를 기반으로해서 pool된다. 만약에 해당 라우트에 대한 커넥션이 풀에 이미 존재하는 경우, 풀에 존재하는 커넥션을 꺼내준다. (새로운 커넥션을 만들지 않는다)

PoolingHttpClientConnectionManager는 라우트별 및 총합에 대한 최대 커넥션 개수를 제한한다. 디폴트는 라우트당 최대 2개의 커넥션을 생성하며, 총 커넥션의 개수는 최대 20개까지 생성한다.

아래의 코드는 PoolingHttpClientConnectionManager의 설정을 커스터마이징하는 코드이다.

``` java
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
// Increase max total connection to 200
cm.setMaxTotal(200);
// Increase default max connection per route to 20
cm.setDefaultMaxPerRoute(20);
// Increase max connections for localhost:80 to 50
HttpHost localhost = new HttpHost("locahost", 80);
cm.setMaxPerRoute(new HttpRoute(localhost), 50);

CloseableHttpClient httpClient = HttpClients.custom()
        .setConnectionManager(cm)
        .build();
```

### Multithreaded request execution

PoolingClientConnectionManager를 커넥션 매니저로 사용하면, HttpClient는 여러개의 스레드를 사용해서 여러개의 요청을 동시에 처리할 수 있다.  PoolingClientConnectionManager는 설정을 기반으로 해서 커넥션을 할당한다. 만약 라우트에 대한 모든 커넥션이 사용중이면, 커넥션이 풀에 반환될때까지 요청이 블락된다. "http.conn-manager.timeout"를 양수로 설정하여 커넥션 매니저가 커넥션 요청 작업에서 무기한 block되지 않도록 할 수 있다. 만약에 http.conn-manager.timeout 시간동안 커넥션이 풀에 반환되지 않으면 ConnectionPoolTimeoutException이 발생한다.

> HttpClient는 thread-safe하기 떄문에 여러 스레드에서 공유해서 사용해도 된다. 하지만 각 스레드는 서로 다른 HttpContext 객체를 사용하는것을 권장한다.

### Connection eviction policy

고전적인 블록킹 I/O 모델의 주요 단점 중 하나는 I/O 작업에 의해 block된 경우에만 네트워크 소켓이 I/O 이벤트에 반응할 수 있다는 것이다. 만약에 커넥션이 관리자에게 다시 release되면, 해당 커넥션을 계속 유지할 수 있지만 소켓의 상태를 모니터링하고 I/O 이벤트에 반응할 수 없다. 

만약에 커넥션이 서버 사이드에서 close된 경우, 클라이언트 사이드에서는 커넥션 상태의 변경을 감지할 수 없다. 

HttpClient는 이런 문제를 해결하기 위해 HTTP 요청을 보내기 위해 커넥션을 사용하기 전에 커넥션이 비정상인지를 테스트한다. 하지만 이러한 테스트를 100% 신뢰할 순 없다. 

장기간 사용되지 않아, 만료된 커넥션을 제거하는 역할을 하는 모니터링 스레드가 존재한다. 모니터링 스레드는 주기적으로 ClientConnectionManager#closeExpiredConnections() 메소드를 호출한다. 해당 메소드는 만료된 커넥션을 전부 close 하고 풀에서 제거한다. 

> 더 자세한 사항은 IdleConnectionEvictor 코드를 참고하자

### Connection keep alive strategy

HTTP 스펙은 지속 커넥션이 얼마나 유지되어야 하는지에 대한 기간을 지정하지 않는다. 일부 HTTP 서버는 비표준 헤더인 Keep-Alive를 사용해서 클라이언트에게 커넥션을 얼마나 유지할 것인지를 알려준다. HttpClient는 가능한 이 정보를 사용한다. 만약에 Keep-Alive 헤더가 응답에 있지 않으면, HttpClient은 커넥션을 무기한으로 유지한다고 가정한다. 그러나 대부분의 HTTP 서버는 클라이언트에 알리지 않고 시스템 리소스를 절약하기 위해 일정 기간 동안 사용하지 않으면 지속 커넥션을 끊는다. 만약에 기본 구현체 전략이 너무 낙관적이라면, 커스텀 keep-alive strategy를 사용하면 된다.

``` java
ConnectionKeepAliveStrategy myStrategy = new ConnectionKeepAliveStrategy() {

    public long getKeepAliveDuration(HttpResponse response, HttpContext context) {
        // Honor 'keep-alive' header
        HeaderElementIterator it = new BasicHeaderElementIterator(
                response.headerIterator(HTTP.CONN_KEEP_ALIVE));
        while (it.hasNext()) {
            HeaderElement he = it.nextElement();
            String param = he.getName();
            String value = he.getValue();
            if (value != null && param.equalsIgnoreCase("timeout")) {
                try {
                    return Long.parseLong(value) * 1000;
                } catch(NumberFormatException ignore) {
                }
            }
        }
        HttpHost target = (HttpHost) context.getAttribute(
                HttpClientContext.HTTP_TARGET_HOST);
        if ("www.naughty-server.com".equalsIgnoreCase(target.getHostName())) {
            // Keep alive for 5 seconds only
            return 5 * 1000;
        } else {
            // otherwise keep alive for 30 seconds
            return 30 * 1000;
        }
    }

};
CloseableHttpClient client = HttpClients.custom()
        .setKeepAliveStrategy(myStrategy)
        .build();
```

### Connection socket factories

ConnectionSocketFactory 인터페이스를 사용해서 소켓을 생성, 초기화, 연결한다. ConnectionSocketFactory 인터페이스 구현체를 커스터마이징하여 소켓을 초기화하는 부분의 코드를 제공할 수 있다. PlainConnectionSocketFactory가 기본 구현체로 사용된다. 

> Apache HttpClient에 대한 더 자세한 사항은 [Chapter 2. Connection management](http://hc.apache.org/httpcomponents-client-ga/tutorial/html/connmgmt.html) 를 참고하길 바란다.

### CloseableHttpClient 구조

HttpClient를 생성할 때는 주로 HttpClientBuilder 클래스를 사용해서 생성한다. 해당 클래스를 사용해서 HttpClient를 생성하면 CloseableHttpClient 타입의 HttpClient가 생성된다. 아래는 간단한 예제 코드이다. 

``` java
public void createHttpClient() {
    HttpClientBuilder builder = HttpClientBuilder.create();

    builder.setMaxConnPerRoute(10);
    builder.setMaxConnTotal(100);
    builder.setConnectionTimeToLive(10, TimeUnit.MINUTES);

    RequestConfig requestConfig = RequestConfig.custom()
                                               .setConnectTimeout(1000)
        									   .setConnectionRequestTimeout(1000)
                                               .setSocketTimeout(2000)
                                               .build();
    builder.setDefaultRequestConfig(requestConfig);
    CloseableHttpClient client = builder.build();
}
```

> 위 코드에서 사용한 각 설정은 아래와 같다.
>
> - maxConnPerRoute : 라우트별 커넥션 최대 개수
> - maxConnTotal : 전체 커넥션 최대 개수
> - connectionTimeToLive : 지속 커넥션이 얼마나 오랫동안 유지되는지를 설정한다.
> - connectTimeout : 소켓 connectTimeout
> - socketTimeout : 소켓 soTimeout
> - connectionRequestTimeout :  커넥션 풀에서 커넥션을 꺼내는데 timeout

HttpClientBuilder 클래스의 build 메소드를 보면 아주 긴 코드를 볼 수 있다. 코드 중간에 아래와 같이 HttpClientConnectionManager를 생성하는 것을 볼 수 있다.

``` java
// HttpClientBuilder.build 메소드

final PoolingHttpClientConnectionManager poolingmgr = new PoolingHttpClientConnectionManager(
     RegistryBuilder.<ConnectionSocketFactory>create()
         .register("http", PlainConnectionSocketFactory.getSocketFactory())
         .register("https", sslSocketFactoryCopy)
         .build(),
     null,
     null,
     dnsResolver,
     connTimeToLive, // HttpClientBuilder.setConnectionTimeToLive
     connTimeToLiveTimeUnit != null ? connTimeToLiveTimeUnit : TimeUnit.MILLISECONDS);

// 생략
if (maxConnTotal > 0) {
     poolingmgr.setMaxTotal(maxConnTotal);  // HttpClientBuilder.setMaxConnTotal
 }
 if (maxConnPerRoute > 0) {
     poolingmgr.setDefaultMaxPerRoute(maxConnPerRoute); // HttpClientBuilder.setMaxConnPerRoute
 }
```

PoolingHttpClientConnectionManager 객체 생성 후에 PoolingHttpClientConnectionManager 클래스의 setMaxTotal, setDefaultMaxPerRoute 메소드를 통해 최대 커넥션 개수, 라우트 당 최대 커넥션 개수를 설정한다. 이 때 파라미터로 들어가는 값은 HttpClientBuilder.setMaxConnTotal, setMaxConnPerRoute 메소드를 통해 전달해준 값이다. (즉 각각 10, 100으로 설정된다)

또한 RequestConfig 객체를 통해 디폴트 connectTimeout, socketTimeout 설정을 할 수 있다.

> 참고로 HttpClientBuilder.build 메소드를 통해 생성되는 HttpClient의 타입은 InternalHttpClient 이다. InternalHttpClient 클래스와 PoolingHttpClientConnectionManager 클래스는 ThreadSafe 하므로 매번 객체를 생성해서 사용하기보다는 공유해서 사용하는것이 좋다.

#### InternalHttpClient#doExecute

실제로 Http 요청을 보내는 부분은 InternalHttpClient의 doExecute 메소드이다. 이 메소드의 코드는 아래와 같다.

``` java
protected CloseableHttpResponse doExecute(
        final HttpHost target,
        final HttpRequest request,
        final HttpContext context) throws IOException, ClientProtocolException {
    
    HttpExecutionAware execAware = null;
    if (request instanceof HttpExecutionAware) {
        execAware = (HttpExecutionAware) request;
    }
    try {
        final HttpRequestWrapper wrapper = HttpRequestWrapper.wrap(request, target);
        final HttpClientContext localcontext = HttpClientContext.adapt(
                context != null ? context : new BasicHttpContext());
        RequestConfig config = null;
        
        // localcontext 세팅
        setupContext(localcontext);
        // route 결정
        final HttpRoute route = determineRoute(target, wrapper, localcontext);
        // Http 요청 보내는 부분
        return this.execChain.execute(route, wrapper, localcontext, execAware);
    } catch (final HttpException httpException) {
        throw new ClientProtocolException(httpException);
    }
}
```

InternalHttpClient는 Http 요청을 보내고 받기 위해 ClientExecChain을 사용한다. ClientExecChain는 데코데이터 패턴을 사용해서 Http 요청을 처리한다. 실제로 Http 요청을 보내고 받는 역할을 하는 ClientExecChain는 MainClientExec이며 MainClientExec를 감싸는 여러개의 데코레이터 객체가 존재한다. 

``` java
public interface ClientExecChain {

    CloseableHttpResponse execute(
            HttpRoute route,
            HttpRequestWrapper request,
            HttpClientContext clientContext,
            HttpExecutionAware execAware) throws IOException, HttpException;

}
```

HttpClientBuilder를 사용해서 InternalHttpClient을 생성한 경우 다음과 같은 순서로 ClientExecChain의 execute 메소드가 호출된다.

- BackoffStrategyExec -> RedirectExec -> ServiceUnavailableRetryExec -> RetryExec -> ProtocolExec -> MainClientExec

> 데코레이터 객체들은 HttpClientBuilder 설정에 따라 추가되지 않을 수 있다. 예를 들어 HttpClientBuilder.disableAutomaticRetries() 메소드를 호출했다면, RetryExec는 제외된다. 

MainClientExec 객체는 HttpClientBuilder의 build 메소드에서 생성되고, 생성된 객체는 InternalHttpClient의 생성자를 통해서 전달한다. 

MainClientExec 객체는 HttpClientConnectionManager 객체로부터 커넥션을 가져온다. 이 때 HttpClientConnectionManager도 HttpClientBuilder의 build 메소드에서 생성된 것으로 PoolingHttpClientConnectionManager이다. 

> HttpClientBuilder.setConnectionManager 메소드를 통해서 MainClientExec 객체가 사용하는 HttpClientConnectionManager 객체를 직접 넘겨줄수도 있다. 

``` java
// MainClientExec#execute

final ConnectionRequest connRequest = connManager.requestConnection(route, userToken);

// ... 생략

final RequestConfig config = context.getRequestConfig();

final HttpClientConnection managedConn;
try {
    final int timeout = config.getConnectionRequestTimeout();
    managedConn = connRequest.get(timeout > 0 ? timeout : 0, TimeUnit.MILLISECONDS);
}

// ... 생략

if (config.isStaleConnectionCheckEnabled()) {
    // validate connection
    if (managedConn.isOpen()) {
        this.log.debug("Stale connection check");
        if (managedConn.isStale()) {
            this.log.debug("Stale connection detected");
            managedConn.close();
        }
    }
}
```

위의 코드는 MainClientExec#execute 메소드 중에서 커넥션을 가져오는 부분이다. HttpClientConnectionManager#requestConnection 메소드를 통해 ConnectionRequest 객체를 가져오고, ConnectionRequest 객체의 get 메소드를 통해서 HttpClientConnection 객체를 가져온다.

전체적인 의존관계는 아래와 같다.

InternalHttpClient#doExecute -> MainClientExec#execute -> PoolingHttpClientConnectionManager#requestConnection

> InternalHttpClient, MainClientExec, PoolingHttpClientConnectionManager 객체는 모두 HttpClientBuilder의 build 메소드에서 생성이 된다.

### PoolingHttpClientManager

아래 메소드는 PoolingHttpClientManager의 requestConnection 메소드이다. PoolingHttpClientManager는 커넥션풀을 관리하기 위해 CPool 클래스를 사용한다.

``` java
@Override
public ConnectionRequest requestConnection(
        final HttpRoute route,
        final Object state) {
    
    // 생략
    final Future<CPoolEntry> future = this.pool.lease(route, state, null);
    return new ConnectionRequest() {

        @Override
        public boolean cancel() {
            return future.cancel(true);
        }

        @Override
        public HttpClientConnection get(
                final long timeout,
                final TimeUnit tunit) throws InterruptedException, ExecutionException, ConnectionPoolTimeoutException {
            final HttpClientConnection conn = leaseConnection(future, timeout, tunit);
            if (conn.isOpen()) {
                final HttpHost host;
                if (route.getProxyHost() != null) {
                    host = route.getProxyHost();
                } else {
                    host = route.getTargetHost();
                }
                final SocketConfig socketConfig = resolveSocketConfig(host);
                conn.setSocketTimeout(socketConfig.getSoTimeout());
            }
            return conn;
        }
    };
}
```

CPool 클래스는 커넥션을 Route별로 관리한다. 각 Route별로 최대 커넥션 개수는 maxPerRoute 속성으로 제한할 수 있다. 

ConnectionRequest의 get() 메소드를 호출해서 HttpClientConnection을 가져할 때 아래 메소드가 실행된다. 아래 메소드는 AbstractConnPool#lease 메소드의 일부이다. 

``` java

// AbstractConnPool#lease

public E get(final long timeout, final TimeUnit tunit) throws InterruptedException, ExecutionException, TimeoutException {
    
    final E entry = entryRef.get();
    if (entry != null) {
        return entry;
    }
    
    synchronized (this) {
        try {
            for (;;) {
                // 1
                final E leasedEntry = getPoolEntryBlocking(route, state, timeout, tunit, this);
                // 2
                if (validateAfterInactivity > 0)  {
                    if (leasedEntry.getUpdated() + validateAfterInactivity <= System.currentTimeMillis()) {
                        if (!validate(leasedEntry)) {
                            leasedEntry.close();
                            release(leasedEntry, false);
                            continue;
                        }
                    }
                }
                // entryRef에 커넥션을 setting한다. get()를 중복해서 호출해도 같은 객체를 반환한다.
                entryRef.set(leasedEntry);
                done.set(true);
                // 콜백 메소드를 실행한다.
                onLease(leasedEntry);
                if (callback != null) {
                    callback.completed(leasedEntry);
                }
                return leasedEntry;
            }
        } catch (final IOException ex) {
            done.set(true);
            if (callback != null) {
                callback.failed(ex);
            }
            throw new ExecutionException(ex);
        }
    }
}

```

위 메소드는 아래와 같은 순서로 실행된다.

- 1 getPoolEntryBlocking 메소드를 통해서 커넥션 풀에서 커넥션을 가져온다. 만약에 커넥션 풀에 커넥션이 존재하지 않으면 새로운 커넥션을 생성한다. 커넥션은 라우트별로 관리된다.
- 2 만약에 커넥션이 validateAfterInactivity 보다 더 오랜기간 동안 사용이 되지 않았으면 해당 커넥션이 유효한지를 체크한다. 만약에 유효하지 않다면 해당 커넥션을 종료하고 풀에서 제거한다.
