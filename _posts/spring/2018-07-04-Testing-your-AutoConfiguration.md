---
layout: post
title: "Testing your Auto-configuration" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring, spring-boot]
---

# Testing your Auto-configuration

ApplicationContextRunner는 Spring Boot 2.0에서 추가된 내용이다. 그 이전 버전에서는 ApplicationContextRunner를 사용할 수 없다.

## ApplicationContextRunner
Spring boot의 auto-configure는 여러 요인에 영향을 받는다. 사용자 설정 (@Bean 정의 그리고 Environment 설정), 특정 라이브러리의 존재 여부 등이 auto-configure에 영향을 준다. 
ApplicationContextRunner는 이러한 auto-configure를 테스트하기 위해서 사용한다. ApplicationContextRunner는 일반적으로 테스트 클래스의 필드로 정의하고 기본적인 configuration을 등록한다. 예를 들어 아래와 같이 ApplicationContextRunner 객체를 필드로 선언하면 UserServiceAutoConfiguration는 항상 등록된다.

``` java
private final ApplicationContextRunner contextRunner = new ApplicationContextRunner()
		.withConfiguration(AutoConfigurations.of(UserServiceAutoConfiguration.class));
```

> 참고
> 여러 auto-configuration을 등록하는 경우, 그 순서는 크게 중요하지 않습니다.

각 테스트는 특정 케이스를 나타내기 위해서 contextRunner를 사용할 수 있다. 예를 들어, 아래의 샘플은 사용자 Configuration 클래스인 UserConfiguration 클래스를 등록한다.
ApplicationContextRunner의 run 메소드는 콜백의 인자로 ApplicationContext 객체를 전달한다. 

``` java
@Test
public void defaultServiceBacksOff() {
	this.contextRunner.withUserConfiguration(UserConfiguration.class)
			.run((context) -> {
				assertThat(context).hasSingleBean(UserService.class);
				assertThat(context.getBean(UserService.class)).isSameAs(
						context.getBean(UserConfiguration.class).myUserService());
			});
}

@Configuration
static class UserConfiguration {

	@Bean
	public UserService myUserService() {
		return new UserService("mine");
	}

}
```

또한 특정 프로퍼티를 설정하려면 withPropertyValues 메소드를 사용하면 된다. 아래는 간단한 샘플이다.

``` java
@Test
public void serviceNameCanBeConfigured() {
	this.contextRunner.withPropertyValues("user.name=test123").run((context) -> {
		assertThat(context).hasSingleBean(UserService.class);
		assertThat(context.getBean(UserService.class).getName()).isEqualTo("test123");
	});
}
```

## 예제 코드

아래는 샘플 코드이다.

``` java
public class ApplicationContextRunnerExample {

    private ApplicationContextRunner contextRunner = new ApplicationContextRunner().withConfiguration(AutoConfigurations.of(SampleAutoConfiguration.class));

    @Test
    public void shouldNot_Load_Bean_With_Disabled_Property() throws Exception {
        contextRunner.withPropertyValues("sample.enabled=false")
                     .run(context -> {
                              Throwable thrown  = Assertions.catchThrowable(() -> context.getBean(SampleBean.class));
                              assertThat(thrown).isInstanceOf(NoSuchBeanDefinitionException.class);
                          }
                     );
    }

    @Test
    public void should_Load_Bean_With_Enabled_Property() throws Exception {
        contextRunner.withPropertyValues("sample.enabled=true")
                     .run(context -> assertThat(context).hasSingleBean(SampleBean.class));
    }

    @Test
    public void should_Load_Bean_With_As_Default() throws Exception {
        contextRunner.run(context -> assertThat(context).hasSingleBean(SampleBean.class));
    }

    @Configuration
    @ConditionalOnProperty(name = "sample.enabled", havingValue = "true", matchIfMissing = true)
    public static class SampleAutoConfiguration {

        @Bean
        public SampleBean sampleBean() throws Exception {
            return new SampleBean();
        }
    }

    public static class SampleBean {

    }
}
```

우선 SampleAutoConfiguration 클래스가 있는데, 이 클래스는 sample.enabled 속성이 true인 경우엔 SampleBean 타입의 객체를 빈으로 등록한다. (기본값은 true이다)

shouldNot_Load_Bean_With_Disabled_Property 테스트에서는 sample.enabled 속성을 false로 하고 있기 때문에 SampleAutoConfiguration에서 SampleBean 타입의 빈을 등록하지 않는다. 테스트를 보면 알 수 있듯이 SampleBean 타입의 빈 객체를 ApplicationContext에서 찾으려고 하는 순간 NoSuchBeanDefinitionException 예외가 발생한다. 

그 아래 2개의 테스트는 "sample.enabled" 속성을 "true"한 경우가 설정을 하지 않은 경우이다. 이 두가지 경우 모두 SampleAutoConfiguration에서 SampleBean 타입의 빈을 등록한다.

## 참고
* [bistro](http://bistros.tistory.com/entry/springboot-20%EC%9D%98-RouterFunction-%EC%8A%A4%EC%BA%90%EB%8B%9D-%EB%B0%A9%EB%B2%95?category=457290)
* [Spring Boot Reference](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/)
