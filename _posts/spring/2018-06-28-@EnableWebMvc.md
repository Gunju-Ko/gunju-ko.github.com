---
layout: post
title: "@EnableWebMvc" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring]
---

## @EnableWebMvc
@Enable로 시작하는 어노테이션은 자바 설정에서 편의를 제공하기 위해서 도입되었다. @Enable 어노테이션은 개발자를 대신해서 많은 설정을 대신해준다. 

이번 글에서 살펴볼 어노테이션 @EnableWebMvc이다. @EnableWebMvc는 @Configuration 설정을 임포트하는 방식이다. 간단하게 코드로 살펴보자.

``` java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

코드를 보면 알겠지만 DelegatingWebMvcConfiguration를 임포트하고 있다. DelegatingWebMvcConfiguration는 @Configuration 설정이다.

그리고 DelegatingWebMvcConfiguration는 WebMvcConfigurer를 통해서 Web 관련 빈들을 커스터마이징 할 수 있도록 해준다.  예를 들어 RequestMappingHandlerAdapter에서 사용하는 HttpMessageConverter를 추가하고 싶다면, WebMvcConfigurer를 상속한 @Configuraion 클래스를 만든후에 configureMessageConverters(List<HttpMessageConverter<?>> converters) 메소드를 구현해주면 된다. WebMvcConfigurer 인터페이스는 WebMvc 설정과 관련된 많은 메소드를 포함하고 있기 때문에, 특정 메소드만 구현하고자 한다면 WebMvcConfigurerAdapter를 사용하면 된다. 

아래는 간단한 예제 코드이다.

``` java
@Configuration
public class SampleWebMvcConfigurerAdapter extends WebMvcConfigurerAdapter {

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new SampleHttpMessageConverter());
    }
}

