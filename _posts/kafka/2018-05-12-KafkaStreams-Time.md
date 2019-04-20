---
layout: post
title: "Kafka Stream - Time" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka, kafka-stream]
---

# Time 

## Time
스트림 프로세싱에서 시간은 중요한 개념이다. 그리고 시간을 어떻게 모델링하고 통합하는지도 중요하다. 예를 들어 Windowing과 같은 연산은 시간 경계를 기반으로해서 정의된다.

카프카 스트림은 다음과 같은 시간 개념을 지원한다.

* Event-time : 이벤트 또는 레코드가 발생한 시점. Event-time semantics를 위해서는 레코드를 생성할 때 레코드에 타임스탬프를 포함해야 한다. 즉 Event-time은 레코드가 생성되는 시점에 시간을 의미한다.
* Processing-time : 스트림 프로세싱 어플리케이션에 의해서 레코드가 처리되는 시점. Processing time은 event-time보다 밀리 초 혹은 몇 시간 정도 늦은 시간이다.
* Ingestion-time :  카프카 브로커가 이벤트나 레코드를 토픽 파티션에 저장하는 시점. Ingestion-time은 레코드 자체에 타임 스탬프가 포함된다는 점에서 event-time과 유사하다. 차이점은 레코드가 생성될 때가 아니라 카프카 브로커가 레코드를 토픽 파티션에 추가할 때 타임스탬프가 생성된다는 점이다. 레코드가 생성되는 시점과 레코드가 토픽에 저장되는 시점의 시간차가 충분히 작다고 가정할 경우, Ingestion-time과 event-time은 거의 비슷하다. 따라서 Event time semantics가 가능하지 않은 경우에는 Ingestion-time이 대안이 될 수 있다. Event time semantics가 가능하지 않은 경우는 Producer가 타임스탬프를 포함시키지 않거나 (낮은 버전의 KafkaProducer) 혹은 Producer가 로컬 시간을 가져오지 못해서 타임스탬프를 직접 할당할 수 없는 경우 등이 있다.

