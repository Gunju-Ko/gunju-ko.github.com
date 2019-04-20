---
layout: post
title: "Spring NestedRuntimeException" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring]
---
# Spring NestedRuntimeException

예외를 Wrapping 할 때 편리하게 사용할 수 있는 예외로 NestedRuntimeException과 NestedCheckedException이 있다. 두 개의 클래스는 스프링에서 제공하고 있는 클래스이다. 주로 Checked Exception을 런타임 예외로 래핑할 때 많이 사용한다.

NestedCheckedException은 Exception 클래스를 상속하고 있기 때문에 반드시 try-catch 문으로 감싸주어야한다. 그리고 두 개의 클래스는 모두 추상 클래스이기 때문에 직접 사용하진 못하고, 개발자가 이 클래스를 상속해서 사용해야 한다. 

이 클래스들은 getMessage() 메소드를 오버라이드하고 있다. getMessage() 메소드의 리턴값은 중첩된 예외의 정보를 포함하고 있다. 만약에 Checked Exception을 런타임 예외로 던지고 싶을땐 NestedRuntimeException을 사용하긴 추천한다. 실제로 스프링에서는 NestedRuntimeException을 많이 사용하고 있다.

아래는 간단한 학습 테스트이다. 

``` java
public class NestedRuntimeExceptionLearningTest {

    @Test
    public void test1() {
        IOException ioException = new SocketTimeoutException("Connect Timeout Exception is occur");
        ClientException nestedRuntimeException = new ClientException("IOException is occur", ioException);

        System.out.println(nestedRuntimeException.getMessage());
        assertThat(nestedRuntimeException.getCause()).isEqualTo(ioException);
        assertThat(nestedRuntimeException.getRootCause()).isEqualTo(ioException);
        assertThat(nestedRuntimeException.getMostSpecificCause()).isEqualTo(ioException);
    }

    @Test
    public void test2() {
        ClientException nestedRuntimeException = new ClientException("IOException is occur");

        System.out.println(nestedRuntimeException.getMessage());
        assertThat(nestedRuntimeException.getCause()).isNull();
        assertThat(nestedRuntimeException.getRootCause()).isNull();
        assertThat(nestedRuntimeException.getMostSpecificCause()).isEqualTo(nestedRuntimeException);
    }

    private static class ClientException extends NestedRuntimeException {
        ClientException(String msg) {
            super(msg);
        }

        ClientException(String msg, Throwable cause) {
            super(msg, cause);
        }
    }
}
```

- test1 : 생성자를 통해 에러 메시지와 예외를 전달했다. 테스트 메소드 중간에 NestedRuntimeException의 getMessage() 메소드의 리턴값을 콘솔을 통해 출력했다. 테스트를 돌려보면 "IOException is occur; nested exception is java.net.SocketTimeoutException: Connect Timeout Exception is occur" 메시지가 나온다. NestedRuntimeException의 getMessage() 메소드의 리턴값은 중첩된 예외 클래스가 무엇인지, 그리고 중첩된 예외 클래스의 에러 메시지가 무엇인지를 포함한다.
- test2 : 생성자를 통해 에러 메시지만 전달했다. 이 경우엔 getMessage() 리턴값은 예외 메시지 정보만 포함하고 있다. 테스트를 돌려보면 콘솔에 "IOException is occur"이란 메시지가 나타난다.
- getRootCause() : 이 메소드는 루트 예외를 리턴한다. 만약 존재하지 않으면 null을 리턴한다. test2의 경우 이 메소드의 리턴값은 null이다.
- getMostSpecificCause() : 만약에 루트 예외가 존재하면 루트 예외를 리턴하고, 존재하지 않으면 자기 자신을 리턴한다.
