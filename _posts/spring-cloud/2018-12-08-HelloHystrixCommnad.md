---
layout: post
title: "Hello @HystrixCommand" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring-cloud, netflixOSS]
---

# Hello @HystrixCommand

이 전글에서는 Hystrix에 관해서 설명을 했다. 이 글은 spring-cloud을 사용해서 HystrixCommand를 쉽게 사용하는 방법에 대해 설명한다. 아래에 글들을 참고해서 작성했다.

- [Spring Cloud Netflix](https://cloud.spring.io/spring-cloud-netflix/single/spring-cloud-netflix.html#_circuit_breaker_hystrix_clients)
- [Hystrix - javanica](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica)

특정 실행을 Hystrix로 감싸기 위해서는 HystrixCommand 인터페이스를 구현하고, Hystrix로 감싸서 실행되어야 하는 부분의 코드를 run() 메소드안에 작성하면 된다. 이는 꽤나 번거로운 작업이 될 수 있는다.

 javanica라는 라이브러리는 애노테이션을 사용해서 이러한 번거라운 작업을 해결했다.

## Getting Started

이 글의 실습코드는 spring boot와 spring cloud-starter 프로젝트를 사용한다. 만약에 spring을 사용하지 않는 프로젝트에서 히스트릭스 애노테이션을 사용하고자 한다면 [Hystrix - javanica](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica)를 참고하길 바란다.

우선 아래와 같이 gradle에 dependency를 추가한다.

```
dependencies {
    compile('org.springframework.cloud:spring-cloud-starter-netflix-hystrix')
    ...
 }
```

그리고 아래와 같이 @EnableCircuitBreaker 애노테이션을 추가해준다. 그럼 이제 @HystrixCommand를 사용할 준비는 모두 끝난다.

``` java
@EnableCircuitBreaker
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Spring Cloud는 @HystrixCommand 애노테이션이 붙은 빈들에 대한 Proxy를 자동으로 생성하고, Proxy들을 빈으로 등록한다. Proxy들은 실제 실행되는 메소드부분을 Hystrix로 감싸서 호출한다. Proxy의 Pointcut과 Advice 부분이 궁금하다면 HystrixCommandAspect 클래스 코드를 보면 된다.

##  How to use

특정 메소드를 Hystrix로 감싸서 호출하고 싶다면 아래와 같이 메소드에 @HystrixCommand 애노테이션을 붙이면 된다.

``` java
@Component
public class RunnableHystrixCommand {
    @HystrixCommand
    public void run(Runnable runnable) {
        runnable.run();
    }
}
```

위와 같이 메소드 위에 @HystrixCommand를 붙이면, 해당 메소드를 동기식으로 실행하며 Hystrix로 감싸져서 실행된다. 또한 commandKey을 값은 메소드의 이름이 된다.

만약 commandKey를 변경하고자 하는 경우 아래와 같이 @HystrixCommand 애노테이션의 commandKey 속성을 사용하면 된다. groupKey의 기본값은 클래스의 이름이다. 만약 groupKey를 변경하고자 한다면, 아래와 같이 애노테이션의 groupKey 속성을 사용하면 된다. 그리고 threadPoolKey를 변경하고 싶다면 threadPoolKey 속성을 사용하면 된다.

``` java
@Component
public class RunnableHystrixCommand {
    @HystrixCommand(commandKey = "Exp1", groupKey = "group1")
    public void run(Runnable runnable) {
        runnable.run();
    }
}
```

위와 같이 동기식으로 호출하는 방법 외에도 @HystrixCommand가 붙은 메소드를 비동기식으로 호출하는것도 가능하다. 비동기로 실행시키려면, 메소드에서 AsyncResult 타입을 리턴해줘야한다. 아래 예제의 경우 메소드가 비동기로 실행된다. 메소드의 리턴 타입을 Future로해서 해당 메소드가 비동기로 실행되어야함을 알려줘야한다.

``` java
@HystrixCommand
public Future<Void> runAsync(Runnable runnable) {
    return new AsyncResult<Void>() {
        @Override
        public Void invoke() {
            runnable.run();
            return null;
        }
    };
}
```

그 외에도 메소드의 리턴값을 Observable로 하는 방법도 있지만 이 글에선 다루지 않겠다. 궁금하다면 javanica 공식문서를 보기 바란다.

#### Fallback

Fallback 메소드를 설정해주기 싶으면 아래와 같이 하면 된다.

``` java
@HystrixCommand(fallbackMethod = "fallback")
public void runWithFallback(Runnable runnable) {
    runnable.run();
}

public void fallback(Runnable runnable, Throwable t) {
    log.error("Exception is occur while run hystrix command", t);
}
```

> 주의할 점은 fallback 메소드는 @HystrixCommand가 붙은 메소드와 같은 클래스에 위치해야한다. 또한 메소드 시그니쳐가 같아야한다. 단 위의 예제와 같이 실행 도중 발생한 예외를 받을 수 있는 파라미터는 추가할 수 있다. (Fallback 메소드의 접근 제어자는 어떤것이든 상관없다.)

또한 아래와 같이 fallback 메소드도 별도의 @HystrixCommand 안에서 실행되도록 할 수 있다.

``` java
@HystrixCommand(fallbackMethod = "fallback")
public void runWithFallback(Runnable runnable) {
    runnable.run();
}

@HystixCommand
public void fallback(Runnable runnable, Throwable t) {
    log.error("Exception is occur while run hystrix command", t);
}
```

Fallback은 동기 혹은 비동기로 실행될 수 있다. 이는 HystrixCommand가 어떤 방식으로 동작하느냐에 달렸다.

- HystrixCommand가 동기로 실행되는 경우 : Fallback은 동기로만 실행된다. 
- HystrixCommand가 비동기로 실행되는 경우 :  Fallback을 동기 혹은 비동기로 실행할 수 있다.

하나의 클래스안에 여러개의 HystrixCommand가 있는 경우, 클래스에 공통 Fallback 메소드를 지정할 수 있다. 단 클래스 공통 Fallback 메소드의 파라미터는 발생한 예외를 전달받기 위한 Throwable 타입만 있어야한다. (혹은 파라미터가 없어도 된다) 아래는 간단한 예제이다.

``` java
@DefaultProperties(defaultFallback = "fallback")
public class Service {
    @RequestMapping(value = "/test1")
    @HystrixCommand
    public APIResponse test1(String param1) {
        // some codes here
        return APIResponse.success("success");
    }

    @RequestMapping(value = "/test2")
    @HystrixCommand
    public APIResponse test2() {
        // some codes here
        return APIResponse.success("success");
    }

    @RequestMapping(value = "/test3")
    @HystrixCommand
    public APIResponse test3(ObjectRequest obj) {
        // some codes here
        return APIResponse.success("success");
    }

    private APIResponse fallback() {
        return APIResponse.failed("Server is busy");
    }
}
```

Fallback 메소드는 아래와 같은 순서로 찾는다. (1번이 가장 높은 우선순위이다)

1. @HystrixCommand의 fallbackMethod 속성
2. 1번이 설정되지 않은 경우, @HystrixCommand의 defaultFallback 속성
3. 2번이 설정되지 않은 경우, @DefaultProperties의 defaultFallback 속성

#### Error Propagation

@HystrixCommand 애노테이션의 ignoreExceptions 속성을 사용해서 특정 예외를 HystrixBadRequestException로 감싸져서 던져지게 할 수 있다. 이 전글에서 살펴봤듯이 HystrixBadRequestException은 실패로 통계되지 않으며, fallback 메소드가 호출되지 않는다.

> HystrixCommand 인터페이스를 상속해서 hystrix를 사용하는 경우와는 달리, @HystrixCommand의 경우 콜러에게 항상 실행도중 발생한 예외를 그대로 던진다. 예외를 HystrixBadRequestException 혹은 HystrixRuntimeException를 감싸서 콜러에게 던지지 않는다.

``` java
@HystrixCommand(ignoreExceptions = {IllegalArgumentException.class},
                fallbackMethod = "fallback")
public void run(Runnable runnable) {
    runnable.run();
}

public void fallback(Runnable runnable, Throwable t) {
    log.error("Exception is occur while run hystrix command", t);
}
```

위의 예제의 경우 run() 메소드 실행 도중에 IllegalArgumentException가 발생하더라도, fallback 메소드가 실행되지 않는다. 또한 Circuit Breaker에 실패로 기록되지 않는다.

@HystrixCommand의 raiseHystrixExceptions 속성을 사용하면, 실행 도중에 발생한 예외를 HystrixRuntimeException 클래스로 감싸서 던지게할 수 있다. 

``` java
@HystrixCommand(raiseHystrixExceptions = HystrixException.RUNTIME_EXCEPTION)
public void execute(Runnable runnable) {
    runnable.run();
}
```

위와 같이 설정하면, 실행 도중 발생한 예외가 ignoreExceptions 속성에 포함되지 않는 경우 HystrixRuntimeException으로 감싸져서 던져진다.

> Fallback을 실행하는 도중에 예외가 발생하면, 실제 실행 시점에 발생한 예외가 콜러에게 던져진다.

#### Configuration

@HystrixCommand는 아래와 같은 방식으로 Hystrix 관련 속성을 설정할 수 있다.

``` java
@HystrixCommand(commandProperties = {
        @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "500")
    })
