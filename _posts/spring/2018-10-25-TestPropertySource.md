---
layout: post
title: "TestPropertySource" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring]
---

## @TestPropertySource

이 글은 [Baeldung - Test Property Source](https://www.baeldung.com/spring-test-property-source)를 번역한 글입니다. 원 글은 링크를 통해 확인하세요.

### Overview

스프링은 테스트에 도움이 되는 많은 기능을 제공한다. 때로는 테스트케이스에서 특정 설정 정보를 사용해야만 할 때가 있다. 이럴때는 @TestPropertySource 애노테이션을 사용하면 된다. @TestPropertySource 애노테이션을 통해 다른 설정보다 더 높은 우선순위를 가지는 설정 sourace를 정의할 수 있다. 

스프링부트에서 제공하는 테스트 관련 기능이 더 궁금하다면 [Testing in Spring Boot](https://www.baeldung.com/spring-boot-testing) 글을 참고하길 바란다.

### Dependencies

spring-boot-starter-test에 대한 의존성을 추가하면, 스프링에서 제공하는 테스트 지원 기능을 사용할 수 있다.

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
    <version>2.0.5.RELEASE</version>
</dependency>
```

### How to use @TestPropertySource

아래와 같이 @Value 애노테이션을 통해 프로퍼티의 값을 주입받는 빈이 있다.

``` java
@Component
public class ClassUsingProperty {
     
    @Value("${baeldung.testpropertysource.one}")
    private String propertyOne;
     
    public String retrievePropertyOne() {
        return propertyOne;
    }
}
```

테스트 클래스에서는 @TestPropertySource 애노테이션을 사용해서 새로운 설정 Source를 정의하고 프로퍼티의 값을 오버라이딩 한다.

``` java
@RunWith(SpringRunner.class)
@ContextConfiguration(classes = ClassUsingProperty.class)
@TestPropertySource
public class DefaultTest {
 
    @Autowired
    ClassUsingProperty classUsingProperty;
 
    @Test
    public void givenDefaultTPS_whenVariableRetrieved_thenDefaultFileReturned() {
        String output = classUsingProperty.retrievePropertyOne();
 
        assertThat(output).isEqualTo("default-value");
    }
}
```

일반적으로 @TestPropertySource는 @ContextConfiguration과 함께 사용된다. @ContextConfiguration는 애플리케이션 컨텍스트를 설정하고 생성하기 위해서 사용된다. 기본적으로 @TestPropertySource 애노테이션은 해당 애노테이션이 추가된 클래스를 기준으로 properties 파일을 로드한다. 예를 들어 테스트 클래스가 "com.baeldung.testpropertysource" 패키지에 있다면, 클래스패스에 "com/baeldung/testpropertysource/DefaultTest.properties" 파일이 있어야한다. 

locations 속성을 통해서 설정 파일의 위치를 변경할 수 있다. 또한 properties 속성을 통해 우선순위가 더 높은 속성을 추가할 수 있다.

``` java
@TestPropertySource(locations = "/other-location.properties",
  properties = "baeldung.testpropertysource.one=other-property-value")
```

### Conclusion

@TestProperties를 사용한 예제는 [Github Repository](https://github.com/eugenp/tutorials/tree/master/testing-modules/spring-testing)에서 확인하실 수 있습니다.