카프카 스트림은 [timestamp extractors](https://docs.confluent.io/current/streams/developer-guide/config-streams.html#streams-developer-guide-timestamp-extractor) 통해 모든 레코드에 타임스탬프를 할당한다. 레코드 별 타임스탬프는 시간과 관련하여 스트림의 진행 상황을 설명한다. (스트림 내에서 레코드의 순서가 맞지 않을 수도 있지만) 그리고 조인과 같은 시간에 의존하는 연산에 의해 활용된다. 이것을 스트림 타임이라고 부르며 wall-clock-time과는 다르다. (wall-clock-time은 어플리케이션이 실행될 때 실제 시간이다) 스트림 타임은 어플리케이션 내에서 여러 Input 스트림을 동기화하는데도 사용된다.

Timestamp Extractor의 구현체는 레코드에 포함된 타임스탬프를 기반으로 해서 타임 스탬프를 계산할 수 있다. 이를 통해 event-time semantics 혹은 ingestion-time semantics를 제공할 수 있다. 또는 Timestamp Extractor 구현체는 레코드를 처리 시점의 wall clock time을 리턴하는 방법으로 구현할 수 있다. 이 경우는 processing-time semantics를 제공한다. 개발자는 비즈니스 요구 사항에 따라서 다른 시간 개념을 적용할 수 있다.

카프카 스트림 어플리케이션이 카프카에 레코드를 보낼 때마다 레코드에 타임스탬프를 할당할 수 있다. 타임 스탬프가 할당되는 방식은 컨텍스트에 따라 다르다.

* Input 레코드를  직접 처리하여 ouput 레코드가 생성된 경우 : output 레코드의 타임 스탬프는 input 레코드의 타임 스탬프로 할당된다.
* Ouput 레코드가 주기적 함수를 통해 생성된 경우 : Ouput 레코드의 타임스탬프는 Stream Task의 내부 시간으로 할당 된다.
* Aggregation의 경우, 업데이트된 레코드의 타임스탬프는 업데이트를 트리거한 최신 input 레코드의 타임스탬프로 할당 된다.

> Tip
> * Know your time : 시간 개념을 사용할 때, time zones과 calendars가 올바르게 동기화 되었는지 등 시간의 추가적인 측면을 확인해야 한다. 그리고 다른 time semantics을 사용하는 토픽을 섞어서 사용하면 안된다.

## default.timestamp.extractor

Timestamp extractor는 레코드에서 타임 스탬프를 꺼내온다. 타임스탬프는 스트림의 진행을 제어하는데 사용된다. 기본 구현체는 [FailOnInvalidTimestamp](https://docs.confluent.io/4.1.0/streams/javadocs/org/apache/kafka/streams/processor/FailOnInvalidTimestamp.html) 이다. FailOnInvalidTimestamp는 KafkaProducer가 카프카 메시지에 자동으로 포함하는 타임스탬프 값을 꺼내온다. 카프카 브로커 설정(`log.message.timestamp.type`)과 토픽 설정(`message.timestamp.type`)에 따라서 time semantic이 달라진다.

* event-time : `log.message.timestamp.type` 설정값이 CreateTime으로 설정된 경우 해당 설정의 기본값이 CreateTime이다. 
* ingestion-time : `log.message.timestamp.type` 설정값이 LogAppendTime으로 설정된 경우

FailOnInvalidTimestamp는 레코드가 유효하지 않는 타임 스탬프를 포함하고 있으면 예외를 발생시킨다. 왜냐하면 Kafka Streams가 유효하지 않는 레코드를 처리하지 않고 버리기 때문이다. 유효하지 않은 타임스탬프는 여러가지 이유로 발생할 수 있다. 예를 들어, 0.10 이전 버전의 KafkaProducer를 사용해서 생성된 레코드에는 타임스탬프가 포함되어 있지 않을 수 있다. 

유효하지 않은 타임스탬프를 가지고 있는 레코드를 처리하고 싶다면 다음 두 개 중 하나의 extractors를 사용하면 된다. 이 extractors는 유효하지 않은 타임 스탬프를 다르게 처리한다.

* LogAndSkipOnInvalidTimestamp : 레코드의 타임 스탬프가 유효하지 않다면, log를 기록하고 유효하지 않는 타임스탬프를 리턴한다. Kafka Streams는 유효하지 않은 레코드를 처리하지 않고 버릴 것이다. LogAndSkipOnInvalidTimestamp을 사용하면 input 데이터에 유효하지 않는 레코드가 있어도 예외가 발생하지 않고 처리를 계속한다. (물론 유효하지 않는 레코드는 버려진다)
* UsePreviousTimeOnInvalidTimestamp : 만약에 레코드의 타임 스탬프가 유효하지 않다면, UsePreviousTimeOnInvalidTimestamp는 같은 토픽 파티션의 이전 레코드의 타임 스탬프 값을 리턴한다. 만약에 같은 토픽 파티션의 이전 레코드의 타임 스탬프값이 없는 경우에는 예외가 발생한다. 

다른 extractor로 WallclockTimestampExtractor가 있다. WallclockTimestampExtractor는 레코드로부터 타임스탬프를 꺼내오지 않는다. 대신 시스템의 현재 시간을 반환한다. (System.currentTimeMillis()) 따라서 WallclockTimestampExtractor를 사용하는 경우, 스트림은 이벤트의 processing time을 기반으로 실행된다.

또한 커스텀한 Timestamp extractor를 구현할 수도 있다. 예를 들어 메시지의 페이로드에 포함된 타임스탬프를 꺼내올 수 있다. 만약에 타임스탬프를 꺼내 올 수 없다면 예외를 발생시켜도 되고, 또는 대략의 타임스탬프를 리턴해도 된다. 음수인 타임스탬프를 리턴할 수도 있는데, 이 경우엔 해당 레코드는 처리되지 않고 버려진다. 즉 데이터 손실이 발생한다. 

아래는 커스텀 TimestampExtractor 구현 예제이다.

``` java
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.streams.processor.TimestampExtractor;

// Extracts the embedded timestamp of a record (giving you "event-time" semantics).
public class MyEventTimeExtractor implements TimestampExtractor {

  @Override
  public long extract(final ConsumerRecord<Object, Object> record, final long previousTimestamp) {
    // `Foo` is your own custom class, which we assume has a method that returns
    // the embedded timestamp (milliseconds since midnight, January 1, 1970 UTC).
    long timestamp = -1;
    final Foo myPojo = (Foo) record.value();
    if (myPojo != null) {
      timestamp = myPojo.getTimestampInMillis();
    }
    if (timestamp < 0) {
      // Invalid timestamp!  Attempt to estimate a new timestamp,
      // otherwise fall back to wall-clock time (processing-time).
      if (previousTimestamp >= 0) {
        return previousTimestamp;
      } else {
        return System.currentTimeMillis();
      }
    }
    return timestamp;
  }
}
```
 
스트림 설정에서 커스텀 Timestamp extractor를 사용하도록 설정할 수 있다.

``` java
import java.util.Properties;
import org.apache.kafka.streams.StreamsConfig;

Properties streamsConfiguration = new Properties();
streamsConfiguration.put(StreamsConfig.TIMESTAMP_EXTRACTOR_CLASS_CONFIG, MyEventTimeExtractor.class);
```

## Flow Control with Timestamps

Kafka Streams는 레코드의 타임스탬프로 스트림의 진행을 규제한다. 이를 통해 모든 Source 스트림을 시간으로 동기화시키려고 노력한다. 기본적으로 Kafka Streams은 event-time processing semantics를 제공한다. 이는 어플리케이션이 여러개의 Topic의 데이터를 처리할 때 특히 더 중요하다. 예를 들어 어플리케이션의 비즈니스 로직이 많이 변경 된 경우 과거의 데이터를 다시 처리하고 싶을 수 있다. 카프카에서 많은 양의 과거 데이터를 가져오는 것은 매우 쉽다. 그러나 적절한 흐름 제어가 없으면 파티션의 데이터 처리가 동기화되지 않고 잘못된 결과가 나올 수 있다.

Kafka Streams에서 각 레코드는 타임스탬프와 연관이 있다. 레코드 버퍼에 있는 레코드들의 타임스탬프 값을 기반으로해서 스트림 Task는 처리할 다음 파티션을 결정한다. 그러나 Kafka Streams는 처리를 위해 단일 스트림 내에서 레코드를 재정렬하지 않는다. 왜냐하면 재정렬은 카프카의 delivery semantics을 깨고 fail 발생시 복구를 어렵게하기 때문이다. 
이러한 흐름 제어가 최선이다. 왜냐하면 항상 레코드의 타임스탬프로 스트림간의 실행 순서를 엄격하게 적용할 수 없기 때문이다. 엄격한 실행 순서를 적용하기 위해서는 시스템이 모든 스트림에서 모든 레코드를 수신할때까지 기다려야하기 때문이다. 혹은 타임스탬프 경계에 대한 추가적인 정보를 넣어야한다.


## 출처
* [Concepts](https://docs.confluent.io/current/streams/concepts.html)
* [Flow control with Timestamps](https://docs.confluent.io/current/streams/architecture.html#streams-architecture-flow-control)
* [default.timestamp.extractor](https://docs.confluent.io/current/streams/developer-guide/config-streams.html#default-timestamp-extractor)
