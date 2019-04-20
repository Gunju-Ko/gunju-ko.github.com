---
layout: post
title: "Reactive Streams" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [reactive]
---

# Reactive Streams

### Reactive
 
Reactive는 아래 4가지 특징을 가지고 있어야 한다. 

### Reactive manifesto

![그림1](https://cdn-images-1.medium.com/max/1600/1*kblZxudSz1qo6O16MLh6ZQ.png)

* Responsive (응답성) : 리액티브 시스템은 가능한 모든 요청에 대해 적시에 응답할 수 있어야한다. 또한 발생한 문제에 대해 빠르고 효율적으로 대응할 수 있어야한다. 시스템은 언제나 응답 가능한 상태로 존재해야한다는 개념이다.
* Elastic (탄력셩) : 시스템은 변화하는 부하에 의해 자원의 변경이 빈번하게 발생하는 상황에서도 응답성을 유지해야 한다. 서버를 동적으로 늘리거나 요청이 줄어들면 서버를 동적으로 줄이는 상황에서도 계속 응답성을 유지해야 한다.
* Resillient (회복성) : 시스템은 장애에 직면한 상황에서도 응답성을 유지할 수 있어야한다. 미션 크리티컬한 시스템 또는 고가용성을 유지해야 하는 시스템에만 국환되지 않는다. 특정 컴포넌트의 장애가 다른 시스템에 번져서는 안된다.
* Message Driven : 리액티브 시스템은 느슨한 결합(loosely coupled)의 방법을 사용하는 컴포넌트 사이에서 비동기의 방법으로 전달되는 메시지를 기반으로 동작한다. 데이터베이스에 데이터를 추가하는 작업을 한다고 가정해보자. 그러면 보통의 경우 서비스 layer에서 레포지토리 layer에게 데이터베이스 작업을 위임하고, 그 작업이 끝날때까지 기다린다. 반면에 메시지 기반 동작은 서비스 layer와 레포지토리 layer 중간에 새로운 layer를 두고, 서비스 layer는 데이터를 추가하라는 메시지를 보내고 바로 리턴된다. 그리고 레포지토리 layer에서는 메시지를 받으면 데이터베이스에 데이터를 추가하는 방식으로 동작한다. 메시지가 처리될 때까지 기다리는 방식이 아니라, 메시지를 던져놓고 다른일에 집중하는 방식이다.

"외부에 이벤트가 발생하면 이에 대응하여 동작하는 프로그램을 작성하는 것"

## Reactive Streams

### Background

* 넷플릭스, 피봇탈 등의 회사들이 모여서 만든 표준이다.
* JVM 상에서 동작하는 언어들의 Reactive 관련된 기술을 개발할 때 중구난방으로 만드는 것을 방지하기 위해서 Reactive 관련된 표준을 만들었다.
* JVM 상에서 동작하는 언어들은 Reactive 관련된 기술을 만들때 이 표준을 지켜서 만들자
* 핵심 인터페이스와 구현의 규칙을 정함으로서 스펙을 만듬

### Goals, Design and Scope

Reactive Streams의 목적은 non-blocking, backpressure로 비동기 스트림 처리의 표준을 제공하는 것이다. 

데이터 스트림(특히 데이터의 크기가 미리 정해지지 않는 실시간 데이터)을 처리하는 것은 비동기 시스템에서 특별한 주의가 필요하다. 가장 중요한 이슈는 리소스 소비가 컨트롤 되어야한다는 것이다. 예를 들어 빠른 데이터소스로 인해 리소스 소비가 심해지면 안된다. 또한 컴퓨팅 리소스를 병렬로 사용할 수 있도록 비동기가 필요하다.

Reactive Streams의 주요 목표는 비동기식 경계에서 스트림 데이터 교환을 관리하는 것이다. 다른 스레드 혹은 스레드 풀로 데이터를 전달한다고 가정해보자. 단 이 때, 데이터를 받는 쪽에서는 데이터를 받기 위한 버퍼가 필요없거나 혹은 버퍼 사이즈가 제한되어야 한다. (즉 버퍼가 무한정 커지거나 하면 안된다) 따라서 back pressure는 이 모델의 필수적인 부분이다. back pressure를 통해서 스레드간 중개를 위한 버퍼의 크기를 제한할 수 있다.

Back pressure를 위한 통신이 동기식이면 비동기 처리의 이점은 무효화된다. ([Reactive Manifesto](https://www.reactivemanifesto.org/))  따라서 Reactive Streams 구현의 모든 측면에서 non-blocking, asynchronous 방식으로 동작하도록 주의를 기울려야 한다. 

규칙을 준수함으로써 스트림 어플리케이션의 전체 프로세싱 그래프에서 앞서 언급한 이점과 특성을 유지하면서, 스펙을 준수함으로써 원활하게 상호 운용 할 수 있는 많은 구현체를 만들 수 있도록 하는것이 스펙의 의도이다.

스트림 조작(manipulations)의 특성은 스펙에 포함되지 않는다. Reactive Streams는 데이터 스트림을 중재하는데에만 관심이 있다. 

요약하면, Reactive Streams는 JVM용 스트림 지향 라이브러리의 표준 및 스펙이다.

* 무한한 데이터를 처리한다.
* 순서가 있다.
* 구성 요소간 비동기적으로 데이터를 전달한다.
* non-blocking backpressure는 필수적이다.

Reactive Streams의 스펙은 다음과 같은 부분으로 구성된다. 

* The API : 인터페이스이다. 인터페이스를 올바르게 구현한 구현체라면, 다른 구현체와도 상호 운용이 가능하다.
* The Technology Compatibility Kit (TCK) : 구현의 적합성 테스트를 위한 표준 테스트 suite

구현시 API 요구 사항을 준수하고 TCK에서 테스트를 통과하는 한 스펙에 포함되지 않은 추가 기능을 구현할 수 있다.

### Reactive Streams란?

논-블록킹 백프레셔를 제공하는 비동기 스트림 프로세싱의 표준이다.

* 논-블록킹 : 어떤 요청에 대해서 응답을 줄 때까지 기다리는게 아니라, 요청을 보내고 바로 리턴되어 다른일을 하는것. 그리고 요청은 별도의 스레드에서 처리되는 것
* backpressue : Publisher가 Subscriber의 처리속도와 관계없이 계속해서 데이터를 전달하게 되면 문제가 발생할 수 있다. 

즉 Reactive Stream은 스트림의 안정적인 처리와 비동기 처리을 제공하기 위함

### Reactive Streams 기본 구조

![그림](http://cfile22.uf.tistory.com/image/998D64465A75C7860E2E25)

API는 다음과 같은 구성요소로 이루어져있으며, Reactive Streams 구현체들은 아래의 구성요소들을 올바르게 구현해야 한다.

* Publisher : 잠재적으로 무한한 수의 연속된 이벤트를 제공한다.
* Subscriber : 이벤트를 받아 이벤트를 처리한다.
* Subscription : Subscription은 Subscriber와 Publisher 사이의 데이터 교환을 중재하는 것이다.
  * 전통적인 옵저버 패턴에서는 Observable이 Observer에게 데이터를 Push하는 방식이다. 따라서 옵저버 패턴에서는 backpressure를 구현하기가 어렵다. 하지만 Reactive Streams은 Pull 방식이다. Subscriber가 다음 이벤트를 처리할 수 있을때 새로운 이벤트를 Subscription을 통해 Publisher에게 요청한다. 
* Processor

Reactive Streams의 중요한 특징중 하나는 PUSH 방식이 아닌 PULL 방식이라는 것이다. Subscriber가 Subscription을 통해서 이벤트를 Publisher에게 요청하면, Publisher는 이벤트를 Subscriber에게 전달한다.

### API Component

자세한 내용은 [Reactive Streams](https://github.com/reactive-streams/reactive-streams-jvm)를 참고하길 바란다.

API는 다음과 같은 구성요소로 이루어져있으며, Reactive Streams 구현체들은 아래의 구성요소들을 올바르게 구현해야 한다.

* Publisher
* Subscriber
* Subscription
* Processor

Publisher는 잠재적으로 무한한 수의 연속된 엘리멘트를 제공한다. Publisher는 Subscriber의 요구에 따라 엘리멘트를 제공한다. Publisher.subscribe(Subscriber) 호출시 다음과 같은 프로토콜에 따라 Subscriber의 메소드가 호출된다. 

```
onSubscribe onNext* (onError | onComplete)?
```

위의 프로토콜을 해석하면 아래와 같다.

* onSubscribe : Subscriber#onSubscribe 메소드는 반드시 호출되어야 한다.
* onNext* : Subscriber#onNext는 0번 또는 한 번 이상 호출될 수 있다. (Subscriber에 의해 요청된다)
* (onError or onComplete)? :  onError, onComplete는 호출해도 되며 호출하지 않아도 된다. 단 호출시에는 반드시 둘 중 하나만 호출해야 한다. 

### Publisher

``` java
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}
```

Publisher는 잠재적으로 무한한 수의 연속된 이벤트를 제공한다. Publisher는 Subscriber의 요구에 따라 이벤트를 제공한다. Publisher.subscribe(Subscriber) 호출 후에는 다음과 같은 프로토콜에 따라 Subscriber의 메소드가 호출된다. 

```
onSubscribe onNext* (onError | onComplete)?
```

위의 프로토콜을 해석하면 아래와 같다.

* onSubscribe : Subscriber#onSubscribe 메소드는 반드시 호출되어야 한다.
* onNext* : Subscriber#onNext는 0번 또는 한 번 이상 호출될 수 있다.
* (onError or onComplete)? :  onError, onComplete는 호출해도 되며 호출하지 않아도 된다. 단 호출시에는 반드시 둘 중 하나만 호출해야 한다. 

#### Rule

* Publisher가 Subscriber의 onNext 메소드를 호출한 총 횟수는 Subscriber가 Subscription을 통해 요청한 총 수보다 작거나 같아야 한다. 이 규칙의 목적은 Publisher가 Subscriber가 요청한 수보다 더 많은 데이터를 전달하지 못하도록 하기 위함이다. 이 규칙은 매우 중요하다. 이 규칙으로 인해 데이터의 요청이 데이터의 수신보다 먼저 일어난다.
* Publisher는 요청받은 것보다 더 적게 onNext를 호출할 수 있다. 그리고 onComplete 또는 onError를 호출해서 Subscription를 종료해야한다. 
  * 이 규칙은 Publisher가 요청받은 수만큼 데이터를 생성하지 않을수도 있다는 것을 의미한다.  예를 들어 Subscription.request(10)을 통해 10개의 데이터를 요청했다고 가정해보자. 하지만 Publisher는 오직 3개의 데이터만 가지고 있다면, Publisher는 onNext를 3번만 호출할 것이다.
*  onSubscribe, onNext, onError, onComplet는 반드시 thread-safe하게 호출해야한다. 만약에 여러개의 스레드에서 호출하는 경우에는 lock 등을 사용해서 thread-safe함을 보장해야한다.
*  Publisher는 실패시에 반드시 onError를 호출해야한다. 이 규칙의 의도는 Publisher가 더이상 진행하지 못하는 경우 이를 Subscriber에게 알릴 책임이 있음을 분명히하기 위함이다. Subscriber는 onError가 호출되면 리소스를 반환하거나 혹은 실패를 처리할 수 있다.
*  Publisher가 성공적으로 완료된 경우에는 onComplete를 호출해야 한다. 이 규칙의 의도는 Publisher가 완료된 경우 이를 Subscriber에게 알릴 책임이 있음을 분명히하기 위함이다.
* Publisher가 onComplete 혹은 onError를 호출한 경우, Subscriber의 Subcription은 취소된 것으로 간주되어야 한다.
* onError, onComplete 메소드 호출 이후에는 Subscriber의 어떠한 메소드도 호출하면 안된다. 
* Subscription이 취소된 경우, Subscriber에 추가적인 호출을 하면 안된다. Publisher는 Subscription.cancel()이 호출되면 Subscriber의 취소 요청을 받아드려야한다. 
  * Subscription.cancel()을 호출하고, Publisher가 취소요청을 전달받는데까지 지연이 있을수 있다. 
* Publisher.subscribe 메소드는 파라미터로 전달받은 Subscriber의 onSubscribe를 반드시 호출해야 하며, Subscriber의 메소드 중에서도 onSubscribe를 가장 먼저 호출해야 한다. 그리고 Publisher.subscribe 메소드는 전달받은 Subscriber가 널이 아닌 이상 정상적으로 종료되어야 한다. 전달받은 Subscripter가 널인 경우에는 NullPointException을 발생시켜야 한다. 실패를 알려야하거나 Subscriber를 거부하고 싶은 경우에는, Subscriber의 onSubscribe를 호출한 이후에 바로 onError을 호출하면 된다. 
  * 이 규칙은 Subscriber의 onSubscribe는 반드시 호출되며, 가장 먼저 호출됨을 보장하기 위함이다. 따라서 onSubscriber메소드에서는 리소스를 초기화하는등의 작업을 할 수 있다. 또한 onSubscribe는 반드시 한번만 호출되어야 한다. 
* Publisher.subscribe는 여러번 호출될 수 있다. 대신 매번 다른 Subscriber를 파라미터로 전달해야한다.
* Publisher는 여러 Subscriber를 지원할 수 있다. 또한 Publisher는 각 Subscription이 유니캐스트인지 멀티캐스트인지 결정한다. 따라서 Publisher 구현에 따라 지원되는 Subscriber가 결정되며, 데이터가 어떻게 분배될지 결정된다. 

### Subscriber

``` java
public interface Subscriber<T> {
    public void onSubscribe(Subscription s);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();
}
```

#### Rule

* Subscription.request(long n) 메소드를 통해 데이터를 요청해야 한다. 그리고 데이터는 onNext 메소드를 통해 받는다. Subscriber는 언제, 얼마나 많은 데이터를 받을지를 알려줘야할 책임이 있다. 
* 만약에 Subscriber가 데이터를 처리하는데 많은 시간이 걸리면(onNext 메소드에서 데이터를 처리하는데 시간이 오래걸린다면), Publisher의 반응성에 부정적인 영향을 미친다. 따라서 데이터 처리하는데 시간이 오래걸리면, 비동기적으로 데이터를 처리하는게 좋다. 이 규칙의 의도는 Subscriber가 Publisher의 진행을 방해하지 않도록 하기 위함이다. 
* Subscriber.onComplete() 메소드나 onError() 메소드에서는 Subscription 또는 Publisher의 어떠한 메소드도 호출해선 안된다. 
* Subscriber.onComplete(), onError() 메소드가 호출되면 Subscription이 취소된 것으로 간주해야 한다. onComplete 또는 onError 메소드가 호출 된 이후에는 Subscription이 더 이상 유효하지 않다. 이 규칙의 의도는 둘 이상의 Publisher가 동일한 Subscriber와 상호 작용할 수 없도록 하기 위함이다. 
* Subscription이 더이상 필요하기 않다면 반드시 Subscription.cancel()를 호출해야 한다. Subscription.cancel()이 호출되면 Subscription은 리소스를 적절하게 반환해야 한다. Subscriber는 Subscription.cancel() 메소드 호출을 통해 Publisher에게 완료되었음을 알린다.
* Subscription 메소드 호출은 동일한 스레드에서 일어나야 한다. Subscriber가 여러 스레드에서 Subscription 메소드를 호출하고 있다면, 락 등을 사용해야 한다.
* Subscriber는 Subscription.cancel()을 호출 한 이후에도 onNext 메소드 호출을 수신 할 수 있도록 준비해야 한다. (아직 요청된 데이터가 보류중인 경우) Subscription.cancel()은 클리닝 작업을 즉시 수행 할 것을 보장하지 않는다. 이 규칙은 cancel()의 호출과 Publisher가 취소를 확인하는 사이에 지연이 있을 수 있음을 강조한다. 
* Subscriber는 앞선 Subscription.request (long n) 호출 유무와 관계없이 onComplete 메소드 호출을 수신 할 수 있어야한다.
* Subscriber는 앞선 Subscription.request (long n) 호출 유무와 관계없이 onError 메소드 호출을 수신 할 수 있어야한다. 
  * onComplete, onError 메소드를 통해서 Publisher의 완료나 에러 발생을 Subscriber가 알게된다. 따라서 Subscriber는 Publisher의 완료여부나 실패여부를 폴링할 필요가 없다.
* Subscriber는 signal 메소드(request(n), cancel)가 먼저 호출되고 난 후에 각 signal를 처리해야 한다. 
* Subscriber.onSubscribe는 한번만 호출되어야 한다. 즉 Subscriber는 최대 한번만 구독할 수 있다.
*  onSubscribe, onNext, onError 또는 onComplete 메소드는 정상적으로 리턴 되어야한다. 단 파라미터로 널을 전달받은 경우에는 NullPointerException를 던져야한다. 그 외에 Subscriber가 실패를 알리는 유일한 방법은 Subscription을 취소하는 것이다. 이 규칙을 위반 한 경우 Subscriber와 관련된 모든 Subscription은 취소 된 것으로 간주되어야하며 Caller는 런타임 환경에 적합한 방식으로 오류을 처리해야 한다. 예를 들어 onNext를 호출했는데, NullPointerException이 아닌 다른 예외가 발생한 경우 Caller는 예외를 로깅하거나 할 수 있다.


### Subscription

``` java
public interface Subscription {
    public void request(long n);
    public void cancel();
}
```

* Subscription.request and Subscription.cancel는 Subscriber의 컨텍스트안에서만 호출되어야 한다. Subscriber는 Subscription를 이용해서 데이터가 언제 더 필요한지 또는 데이터가 더이상 필요하지 않음을 컨트롤한다.
* Subscriber는 onNext 또는 onSubscribe 메소드 내에서 Subscription.request를 동기적으로 호출할 수 있다.
  * request 구현은 reentrant 해야한다. 이는 request와 onNext 사이의 상호 재귀의 경우 스택 오버플로를 피하기 위함이다. 이것은 Publisher가 synchronous 할 수 있음을 의미한다.
* Subscription.request는 Publisher와 Subscriber간에 동기식 재귀에 대한 상한선을 정해야한다. 이 규칙의 목적은 request와 onNext 사이의 상호 재귀에 상한을 두어 위의 규칙을 보완하는 것이다. 상호 재귀에 상환을 1로 두는 것을 권장한다. (스택 공간을 절약하기 위해서) 바람직하지 않는 동기 구현은 Subscriber.onNext -> Subscription.request -> Subscriber.onNext -> 식으로 계속해서 재귀를 통해 onNext와 request가 호출되는 것이다. 이 경우 스택 오버플로우가 발생할 수 있다.
* Subscription.request는 적시에 반환되어야한다. request는 non-obstructing 메소드이다. (매우 빠르게 종료되는 메소드이다) 호출 스레드에서 가능한 빠르게 실행되어야하며, caller의 스레드의 실행을 지연시킬만한 과도한 계산 및 기타작업은 하지 않는게 좋다. 
* Subscription.cancel는 적시에 반환되어야한다. 멱등성이 있어야 하며 thread-safe 해야한다.
* Subscription이 취소된 이후에 추가적인 Subscription.request(long n)은 비작동(non-operation)해야한다. 즉 Subscription이 취소된 이후에 Subscription.request를 호출하더라도 데이터를 추가적으로 전달해서는 안된다.
* Subscription이 취소된 이후에 추가적인 Subscription.cancel()은 비작동(non-operation)해야한다.
* request-ing은 데이터에 대한 요청이 Publisher에게 전달되도록 보장하는 것뿐만 아니라 추가적인 작업을 한다.
* Subscription.request(long n) 메소드는 전달된 파라미터가 0보다 작거나 같은 경우 IllegalArgumentException으로 onError를 호출해야한다. 
* Subscription.request (long n) 메소드는 Subscriber의 onNext를 동기식으로 호출 할 수 있다. 호출하는 스레드에서 로직을 실행하는 Publisher와 같은 synchronous Publishers를 만들 수 있다.
* Subscription.request (long n) 메소드는 Subscriber의 onComplete 또는 onError를 동기식으로 호출 할 수 있다.
* Subscription.cancel()은 Publisher에게 Subscriber 메소드 호출을 중단하도록 요청 해야한다. cancel()을 호출하는 즉시 Subscription에 영향을 미치지 않아도된다. Subscription을 취소하려는 의도가 결국 Publisher에 의해 존중되며 신호가 수신되기까지 시간이 걸릴 수 있다.
* Subscription.cancel()은 결국 해당 Subscriber에 대한 참조를 삭제하도록 Publisher에 요청 해야한다. Subscribers는 subscription이 더이상 유효하지 않으면 가비지컬렉터의 대상이 되어야한다. 동일한 Subscriber 객체로 다시 구독하는 것은 권장되지 않는다.
* Subscription.cancel이 호출된 시점에 다른 Subscription이 존재하지 않으면 Stateful한 Publisher의 경우 종료 상태로 전환 될 수 있다. 
* Subscription.cancel는 반드시 정상적으로 리턴되어야 한다.
* Subscription.request는 반드시 정상적으로 리턴되어야 한다.
* Subscription의 request 메소드는 무한대로 호출될 수 있어야하며, request 메소드로 최대 Long.MAX_VALUE개의 데이터를 요구할 수 있어야 한다. Publisher는 Long.MAX_VALUE를 무한대로 간주 할 수 있다.


Subscription 오직 한 쌍의 Subscriber, Publisher에 의해 공유된다. 따라서 subscribe() 메서드는 생성된 Subscription 객체를 반환하지 않는다. Subscription 객체는 onSubscribe 콜백을 통해 Subscriber에게만 전달된다. 
Subscription의 목적은 Subscriber와 Publisher 사이의 데이터 교환을 중재하는 것이다. 


### Publisher, Subscription, Subscriber 정리

Publisher.subscribe(Subscriber sub) 메소드를 통해 Publisher에 새로운 Subscriber를 등록하면, 아래와 같은 방식으로 Subscriber의 메소드가 차례로 호출된다.

![Alt text]({{ site.url }}/assets/img/posts/reactive-streams/publisher-subscriber.png)

```
onSubscribe onNext* (onError | onComplete)?
```

1. Subscriber.onSubscribe(Subscription s) : Subscriber가 Subscription 객체를 전달받는다. Subscriber는 새로운 이벤트를 Subscription.request 메소드를 통해서 요청할 수 있다.
2. Subscription.request(long n) : 메소드를 통해서 n개의 이벤트를 요청한다. 새로운 이벤트는 Subscriber.onNext 메소드를 통해서 전달받는다.
3. Subscriber.onNext : Subscribe가 새로운 이벤트를 받는다. 이벤트를 처리하고 난 뒤에 Subscription.request(long n) 메소드를 통해 다음 이벤트를 요청할 수 있다.
4. Subscriber.onComplete() : Publisher는 더 이상 전달할 이벤트가 없을때 Subscriber.onComplete() 메소드를 호출해야 한다.

### Processor

``` java
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
}
```

* Processor는 Subscriber, Publisher의 스펙을 모두 만족해야한다. 주로 map, filter, subscribeOn, publishOn 등 오퍼레이터를 구현할 때 사용한다.
* 프로세서는 onError 신호를 복구 할 수 있다. onError 신호를 복구하는 경우 Subscription은 취소된 것으로 간주해야한다. 만약에 onError 신호를 복구하지 않았다면, onError 신호를 Subscribers에 전파해야한다. 이는 Processor 구현이 단순한 transformation 작업 이상이 될 수 있음을 의미한다.


필수 사항은 아니지만 Processor의 마지막 Subscriber가 Subscription를 취소할 때, Processor의 업스트림 Subscription을 취소시키는 것이 좋다. 이를 통해 취소 신호가 업스트림으로 전파 될 수 있다.

![Alt text]({{ site.url }}/assets/img/posts/reactive-streams/processor.png)

* Processor는 오퍼레이터(map, filter, reduce)등을 구현할 때 사용되며, Publisher와 Subscriber 2가지 역할을 모두 수행한다.

### Asynchronous vs Synchronous Processing

Reactive Streams에서는 onNext, onError, onComplete 메소드가 Publisher를 block하지 않아야한다고 규정한다. onNext, onError, onComplete는 이벤트를 동기적/비동기적으로 처리할 수 있다. 

## More 

* [What Are Reactive Streams in Java?](https://dzone.com/articles/what-are-reactive-streams-in-java)
* [Reactive Stream](http://www.reactive-streams.org/)
* [Reactive Stream github](https://github.com/reactive-streams/reactive-streams-jvm)
* [Spring Camp - Reactive Stream](https://www.youtube.com/watch?v=UIrwrW5A2co)
* [Reactive Spring Workshop](https://speakerdeck.com/simonbasle/reactive-spring-workshop-at-javaland)
* [Reactive Streams로의 여행](http://eyeahs.github.io/blog/2017/01/24/a-journey-into-reactive-streams/)
* [Reactive Programming VS Reactive System](http://blog.lespinside.com/reactive-programming-versus-reactive-systems/)
