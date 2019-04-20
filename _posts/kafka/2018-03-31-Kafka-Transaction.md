---
layout: post
title: "Kafka Transaction"
author: Gunju Ko
categories: [kafka]
cover:  "/assets/instacode.png"
---

# Kafka Transaction
Kafka Client 0.11에 추가된 Kafka Transaction에 대해 학습한 내용을 정리한 글이다

----------

## Overview of delivery semantics

### At least once semantics
- At least once는 말 그대로 메시지를 적어도 한 번 전달한다는 의미이다. 이 때 발생할 수 있는 문제점은 메시지가 중복해서 전달될 수 있다는 것이다.
- Messages are never lost but may be redelivered.

> **Producer Configuration*
> - acks=all
> - retries : Setting a value greater than zero will cause the client to resend any record whose send fails with a potentially transient error.
> - if a producer ack times out or receives an error, it might retry sending the message assuming that the message was not written to the Kafka topic.

#### Example

1 sending the message

![at-lest-once1]({{ site.url }}/assets/img/posts/kafka-transaction/at-lest-once1.png)

![at-lest-once2]({{ site.url }}/assets/img/posts/kafka-transaction/at-lest-once2.png)

2 broker had failed right before it sent the ack but after the message was successfully written to the Kafka topic

![at-lest-once3]({{ site.url }}/assets/img/posts/kafka-transaction/at-lest-once3.png)

3 retry sending the message assuming that the message was not written to the Kafka topic.

![at-lest-once4]({{ site.url }}/assets/img/posts/kafka-transaction/at-lest-once4.png)

4 retry leads to the message being written twice and hence delivered more than once to the end consumer. 

![at-lest-once5]({{ site.url }}/assets/img/posts/kafka-transaction/at-lest-once5.png)

![at-lest-once6]({{ site.url }}/assets/img/posts/kafka-transaction/at-lest-once6.png)

### At most once semantics
Messages may be lost but are never redelivered.

> **Producer Configuration**
> - retries=0 
>   - if the producer does not retry when an ack times out or returns an error, then the message might end up not being written to the Kafka topic, and hence not delivered to the consumer
> - ack=0
>   - If set to zero then the producer will not wait for any acknowledgment from the server at all. 

#### Example

 1 sending the message

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/at-most-once1.png)

 2 producer receives error

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/at-most-once2.png)

 3 producer does not retry when an ack times out or returns an error, then the message might end up not being written to the Kafka topic

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/at-most-once3.png)

## Exactly once semantics in Apache Kafka
### Idempotence

- Idempotent Producer를 통해서 메시지를 정확히 한번만 전달할 수 있다. Retry가 더이상 메시지 중복을 발생시키지 않는다.
- even if a producer retries sending a message, it leads to the message being delivered exactly once to the end consumer.

----------

#### Example

1 sending the message with metadata
  - sequence number : identifies the record
  - producer id : identifies the producer whether the broker can distinguish between different producer

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/idempotence1.png)
![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/idempotence2.png)


2 producer receives error

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/idempotence3.png)

3 retry sending the message assuming that the message was not written to the Kafka topic.
  - send message with the same metadata
  - producer retries will no longer introduce duplicates. 
  - broker can actually identify that that's the retry

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/idempotence4.png)
![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/idempotence5.png)

> **Producer Configuration**
>- To enable idempotence, the enable.idempotence configuration must be set to true.
>- it is recommended to leave the retries config unset, as it will be defaulted to Integer.MAX_VALUE. 
>- acks config will default to all. 

#### Code Example
``` java
private static Properties producerProperties() {
        Properties props = new Properties();
        props.put(CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        // ...
        // ...
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
        return props;
 }
```

### Transactions : Atomic writes across multiple partition
- Kafka 0.11에 추가된 transactions API를 사용하면 여러 partition에 여러 개의 메시지를 atomic하게 보낼 수 있다.

----------
1 3개의 partiton에 5개의 메시지를 보낸다. 이 때 message의 트랜잭션이 아직 진행중이라고 표시한다. 아직 트랜잭션이 진행중이기 때문에 Consumer는 이 메시지를 읽을 수 없다. (read_committed인 경우)

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/transaction1.png)
![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/transaction2.png)