```

#### DelegatingWebMvcConfiguration

WebMvcConfigurationSupport의 하위 클래스이다.  DelegatingWebMvcConfiguration는 WebMvcConfigurer 타입의 빈들을 통해 웹 관련 설정을 커스터마이징 할 수 있도록 해준다. 이 클래스는 @EnableWebMvc 어노테이션을 통해서 임포트된다. 아래의 코드를 보면 알 수 있듯이 WebMvcConfigurer 타입의 빈을 통해서 HttpMessageConverters를 등록할 수 있게 되어있다. 

``` java
@Configuration
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {

	private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();


	@Autowired(required = false)
	public void setConfigurers(List<WebMvcConfigurer> configurers) {
		if (!CollectionUtils.isEmpty(configurers)) {
			this.configurers.addWebMvcConfigurers(configurers);
		}
	}
		
	// skip 
	
	@Override
	protected void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
		this.configurers.configureMessageConverters(converters);
	}
	// skip

}
```

위의 코드를 보면 setConfigurers 메소드에서 WebMvcConfigurer 타입의 빈을 모두 주입받아서 WebMvcConfigurerComposite 타입의 객체에 주입하고 있다. 따라서 WebMvcConfigurer 타입의 빈들은 전부 WebMvcConfigurerComposite 객체에 주입이 된다. WebMvcConfigurer 타입의 빈들은 Web 관련 빈들을 초기화할 때 사용된다. 

DelegatingWebMvcConfiguration는 다음과 같은 빈들을 등록한다. 빈을 등록하는 메소드는 WebMvcConfigurationSupport에 있다. 따라서 WebMvcConfigurationSupport도 아래와 같은 빈들을 등록한다. 다만 DelegatingWebMvcConfiguration는 WebMvcConfigurer 타입의 빈들을 통해서 등록되는 빈들을 커스터마이징 할 수 있게 해주는 정도의 역할만 하고 있다.

HandlerMapping
* RequestMappingHandlerMapping
* HandlerMapping
* BeanNameUrlHandlerMapping
* HandlerMapping
* HandlerMapping

HandlerAdapter
* RequestMappingHandlerAdapter
* HttpRequestHandlerAdapter
* SimpleControllerHandlerAdapter

HandlerExceptionResolerComposite
* ExceptionHandlerExceptionResolver
* ResponseStatusExceptionResolver
* DefaultHandlerExceptionResolver

기타
* AntPathMather
* UrlPathHelper 
* ContentNegotiationManager

AntPathMatcher와 UrlPathMatcher는 PathMatchConfigurer 타입의 빈을 통해서 설정할 수 있다.

#### WebMvcConfigurationSupport

MVC 자바 설정에서 기본 설정을 제공하는 메인 클래스이다. 주로 @EnableWebMvc를 통해서 임포트된다. (더 정확히 말하면 이 클래스를 상속한 DelegatingWebMvcConfiguration가 임포트된다.) 커스터마이징이 필요하다면 WebMvcConfigurationSupport 클래스를 상속해서 필요한 메소드를 오버라이딩해도 된다. 단 이 때 서브클래스에 @Configuration 어노테이션을 반드시 붙여야한다. 그리고 @Bean 메소드가 붙은 메소드를 오버라이딩하는 경우에 @Bean 메소드를 꼭 붙여야한다. 하지만 가능하면 WebMvcConfigurer를 통해서 커스터마이징 하는것이 훨씬 좋다. 하지만 WebMvcConfigurer를 통해서 커스터마이징 하는게 불가능하다면 WebMvcConfigurationSupport 혹은 DelegatingWebMvcConfiguration 클래스를 상속해서 필요한 메소드를 오버라이딩 하는 것도 방법이다. 이 경우에는 @EnableWebMvc를 붙여서는 안된다. 

#### WebMvcConfigurer

WebMvcConfigurer 인터페이스는 MVC 네임스페이스를 이용한 설정과 동일한 설정을 하는데 필요한 메소드를 정의하고 있다. Spring MVC를 자바 기반으로 설정할 때 사용한다. 위에서 설명했듯이 HttpMessageConverter를 추가할 때 WebMvcConfigurer 인터페이스를 상속한 @Configuration 클래스를 만들고, configureMessageConverters 메소드를 구현해주면 된다. 

#### WebMvcConfigurerAdapter

WebMvcConfigurer 인터페이스는 10개가 넘는 메소드를 정의하고 있는데, 이들 메소드를 모두 구현하는 경우는 드물다. 대신 WebMvcConfigurer 인터페이스를 구현하고 있는 WebMvcConfigurerAdapter 클래스를 상속받아 필요한 메소드만 구현하는 것이 일반적이다. 
WebMvcConfigurerAdapter 클래스를 상속받아 구현한 클래스는 @Configuration 어노테이션을 적용해서, 설정 정보로 사용해야 한다. 

#### WebMvcAutoConfiguration

Spring Boot를 사용했을때 WebMvcAutoConfiguration 클래스를 통해서 Spring MVC를 자동으로 설정한다. 여기서 자동설정이 되는 조건이 중요하다. 

``` java
@Configuration
@ConditionalOnWebApplication
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class,
		WebMvcConfigurerAdapter.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class,
		ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {

```

Spring Boot에서 WebMvcAutoConfiguration이 자동으로 설정될 조건은 아래와 같다.
* 클래스패스에 Servlet.class, DispatcherServlet.class, WebMvcConfigurerAdapter.class가 존재해야한다.
* WebMvcConfigurationSupport 타입의 빈이 존재해서는 안된다.
* 웹 어플리케이션으로 실행된다. (ApplicationContext가 WebApplicationContext인 경우)

위에서 두번째 조건이 WebMvcConfigurationSupport 타입의 빈이 존재해서는 안된다는 것이다. 만약에 @EnableWebMvc 어노테이션을 사용하는 경우에는 DelegatingWebMvcConfiguration 타입의 빈을 등록한다. (그리고 DelegatingWebMvcConfiguration는 WebMvcConfigurationSupport의 하위 클래스이다) 따라서 @EnableWebMvc를 추가하는 경우에는 WebMvcAutoConfiguration 클래스가 설정으로 등록되지 않는다. WebMvcAutoConfiguration는 HttpMessageConverter를 자동으로 등록해주는 등 꽤 많은 것들을 자동으로 설정해주고 있다. 따라서 가능하면 @EnableWebMvc보다는 WebMvcAutoConfiguration를 사용하는게 좋을거 같다. (즉 Spring Boot을 사용하고 있다면 @EnableWebMvc를 추가하지 않는게 좋다)