public User getUserById(String id) {
    return userResource.getUserById(id);
}
```

Javanica는 Hystrix의 ConfigurationManager을 사용해서 동적으로 속성을 변경한다. 위의 예제의 경우 Javanica는 내부적으로 아래 코드를 실행한다.

``` java
ConfigurationManager.getConfigInstance().setProperty("hystrix.command.getUserById.execution.isolation.thread.timeoutInMilliseconds", "500");
```

threadPool 관련 속성은 @HystrixCommand의 threadPoolProperties 속성을 통해 설정할 수 있다. 아래는 간단한 예제이다.

``` java
@HystrixCommand(commandProperties = {
        @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "500")
    },
            threadPoolProperties = {
                    @HystrixProperty(name = "coreSize", value = "30"),
                    @HystrixProperty(name = "maxQueueSize", value = "101"),
                    @HystrixProperty(name = "keepAliveTimeMinutes", value = "2"),
                    @HystrixProperty(name = "queueSizeRejectionThreshold", value = "15"),
                    @HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "12"),
                    @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "1440")
    })
public User getUserById(String id) {
    return userResource.getUserById(id);
}
```

#### 주요 설정

다음은 Hystrix의 주요 설정이다. commandKey를 사용해서 Hysyrix별로 설정을 다르게 줄 수 있다.

```yaml
hystrix.command.{commandKey}.{property}
```

개인적으로 생각하는 중요한 설정은 아래와 같다. 만약에 Hystrix 기본설정을 변경하고 싶은 경우엔 아래와 같이 하면 된다.

```yaml
hystrix.command.default.{property}
```

#### Execution

- hystrix.command.{commandKey}.execution.isolation.strategy : ISOLATION 전략
  - THREAD (default)
  - SEMAPHORE

- hystrix.command.{commandKey}.execution.isolation.thread.timeoutInMilliseconds : Timeout 값
  - default : 1000
- hystrix.command.{commandKey}.execution.timeout.enabled : Timeout 사용여부
  - default : true
- hystrix.command.{commandKey}.execution.isolation.semaphore.maxConcurrentRequests : 세마포어 사용시 동시 호출량 제한
  - default : 10

#### Fallback

아래 설정들은 Fallback 메소드 실행과 관련된 설정이다. 이 속성들은 ISOLATION 전략과 상관없이 모두 적용된다.

- hystrix.command.{commandKey}.fallback.isolation.semaphore.maxConcurrentRequests : Fallback 메소드 실행의 동시 호출량을 제한한다. 동시 호출량을 넘어선 경우 Fallback 메소드는 실행되지 않고 예외가 발생한다.
  - default : 10
- hystrix.command.{commandKey}.fallback.enabled : Fallback 사용여부
  - default : true

#### Circuit Breaker

- hystrix.command.{commandKey}.circuitBreaker.enabled : circuit breaker 사용여부
  - default : true

- hystrix.command.{commandKey}.circuitBreaker.requestVolumeThreshold : 특정 시간동안 일정 개수 이상의 호출이 발생한 경우에만 circuit이 open 된다. 호출 개수는 이 설정을 통해 정할 수 있다.
  - default : 20

- hystrix.command.{commandKey}.circuitBreaker.sleepWindowInMilliseconds : circuit open시 open이 지속되는 시간
  - default : 5000

- hystrix.command.{commandKey}.circuitBreaker.errorThresholdPercentage : circuit이 open되기 위한 최소 에러 비율
  - default : 50

- hystrix.command.{commandKey}.circuitBreaker.forceOpen : circuit을 강제로 open시켜 모든 요청을 차단한다. 이 속성은 circuitBreaker.forceClosed 보다 우선순위가 높다.
  - default : false

- hystrix.command.{commandKey}.circuitBreaker.forceClosed : circuit을 강제로 close시켜 모든 요청을 처리한다.
  - default : false

#### Metrics

- hystrix.command.{commandKey}.metrics.rollingStats.timeInMilliseconds : Circuit Breaker가 이 속성을 time window로 사용해서 Circuit의 상태를 계산한다.
  - default : 10,000

> circuit breaker 설정과 metrics 설정을 default로 사용하는 경우 "10초간 20개 이상의 호출이 발생한 경우, 50% 이상의 에러가 발생하면 5초간 Circuit이 Open된다."

####  Thread Pool Properties

ThreadPool은 ThreadPoolKey 별로 다르게 설정할 수 있다. 위의 설정과는 다르게 "hystrix.threadpool.{threadPoolKey}.{property}" 이런 형식으로 설정한다.

- hystrix.threadpool.{threadPoolKey}.coreSize : 스레드풀 core size
  - default : 10
- hystrix.threadpool.{threadPoolKey}.maximumSize : 스레드풀 max size로 allowMaximumSizeToDivergeFromCoreSize를 true로 설정한 경우에만 의미가 있다.
  - default : 10
- hystrix.threadpool.{threadPoolKey}.maxQueueSize : BlockingQueue의 크기이다. -1로 설정하는 경우 BlockingQueue의 구현체로 SynchronousQueue가 사용된다. 0보다 큰 값으로 설정하는 경우 LinkedBlockingQueue를 사용한다. queue의 크기를 런타임 시점에 바꿀 수 없다. 이런 한계를 해결하고 싶다면 queueSizeRejectionThreshold 설정을 사용하길 바란다.
  - default : -1
- hystrix.threadpool.{threadPoolKey}.queueSizeRejectionThreshold : 해당 값을 설정하면 maxQueueSize에 도달하지 않더라도 rejection이 일어나도록 해준다. 이 설정이 존재하는 경우는 BlockingQueue의 maxQueueSize는 런타임시에 동적으로 변경할 수 없기 때문이다. 이 설정은 maxQueueSize가 -1인 경우엔 의미가 없습니다.
  - default : 5
- hystrix.threadpool.{threadPoolKey}.keepAliveTimeMinutes : 스레드풀의 스레드 개수가 coreSize보다 많고, 특정 스레드가 keepAliveTimeMinutes보다 긴 시간동안 사용되지 않았다면, 해당 스레드를 제거한다.
  - default : 1
- hystrix.threadpool.{threadPoolKey}.allowMaximumSizeToDivergeFromCoreSize : 이 값을 true로 설정해야만 maximumSize 속성이 의미가 있게 된다.
  - default : false

>  Hystrix 관련 모든 설정들이 궁금하다면 [Hystrix-Configuration](https://github.com/Netflix/Hystrix/wiki/Configuration#CommandExecution)를 참고하길 바란다.

#### More

이 글에서는 Request Cache, Hystrix collapser에 대해선 다루지 않는다. 해당 내용이 궁금하다면 공식문서를 참고하길 바란다.

## Spring Cloud Netflix Hystrix

Spring Cloud는 application.yml에서 Hystrix 관련 설정을 읽어서 Hystrix에 적용시켜준다. 따라서 아래와 같이 Hystrix 설정을 yml에 추가해주면 된다.

``` yaml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000
```

위와 같이 설정하는 경우 Hystrix의 기본 timeout은 3초가 된다. 만약 특정 threadPoolKey 설정을 변경하고 싶다면 아래와 같이 하면 된다.

``` yaml
hystrix:
  threadpool:
    group1:
      coreSize: 10
      maximumSize: 20
      allowMaximumSizeToDivergeFromCoreSize: true
```