2 commit all those message in an atomic manner. 
- commit이 되면 Consumer는 이 메시지를 읽을 수 있다.
- 트랜잭션이 commit이 되지 않고 abort된 경우에 이 메시지들을 읽을 수 없다. (read_committed인 경우)

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/transaction3.png)

> **Producer Configuration**
>- set a producer config “transactional.id” to some unique ID.
>- "transactional.id"가 설정이 되면 "enable.idempotence"는 자동으로 true가 된다.
>- 여러 Application에서 KafkaProducer를 사용하고 있다면, 각 어플리케이션은 서로다른 transactional.id를 사용해야 한다.
>- there can be only one open transaction per producer. 
>- All messages sent between the beginTransaction() and commitTransaction() calls will be part of a single transaction.

#### Code Example
``` java
public static void main(String[] args) {
        Producer<Long, String> producer = new KafkaProducer<>(producerProperties());

        producer.initTransactions();

        producer.beginTransaction();
        for (Meesage message : messages) {
            producer.send(new ProducerRecord<>(TOPIC, meaage.getKey(), message.getValue()));
        }

        producer.commitTransaction();
        producer.close();
    }

    private static Properties producerProperties() {
        Properties props = new Properties();
        props.put(CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        // ...
        // ...
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "sample-producer");
        return props;
    }
```

> **Note**
> - initTransactions() :  Needs to be called before any other methods when the transactional.id is set in the configuration.
>   - initTransaction을 호출하면 같은 transactional.id를 사용하던 KafkaProducer의 트랜잭션을 종료를 기다린다. 그리고 producer id와 새로운 epoch를 가지게 된다. 이전 epoch를 사용하던 KafkaProducer는 더이상 사용를 하지 못한다.

### Reading Transactional Mesesage
Kafka guarantees that a consumer will eventually deliver only non-transactional messages or committed transactional messages. 
 
 > **Consumer Configuration**
> - isolation.level
>  - read_committed : In addition to reading messages that are not part of a transaction, also be able to read ones that are, after the transaction is committed.
>  - read_uncommitted(default) : Read all messages in offset order without waiting for transactions to be committed.

#### Example

1 It will withhold messages from open transactions

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/read_commit2.png)

2 The Kafka consumer will only deliver transactional messages to the application if the transaction was actually committed.

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/read_commit1.png)

#### Code Example
``` java
    private static Properties consumerProperties() {
        Properties props = new Properties();
        props.put(CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
	    // .... 
	     
        props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
        return props;
    }
```

###  Read-Process-Write
- Marking an offset as consumed is called committing an offset. 
- 카프카에서 record의 offset를 commit하는 것은 offsets topic에 해당 offset를 write하는 것이다. 
- offset를 commit하는 것은 Kafka Topic에 메시지를 보내는 것이기 때문에 트랜잭션으로 포함시킬 수 있다.

#### Example
-----------

1 consumes a message M1 

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/readProcessWrite1.png)

2 write message M2 to Topic B after doming some processing on message M1

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/readProcessWrite2.png)

3 commit offsets

- The message M1 will be considered consumed from Topic A only when its offset is marked as consume
- commit of the offset and wirte of message M2 to Topic B will be part of a single transaction

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/readProcessWrite3.png)

4 Exactly Once Processing
-  exactly once processing means that M1 is considered consumed if and only if M2 is successfully produced, and vice versa
-  messages M1 and M2 are considered successfully consumed and published together

![Alt text]({{ site.url }}/assets/img/posts/kafka-transaction/readProcessWrite4.png)

#### Code Example
``` java
while (true) {
  ConsumerRecords records = consumer.poll(Long.MAX_VALUE);
  producer.beginTransaction();
  for (ConsumerRecord record : records) {
    // processing 
    // ...
 
    // write
    producer.send(producerRecord(“outputTopic”, record));
  }
  // commit offset
  producer.sendOffsetsToTransaction(currentOffsets(consumer), group);  
  producer.commitTransaction();
}
```
> **Note**
> - offsets will be considered committed only if the transaction is committed successfully
> - enable.auto.commit=false and should also not commit offsets manually (via sync or async commits).


### 출처
- https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/
- https://www.confluent.io/blog/transactions-apache-kafka/
- https://kafka.apache.org/10/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html
- https://kafka.apache.org/10/javadoc/index.html?org/apache/kafka/clients/consumer/KafkaConsumer.html
