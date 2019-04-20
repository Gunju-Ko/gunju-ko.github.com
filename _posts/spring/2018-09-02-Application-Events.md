---
layout: post
title: "Application Events and Listeners" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring]
---

## Application Events and Listeners

SpringApplication은 ContextRefreshedEvent와 같은 일반적인 스프링 프레임워크 이벤트 외에도 몇몇 추가적인 애플리케이션 이벤트들을 발생시킨다.

> 몇몇 이벤트들은 ApplicationContext가 생성되기 전에 발생하기 때문에 리스너를 @Bean으로 등록할 수 없다. 리스너들은 SpringApplication.addListeners(..)나 SpringApplicationBuilder.listeners(...) 메소드들을 사용해 등록할 수 있다. 리스너들이 자동적으로 등록되게 하기 위해서는, META-INF/spring.factories 파일을 프로젝트에 추가하고 org.springframework.context.ApplicationListener 키값을 사용해서 리스너를 참조시킨다. 아래는 간단한 예제이다.

``` yaml
org.springframework.context.ApplicationListener=com.example.project.MyListener
```

애플리케이션 실행시 이벤트는 아래와 같은 순서대로 발생한다.

1. ApplicationStartingEvent은 실행 시작 시점에 발생한다. 이 때는 리스너들과 Initializer들의 등록을 제외하고 어떤 것도 진행 되기 전이다.
2. ApplicationEnvironmentPreparedEvent은 컨텍스트에서 사용될 Environment를 알고 있지만, 아직 컨텍스트가 생성되기 전에 전송된다.
3. ApplicationPreparedEvent은 정의된 빈들이 로딩되고 refresh가 시작되기 직전에 전송된다.
4. ApplicationStartedEvent는 컨텍스트가 refresh되고 애플리케이션과 커맨드라인 러너가 호출되기 전에 전송된다.
5. ApplicationReadyEvent는 모든 애플리케이션과 커맨드라인 러너가 호출된 이후에 전송된다. 이 시점은 애플리케이션이 요청을 처리할 준비 되었음을 가리킨다.
6. ApplicationFailedEvent은 시작 도중 예외가 발생하면 전송된다.

> 보통 애플리케이션 이벤트들을 사용할 필요가 없다. 그러나 이벤트들을 알고 있으면 편리할 수있다. 내부적으로 스프링 부트는 많은 일들을 처리하기 위해 이벤트들을 사용한다.

애플리케이션 이벤트들은 스프링 프레임워크의 이벤트 발생 메커니즘을 사용해 전송된다. 스프링 이벤트 발생 메커니즘에서는 자식 컨텍스트의 리스너에 발생된 이벤트는 부모 컨텍스트들의 리스너에게도 발생되도록 보장한다. 따라서 애플리케이션이 SpringApplication 객체의 계층관계를 사용한다면, 리스너는 같은 타입의 애플리케이션 이벤트 객체들을 여러개 받게 된다.

리스너가 자신의 컨텍스트에서 발생한 이벤트인지 자식 컨텍스트에서 발생한 이벤트인지를 구별하게 하기 위해, 애플리케이션 컨텍스트를 주입받아서 이벤트의 컨텍스트와 비교해야 한다. 컨텍스트는 ApplicationContextAware를 구현하거나 리스너가 빈일 경우 @Autowired를 사용해서 주입받을 수있다.

## 출처

* [spring boot docs](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-application-events-and-listeners)
