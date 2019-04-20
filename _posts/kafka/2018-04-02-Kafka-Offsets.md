---
layout: post
title: "Kafka Offsets" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka]
---

# Kafka Offsets
본 글에서는 Consumer의 Offset에 대해 알아보도록 하자

## Offsets and Consumer Position
Kafka는 각각의 Record에 대해 오프셋을 가지고 있다. 이 오프셋은 파티션 내에서 Record에 대한 고유한 식별자로 사용된다. 또한 Consumer의 파티션내의 position를 나타낸다. 예를 들어 Consumer의 Position이 5인 경우에 오프셋이 0부터 4까지인 Record는 이미 처리하였음을 의미한다. 또한 다음 poll()에서 받는 첫번째 Record의 오프셋은 5가 된다. 
사실 Consumer에는 두 가지 개념의 Position이 있다. 
* Position : 다음 poll()에서 가져올 첫 번째 Record의 오프셋을 의미한다. 따라서 Consumer가 처리한 Record 중 가장 높은 오프셋보다 1 크다. Position 값은 Consumer가 poll()를 호출할 때마다 자동으로 증가한다.
* Committed Position : Offset Topic에 저장된 마지막 오프셋으로 Consumer 프로세스가 재시작되는 경우 Consumer는 Committed Position부터 Record를 읽는다. AutoCommit 기능을 자용해서 주기적으로 오프셋를 자동으로 커밋할 수 있다. 혹은 commitSync나 commitAsync 메소드를 이용해서 오프셋을 직접 커밋 할 수 있다.

## Storing Offsets Outside Kafka
Consumer 어플리케이션은 오프셋을 Offset Topic에 저장해야만 하는 것은 아니다. 오프셋을 다른 저장소에 저장할 수 있다. 따라서 특정 저장소에 Record 처리 결과를 저장하는 것과, 오프셋을 저장하는 행위를 Atomic하게 처리할 수 있다. 그리고 이를 통해 Exactly Once semantic를 구현할 수 있다. 아래는 오프셋을 다른 저장소에 저장하는 적절한 예이다.
* 오프셋을 관계형 데이터베이스에 저장해서 관리한다. 그러면 Record를 처리하는 과정에서 일어나는 DB 작업과 오프셋을 커밋하는 행위를 하나의 트랜잭션으로 처리할 수 있다.

각각의 Record는 Offset를 가지고 있다. 따라서 Offset를 직접 관리하려면 다음과 같은 작업이 필요하다.

* `enable.autocommit`설정을 `false`로 해야한다.
* ConsumerRecord에서 제공하는 오프셋을 이용해서 오프셋을 저장해라
* 재시작하는 경우에는 Consumer의 Position을 복구 해야한다. 이 때 `seek(TopicPartition, long)`메소드를 사용해라

만약에 Partition 할당이 자동으로 이뤄진다면 Partition이 재할당 될 경우를 고려해야 한다. Partition이 재할당 되는 경우를 처리하기 위해서는 `ConsumerRebalanceListener`를 사용하면 된다. 자세한 건 아래에서 설명하도록 하겠다.

## Controlling The Consumer's Position
Consumer는 Position을 수동으로 관리할 수 있다. 이 말은 Consumer는 이미 처리한 Record를 다시 읽도록 Position을 조정하거나 특정 Record를 읽지 않고 가장 최근의 Record를 읽도록 Position을 조정할 수 있다. Consumer의 다음과 같은 메소드를 이용해서 Position을 조정할 수 있다.
* seek(TopicPartition, long) : Consumer의 
* seekToBegining(Collection) 
* seekToEnd(Collection)

## ConsumerRebalanceListener
파티션이 재조정 될 때 실행되는 콜백 메소드를 가진 인터페이스이다. ConsumerRebalanceListener 인터페이스는 Consumer가 파티션을 직접 할당하지 않고, 컨슈머 그룹 개념을 사용해서 파티션이 자동으로 할당 되도록 했을 경우만 사용할 수 있다. 만약 파티션을 직접 할당 했다면, 재조정은 발생하지 않으며 콜백 메소드는 호출되지 않는다. 
ConsumerRebalanceListener를 사용하면 오프셋을 DB나 다른 저장소에 저장할 수 있다. 예를 들어, onPartitionRevoked 메소드에서 파티션이 재조정 될 때 파티션의 오프셋을 저장 할 수 있다. 

