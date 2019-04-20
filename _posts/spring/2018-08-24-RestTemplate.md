---
layout: post
title: "RestTemplate" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring]
---

# RestTemplate

Spring에서 제공하는 RestTemplate은 강력하고, 인기있는 자바 기반 REST 클라이언트이다. 개념적으로는 다른 템플릿 클래스와 유사하다. RestTemplate은 동기 API를 지원하며, blocking I/O에 의존한다. 동시성이 낮은 경우에 사용하면 괜찮다. 

HttpMessageConverter는 객체를 Http Request Body로 변환할 때 사용된다. 또한 response을 객체로 변환할 때도 HttpMessageConverter를 사용한다. HttpMessageConverter는 설정할 수 있다.

RestTemplate은 HTTP 클라이언트 라이브러리보다 더 높은 수준의 API를 제공한다. REST 엔드 포인트를 아주 쉽게 호출할 수 있다. 이 글에서는 RestTemplate의 HttpClient을 어떤식으로 설정하는지에 대해 주로 설명한다. 자세한 사용법은 아래 링크를 참고하길 바란다.

## HTTP Client

RestTemplate은 RESTful HTTP request를 만드는 추상화를 제공하며, 내부적으로 RestTemplate은 이러한 요청을 보내기 위해 native HttpClient을 사용한다. 

기본 생성자로 RestTemplate를 생성하면, HttpURLConnection 객체를 사용해서 요청을 보낸다. 만약에 다른 HTTP 라이브러리를 사용하고 싶다면, ClientHttpRequestFactory 구현체를 RestTemplate의 생성자로 넘겨주면 된다. 예를 들어 apache에서 제공하는 HttpClient를 사용해서 요청을 보내고 싶다면, HttpComponentsClientHttpRequestFactory 객체를 RestTemplate의 생성자로 넘겨주면 된다. 또는 setRequestFactory(ClientHttpRequestFactory requestFactory) 메소드로 HttpComponentsClientHttpRequestFactory 객체를 설정해주면 된다.

``` java
RestTemplate template = new RestTemplate(new HttpComponentsClientHttpRequestFactory());
```

다음과 같은 HttpClient를 지원한다.

* Apache HttpComponents
* Netty
* OkHttp

각각의 ClientHttpRequestFactory는 설정을 통해 커넥션 풀이나, credentials 등을 설정할 수 있다.

#### HttpClient timeout 설정

HttpComponentsClientHttpRequestFactory의 timeout 설정은 매우 간단하다.

``` java
RestTemplate restTemplate = new RestTemplate(getClientHttpRequestFactory());
 
private ClientHttpRequestFactory getClientHttpRequestFactory() {
    int timeout = 5000;
    HttpComponentsClientHttpRequestFactory clientHttpRequestFactory
      = new HttpComponentsClientHttpRequestFactory();
    clientHttpRequestFactory.setConnectTimeout(timeout);
    return clientHttpRequestFactory;
}
```

위와 같이 설정하면 connection timeout을 설정할 수 있다. 하지만 HttpComponentsClientHttpRequestFactory 메소드를 통해서 설정할 수 있는 request 설정은 connectTimeout, connectionRequestTimeout, readTimeout 3가지 설정 뿐이다. 만약 그 외에 설정을 하고 싶으면 아래와 같이 할 수 있다.

``` java
private ClientHttpRequestFactory getClientHttpRequestFactory() {
    int timeout = 5000;
    RequestConfig config = RequestConfig.custom()
      .setConnectTimeout(timeout)
      .setConnectionRequestTimeout(timeout)
      .setSocketTimeout(timeout)
      .build();
    CloseableHttpClient client = HttpClientBuilder
      .create()
      .setDefaultRequestConfig(config)
      .build();
    return new HttpComponentsClientHttpRequestFactory(client);
}
```

#### Socket connection pool

RestTemplate을 기본 생성자로 생성하는 경우, ClientHttpRequestFactory 구현체로 SimpleClinetHttpRequestFactory를 사용한다. SimpleClinetHttpRequestFactory는 REST API를 호출할 때마다, HttpURLConnection 객체를 생성한다. 즉 소켓을 재사용하지 않고, 요청을 보낼때마다 새로운 소켓 연결을 시도한다. 따라서 외부 호출이 많은 서비스에서 RestTemplate을 사용한다면, 반드시 connection pooling을 해야한다. 

