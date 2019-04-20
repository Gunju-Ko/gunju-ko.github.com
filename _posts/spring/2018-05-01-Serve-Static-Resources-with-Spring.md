---
layout: post
title: "Serve static resources with spring" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring]
---

# ResourceHttpRequestHandler

ResourceHandlerRegistry는 ResourceHttpRequestHandler를 설정하여 클래스패스, WAR, 파일시스템에 존재하는 정적 자원을 제공하기 위해서 사용된다. ResourceHandlerRegistry는 설정 클래스 내에서 프로그래밍 방식으로 설정할 수 있다.

## Serving a Resource Stored in the WAR
만약에 webapp/resources 폴더에 위치한 특정 css 파일을 가져온다고 해보자. 주로 정적 파일은 webapp/resources 폴더에 위치한다. 그러면 아래와 같은 설정을 추가해주면 된다.

``` java
@Configuration
@EnableWebMvc
public class MvcConfig extends WebMvcConfigurerAdapter {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry
          .addResourceHandler("/resources/**")
          .addResourceLocations("/resources/"); 
    }
}
```

위의 예제에 대해서 설명을 해보면, 우선 리소스 핸들러 정의를 추가하여 외부 URL 경로를 설정한다. 그런 다음 외부 URL 경로를 내부적으로 리소스가 실제로 위치한 경로에 매핑한다. 여러 개의 리소스 핸들러를 정의할 수도 있다.

아래 html 파일의 다음 라인은 webapp/resources 디렉토리 내에 위치한 myCss.css 파일을 가져온다.

``` htmlbars
<link href="<c:url value="/resources/myCss.css" />" rel="stylesheet">
```

## Serving a Resource Stored in the File System
`/files/**` URL 패턴과 일치하는 요청이 들어올 때마다 `/opt/files/` 디렉토리에 저장된 리소스를 제공하려고 한다. 이는 아래와 같이 설정을 해주면 된다.

``` java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry
      .addResourceHandler("/files/**")
      .addResourceLocations("file:/opt/files/");
 }
```

리소스 위치를 설정하고 나면, 매핑된 URL 패턴을 사용해서 파일 시스템에 저장된 이미지등을 불러올 수 있다.

## Configuring Multiple Locations for a Resource
둘 이상의 위치에서 리소스를 검색하려면 다음과 같이 하면 된다. addResourceLocations 메소드에서 여러 개의 위치를 설정할 수 있다. 리소스를 찾을때까지 위치의 목록을 순서대로 찾는다. 아래는 간단한 예제이다.

``` java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry
      .addResourceHandler("/resources/**")
      .addResourceLocations("/resources/","classpath:/other-resources/");
}
```

## The New ResourceResolvers
Spring 4.1은 새로운 클래스인 ResourceResolver와 ResourceChainRegistration를 제공한다. 새로운 클래스들을 사용하면, 정적 리소스를 로딩할 때 브라우저의 성능을 최적화할 수 있다.

### The PathResourceResolver
가장 간단한 ResourceResolver로 URL 패턴을 가지고 리소스를 찾는다. 사실 ResourceChainRegistration에 어떠한 ResourceResolver도 추가하지 않았다면 PathResourceResolver를 기본으로 사용한다. 아래는 간단한 예제이다.

``` java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry
      .addResourceHandler("/resources/**")
      .addResourceLocations("/resources/","/other-resources/")
      .setCachePeriod(3600)
      .resourceChain(true)
      .addResolver(new PathResourceResolver());
```

* Resource 체인에 PathResourceResolver만 등록했다. 밑에서 하나 이상의 ResourceResolver를 등록하는 것에 대해서 설명하도록 하겠다.
* 리소스는 3600초 동안 브라우저에서 캐시될 것이다.
* resourceChain 메소드의 리턴 타입은 ResourceChainRegistration이다. ResourceChainRegistration 클래스의 addResolver 메소드를 통해서 하나 이상의 ResourceResolver를 추가할 수 있다.

### The GzipResourceResolver

정적 리소스를 압축함으로써 네트워크 bandwith를 최적화할 필요가 있을 때 GzipResourceResolver를 사용할 수 있다. GzipResourceResolver는 요청된 리소스에 대한 검색을 다른 resolver에게 위임한다. 만약 리소스를 찾고, request의 "Accept-Encoding" 헤더에 gzip이 포함된 경우에는 해당 리소스를 압축해서 보낸다.
GzipResourceResolver는 PathResourceResolver를 설정했던 방식대로 설정하면 된다.

