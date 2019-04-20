---
layout: post
title: "Spring Boot Embedded Tomcat" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring, spring-boot]
---

## Spring Boot Embedded Tomcat

이 글은 [spring boot reference](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-embedded-web-servers.html) 를 번역한 글입니다. 자세한 내용은 공식문서를 확인하시길 바랍니다.

스프링부트 웹  애플리케이션은 임베디드 웹 서버를 포함하고 있다. 

- sprinb-boot-starter : 서블릿 스택 애플리케이션으로 기본적으로 tomcat을 사용한다. (spring-boot-starter-tomcat) 하지만 spring-boot-starter-jetty 혹은 spring-boot-starter-undertow을 대신 사용할 수도 있다.
- spring-boot-starter-webflux : Reactive 스택 애플리케이션은 netty를 기본으로 사용한다. (spring-boot-starter-reactor-netty) 하지만 tomcat, jetty, undertow를 사용할 수도 있다.

만약에 기본 HTTP 서버가 아닌 다른 서버를 사용하고자 한다면 기본 dependecy를 제외시키고, 사용하고자 하는 HTTP 서버의 dependency를 추가해야 한다. 예를 들어 아래와 같이 jetty를 사용하고자 하는 경우 tomcat을 의존성에서 제외시키고, jetty를 포함한다.

``` xml
<properties>
	<servlet-api.version>3.1.0</servlet-api.version>
</properties>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
	<exclusions>
		<!-- Exclude the Tomcat dependency -->
		<exclusion>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-tomcat</artifactId>
		</exclusion>
	</exclusions>
</dependency>
<!-- Use Jetty instead -->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

### Disabling the Web Server

Embedded 웹서버를 사용하고 싶지 않다면, 아래 설정을 추가하면 된다.

``` yaml
spring.main.web-application-type: none
```

### Change the HTTP Port

포트는 디폴트로 8080을 사용한다. 만약에 포트를 변경하고 싶다면 `server.port` 속성을 사용해서 변경하면 된다. 만약에 WebApplicatonContext는 그대로 사용하면서 Http 엔드포인트를 없애고 싶다면 server.port 속성을 -1로 하면 된다.

server.port 속성을 0으로 하면, 현재 사용하고 있지 않은 포트 중에서 하나를 랜덤하게 사용한다.

### Discover the HTTP Port at Runtime

서버가 실행중인 포트는 로그를 통해 확인할 수 있다. 만약에 @SpringBootTest(webEnvironment=WebEnvironment.RANDOM_PORT)를 사용해서 테스트를 하는 경우 @LocalServerPort 애노테이션을 통해서 포트 정보를 필드로 주입받을 수 있다. 아래는 간단한 예제이다.

``` java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(webEnvironment=WebEnvironment.RANDOM_PORT)
public class MyWebIntegrationTests {

	@Autowired
	ServletWebServerApplicationContext server;

	@LocalServerPort
	int port;