connection pooling을 위해서 아래와 같이 RestTemplate을 생성하면 된다.

``` java
HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
CloseableHttpClient httpClient = HttpClientBuilder.create()
                                                  .setMaxConnTotal(appOrderProperties.getMaxConnection())
                                                  .setMaxConnPerRoute(appOrderProperties.getMaxConnectionPerRoute())
                                                  .build();
factory.setHttpClient(httpClient);
return new RestTemplate(factory);
```

* maxConnPerRoute : 단일 IP:PORT 쌍에 대한 커넥션의 수를 제한한다.
* maxConnTotal : 총 커넥션 수를 제한한다.

## 특징

#### Header

요청 헤더를 명시하기 위해서는 exchange() 메소드를 사용해라.

``` java
String uriTemplate = "http://example.com/hotels/{hotel}";
URI uri = UriComponentsBuilder.fromUriString(uriTemplate).build(42);

RequestEntity<Void> requestEntity = RequestEntity.get(uri)
        .header(("MyRequestHeader", "MyValue")
        .build();

ResponseEntity<String> response = template.exchange(requestEntity, String.class);

String responseHeader = response.getHeaders().getFirst("MyResponseHeader");
String body = response.getBody();
```

응답해더를 받고 싶다면, ResponseEntity 객체를 반환하는 메소드를 사용하면 된다.

#### Body

RestTemplate는 API 호출 결과를 HttpMessageConverter로 사용해서 객체로 변환한다. POST의 경우, 두번째 파라미터 객체는 요청 바디로 serialized된다.

``` java
URI location = template.postForLocation("http://example.com/people", person);
```

ContentType 헤더는 명시적으로 설정할 필요가 없다. 대부분의 경우, source 객체 타입을 기반으로 호환 가능한 MessageConverter를 찾는다. 그러면 선택된 MessageConverter가 ContentType 헤더를 적절하게 설정한다. 필요한 경우, exchange 메소드를 사용해서 "ContentType" 요청 헤더를 명시적으로 제공할 수 있다. ContentType 헤더를 명시적으로 제공한 경우, ContentType 헤더에 따라 MessageConverter가 결정된다.

아래와 같은 GET 메소드의 응답 바디는 결과 객체로 deserialized 된다.

``` java
Person person = restTemplate.getForObject("http://example.com/people/{id}", Person.class, 42);
```

요청의 Accept 헤더는 명시적으로 설정할 필요가 없다. 대부분의 경우 응답 객체 타입을 기반으로 호환되는 MessageConverter를 찾으며, 선택된 MessageConverter는 Accept 헤더를 채운다. 마찬가지로 Accept 헤더를 명시적으로 주고 싶은 경우엔 exchange 메소드를 사용하면 된다.

기본적으로 RestTemplate는 내장된 MessageConverter를 등록한다. MessageConverter는 클래스패스에 따라 선택적으로 등록된다. MessageConverter를 직접 추가할 수도 있다.

#### Message Conversion

Spring web 모듈은 HTTP 요청과 응답의 바디를 읽고 쓸 수 있는 HttpMessageConverter 클래스를 포함하고 있다. HttpMessageConverter는 클라이언트쪽에서는 주로 RestTemplate 안에서 사용되고, 서버쪽에서는 Spring MVC RestController에서 사용된다.

MIME(Main Media) 타입을 위한 HttpMessageConverter 구현 클래스는 프레임워크에서 제공한다. 그리고 이러한 MessageConverter는 자동으로 등록된다. (RestTemplate, RequestMethodHandlerAdapter)

모든 MessageConverter는 기본 미디어 타입을 사용된다. 하지만 supportedMediaTypes 빈 프로퍼티를 설정함으로써 오버라이드 될 수 있다. 스프링은 아래와 같은 기본 HttpMessageConverter들을 제공한다.

* StringHttpMessageConverter
* FormHttpMessageConverter
* ByteArrayHttpMessageConverter
* MarshallingHttpMessageConverter
* MappingJackson2HttpMessageConverter
* MappingJackson2XmlHttpMessageConverter
* SourceHttpMessageConverter
* BufferedImageHttpMessageConverter


