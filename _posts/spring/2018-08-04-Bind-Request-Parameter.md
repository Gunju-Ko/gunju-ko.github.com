---
layout: post
title: "Bind request parameter" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring]
---

# HandlerMethodArgumentResolver

이 글에서는 Converter, ConverterFactory, HandlerMethodArgumentResolver에 대해 알아보고, 어떤 용도로 사용할 수 있는지에 대해 소개한다. 

### Bind Request Parameters

기본적으로 Spring은 Request Parameter를 단순한 타입으로만 변환해준다. Request Parameter로 String, int, boolean 타입의 데이터를 전달하면, 컨트롤러 메소드 파리미터에 자동으로 바인딩된다. 하지만 컨트롤러 메소드에서 다양한 타입을 바인딩해야할 때가 있다.

### Converter

컨트롤러 메소드에서 특정 타입을 받고 싶은 경우, 사용할 수 있는 첫번째 방법은 Converter<S, T> 구현체를 제공하는 것이다. Converter 구현체는 Thread-safe 해야한다. Converter 인터페이스는 convert(S source) 메소드를 정의한다. convert 메소드에서 source를 변환하지 못하는 경우 IllegalArgumentException를 던져야한다. 

``` java

/**
 * A converter converts a source object of type {@code S} to a target of type {@code T}.
 *
 * <p>Implementations of this interface are thread-safe and can be shared.
 *
 * <p>Implementations may additionally implement {@link ConditionalConverter}.
 *
 * @author Keith Donald
 * @since 3.0
 * @param <S> the source type
 * @param <T> the target type
 */
public interface Converter<S, T> {

	/**
	 * Convert the source object of type {@code S} to target type {@code T}.
	 * @param source the source object to convert, which must be an instance of {@code S} (never {@code null})
	 * @return the converted object, which must be an instance of {@code T} (potentially {@code null})
	 * @throws IllegalArgumentException if the source cannot be converted to the desired target type
	 */
	T convert(S source);

}

```

> * Converter의 경우 빈으로 등록만 해주면 FormatterRegistry에 자동으로 등록된다. 이는 Spring Boot의 WebMvcAutoConfigurationAdapter에 의해 등록이 된다. 따라서 Spring Boot를 사용하고 있지 않다면 WebMvcConfigurer의 addFormatters 메소드를 통해 FormatterRegistry에 Converter를 등록해줘야한다.

### ConverterFactory

특정 타입의 하위 타입까지 전부 변환하고 싶은 경우가 있다. 이 경우에 각 하위타입마다 Converter를 구현하는 것보다는 ConverterFactory를 사용하는게 좋다. ConverterFactory를 사용하면 특정 타입의 하위 타입까지 전부 변환하도록 할 수 있다.

이 경우에는 ConverterFactory<S, R>를 사용하면 된다. 여기서 R는 변화하고자 하는 타입들중 가장 상위 타입이 오도록 하면 된다. 

``` java
/**
 * A factory for "ranged" converters that can convert objects from S to subtypes of R.
 *
 * <p>Implementations may additionally implement {@link ConditionalConverter}.
 *
 * @author Keith Donald
 * @since 3.0
 * @see ConditionalConverter
 * @param <S> the source type converters created by this factory can convert from
 * @param <R> the target range (or base) type converters created by this factory can convert to;
 * for example {@link Number} for a set of number subtypes.
 */
public interface ConverterFactory<S, R> {

	/**
	 * Get the converter to convert from S to target type T, where T is also an instance of R.
	 * @param <T> the target type
	 * @param targetType the target type to convert to
	 * @return a converter from S to T
	 */
	<T extends R> Converter<S, T> getConverter(Class<T> targetType);

}
```

아래의 코드는 String을 Enum 타입으로 변환할 수 있는 ConverterFactory 구현체이다. ConverterFactory 구현체는 getConverter 메소드만 구현해주면 된다. getConverter 메소드는 특정 타입에 대한 Converter를 리턴한다. 그럼 리턴된 Converter가 변환 과정을 진행한다. 

