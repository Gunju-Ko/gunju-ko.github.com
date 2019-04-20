---
layout: post
title: "Configuring a Streams Application" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka, kafka-stream]
---

# Configuring a Streams Application

## StreamsConfig
Kafka Streams를 설정하려면 `StreamsConfig` 객체를 사용하면 된다. 다음과 같은 순서로 설정을 할 수 있다.
1. `java.util.Properties` 객체를 생성
2. 파라미터를 세팅
3. `StreamsConfig` 객체를 생성할 때 `Properties`를 생성자를 통해서 전달한다.

아래는 `StreamsConfig` 객체를 생성하는 간단한 샘플코드이다.

``` java
import java.util.Properties;
import org.apache.kafka.streams.StreamsConfig;

Properties settings = new Properties();
// Set a few key parameters
settings.put(StreamsConfig.APPLICATION_ID_CONFIG, "my-first-streams-application");
settings.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-broker1:9092");
// Any further settings
settings.put(... , ...);

// Create an instance of StreamsConfig from the Properties instance
StreamsConfig config = new StreamsConfig(settings);
```

## Configuration Parameter Reference
Stream 설정 파라미터에서 가장 많이 사용하는 속성에 대해서 설명을 하도록 하겠다. 더 자세한 내용은 Javadocs를 참고하자 [StreamsConfig](https://kafka.apache.org/11/javadoc/org/apache/kafka/streams/StreamsConfig.html) 

### 필수 파라미터
application.id : Stream Processing 어플리케이션의 식별자이다. Kafka Cluster 안에서 고유한 값을 가져야 한다.
* 각각의 Stream Processing 어플리케이션은 고유한 application.id를 가져야한다.
* 같은 Stream Processing 어플리케이션을 여러 서버에 실행시킨다면, 다른 서버에서 실행되는 각각의 인스턴스는 같은 application id를 가져야 한다. 
* `.`, `-`, `_`를 포함해서 영숫자만을 사용하는것을 권장한다.
* application.id 값은 다음과 같은 값에 영향을 미친다.
  * `client.id`의 prefix값
  * KafkaConsuemr의 `group.id`
  * state 디펙토리의 서브 디렉토리 이름 (`state.dir`)
  * 인터널 Kafka Topic 이름의 Prefix

bootstrap.servers : Kafka Cluster에 연결을 맺기위한 부트스트랩 서버의 IP/PORT값

### 선택적 파라미터
모든 파라미터를 소개할 수는 없기 때문에 몇가지 중요하다고 생각되는 속성만 정리하겠다.

* client.id : 요청을 보낼때 사용되는 ID 값이다. 이 값은 Kafka Streams이 내부적으로 사용하는 Consumer와 Producer에게 전달된다.
* commit.interval.ms : 오프셋을 커밋하는 주기
* default.deserialization.exception.handler	: 예외 처리 클래스로 `DeserializationExceptionHandler` 인터페이스 구현체이어야 한다.
* default.production.exception.handler : 예외 처리 클래스로 `ProductionExceptionHandler` 인터페이스 구현체이어야 한다.
* default.key.serde : Record key를 위한 디폴트 serializer/deserializer 클래스로, `Serde` 인터페이스의 구현체여야 한다. Serialization과 Deserialization은 데이터를 구체화할 때 필요하다. 예를 들어, 카프카 토픽으로부터 데이터를 읽어드리거나 혹은 카프카 토픽에 데이터를 보내는 경우이다. 또한 State Store에서 데이터를 읽어드리거나 혹은 State Store에 데이터를 쓰는 경우에도 Serialization과 Deserialization이 필요하다. 자세한 경우는 [Data types and serialization](https://kafka.apache.org/11/documentation/streams/developer-guide/datatypes.html#streams-developer-guide-serdes)을 참고해라
* default.value.serde : Record value를 위한 디폴트 serializer/deserializer 클래스로, `Serde` 인터페이스의 구현체여야 한다.
* num.standby.replicas : Standy Replicas의 수이다. Standy Replicas라는 것은 Status Store의 복사본이다. Kafka Stream은 명시된 수의 복사본을 생성하려고 시도한다. 그리고 가능한한 최신 상태로 유지하려고 한다. Standy Replicas는 장애 조치의 지연을 최소화하는데 사용된다. 만약에 특정 인스턴스가 다운되면 각각의 Task들은 standby replica를 가지고 있는 인스턴스에서 재실행 되도록 한다. 따라서 State Store를 복원하는데 걸리는 시간을 최소화할 수 있다. Kafka Stream이 standby replicas를 사용해서 어떻게 State Store 복원을 최소화하는지 더 자세히 알고 싶으면 [State](https://kafka.apache.org/11/documentation/streams/architecture.html#streams-architecture-state) 섹션을 참고해라
* num.stream.threads : 스트림 프로세싱을 처리하기 위한 스레드의 수이다. 스트림 처리코드는 이 스레드에서 실행된다. 더 자세한 내용은 [Thread Model](https://kafka.apache.org/11/documentation/streams/architecture.html#streams-architecture-state)를 참고해라
* replication.factor : Kafka Stream이 내부적으로 생성하는 토픽의 replication 수를 지정할 때 사용한다. replication는 fault tolerane를 위해 중요하다. replication이 없으면 브로커 중 하나라도 오류가 발생해도 Stream 어플리케이션에게 영향을 미친다. Source 토픽의 replication 수와 비슷한 수의 replication을 생성하는 것이 좋다.
* state.dir	: State Store의 위치이다. Kafka Stream은 state store를 해당 디렉토리 안에 저장한다. `state.dir` 디렉토리 안에 Stream 어플리케이션 별로 서브 디렉토리가 생성된다. 이 때 생성되는 서브 디렉토리의 이름은 application id이다. 
* partition.grouper : partiton grouper는 Source 토픽에 해당하는 Task들을 만든다.  이 때 각각의 Task들은 파티션을 할당 받는다. partition grouper의 기본 구현체는 DefaultPartitionGrouper 클래스이다. 이 클래스는 Task에게 각각의 Source 토픽의 파티션을 하나씩 할당한다. 따라서 생성되는 Task의 수는 Source 토픽 중에서 가장 많은 파티션를 가진 토픽의 파티션 개수이다.

### KafkaConsumer, KafkaProducer 설정 파라미터
Kafka Stream에서 내부적으로 사용하는 Kafka Producer와 Kafka Consumer를 설정할 수 있다. Consumer와 Producer 설정은 StreamConfigs 객체를 통해서 할 수 있다.

아래는 Kafka Consumer의 `session.timeout`값을 설정하는 예제이다.

``` java
Properties streamsSettings = new Properties();
// Example of a "normal" setting for Kafka Streams
streamsSettings.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-broker-01:9092");
// Customize the Kafka consumer settings of your Streams application
streamsSettings.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 60000);
StreamsConfig config = new StreamsConfig(streamsSettings);
```

KafkaConsumer 속성과 KafkaProducer 속성중에 이름이 겹치는 부분이 있다. 이 경우에는 `consumer` 혹은 `producer` prefix를 붙임으로써 해결할 수 있다. 아래는 간단한 샘플 코드이다.

``` java
Properties streamsSettings = new Properties();
// same value for consumer and producer
streamsSettings.put("PARAMETER_NAME", "value");
// different values for consumer and producer
streamsSettings.put("consumer.PARAMETER_NAME", "consumer-value");
streamsSettings.put("producer.PARAMETER_NAME", "producer-value");
// alternatively, you can use
streamsSettings.put(StreamsConfig.consumerPrefix("PARAMETER_NAME"), "consumer-value");
streamsSettings.put(StreamsConfig.producerPrefix("PARAMETER_NAME"), "producer-value");
```

### Default 값
Kafka Stream은 KafkaConsumer, KafkaProducer의 속성 중 일부 속성을 디폴트값과 다른 값으로  세팅해서 사용한다. 각각의 설정에 대한 자세한 내용은 설명하지 않도록 하겠다. 다만 retries 속성이나 max.block.ms 속성을 변경하는 경우에는 max.poll.interval.ms 값 설정을 다음 아래와 같이 설정하는게 좋다. 

> max.poll.interval.ms > min ( max.block.ms, (retries +1) * request.timeout.ms ) 

#### Consumer
* auto.offset.reset : earliest
* enable.auto.commit : false
* max.poll.interval.ms : Integer.MAX_VALUE
* max.poll.records : 1000
* rocksdb.config.setter

`enable.auto.commit` 속성은 사용자가 오버라이딩 할 수 없다. 따라서 Kafka Stream은 항상 auto commit 기능을 사용하지 않는다.

#### Producer
* linger.ms : 100
* retries : 10

#### rocksdb.config.setter
RocksDB를 설정할 때 사용한다. Kafka Stream은 디폴트 스토리지 엔진으로 RocksDB를 사용한다. RocksDB의 기본 설정을 바꾸기 위해서는 `RocksDBConfgSetter` 인터페이스를 구현해야 한다. 그리고 `rocksdb.config.setter` 속성을 통해서 해당 클래스를 설정하면 된다. 아래는 RocksDB의 메모리 사이즈를 조정하는 간단한 샘플 예제이다.

``` java
    public static class CustomRocksDBConfig implements RocksDBConfigSetter {

       @Override
       public void setConfig(final String storeName, final Options options, final Map<String, Object> configs) {
         // See #1 below.
         BlockBasedTableConfig tableConfig = new org.rocksdb.BlockBasedTableConfig();
         tableConfig.setBlockCacheSize(16 * 1024 * 1024L);
         // See #2 below.
         tableConfig.setBlockSize(16 * 1024L);
         // See #3 below.
         tableConfig.setCacheIndexAndFilterBlocks(true);
         options.setTableFormatConfig(tableConfig);
         // See #4 below.
         options.setMaxWriteBufferNumber(2);
       }
    }

Properties streamsSettings = new Properties();
streamsConfig.put(StreamsConfig.ROCKSDB_CONFIG_SETTER_CLASS_CONFIG, CustomRocksDBConfig.class);
```

### Resiliency를 위한 추천 파라미터

| Parameter Name      |    Corresponding Client | Default value  | Consider setting to|
| :-------- | :--------| :--------: | :--------: |
| acks  | Producer |  acks=1   | acks=all |
| replication.factor | Streams |  1  | 2 |
| min.insync.replicas  | Broker | 1  | 2 |

만약에 replication factor을 3으로 증가시킨다면 최대 2개까지 브로커가 다운되더라도 문제가 발생하지 않는다. 또한 acks=all로 설정함으로써 Record를 더 안전하게 카프카 토픽에 저장할 수 있다. default 값을 변경하는 것은 약간의 성능 저하와 추가적인 스토리지 공간을 필요로 하지만 해당 값들은 default 값보다는 위에서 말한 값으로 세팅하는 것을 추천한다.

## Processing.guaruntee
만약에 `processing.guarantee`를 `exactly_once`로 설정했다면, Kafka Streams는 다음과 같은 속성을 오버라이딩하는 것을 허용하지 않는다. Proceesing.gurantess는 굉장히 중요하기 때문에 나중에 다시 정리하도록 하겠다.

* isoloation.level : read_commit
* enable.idempotence : true
* max.in.flight.requests.per.connection : 1


## 출처
위에 글은 [Apache Document](https://kafka.apache.org/11/documentation/streams/developer-guide/config-streams.html#optional-configuration-parameters)와 [StreamsConfig](https://kafka.apache.org/11/javadoc/org/apache/kafka/streams/StreamsConfig.html)를 토대로 정리한 내용이다. 따라서 더 자세한 내용을 알고 싶은 경우 이 링크를 통해 확인하길 바란다.