#### Jackson Json View

Jackson JSON 뷰를 지정하여 객체의 하위 속성만 serialize할 수 있다.

``` java
MappingJacksonValue value = new MappingJacksonValue(new User("eric", "7!jd#h23"));
value.setSerializationView(User.WithoutPasswordView.class);

RequestEntity<MappingJacksonValue> requestEntity =
    RequestEntity.post(new URI("http://example.com/user")).body(value);

ResponseEntity<String> response = template.exchange(requestEntity, String.class);
```

## ResponseErrorHandler

ResponseErrorHandler는 전략 인터페이스로 API 호출 응답을 보고, 에러가 있는지 여부와, 에러가 있다면 에러를 어떻게 처리할 것인지를 결정한다.

``` java
/**
 * Strategy interface used by the {@link RestTemplate} to determine
 * whether a particular response has an error or not.
 *
 * @author Arjen Poutsma
 * @since 3.0
 */
public interface ResponseErrorHandler {

	/**
	 * Indicate whether the given response has any errors.
	 * <p>Implementations will typically inspect the
	 * {@link ClientHttpResponse#getStatusCode() HttpStatus} of the response.
	 * @param response the response to inspect
	 * @return {@code true} if the response has an error; {@code false} otherwise
	 * @throws IOException in case of I/O errors
	 */
	boolean hasError(ClientHttpResponse response) throws IOException;

	/**
	 * Handle the error in the given response.
	 * <p>This method is only called when {@link #hasError(ClientHttpResponse)}
	 * has returned {@code true}.
	 * @param response the response with the error
	 * @throws IOException in case of I/O errors
	 */
	void handleError(ClientHttpResponse response) throws IOException;

	/**
	 * Alternative to {@link #handleError(ClientHttpResponse)} with extra
	 * information providing access to the request URL and HTTP method.
	 * @param url the request URL
	 * @param method the HTTP method
	 * @param response the response with the error
	 * @throws IOException in case of I/O errors
	 * @since 5.0
	 */
	default void handleError(URI url, HttpMethod method, ClientHttpResponse response) throws IOException {
		handleError(response);
	}
}
```

* boolean hasError(ClientHttpResponse response) throws IOException : Response가 에러가 있는지 여부를 판단한다. 일반적으로 ClientHttpResponse#getStatusCode 메소드 호출해서 응답의 HttpStatus 가지고 에러 유무를 판단한다. 만약에 에러가 있다면 true를 리턴한다. 
* void handleError(ClientHttpResponse response) throws IOException : ClientHttpResponse 객체의 에러를 처리한다. 이 메소드는 hasError(ClientHttpResponse response) 메소드가 true를 리턴한 경우에만 호출된다. 
* default void handleError(URI url, HttpMethod method, ClientHttpResponse response) throws IOException : handleError(ClientHttpResponse response) 메소드와 기본적으로 같지만, requestURL과 Http Method 정보를 추가적으로 제공한다. **참고로 RestTemplate은 hasError(ClientHttpResponse response)가 true를 반환하면, 이 메소드를 호출해서 에러를 처리한다**

### DefaultResponseErrorHandler

ResponseErrorHandler 인터페이스의 기본 구현체이다. DefaultResponseErrorHandler는 ClientHttpResponse의 Status code를 체크한다. Status Code가 4XX이거나 5XX인 경우에는 에러로 간주된다. hasError(HttpStatus) 메소드를 오버라이딩하면 Status code로 에러 여부를 판단하는 부분을 커스터마이징 할 수 있다.

``` java
protected boolean hasError(HttpStatus statusCode) {
	return (statusCode.series() == HttpStatus.Series.CLIENT_ERROR ||
			statusCode.series() == HttpStatus.Series.SERVER_ERROR);
}
```

위의 코드는 DefaultResponseErrorHandler#hasError(HttpStatus statusCode) 메소드이다. 이 메소드는 boolean hasError(ClientHttpResponse response) 메소드에서 호출이 된다. protected 접근 제어자를 가지므로 하위 클래스에서 오버라이딩 할 수 있다.

