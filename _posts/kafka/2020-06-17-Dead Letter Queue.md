---
layout: post
title: "Dead Letter Queue in Kafka" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka]
---

# Dead Letter Queue in Kafka

### Dead Letter Queue

* 메시지를 어떠한 이유로 처리할 수 없는 경우엔 Dead Letter Queue(토픽)로 보낸다. 아래와 같은 이유로 메시지 처리가 실패할 수 있다
  * 메시지를 deserialize할 수 없는 경우
  * 데이터가 예상과 다른 경우 (예를 들어 값이 항상 양수인 필드에 음수가 들어간 경우)

### Kafka Connect

Kafka Connect에서는 처리할 수 없는 메시지를 dead letter queue로 보내도록 설정할 수 있다.  dead letter로 보내진 메시지는 무시되거나 수정 및 재처리 될 수 있다.

![Source topic messages --> Kafka Connect --> Sink messages --> Dead letter queue](https://cdn.confluent.io/wp-content/uploads/DLQ_Source_Topic_Messages_Kafka_Connect_Sink_Messages-e1552339900964.png)

> 이미지 출처 : https://www.confluent.io/blog/kafka-connect-deep-dive-error-handling-dead-letter-queues/

dead letter queue을 사용하려면 아래 2가지 속성을 설정해주어야 한다.

* errors.tolerance = all
* errors.deadletterqueue.topic.name

더 자세한 설정은 [confluent - Error Handling and Dead Letter Queue](https://www.confluent.io/blog/kafka-connect-deep-dive-error-handling-dead-letter-queues/) 를 참고하길 바란다.

### Spring Kafka

* Spring Kafka에서는 `SeekToCurrentErrorHandler` 를 통해서 메시지 처리를 재시도할 수 있다. 그리고 최대 실패수만큼 재시도가 실패한 경우에 Recorerer가 동작하도록 설정할 수 있다. Spring Kafka는 `DeadLetterPublishingRecoverer` 를 제공하는데 해당 Recoverer는 실패한 메시지를 다른 토픽으로 보낸다. 
* 실패한 메시지는 `<originalTopic>.DLT` 로 보내진다. 그리고 실패한 메시지의 파티션과 같은 파티션으로 보내진다. 그러므로 기본 동작으로 설정한 경우엔 Dead Letter 토픽은 Original 토픽의 파티션보다 크거나 같아야한다. Dead Letter 토픽의 이름과 파티션을 직접 정하도록 설정하는것도 가능하다.
* Dead Letter 토픽에 레코드를 보낼때 아래 헤더가 추가되어 보내진다.
  * ` KafkaHeaders.DLT_EXCEPTION_FQCN`: The Exception class name.
  * `KafkaHeaders.DLT_EXCEPTION_STACKTRACE`: The Exception stack trace.
  * `KafkaHeaders.DLT_EXCEPTION_MESSAGE`: The Exception message.
  * `KafkaHeaders.DLT_ORIGINAL_TOPIC`: The original topic.
  * `KafkaHeaders.DLT_ORIGINAL_PARTITION`: The original partition.
  * `KafkaHeaders.DLT_ORIGINAL_OFFSET`: The original offset.
  * `KafkaHeaders.DLT_ORIGINAL_TIMESTAMP`: The original timestamp.
  * `KafkaHeaders.DLT_ORIGINAL_TIMESTAMP_TYPE`: The original timestamp type.

### 출처

* [medium - Dead Letter Queues in Kafka](https://medium.com/@sannidhi.s.t/dead-letter-queues-dlqs-in-kafka-afb4b6835309)
* [confluent- Error Handling and Dead Letter Queue](https://www.confluent.io/blog/kafka-connect-deep-dive-error-handling-dead-letter-queues/)