``` java
registry
  .addResourceHandler("/other-files/**")
  .addResourceLocations("file:/Users/Me/")
  .setCachePeriod(3600)
  .resourceChain(true)
  .addResolver(new GzipResourceResolver());
```

아래와 같은 curl 요청은 /Users/Me 디렉토리에 위치한 Home.html 파일의 압축 버전을 가져온다.

```
curl -H  "Accept-Encoding:gzip,deflate" http://localhost:8080/handling-spring-static-resources/other-files/Hello.html
```

위의 요청에서 주목할 만한 점은 요청 헤더에 "Accept-Encoding:gzip,deflate"를 추가했다는 것이다. 이 헤더가 중요한 이유는 GzipResourceResolver는 gzip으로 응답을 보내도 되는 경우에만 실행되기 때문이다.

### Chaining ResourceResolvers
리소스 찾는것을 최적화하기 위해서, ResourceResolvers는 리소스 처리를 다른 Resolver에게 위임할 수 있다. 위임을 할 수 없는 유일한 Resolver는 PathResourceResolver으로 항상 체인의 맨 마지막에 추가를 해야한다. 만약에 
resourceChain를 false로 설정하는 경우엔 디폴트로 PathResourceResolver만 사용한다. 아래의 예제는 GzipResourceResolver와 PathResourceResolver를 체이닝했다. GzipResourceResolver는 요청된 리소스에 대한 검색을 PathResourceResolver에 위임한다.

``` java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry
      .addResourceHandler("/js/**")
      .addResourceLocations("/js/")
      .setCachePeriod(3600)
      .resourceChain(true)
      .addResolver(new GzipResourceResolver())
      .addResolver(new PathResourceResolver());
}
```

## Additional Security Configuration

만약에 Spring Security를 사용한다면, 정적자원에 대한 접근을 허용해야한다. 리소스 URL에 접근하기 위해 아래와 같은 권한을 추가해야한다.

```
<intercept-url pattern="/files/**" access="permitAll" />
<intercept-url pattern="/other-files/**/" access="permitAll" />
<intercept-url pattern="/resources/**" access="permitAll" />
<intercept-url pattern="/js/**" access="permitAll" />
```

## 추가

### 핸들러와 HandlerAdapter
DispatcherServlet은 웹 요청을 실제로 처리하는 객체의 타입을 @Controller 어노테이션을 구현한 클래스로 제한하지 않는다. 실제로 거의 모든 종류의 객체로 웹 요청을 처리할 수 있다. 그래서 웹 요청을 처리하는 객체를 좀 더 범용적인 의미로 "핸들러"라고 부른다.
DispatcherServlet은 핸들러 객체의 실제 타입이 무엇인지는 상관하지 않는다. 단지, 웹 요청 처리 결과로 ModelAndView만 리턴하면 DispatcherServlet이 올바르게 동작한다. HandlerAdapter는 핸들러의 실행 결과를 DispatcherServlet이 요구하는 ModelAndView로 변환해준다. 따라서, 어떤 종류의 핸들러 객체가 사용되더라도 알맞은 HandlerAdapter만 있으면 스프링 MVC 프레임워크에 기반해서 웹 요청을 처리할 수 있다. 

### HandlerMapping
MVC 설정을 이용하면 최소 두 개 이상의 HandlerMapping이 등록된다. 각 HandlerMapping은 우선순위를 갖고 있으며, 요청이 들어왔을때 DispatcherServlet은 우선순위에 따라 HandlerMapping에 요청을 처리할 핸들러 객체를 의뢰한다.
우선순위가 높은 HandlerMapping이 요청을 처리할 핸들러 객체를 리턴하면, 그 핸들러 객체를 이용한다. 만약 HandlerMapping이 null을 리턴하면 그 다음 우선순위를 갖는 HandlerMapping을 이용한다. 이렇게 HandlerMapping이 null을 리턴하지 않을 때까지 이 과정을 반복하고 마지막 HandlerMapping까지 null을 리턴하면, 404 에러 코드를 응답한다.
HandlerMapping이 요청을 처리할 핸들러 객체를 리턴하면, 핸들러 객체를 처리할 HandlerAdapter를 찾은 뒤에 HandlerAdapter에 핸들러 객체 실행을 위임한다.

아래의 코드는 DispatcherServlet의 doDispatch 메소드이다. 코드를 보면 위의 설명한 순서대로 실행됨을 알 수 있다.
1. getHandler 메소드를 통해서 요청을 처리할 핸들러 객체를 리턴한다. getHandler 메소드는 HandlerMapping를 사용해서 요청을 처리한 핸들러 객체를 찾는다.
2. 핸들러 객체를 처리할 HandlerAdapter를 찾는다. (getHandlerAdapter)
3. HandlerAdapter에 핸들러 객체 실행을 위임한다.