DefaultResponseErrorHandler는 4XX 에러인 경우에는 HttpClientErrorException를 던진다. 그리고 5XX 에러인 경우에는 HttpServerErrorException를 던진다. 아래는 void handleError(ClientHttpResponse response, HttpStatus statusCode) 메소드인데, handleError(ClientHttpResponse response) 메소드에서 호출이된다. 

``` java
protected void handleError(ClientHttpResponse response, HttpStatus statusCode) throws IOException {
	switch (statusCode.series()) {
		case CLIENT_ERROR:
			throw new HttpClientErrorException(statusCode, response.getStatusText(),
					response.getHeaders(), getResponseBody(response), getCharset(response));
		case SERVER_ERROR:
			throw new HttpServerErrorException(statusCode, response.getStatusText(),
					response.getHeaders(), getResponseBody(response), getCharset(response));
		default:
			throw new UnknownHttpStatusCodeException(statusCode.value(), response.getStatusText(),
					response.getHeaders(), getResponseBody(response), getCharset(response));
	}
}
```

아래 코드는 TestRestTemplate의 inner 클래스인 NoOpResponseErrorHandler 클래스다. TestRestTemplate은 ResponseErrorHandler로 NoOpResponseErrorHandler를 사용한다. 아래 코드를 보면 알 수 있듯이 handlerError 코드에서 아무런 작업을 하지 않는다. 따라서 Status code가 4XX 혹은 5XX인 경우에도 예외를 발생시키지 않는다. (DefaultResponseErrorHandler를 상속했기 때문에 Status code가 4XX 혹은 5XX인 경우 에러로 간주하긴 할 것이다)

``` java
private static class NoOpResponseErrorHandler extends DefaultResponseErrorHandler {

	@Override
	public void handleError(ClientHttpResponse response) throws IOException {
	}

}
```

> RestTemplate은 ResponseErrorHandler를 지정하지 않으면, DefaultResponseErrorHandler를 사용한다. 따라서 RestTemplate은 Status code가 4XX이거나 5XX인 경우에는 각각 HttpClientErrorException, HttpServerErrorException를 던진다. 만약, 4XX이거나 5XX인 경우에도 예외를 던지고 싶지 않다면, ResponseErrorHandler를 적절히 구현해주면 된다.

### ResponseErrorHandler With RestTemplate

그럼 RestTemplate은 언제 ResponseErrorHandler의 메소드를 호출할까? RestTemplate의 doExecute 메소드를 보면 알 수 있는데, RestTemplate은 ClientHttpResponse#execute 메소드를 호출해서 요청을 보내고, 응답을 받은뒤에 ResponseErrorHandler의 메소드를 호출한다. 아래는 ResponseErrorHandler 메소드를 호출하는 부분의 코드이다.

``` java
protected void handleResponse(URI url, HttpMethod method, ClientHttpResponse response) throws IOException {
	ResponseErrorHandler errorHandler = getErrorHandler();
	boolean hasError = errorHandler.hasError(response);
	
	// skip

	if (hasError) {
		errorHandler.handleError(url, method, response);
	}
}
```

아래와 같은 순서로 동작한다.
* boolean hasError(ClientHttpResponse response) 메소드를 호출한다. 호출 결과는 ResponseErrorHandler의 구현에 따라 다를 것이다.
* 만약에 boolean hasError(ClientHttpResponse response) 메소드가 true를 리턴한다면 (즉 에러라면) void handleError(URI url, HttpMethod method, ClientHttpResponse response) 메소드를 호출한다. handleError(URI url, HttpMethod method, ClientHttpResponse response) 메소드는 handleError(ClientHttpResponse response) 메소드를 곧바로 호출할 것이다.

## More reading
* [spring docs](https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#rest-client-access)
* [spring.io - RestTemplate Module](https://docs.spring.io/autorepo/docs/spring-android/1.0.x/reference/html/rest-template.html)
* [Baeldung - HttpClient Connection Management](https://www.baeldung.com/httpclient-connection-management)
* [Baeldung - RestTemplate](https://www.baeldung.com/rest-template)
* [Baeldung - HttpClient](https://www.baeldung.com/httpclient-timeout)
* [StackOverflow](https://stackoverflow.com/questions/31869193/using-spring-rest-template-either-creating-too-many-connections-or-slow/31892901#31892901)
* [WebClient](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-client)
