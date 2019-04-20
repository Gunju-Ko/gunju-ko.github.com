---
layout: post
title: "Deep dive into Hystrix" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring-cloud, netflixOSS]
---

# Deep dive into Hystrix

이 글은 Netflix에서 만든 Hystrix라는 오픈 소스에 대한 소개 글입니다. 출처는 아래와 같습니다.

- [11번가 Spring Cloud 기반 MSA로의 전환](https://www.youtube.com/watch?v=J-VP0WFEQsY&t=1456s)
- [Hystrix - 디지털 세상을 만드는 아날로거](https://medium.com/@goinhacker/hystrix-500452f4fae2)
- [Hystrix! API Gateway를 도와줘](http://woowabros.github.io/experience/2017/08/21/hystrix-tunning.html)
- [Hystrix 공식문서](https://github.com/Netflix/Hystrix/wiki)

## What Is Hystrix?

Netflix가 만든 Fault Tolerance Library이다.

Hystrix를 Circuit Breaker로 보는 경우가 있는데, 사실 Circuit Breaker는 Hystrix가 제공하는 기능 중 일부일 뿐이다. 기능적 관점에서 본다면 Hystrix의 주요 4가지 기능은 아래와 같다.

- Circuit Breaker : 분산환경에서는 한 개의 서비스에서 실패가 발생하면 해당 서비스의 실패가 의존성있는 다른 서비스에 전파될 수 있다. 이렇게 외부 서비스를 호출하는 부분을 Hystrix로 Wrapping하면 실패가 전파되는 것을 막을수 있다.
- Fallback
- Isolation : 각각의 HystrixCommand 실행은 별로의 스레드로 분리되어 실행될 수 있다. 스레드풀이 가득차는 경우 추가적인 요청은 Reject된다. (요청을 큐에 넣지 않는다)
- Timeout

> 그 외에도 Hystrix는 실시간 모니터링과 설정 변경을 지원한다. 또한 내부적으로 중복된 Request 처리를 줄이기 위한 Request Caching을 지원하고, Request를 일괄 처리하기 위한 Request Collapsing을 제공한다.

Hystrix는 서비스간의 의존성이 발생하는 접근 포인트를 분리시켜 장애의 전파를 막고, Fallback을 제공하여 시스템 장애로부터의 복구를 유연하게 한다. (Resiliency)

Hystrix의 특징은 아래와 같다.

- Third-party 클라이언트 라이브러리 의존성으로 인한 지연 및 장애로부터 보호하고 제어한다.
- 복잡한 분산 시스템에서 장애가 전파되는 것을 막는다.
- 빨리 실패하고, 빠르게 회복한다. (Fast fail, Rapidly recover)
- 준 실시간 모니터링이 가능하며, 알람 및 작동 제어가 가능하다.

## What Problem Does Hystrix Solve?

모든 서버들이 정상적인 경우 요청의 흐름은 아래와 같다.

![그림](https://github.com/Netflix/Hystrix/wiki/images/soa-1-640.png)

여러 서버 중 하나에 지연이 발생하면, 모든 요청에 지연이 발생할 수 있다. 아래 그림을 처럼 Dependecy I에 지연이 발생하면 모든 요청에 지연이 발생한다.

![그림](https://github.com/Netflix/Hystrix/wiki/images/soa-2-640.png)

트래픽이 많은 경우 하나의 지연이 모든 서버의 리소스를 포화 상태로 만들 수 있다. 애플리케이션에서 네트워크 요청을 보내는 구간은 잠재적으로 예외가 발생할 수 있다. 예외보다 더 심각한 것은 네트워크 오류로 인해서 시스템 전체의 리소스가 고갈될 수도 있다는 것이다. 결과적으로 네트워크 오류로 인해 대부분의 사용자 요청을 처리하지 못하게 되고 심각한 경우 시스템이 다운될 수도 있다.

![그림](https://github.com/Netflix/Hystrix/wiki/images/soa-3-640.png)

이러한 이슈는 써드파티 클라이언트를 통해 네트워크 액세스가 수행될 때 더 악화된다. 써드파티 라이브러리의 경우 구현 세부 사항이 감쳐진 경우도 있으며, 개발자가 구현 세부사항에 대해 잘 알지 못하는 경우도 있다. 또한 클라이언트마다 모니터링 및 변경이 어려운 경우가 있다. 

장애와 지연은 고립되어야하며 관리되어야한다. 그래서 하나의 장애가 전체 애플리케이션 퍼지지 않도록 해야한다.

Hystrix는 아래와 같은 기능을 제공하여 외부 의존성으로부터 받는 영향을 최소화한다.

- 하나의 의존성이 컨테이너의 모드 유저 스레드를 사용하는 것을 방지한다.
- 부하로 인해 더이상 실행할 수 없는 경우 빠르게 실패한다.
- 예외가 발생하면 Fallback을 실행시켜준다.
- 의존성에 대한 영향을 최소화하기 위해 Isolation 기술을 사용한다.
- 준 실시간에 가까운 모니터링 및 알람 기능 제공한다.

## How it works

![그림](https://raw.githubusercontent.com/wiki/Netflix/Hystrix/images/hystrix-command-flow-chart.png)

#### 1. Construct HystrixCommand or HystrixObservableCommand

HystricCommand 혹은 HystrixObservableCommand 객체를 생성한다.

- HystrixCommand : 단일값 형태로 리턴받고 싶은 경우에 사용한다.
- HystrixObservableCommand : Observable 형태로 응답을 받고 싶은 경우에 사용한다.

#### 2. execute the command 

1. execute() : command를 실행시키고 결과를 리턴받는다. (혹은 예외가 발생한다) 실행이 완료될 때까지 block된다.
2. queue() : 실행 결과를 받을수 있는 Future 객체를 리턴한다.
3. observe() : 실행 결과 Observable을 subscribe하고, 실행 결과 Observable의 복제본을 리턴한다. 
4. toObservable() : Observable을 리턴한다. 리턴된 Observable을 subscribe하는 순간 command가 실행되고 결과를 내보낼 것이다. 실제 실행을 Lazy하게 하고싶을때 사용할 수 있다.

여기까지가 HystrixCommand를 사용하는 클라이언트가 해줘야하는 부분이다. 3번부터 9번까지는 HystrixCommand 내부의 동작 순서를 나타낸다.

#### 3. available in cache?

결과가 캐시에 저장되어 있으면, 캐시된 결과를 리턴한다.

#### 4. circuit-breaker open?

circuit이 오픈된 경우 8번으로 간다.

#### 5. Semaphore/Thread pool rejected?

세마포어/스레드풀이 가득찬 경우 8번으로 간다.

#### 6. construct() or run()

HystrixCommand.run() 메소드 혹은 HystrixObservableCommand.construct() 메소드를 실행한다. 만약에 Isolation전략으로 스레드를 사용하고 있다면 HystrixCommand.run() 메소드는 별도의 스레드풀에서 동작한다.

1. 실행도중 예외가 발생하면 8번으로 간다.
2. 실행도중 타임아웃이 발생하면 8번으로 간다. 타임아웃이 발생하면 TimeoutException을 발생시킨다. 
3. 정상적으로 실행된 경우 9번으로 간다.

> 별도의 스레드풀에서 동작 중인 작업은 타임아웃이 발생했더라도 계속해서 실행될 수 있다. 별도의 스레드 작업을 중단시킬수 있는 방법은 기술적으로 없다. (최선의 방법이 Thread.interrupt()를 호출하는것 뿐이다) 결과적으로 클라이언트가 TimeoutException을 받았더라도 스레드는 작업을 계속할 것이다.
>
> 이러한 기술적 한계 때문에 스레드풀이 쉽게 포화상태가 될 수 있다. 따라서 만약에 HystrixCommand.run() 메소드 안에서 HttpClient를 사용하고 있다면, connectTimeout, readTimeout을 적절히 세팅해주어야한다.

#### 7. calculate circuit health

Circuit Breaker의 상태를 계산한다. Hystrix는 Circuit Breaker에게 성공, 실패, rejection, timeout에 대한 정보를 전달한다. Circuit Breaker는 각각에 대한 횟수를 기록하고 통계를 낸다. Fallback 메소드가 실행되는 경우도 Circuit Breaker에 실패로 기록된다.

#### 8. getFallback

다음과 같은 경우에 Fallback을 실행한다.

1. circuit이 오픈된 경우
2. 세마포어/스레드풀이 가득찬 경우
3. HystrixCommand.run() 메소드 혹은 HystrixObservableCommand.construct() 메소드 실행도중 예외가 발생한 경우
4. HystrixCommand.run() 메소드 혹은 HystrixObservableCommand.construct() 메소드를 실행도중 타임아웃이 발생한 경우

Fallback 메소드에서는 보통 일반적인 응답을 리턴한다. 예를 들어 추천 상품을 리턴해야 하는 경우 Fallback 메소드에서는 default 상품을 리턴하도록 작성한다. Fallback 메소드에서 다른 API를 호출하는 등 네트워크 작업을 하는건 좋지 않다. 또한 Fallback 메소드에서는 가능하면 예외를 발생시키지 않는것이 좋다. 

Fallback 메소드는 다음과 같은 방법으로 제공할 수 있다.

- HystrixCommand.getFallback() 메소드를 구현한다.
- HystrixObservableCommand.resumeWithFallback() 메소드를 구현한다. 

Fallback 메소드가 존재하지 않거나, Fallback 메소드 실행 도중에 예외가 발생하는 경우 아래와 같이 동작한다. 

- execute() : 예외가 발생한다. 
- queue() : Future의 get() 메소드를 호출하면 예외가 발생한다. 
- observe() : subscribe하면 subscriber의 onError을 호출하고 즉시 종료되는 Observable을 리턴한다.
- toObservable() : subscribe하면 subscriber의 onError을 호출하고 즉시 종료되는 Observable을 리턴한다.

#### 9. return resulting Observable

Hystrix command가 성공하면 Observable 형태로 결과가 리턴된다. 그리고 Hystrix command의 실행 방법(2번)에 따라 Observable은 아래와 같은 방식으로 변형되어 클라이언트에게 최종적으로 리턴된다.

![그림](https://raw.githubusercontent.com/wiki/Netflix/Hystrix/images/hystrix-return-flow.png)

- execute() : queue() 메소드를 호출해서 Future() 객체를 얻은다음 get() 메소드의 리턴값을 리턴한다.
- queue() : Observable 객체를 Future 객체로 변환해서 리턴한다.
- observe() : Observable를 subscribe해서 command를 즉시 실행한다. 그리고 해당 Observable의 복제본을 리턴한다.
- toObservable() : Observable를 그래도 리턴한다. 실제로 command가 실행되기 위해서는 리턴된 Observable 객체를 subscribe 해야한다.

## Hystrix 적용하기

Hystrix는 Spring Cloud 없이 그냥 JVM 환경에서 사용할 수 있다. 사용하는 방법은 크게 2가지이다. 하나는 애노테이션을 사용하는 방법이고, 다른 하나는 특정 인터페이스를 상속받아 구현하는 방법이다. 이 글에서는 특정 인터페이스를 상속해서 Hystrix를 적용하는 방법에 대해서만 설명한다.

#### HystrixCommand 상속 (혹은 HystrixObservableCommand)

이 글에서는 HystrixCommand를 상속하는 방법에 대해서만 다룬다. 따라서 HystrixObservableCommand를 사용하는 방법은 [공식문서](https://github.com/Netflix/Hystrix/wiki/How-To-Use)를 참고하길 바란다.

``` java
public static class CommandHelloWorld extends HystrixCommand<String> {

    public CommandHelloWorld(String group) {
        super(HystrixCommandGroupKey.Factory.asKey(group));
    }

    @Override
    protected String run() {
        return "Hello World";
    }
}
```

HystrixCommand의 execute() 메소드를 호출해서 동기식으로 실행시킬 수 있다. 

``` java
@Test
public void testSynchronous() throws Exception {
    String name = "gunju";
    CommandHelloWorld hello = new CommandHelloWorld(name);
    assertThat(hello.execute()).isEqualTo("Hello " + name);
}
```

그리고 HystrixCommand의 queue() 메소드를 호출해서 비동기로 실행시킬 수도 있다.

``` java
@Test
public void testAsynchronous() throws Exception {
    String name = "gunju";
    CommandHelloWorld hello = new CommandHelloWorld(name);

    Future<String> result = hello.queue();
    assertThat(result.get()).isEqualTo("Hello " + name);
}
```

이 외에도 호출 결과를 Observable로 받는 방법이 있다. 이 방법에 대해선 설명하지 않으니 공식문서를 참고하길 바란다.

#### Hystrix Command를 호출할 때 벌어지는 일

- 이 메소드를 Intercept하여 대신 실행한다. (Thread Isolation)
  - 더이상 요청을 처리하지 못하는 경우는 바로 Reject한다.
- 메소드의 실행 결과 성공 혹은 실패(Exception) 발생 여부를 기록하고 통계를 낸다. 통계에 따라 Circuit Open 여부를 결정한다. (Circuit Breaker)
- 실패한 경우(Exception) 사용자가 제공한 메소드를 대신 실행한다. (Fallback)
- 특정시간 동안 메소드가 종료되지 않은 경우 Exception을 발생시킨다. (Timeout)
- 준실시간으로 메트릭 및 설정 변화를 모니터링한다.

Hystrix를 사용하는 경우, 각각의 의존성은 고립된다. 따라서 특정 의존성이 지연으로 인해 리소스가 고갈되는 것을 막을 수 있다. 뿐만 아니라 예외가 발생한 경우 Fallback 메소드를 실행시켜준다.

## Circuit Breaker

- Circuit Open : 메소드를 호출하더라도 실제 메소드는 실행되지 않으며 호출이 차단됌. Circuit Open의 경우 아래와 같은 조건에서 발생함
  - 일정시간 동안 + 일정 개수 이상의 호출이 발생한 경우 + 일정 비율 이상의 에러가 발생한 경우
- Circuit Close : 호출 허용
  - 일정 시간 경과 후에 단 한개의 요청에 대해서 호출을 허용하며 (Half Open)
  - 이 호출이 성공하면 Circuit이 Close됌
- 특정 서버가 장애가 났음에도 해당 서버을 계속해서 호출하는 것은 장애 상황을 악화시킬뿐만 아니라 장애가 다른 서버로 전파될 가능성이 있다. Hystrix의 경우 일정 비율 이상의 에러가 발생한 경우 메소드의 실행을 차단시켜주기 때문에, 다른 서버의 장애가 해당 서버를 호출하는 서버에게까지 전파되는 것을 막을 수 있다.

아래 그림은 HystrixCircuitBreaker의 플로우 차트와 이에 대한 설명이다.

![그림](https://raw.githubusercontent.com/wiki/Netflix/Hystrix/images/circuit-breaker-1280.png)

- 일정 개수 이상의 호출이 발생한 경우 (HystrixCommandProperties.circuitBreakerRequestVolumeThreshold())
- 일정 비율 이상의 에러가 발생한 경우 (HystrixCommandProperties.circuitBreakerErrorThresholdPercentage())
- circuit이 오픈된다. circuit이 오픈되면 모든 호출이 차단된다.
- 특정 시간 이후에 (HystrixCommandProperties.circuitBreakerSleepWindowInMilliseconds()) 하나의 요청을 허용한다 (half-open) 해당 요청이 실패하면 계속해서 open 상태가 된다. 만약에 해당 요청이 성공하면 circuit이 close 된다.

#### CommandKey - Circuit Breaker의 단위

- Hystrix Circuit Breaker는 한 프로세스 내에서 주어진 CommandKey 단위로 통계를 내고 동작한다. 즉 Circuit Breaker는 CommandKey 단위로 생성이 된다. 
- 장애의 포인트가 같은 경우는 같은 CommandKey를 같게 해주는게 좋다. 예를 들어 a라는 메소드와 b라는 메소드가 같은 서버를 호출하고 있다면 두 개의 메소드의 CommandKey를 같게 해준다면, 호출하는 서버가 장애가 발생한 경우 a, b 메소드가 모두 차단이 되기 때문에(Circuit Open) 장애 전파를 효과적으로 막을 수 있다.

- Circuit Breaker의 단위가 너무 크다면? 하나의 메소드가 오동작해서 여러개의 메소드의 호출이 차단될 가능성도 있다.
- Circuit Breaker의 단위가 너무 작다면? 메소드가 20번 이상 호출이 안될수도 있다. 이러면 예외가 50% 이상 발생하더라도 Circuit이 Open되지 않기 때문에 주의가 필요하다. 혹은 "circuitBreaker.requestVolumeThreshold" 설정값을 조정하는게 좋다.
- 동일한 의존성을 갖는 HystrixCommand 들을 동일한 commandKey로 묶는 것을 고려해야한다.

HystrixCommand 인터페이스 구현체의 CommandKey 기본값은 클래스 이름이다. 아래와 같이 명시적으로 CommandKey을 세팅할 수도 있다.

``` java
public CommandHelloWorld(String name) {
    super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"))
            .andCommandKey(HystrixCommandKey.Factory.asKey("HelloWorld")));
    this.name = name;
}
```

HystrixCommandKey는 인터페이스이므로 이를 implements해서 구현해줘도 된다. 그리고 Factory 클래스를 이용해서 아래와 같이 간단하게 해결할 수도 있다.

``` java
HystrixCommandKey.Factory.asKey("HelloWorld")
```

#### CommandGroupKey

HystrixCommandKey와 마찬가지로 HystrixCommandGroupKey는 Factory 클래스를 통해 생성할 수 있다.

``` java
HystrixCommandGroupKey.Factory.asKey("ExampleGroup")
```

#### Command Thread-Pool

Thread isolation을 사용하는 경우 threadPoolKey를 통해 Hystrix가 사용할 스레드풀(HystrixThreadPool)을 지정할 수 있다. HystrixThreadPoolKey를 따로 지정하지 않는 경우엔 HystrixCommandGroupKey가 사용된다.

마찬가지로 생성자를 통해 값을 주입해줄 수 있다.

``` java
public CommandHelloWorld(String name) {
    super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"))
            .andCommandKey(HystrixCommandKey.Factory.asKey("HelloWorld"))
            .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("HelloWorldPool")));
    this.name = name;
}
```

위의 코드를 보면 알겠지만, Factory 클래스를 통해 HystrixThreadPoolKey 객체를 생성했다. 

HystrixThreadPoolKey와 HystrixCommandGroupKey가 분리되어 있는 이유는, 같은 command group이라도 다른 스레드풀에서 동작해야하는 경우가 있기 때문이다.

## Fallback

- Fallback으로 지정된 메소드는 다음의 경우에 원본 메소드 대신 실행된다.
  - Circuit Open
  - Any Exception : 단 HystrixBadRequestException은 제외
  - Semaphore / ThreadPool Rejection
  - Timeout

- Fallback으로 인해서 예외가 많이 감쳐지게 될 수 있다.  또한 Fallback 메소드가 실행되더라도 circuit breaker에는 에러로 기록된다.

HystrixCommand를 상속하는 경우 아래와 같이 fallback 메소드를 제공해주면 된다.

``` java
@Test
public void testFallback1() throws Exception {
    String name = "gunju";
    CommandHelloFailure command = new CommandHelloFailure(name);
    assertThat(command.execute()).isEqualTo("Hello Failure " + name);
}

public static class CommandHelloFailure extends HystrixCommand<String> {

   private final String name;

   public CommandHelloFailure(String name) {
       super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
       this.name = name;
   }

   @Override
   protected String run() throws Exception {
       throw new IllegalStateException();
   }

   @Override
   protected String getFallback() {
       return "Hello Failure " + name;
   }
}
```

만약에 run() 메소드를 실행하는 도중에 예외가 발생했지만 fallback 메소드를 제공하지 않은 경우엔 예외가 발생한다. 반면에 fallback 메소드를 제공하면 getFallback() 메소드의 리턴값을 리턴한다.

#### HystrixBadRequestException

- 사용자의 코드에서 HystrixBadRequestException을 발생시키면, 이 오류는 Fallback을 실행하지 않으며, Circuit Open을 위한 통계에도 집계되지 않는다. 즉 run() 메소드에서 HystrixBadRequestException이 발생하면 Fallback 메소드의 리턴값을 리턴하지 않고, HystrixBadRequestException을 던진다.
- 비즈니스 로직이나 외부 호출로 인한 예외가 아니라, 사용자가 잘못된 값을 입력한 경우엔 HystrixBadRequestException를 던짐으로서 Circuit Open이 되는것을 방지할 수 있다.
- 만약 사용자의 실수를 다른 예외로 던진다면, Circuit Breaker의 통계에 집계되어 사용자의 잘못으로 Circuit이 Open되는 경우가 발생할 수 있다. 또한 Fallback이 실행되어, 개발 과정에서 오류를 인지하지 못할 수 있다.

#### Error Propagation

run() 메소드에서 발생한 예외는 HystrixRuntimeException으로 감싸져서 던져진다. run() 메소드에서 발생한 예외를 알고 싶은 경우에는 HystrixRuntimeException 객체의 getCause() 메소드를 호출하면 된다. 아래 코드를 보면 알수있듯이 HystrixBadRequestException의 경우엔 HystrixRuntimeException으로 감싸져서 던져지지 않는다.

``` java
@Test
public void testException1() throws Exception {
    CommandHelloException command = new CommandHelloException(new IllegalStateException());
    Throwable e = catchThrowable(command::execute);
    assertThat(e).isInstanceOf(HystrixRuntimeException.class);
    assertThat(e.getCause()).isInstanceOf(IllegalStateException.class);
}

@Test
public void testHystrixBadRequestException() {
    CommandHelloException command = new CommandHelloException(new HystrixBadRequestException("bad request"));
    Throwable e = catchThrowable(command::execute);
    assertThat(e).isInstanceOf(HystrixBadRequestException.class);
}

public static class CommandHelloException extends HystrixCommand<String> {

    private final Exception exception;

    public CommandHelloException(Exception exception) {
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.exception = exception;
    }

    @Override
    protected String run() throws Exception {
        throw exception;
    }
}
```

아래는 실행도중 발생하는 에러의 타입에 따라 던져지는 예외를 정리한 것이다.

- run() 메소드 실행 도중 예외 발생 : HystrixRuntimeException
- 타임아웃 : HystrixRuntimeException
- circuit open : HystrixRuntimeException
- Thread pool, Semaphore rejection : HystrixRuntimeException
- run() 메소드 실행 도중 HystrixBadRequestException 발생 : HystrixBadRequestException

타임아웃으로 인해 HystrixRuntimeException이 발생한 경우, getCause() 메소드는 TimeoutException을 리턴한다. Thread pool의 rejection으로 인해 HystrixRuntimeException이 발생한 경우는 getCause() 메소드는 RejectedExecutionException을 리턴한다.

## Timeout

- Hystrix에서는 Circuit Breaker 단위로 (CommandKey 단위로) Timeout을 설정할 수 있다.

``` yaml
hystrix.command.<commandKey>.execution.isolation.thread.timeoutInMilliseconds
```

- 기본값은 1초로 짧기 때문에 주의가 필요하다.

``` java
@Test
public void testTimeout1() {
    long executionTime = 1500;
    int timeout = 2000;

    CallableHystrixCommand<String> command = new CallableHystrixCommand<>(() -> {
        Thread.sleep(executionTime);
        return "Hello";
    }, timeout);

    assertThat(command.execute()).isEqualTo("Hello");
}

@Test
public void testDefaultTimeout() {
    long executionTime = 1500;

    CallableHystrixCommand<String> command = new CallableHystrixCommand<>(() -> {
        Thread.sleep(executionTime);
        return "Hello";
    });
    
    Throwable t = catchThrowable(command::execute);
    assertThat(t).isInstanceOf(HystrixRuntimeException.class);
    assertThat(t.getCause()).isInstanceOf(TimeoutException.class);
}

@Test
public void testTimeout2() {
    long executionTime = 2500;
    int timeout = 2000;

    CallableHystrixCommand<String> command = new CallableHystrixCommand<>(() -> {
        Thread.sleep(executionTime);
        return "Hello";
    }, timeout);

    Throwable t = catchThrowable(command::execute);
    assertThat(t).isInstanceOf(HystrixRuntimeException.class);
    assertThat(t.getCause()).isInstanceOf(TimeoutException.class);
}

public static class CallableHystrixCommand<V> extends HystrixCommand<V> {

    private final Callable<V> callable;

    public CallableHystrixCommand(Callable<V> callable) {
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.callable = callable;
    }

    public CallableHystrixCommand(Callable<V> callable, int timeout) {
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"), timeout);
        this.callable = callable;
    }

    @Override
    protected V run() throws Exception {
        return callable.call();
    }
}
```

좀 길긴 하지만 위에 학습 테스트를 차례로 살펴보자

- testTimeout1() : 실행시간은 1500ms이지만 timeout이 2000ms 이므로 타임아웃이 발생하지 않는다.
- testDefaultTimeout() : 실행시간은 1500ms이지만 timeout이 1000ms 이므로 타임아웃이 발생한다. TimeoutException은 HystrixRuntimeException으로 감싸져서 던져진다.
- testTimeout2() : 실행시간은 2500ms이지만 timeout이 2000ms 이므로 타임아웃이 발생한다

## Isolation

두가지 Isolation 방식을 Circuit Breaker 별로 지정 가능하다. Isolation 덕분에 하나의 지연 때문에 전체 시스템이 지연되는 것을 막을 수 있으며, 특정 지연이 시스템의 너무 많은 리소스를 사용하지 못하도록 막을 수 있다. 공식문서에 따르면 두가지 방식중 Thread 방식을 권장하고 있다. 

- Semaphore
- Thread (default)

![그림](https://raw.githubusercontent.com/wiki/Netflix/Hystrix/images/request-example-with-latency-1280.png)

위의 그림을 보면 알 수 있듯이, Dependency I에 지연이 생기더라도, 시스템 전체에는 영향을 주진 않는다. 외부 시스템에 많이 의존하고 있는 경우 이러한 격리작업은 매우 중요하다. 물론 read/connect Timeout 설정을 통해서도 네트워크 장애에 대한 대비를 할 수 있다. 하지만 Hystrix는 보다 빠르게 실패할 수 있다는 장점이 있다. 또한 이러한 격리를 통해 특정 종속성으로 인해 시스템 전체가 마비되지 않도록 할 수 있다. 따라서 외부 서버의 내부 구현을 잘모르거나 믿지 못하는 경우 반드시 Hystrix를 통해 외부 서버 호출 구간을 격리시켜야 한다.

#### Semaphore

- Circuit Breaker마다 세마포어가 붙어 있게 된다.
- 세마포어을 이용해서 Circuit Breaker의 최대 동시 호출을 제한할 수 있다. 
- 세마포어 이상의 동시 호출을 하려는 경우는 Rejection이 된다. 그리고 Fallback 메소드가 실행된다.
- Command를 호출한 Caller의 스레드에서 메소드가 실행된다. 
- Timeout이 동작하지 않는다.
  - 자바에서는 다른 스레드를 멈출 수 있는 방법이 딱히 없기 때문.
- Semaphore Isolation는 외부 서버나 종속성에 대한 신뢰가 있을 경우에만 사용하는게 좋다. 별도의 스레드에서 실행하는 것이 아니라 Caller 스레드에서 실행되는 것이기 때문에 외부 서버나 종속성에서 오랜 시간동안 block된다면, Caller 스레드도 오랜 시간동안 block 상태가 될 수밖에 없다.

#### Thread Isolation

- Circuit Breaker 별로 사용할 Thread Pool을 지정할 수 있다. (ThreadPoolKey)
  - Caller는 메소드 호출 후 특정 시간이 지난다면, 바로 리턴된다. 하지만 Thread Pool에서 실행중인 태스크는 계속해서 실행된다. 따라서 Thread Isolation을 사용하더라도 태스크가 무한정 block되도록 코드를 작성하면 안된다.
  - Timeout이 제시간에 발생한다.
- 서로 다른 Circuit Breaker가 같은 Thread Pool를 사용할 수도 있다.
- 최대 개수 초과시엔 Thread Pool Rejection이 발생하고, Fallback이 실행된다.
- Command를 호출한 Thread가 아닌 Thread Pool에서 메소드를 실행한다.
- 실제 메소드의 실행은 다른 Thread에서 실행되므로 Thread Local 사용시 주의가 필요하다.
- 별도의 스레드에서 실행하는것이기 때문에 추가적인 오버헤드가 발생할 수 있다. (context swithing, scheduling)

``` java
public static ThreadLocal<String> sync = new ThreadLocal<>();

@Test
public void testThreadPoolIsolation() {
    String commandKey = "testThreadPoolIsolation";
    String value = "value";

    RunnableHystrixCommand command = new RunnableHystrixCommand(commandKey, () -> {
        // run() 메소드 부분은 별도의 스레드에서 실행된다.
        sync.set(value);
    });

    assertThat(command.execute()).isEqualTo("success");
    // ThreadLocal의 값이 변경되어있지 않음
    assertThat(sync.get()).isEqualTo(null);
}

public static class RunnableHystrixCommand extends HystrixCommand {

    private final Runnable runnable;

    public RunnableHystrixCommand(String commandKey, Runnable callable) {
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("TestGroup"))
                    .andCommandKey(HystrixCommandKey.Factory.asKey(commandKey)));
        this.runnable = callable;
    }

    @Override
    protected String run() throws Exception {
        runnable.run();
        return "success";
    }
}
```

위와 같은 테스트 코드를 작성해봤다. HystrixCommand의 run() 메소드 부분은 별도의 스레드풀에서 실행되기 때문에, 테스트를 실행하는 스레드에서 ThreadLocal의 값이 null이다. HystrixCommand에서 ThreadLocal을 사용하는 경우 이 점을 유의하길 바란다.

``` java
@Test
public void testRejectExecutionException() {
    String commandKey = "commandKey1";
    String commandGroupKey = "group1";

    int poolSize = 10;
    IntStream.range(0, poolSize)
             .forEach(i -> executeNotFinishJob(commandKey, commandGroupKey));

    // <1>
    shouldThrowHystrixRuntimeExceptionIfUseSameGroupKey(commandKey);
    // <2>
    shouldSuccessIfUseDifferentGroupKey("group2");
    // <3>
    shouldThrowHystrixRuntimeExceptionIfUseSameGroupKey("commandKey2");
}

private void shouldThrowHystrixRuntimeExceptionIfUseSameGroupKey(String commandKey) {
    RunnableHystrixCommand command = new RunnableHystrixCommand(commandKey, "group1", () -> {});
    Throwable t = catchThrowable(command::execute);
    assertThat(t).isInstanceOf(HystrixRuntimeException.class);
    assertThat(t.getCause()).isInstanceOf(RejectedExecutionException.class);
}

private void shouldSuccessIfUseDifferentGroupKey(String groupKey) {
    RunnableHystrixCommand command = new RunnableHystrixCommand("commandKey1", groupKey, () -> {});
    assertThat(command.execute()).isEqualTo("success");
}

private void executeNotFinishJob(String commandKey, String commandGroupKey) {
    new RunnableHystrixCommand(commandKey, commandGroupKey, () -> {
        while (true) {
        }
    }).queue();
}

public static class RunnableHystrixCommand extends HystrixCommand<String> {

    private final Runnable runnable;

    public RunnableHystrixCommand(String commandKey, Runnable runnable) {
        this(commandKey, "TestGroup", runnable);
    }

    public RunnableHystrixCommand(String commandKey, String commandGroupKey, Runnable runnable) {
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey(commandGroupKey))
                    .andCommandKey(HystrixCommandKey.Factory.asKey(commandKey)));
        this.runnable = runnable;
    }

    @Override
    protected String run() throws Exception {
        runnable.run();
        return "success";
    }
}
```

위의 코드는 스레드풀 관련 학습 테스트이다. 

- testRejectExecutionException : 종료되는 않는 작업을 HystrixCommand로 감싸서 10번 호출한다. 이때 commandGroupKey는 "group1"이다
  - <1> : commandGroupKey가 "group1"인 HystrixCommand 객체를 생성한 다음에 실행시키면, HystrixRuntimeException이 발생한다. 그 이유는 현재 스레드풀이 가득찼기 때문이다. (스레드풀 사이즈의 기본값은 10이다) 
  - <2> :  같은 commandKey를 가지더라도 threadPoolKey가 다른 경우 (위의 경우 threadPoolKey를 따로 지정하지 않았기 때문에 commandGroupKey가 사용된다) 별도의 스레드풀에서 실행되기 된다. 따라서 commandGroupKey가 "group2"인 HystrixCommand는 정상적으로 실행된다.
  - <3> : 다른 commandKey를 가지더라도 같은 threadPoolKey를 사용할 수도 있다. 테스트 코드의 마지막 부분을 보면 commandKey는 다르지만 commandGroupKey가 같기 때문에 HystrixRuntimeException이 발생한다.
- 스레드풀이 가득차서 예외가 발생하는 경우 getCause() 메소드로 리턴되는 객체의 타입은 RejectedExecutionException 이다.