``` java
@Component
public class StringToEnumConverterFactory
  implements ConverterFactory<String, Enum> {
 
    private static class StringToEnumConverter<T extends Enum> 
      implements Converter<String, T> {
 
        private Class<T> enumType;
 
        public StringToEnumConverter(Class<T> enumType) {
            this.enumType = enumType;
        }
 
        public T convert(String source) {
            return (T) Enum.valueOf(this.enumType, source.trim());
        }
    }
 
    @Override
    public <T extends Enum> Converter<String, T> getConverter(
      Class<T> targetType) {
        return new StringToEnumConverter(targetType);
    }
}

```

### HandlerMethodArgumentResolver

HttpSession이나 데이터베이스에서 가져온 데이터를 컨트롤러 메소드 파라미터 객체에 바인딩하고자 할 경우가 있다. 이런 경우에는 HandlerMethodArgumentResolver를 사용하면 된다. 

우선 아래와 같은 어노테이션을 정의해야 한다.

``` java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface Version {
}
```

그 다음에 HandlerMethodArgumentResolver 인터페이스 구현체를 만들면 된다. 아래의 HeaderVersionArgumentResolver 클래스는 컨트롤러 메소드 파라미터에 @Version 어노테이션이 붙은 경우에, Version 헤더  값을 파라미터에 바인딩해준다.

``` java
public class HeaderVersionArgumentResolver
  implements HandlerMethodArgumentResolver {
 
    @Override
    public boolean supportsParameter(MethodParameter methodParameter) {
        return methodParameter.getParameterAnnotation(Version.class) != null;
    }
 
    @Override
    public Object resolveArgument(
      MethodParameter methodParameter, 
      ModelAndViewContainer modelAndViewContainer, 
      NativeWebRequest nativeWebRequest, 
      WebDataBinderFactory webDataBinderFactory) throws Exception {
  
        HttpServletRequest request 
          = (HttpServletRequest) nativeWebRequest.getNativeRequest();
 
        return request.getHeader("Version");
    }
}
```

HandlerMethodArgumentResolver 인터페이스에 정의된 메소드는 아래와 같다.

* boolean supportsParameter(MethodParameter methodParameter) : MethodParameter가 이 resolver에 의해서 처리될 지 여부를 결정한다. resovler에서 해당 MethodParameter를 처리하고자 할 경우에 true를 리턴한다. 
* Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception : 파라미터에 바인딩될 객체를 리턴한다. 여기서 parameter는  supportsParameter 메소드의 결과가 true인 객체만 올 수 있다. NativeWebRequest webRequest는 현재 요청 객체이다.

HandlerMethodArgumentResolver 구현체는 WebMvcConfigurer 인터페이스를 통해 등록을 해주어야 한다.

``` java
@Configuration
public class WebConfig implements WebMvcConfigurer {
 
    //...
 
    @Override
    public void addArgumentResolvers(
      List<HandlerMethodArgumentResolver> argumentResolvers) {
        argumentResolvers.add(new HeaderVersionArgumentResolver());
    }
}
```

그러면 컨트롤러에서 아래와 같이 사용할 수 있다.

``` java
@GetMapping("/entity/{id}")
public ResponseEntity findByVersion(
  @PathVariable Long id, @Version String version) {
    return ...;
}
```

보시다시피 HandlerMethodArgumentResolver.resolveArgument 메소드의 리턴타입은 Object이다. 따라서 어떠한 객체도 리턴될 수 있다. 

### 결론

위에서 설명한 것들을 사용하면, 지루한 변환 작업을 대부분 제거할 수 있다. 그리고 변환 작업을 Spring에게 위임할 수 있다. 

* 간단한 타입 변환은 Converter를 사용하면 된다.
* 특정 타입의 하위 타입까지 전부 변환하고 싶은 경우 ConverterFactory를 사용하면 된다.
* 바인딩할 데이터를 HttpSession이나 데이터베이스에서 가져와야하거나, 데이터를 가져오기 위해서 추가적인 로직이 필요하다면, HandlerMethodArgumentResolver를 사용하면 된다.

### 출처
* [baeldung](http://www.baeldung.com/spring-mvc-custom-data-binder)
 

