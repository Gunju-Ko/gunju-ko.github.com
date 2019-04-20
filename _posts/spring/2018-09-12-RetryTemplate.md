---
layout: post
title: "Spring Retry" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring]
---

# RetryTemplate

Spring Retry에서 제공하는 RetryTemplate을 사용하면, 재시도 구현을 아주 쉽게 할 수 있다. 본 글에서는 Spring Retry에서 제공하고 있는 RetryTemplate의 특징과 사용법에 대해서 소개한다.


## RetryTemplate

실패한 작업을 자동으로 재시도하는 것은 작업에 대한 실패 가능성을 줄이는데 도움을 준다. 보통 일시적인 오류인 경우에 이러한 재시도는 큰 도움을 준다. 예를 들어 다른 웹 서버로 요청을 보내는데 네트워크로 인해 오류가 발생하거나, 데이터베이스에 업데이트하는데 오류가 발생하는 경우 잠시 후에 해결되는 경우가 많다. 

Spring Retry에는 작업에 대한 재시도를 자동화하기 위한 전략 인터페이스(RetryOperations)가 있다. 아래는 RetryOperations 인터페이스 코드이다.

``` java
public interface RetryOperations {

    <T> T execute(RetryCallback<T> retryCallback) throws Exception;

    <T> T execute(RetryCallback<T> retryCallback, RecoveryCallback<T> recoveryCallback)
        throws Exception;

    <T> T execute(RetryCallback<T> retryCallback, RetryState retryState)
        throws Exception, ExhaustedRetryException;

    <T> T execute(RetryCallback<T> retryCallback, RecoveryCallback<T> recoveryCallback,
        RetryState retryState) throws Exception;

}
```

아래는 RetryCallback 인터페이스이다. RetryCallback는 doWithRetry라는 메소드를 하나 가지고 있는 간단한 인터페이스이다. doWithRetry 메소드에는 재시도를 할 비즈니스 로직이 들어간다.

``` java
public interface RetryCallback<T> {

    T doWithRetry(RetryContext context) throws Throwable;

}
```

콜백이 실패하면 (Exception이 발생하면)  재시도 정책에 따라서 특정 횟수 혹은 특정 시간동안 재시도를 할 것이다. RetryOperations에는 재시도가 모두 실패했을때, 실패에 대한 처리하는 콜백 메소드를 인자로 받는 execute 메소드를 제공한다. 

아래는 RetryOperations의 구현체인 RetryTemplate을 이용해서 재시도를 구현한 간단한 예제이다. 

``` java
RetryTemplate template = new RetryTemplate();

TimeoutRetryPolicy policy = new TimeoutRetryPolicy();
policy.setTimeout(30000L);

template.setRetryPolicy(policy);

Foo result = template.execute(new RetryCallback<Foo>() {

    public Foo doWithRetry(RetryContext context) {
        // Do stuff that might fail, e.g. webservice operation
        return result;
    }

});
```

위의 예제는 작업이 실패하는 경우 timeout이 발생할 때까지 계속해서 재시도한다. 

## RetryContext

RetryCallback 메소드의 파라미터는 RetryContext이다. 대부분의 콜백은 RetryContext를 무시할 것이다. RetryContext는 데이터를 저장하기 위한 용도로 사용할 수 있다. 만약 동일한 스레드에서 재시도가 중첩해서 일어나는 경우 RetryContext 상위 컨텍스트를 가진다. 상위 컨텍스트는 실행간에 공유해야할 데이터를 저장하는데 유용하다.

> 상위 컨텍스트는 RetryCallback안에서 또 다시 RetryTemplate 등을 사용해서 중첩된 재시도를 할 경우에 생긴다. 

## Recovery Callback

재시도가 전부 실패하면, RetryOperations는 RecoveryCallback을 호출한다. 이 기능을 사용하려면 클라이언트는 execute 메소드를 호출할 때 RecoveryCallback 객체를 전달해주어야한다. 

``` java
Foo foo = template.execute(new RetryCallback<Foo>() {
    public Foo doWithRetry(RetryContext context) {
        // business logic here
    },
  new RecoveryCallback<Foo>() {
    Foo recover(RetryContext context) throws Exception {
          // recover logic here
    }
});
```

모든 재시도가 실패하고 더이상 재시도할 수 없는 경우, RecoveryCallback 메소드를 호출한다. RecoveryCallback의 recover 메소드에서는 재시도가 전부 실패한 경우에 대한 대체 로직을 수행한다. 

## Stateless Retry

가장 간단한 재시도는 단순한 while 루프이다. RetryTemplate는 성공하거나 혹은 실패할 때까지 계속해서 재시도할 수 있다. RetryContext에서는 상태를 포함하고 있다. 그리고 그 상태를 가지고 재시도를 할지 혹은 중단할지 여부를 결정한다. 이 상태는 보통의 경우 전역으로 저장할 필요가 없다. 이러한 재시도를 stateless retry라 한다. stateless 재시도에서는 콜백이 항상 동일한 스레드에서 실행된다. 

> 이 글에서는 Stateful Retry에 대해서는 설명하지 않는다. 해당 내용은 공식문서를 참고하길 바란다.
 
## Retry Policies