콜백 메소드는 파티션이 재조정 될 때마다 poll 메소드 내부에서 호출이 된다. 따라서 콜백 메소드는 유저 스레드에서만 실행이 된다. 

모든 Consumer가 onPartitionRevoked 메소드를 호출 한 후에 onPartitionAssigned 메소드가 호출된다. 따라서 파티션 오프셋을 onPartitionRevoked 메소드에서 저장하고, onPartitionAssigned 메소드에서 파티션 오프셋을 읽는 것이 바람직하다. 이 경우엔 모든 파티션의 오프셋이 저장된 후에 onPartitionAssigned 메소드에서 저장된 오프셋을 읽을 것이다.

아래의 코드는 오프셋을 저장하는 pseudo-code이다.

```java   
    public class SaveOffsetsOnRebalance implements ConsumerRebalanceListener {
       private Consumer<?,?> consumer;

       public SaveOffsetsOnRebalance(Consumer<?,?> consumer) {
           this.consumer = consumer;
       }

       public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
           // save the offsets in an external store using some custom code not described here
           for(TopicPartition partition: partitions)
              saveOffsetInExternalStore(consumer.position(partition));
       }

       public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
           // read the offsets from an external store using some custom code not described here
           for(TopicPartition partition: partitions)
              consumer.seek(partition, readOffsetFromExternalStore(partition));
       }
   }
```

ConsumerRebalanceListener 클래스의 2개의 콜백 메소드에 대해 좀 더 알아보자

#### ConsumerRebalanceListener.onPartitionRevoked(Collection<`TopicPartition`> partitions)

파티션 재조정이 일어나는 경우 콜백 메소드가 실행이 된다. 따라서 onPartitionRevoked 메소드에서 오프셋을 다른 저장소에 저장할 수 있다. 이 메소드는 Consumer가 데이터를 가져오는 것을 멈추고 rebalance 작업이 시작되기 전에 호출이 된다. 오프셋을 다른 저장소에 저장할 때 이 콜백 메소드에서 저장하는 것을 권장한다. 

* 이 메소드는 rebalance가 일어나기 전에만 호출이 된다. KafkaConsumer.close() 메소드가 호출되기 전에 호출이 되거나 하진 않는다.
* 보통 오프셋을 저장하기 위해서 Consumer 인스턴스를 사용한다. 그리고 Consumer 인스턴스의 메소드 호출에서 WakeupException 혹은 InterruptException이 발생할 수 있다. 이 때 예외는 현재 KafkaConsumer.poll() 메소드로 전파된다. 즉 이 예외를 catch해서 따로 처리할 필요가 없다.

파라미터
* partitions : 파티션 재조정이 일어나기 전에 할당되었던 파티션들의 목록

#### ConsumerRebalanceListener.onPartitionAssgined(Collection<`TopicPartition`> partitions)
파티션이 재조정되면 콜백 메소드가 실행된다. 따라서 할당된 파티션에 대한 오프셋을 저장소에서 가져오고 Consumer의 Posititon을 초기화하는 작업을 이 메소드에서 실행하면 된다.

* 이 메소드는 파티션 재조정이 완료되고 Consumer가 데이터를 읽어오기 전에 호출이 된다.
* 이 메소드는 그룹내의 모든 Consumer들이 onPartitionRevoked 메소드 호출을 완료한 뒤에 호출된다.
* 오프셋을 조정하기 Consumer 인스턴스를 사용하는데 이 때 WakeupException이나 InterruptException이 발생할 수 있다. 이 경우에 예외가 Consumer의 poll() 메소드로 전파된다. 따라서 예외를 catch해서 따로 처리할 필요가 없다.

파리미터
* partitions : Consumer에 할당된 파티션

더 자세한 내용은 아래 출처에서 확인할 수 있다.

## 출처
* [KafkaConsumer](https://kafka.apache.org/10/javadoc/index.html?org/apache/kafka/clients/consumer/KafkaConsumer.html)
* [ConsumerRebalanceListener](https://kafka.apache.org/10/javadoc/org/apache/kafka/clients/consumer/ConsumerRebalanceListener.html)