``` java
processedRequest = checkMultipart(request);
multipartRequestParsed = (processedRequest != request);

// 1. Determine handler for the current request.
mappedHandler = getHandler(processedRequest);
if (mappedHandler == null || mappedHandler.getHandler() == null) {
	noHandlerFound(processedRequest, response);
	return;
}

// 2. Determine handler adapter for the current request.
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

// Process last-modified header, if supported by the handler.
// ...skip

// 3. Actually invoke the handler.
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

if (asyncManager.isConcurrentHandlingStarted()) {
	return;
}

```

### 기본으로 등록되는 HandlerMapping, HandlerAdapter

#### HandlerMapping
* RequestMappingHandlerMapping : @Contoller 적용 빈 객체를 핸들러로 사용하는 HandlerMapping 구현체로 적용 우선순위가 높다.
* SimpleUrlHandlerMapping : `<mvc-default-servlet-handler>`, `<mvc:view-controller>` 또는`<mvc:resources>` 태그를 사용할 때 등록되는 HandlerMapping 구현체로 URL과 핸들러 객체를 매핑한다. 적용 우선순위는 낮다.

#### HandlerAdapter
* RequestMappingHandlerAdapter : @Controller 적용 빈 객체에 대한 어댑터로 RequestMappingHandlerMapping/Adapter를 이용해서 @Controller 기반 컨트롤러 객체를 핸들러로 사용하게 된다.
* HttpRequestHandlerAdapter : HttpRequestHandler 타입의 객체에 대한 어댑터이다. HttpRequestHandler 인터페이스는 주로 스프링이 기본으로 제공하는 핸들러 클래스가 구현하고 있다. 예를 들어 DefaultServletHttpRequestHandler, ResourceHttpRequestHandler 등이 있다.

> Note
> - RequestMappingHandlerMapping이 우선순위가 SimpleUrlHandlerMapping의 우선순위보다 높다. 따라서, 특정 요청이 들어올 경우 RequestMappingHandlerMapping을 먼저 확인하고, 그 다음에 SimpleUrlHandlerMapping를 확인한다.

### HttpRequestHandler
HttpRequestHandler 인터페이스에서 DefaultServletHttpRequestHandler 클래스와 ResourceHttpRequestHandler에 대해서 간략하게 설명을 하도록 하겠다.

* DefaultServletHttpRequestHandler : 클라이언트의 요청을 WAS(웹 어플리케이션 서버, 톰캣이나 웹로직 등)가 제공하는 디폴트 서블릿에 전달한다. 예를 들어, "/index.html"에 대한 처리를 DefaultServletHttpRequestHandler에 요청하면, 이 요청을 다시 디폴트 서블릿에 전달해서 처리하도록 한다. 각 WAS는 서블릿 매핑에 존재하지 않는 요청을 처리하기 위한 디폴트 서블릿을 제공한다. 요청 URL에 매핑되는 핸들러가 존재하지 않을 경우 404응답 대신, 디폴트 서블릿이 해당 요청 URL을 처리하도록 할 수 있다.
* ResourceHttpRequestHandler : 클래스패스, WAR, 파일시스템에 존재하는 정적 리소스을 제공하기 위해서 사용되며, 내부적으로는 ResourceResolver를 사용해서 리소스를 찾는다.

> Note
> - DefaultServletHttpRequestHandler와 ResourceHttpRequestHandler 모두 SimpleUrlHandlerMapping 클래스를 통해서 URL과 Handler를 매핑한다. 만약 DefaultServletHttpRequestHandler와 ResourceHttpRequestHandler를 모두 사용하도록 설정 된 경우에는 각각의 SimpleUrlHandlerMapping 객체가 HandlerMapping으로 등록된다.  이 때 ResourceHttpRequestHandler용 SimpleUrlHandlerMapping 객체의 우선순위가 더 높다. 따라서 ResourceHttpRequestHandler에 매핑되지 않는 경로에 대해서만 DefaultServletHttpRequestHandler가 처리를 하게 된다.

### ResourceHttpRequestHandler
* HanlderMapping : SimpleUrlHandlerMapping 사용
* HandlerAdapter : HttpRequestHandlerAdapter 사용
* Handler : ResourceHttpRequestHandler


## 출처
http://www.baeldung.com/spring-mvc-static-resources
책 : Spring 4.0 프로그래밍 (최범균 저)