RetryTemplate에서 재시도 할지 여부는 RetryPolicy에 의해 결정된다. RetryTemplate은 RetryPolicy의 open 메소드를 통해서 RetryContext 객체를 생성한다. 그리고 RetryCallback의 doWithRetry 메소드 인자로 생성된 RetryContext 객체를 전달한다. 
RetryTemplate은 콜백이 실패한 경우 RetryPolicy에게 상태를 업데이트 하도록 요청한다. 그리고 RetryPolicy의 canRetry 메소드를 호출하여, 재시도가 가능한지 여부를 묻는다. 만약 재시도가 불가능한 경우, RetryTemplate는 마지막 콜백 실행시 발생한 예외를 던진다. 단 RecoveryCallback이 있는 경우 RecoveryCallback 메소드를 호출한다. 

Spring Retry는 SimpleRetryPolicy 및 TimeoutRetryPolicy와 같은 간단한 stateless RetryPolicy 구현체를 제공한다. SimpleRetryPolicy는 설정된 예외가 발생한 경우에만 재시도를 하며, 재시도 최대 횟수를 제한한다. 

``` java
// Set the max attempts including the initial attempt before retrying
// and retry on all exceptions (this is the default):
SimpleRetryPolicy policy = new SimpleRetryPolicy(5, Collections.singletonMap(Exception.class, true));

// Use the policy...
RetryTemplate template = new RetryTemplate();
template.setRetryPolicy(policy);
template.execute(new RetryCallback<Foo>() {
    public Foo doWithRetry(RetryContext context) {
        // business logic here
    }
});
```

ExceptionClassifierRetryPolicy라는 보다 유연한 RetryPolicy도 있다. 이는 예외 유형에 따라 다르게 재시도할 수 있도록 해준다. ExceptionClassifierRetryPolicy는 예외 유형에 따라 RetryPolicy를 결정한다. ExceptionClassifierRetryPolicy는 콜백 메소드에서 발생하는 예외 유형에 따라 RetryPolicy를 다르게 하고 싶을때 유용하다. 

> 모든 예외를 재시도하는 것은 비효율적이다. 따라서 모든 예외에 대해 재시도 하지말고, 재시도 가능할 것 같은 예외에 대해서만 재시도 해라. 재시도해도 또 다시 예외가 발생할 것이 확실한 경우에 재시도를 하는것은 비효율적이다.

## Backoff Policies

오류가 발생하여 재시도를 할 때 재시도를 하기전에 잠깐 기다리는 것이 유용할때가 많다. 일반적으로 오류는 잠깐 동안 기다리기만 해도 해결되는 경우가 많다. RetryCallback이 실패하면 RetryTemplate은 BackoffPolicy에 따라 실행을 일시적으로 중지할 수 있다. 

``` java
public interface BackoffPolicy {

    BackOffContext start(RetryContext context);

    void backOff(BackOffContext backOffContext)
        throws BackOffInterruptedException;

}
```

BackoffPolicy 인터페이스의 backOff 메소드는 원하는 방식으로 구현하면 된다. 일반적으로는 backoff 시간을 기하급수적으로 증가시킨다. backoff 시간을 기하급수적으로 증가시키고 싶으면 ExponentialBackoffPolicy를 사용하면 된다. 고정된 시간으로 backoff 시키고자 한다면 FixedBackOffPolicy를 사용하면 된다.

## Listeners

재시도를 할 때마다 호출되는 콜백 메소드를 추가할 수 있다. Spring Retry는 RetryListener 인터페이스를 제공하고 있는데, 이 인터페이스의 구현체를 RestTemplate에 등록할 수 있다.

인터페이스는 아래와 같다.

``` java
public interface RetryListener {

    void open(RetryContext context, RetryCallback<T> callback);

    void onError(RetryContext context, RetryCallback<T> callback, Throwable e);

    void close(RetryContext context, RetryCallback<T> callback, Throwable e);
}
```

* open(RetryContext context, RetryCallback<\T\> callback) : 첫번째 시도 전에 호출이된다. 메소드의 인자로 RetryContext 객체와 실행될 콜백 메소드가 전달된다. 만약에 등록된 RetryListener 중에서 open 메소드의 결과로 false을 리턴한 리스너가 하나라도 있다면, TerminatedRetryException이 발생한다. 그리고 이 경우에는 한번도 시도 하지 않는다.
* void close(RetryContext context, RetryCallback<\T\> callback, Throwable e) : 성공 실패 여부와 상관없이 재시도가 모두 끝나면 close 메소드가 호출된다. 만약에 모든 재시도가 실패한 경우, 마지막에 발생한 예외가 두번째 인자로 전달된다.
* void onError(RetryContext context, RetryCallback<\T\> callback, Throwable e) : 재시도 중 에러가 발생할 때마다 onError가 호출된다. 

> 참고로 RetryListener는 하나 이상 등록될 수 있으며, RetryListner는 리스트에 저장된다. 그리고 등록된 RetryListener의 open 메소드가 호출되는 순서와 onError, close 메소드가 호출되는 순서는 반대이다.

## Declarative Retry

Spring Retry는 어노테이션을 통해서 재시도를 할 수 있도록 해준다. 자세한 내용은 출처에서 확인하길 바란다.


## 출처

* [Spring Retry github](https://github.com/spring-projects/spring-retry) 