	// ...

}
```

> @LocalServerPort는 @Value("${local.server.port}")의 메타 애노테이션이다. @LocalServerPort는 테스트에서만 사용 가능하다.

### Enable HTTP Response Compression

Http 응답 압축은 Jetty, Tomcat, Undertow에서 가능하다. 아래와 같이 설정하면 응답이 압축된다.

```yaml
server.compression.enabled: true
```

응답의 크기가 2048byte 보다 큰 경우에만 압축을 한다. `server.compression.min-response-size` 속성을 통해서 이 값을 설정할 수 있다. 

> 단 Transfer-Encoding: chunked의 경우 응답의 크기를 알 수 없기 때문에 무조건 압축을해서 보낸다.

기본적으로 응답의 content-type이 아래와 같은 경우에만 압축을한다. (`server.compression.mime-types` 속성을 통해서  설정이 가능하다)

- text/html
- text/xml
- text/plain
- text/css
- text/javascript
- application/javascript
- application/json
- application/xml

### Configure the Web Server

웹 서버를 커스터마이징 하려면 application.properties에 새로운 속성을 추가하면 된다.  `server.*` 설정이나 `server.tomcat.*` 설정들을 통해서 웹서버를 커스터마이징할 수 있다. 하지만 커스터마이징 하고자 하는 부분이 설정으로 지원되지 않는 경우, [WebServerFactoryCustomizer](https://docs.spring.io/spring-boot/docs/2.1.2.RELEASE/api/org/springframework/boot/web/server/WebServerFactoryCustomizer.html) 클래스를 이용해야 한다. WebServerFactoryCustomizer에서는 Server Factory에 접근할 수 있다. 예를 들어 아래는 Tomcat을 커스터마이징 하는 WebServerFactoryCustomizer이다.

``` java
@Component
public class MyTomcatWebServerCustomizer
		implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

	@Override
	public void customize(TomcatServletWebServerFactory factory) {
		// customize the factory here
	}
}
```

WebServerFactory에 액세스하면 커스터마이저를 추가하여 커넥터, 서버 리소스 또는 서버 자체와 같은 특정 부분을 커스트마이징 할 수 있다.

최후의 수단으로, 커스터마이징한 WebServerFactory 타입의 빈을 등록할수도 있다. (이는 스프링부트가 제공하는 WebServerFactory 타입의 빈을 대체한다) 직접 WebServerFactory 타입의 빈을 등록하는 경우 `server.*` 프로퍼티는 더이상 적용되지 않는다. 

> 임베디드 Tomcat에 server.* 프로퍼티를 적용하는 부분은 TomcatServletWebServerFactoryCustomizer 클래스 코드를 보면 된다. 해당 클래스는 WebServerFactoryCustomizer 인터페이스의 구현 클래스이다.

### Embedded Tomcat KeepAlive

Embedded Tomcat을 사용하는 경우 하나의 커넥션으로 HTTP 요청을 최대 100개까지 보낼 수 있다. Embedded Tomcat의 경우 기본적으로 요청에 "Connection: close" 헤더가 포함되어 있지 않은 경우 커넥션을 계속 유지한다. 따라서 클라이언트는 하나의 커넥션으로 여러개의 HTTP 요청을 보낼 수 있다. 하나의 커넥션으로 100개의 요청을 보내는 경우, Embedded Tomcat은 응답 헤더에 Connection:close 를 포함시키고, 커넥션의 연결을 끊는다. 그리고 커넥션으로 60초 이상동안 새로운 요청이 오지 않으면 Tomcat은 해당 커넥션의 연결을 끊는다. 만약에 하나의 커넥션으로 더 많은 요청을 처리하고 싶거나, 사용하지 않는 커넥션을 더 빨리 끊고 싶다면 아래은 코드를 추가하면 된다.

``` java
@Component
public class TomcatCustomizer implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.addConnectorCustomizers((TomcatConnectorCustomizer) connector -> {
            ProtocolHandler protocolHandler = connector.getProtocolHandler();
            if (protocolHandler instanceof AbstractHttp11Protocol) {
                applyProperties((AbstractHttp11Protocol) protocolHandler);
            }
        });
    }

    private void applyProperties(AbstractHttp11Protocol protocolHandler) {
        protocolHandler.setMaxKeepAliveRequests(300);
        protocolHandler.setKeepAliveTimeout(30000);
    }
}

```

다음과 같은 클래스를 빈으로 등록하면 maxKeepAliveRequest는 300이 되며, keepAliveTimeout은 30초가 된다.

> 이 글에서는 아래와 같은 주제에 대해선 다루지 않습니다. 궁금하신분들은 공식문서를 찾아보시길 바랍니다.
>
> - SSL
> - HTTP/2
> - Add a Servlet, Filter, or Listener to an Application
> - Configure Access Logging
> - Running Behind a Front-end Proxy Server
> - Enable Multiple Connectors with Tomcat
> - Create WebSocket Endpoints Using @ServerEndpoint 



