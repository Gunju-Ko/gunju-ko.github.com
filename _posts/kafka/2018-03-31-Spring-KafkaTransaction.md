---
layout: post
title: "Spring Kafka Transaction" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka, spring-kafka]
---

## Spring Kafka Transaction
Spring Kafka Transaction과 관련된 라이브러리를 정리한 글이다. 더 자세한 내용은 [레퍼런스](https://docs.spring.io/spring-kafka/docs/2.1.4.RELEASE/reference/html/)를 참고하길 바란다.

Spring for Apache Kafka adds transaction supoort in several ways
- KafkaTransactionManager
- Transactional KafkaMessageListenerContainer
- Local Transaction with KafkaTemplate

### Transactions
- Transactions을 사용하려면 DefaultKafkaProducerFactory의 setTransactionIdPrefix 메소드를 통해서 transactionIdPrefix를 설정하면 된다. 
- KafkaTemplate, KafkaTransactionManager는 Producer 생성을 DefaultKafkaProducerFactory에 위임한다. DefaultKafkaProducerFactory는 트랜잭션 모드일 때 Producer를 캐시로 관리한다.
- DefaultKafkaProducerFactory의 createProducer() 메소드는 트랜잭션 모드일 때 cache에 존재하는 Producer를 리턴한다. 이 때 cache에 Producer가 존재하지 않는 경우에 새로운 Producer를 생성하며 이 때 Producer의 transactionId는 transactionIdPrefix+n이 된다.
n은 0부터 시작되며 새로운 Producer를 생성할 때마다 1씩 증가한다.
- KafkaTemplate이나 KafkaTransactionManager는 트랜잭션을 커밋하고, Producer를 close() 하는데 이 때 Producer를 캐시로 반환하다.

![KafkaTemplate]({{ site.url }}/assets/img/posts/spring-kafka-transaction/KafkaTemplate.png)

![Alt text]({{ site.url }}/assets/img/posts/spring-kafka-transaction/KafkaTransactionManager.png)

#### Code Example
``` java
    @Bean
    public ProducerFactory<Object, Object> kafkaProducerFactory() {
        DefaultKafkaProducerFactory<Object, Object> factory = new DefaultKafkaProducerFactory<>(config);

        // ...
        factory.setTransactionIdPrefix("transactionIdPrefix");
        return factory;
    }
    
    @Bean
    public KafkaTransactionManager<?, ?> kafkaTransactionManager(
        ProducerFactory<?, ?> kafkaProducerFactory) {
        KafkaTransactionManager<?, ?> kafkaTransactionManager = new KafkaTransactionManager<>(kafkaProducerFactory);
        // ...
        return kafkaTransactionManager;
    }
    
    @Bean
    public KafkaTemplate<?, ?> kafkaTemplate(
        ProducerFactory<Object, Object> kafkaProducerFactory) {
        KafkaTemplate<Object, Object> kafkaTemplate = new KafkaTemplate<>(
            kafkaProducerFactory);
       // ...
        return kafkaTemplate;
    }
```

### KafkaTransactionManager
- KafkaTransactionManager는 PlatformTransactionManager의 구현체이다. 생성자로 ProducerFactory를 주입받는다. ProducerFactory의 transactionCapable() 메소드의 리턴값은 true여야 한다.
- KafkaTransactionManager를 사용하면 스프링에서 지원하는 @Transactional 어노테이션이나 TransactionTemplate등을 사용해서 트랜잭션을 관리할 수 있다.
- KafkaTransactionManager에 의해 현재 Transaction이 진행중이면 KafkaTemplate은 현재 Transaction이 진행중인 KafkaProducer를 사용한다. 
- 단 이 때 주의해야 할 점은 KafkaTemplate과 KafkaTransactionManager가 같은 ProducerFactory를 사용해야 한다.


##### @Transactional
``` java
    @Transactional
    public void sample1(ProducerRecord<Long, String>... records) {
        for (ProducerRecord<Long, String> record : records) {
            kafkaTemplate.send(record);
        }
    }
```


![Alt text]({{ site.url }}/assets/img/posts/spring-kafka-transaction/@Transactional.png)

- KafkaTransactionManager은 @Transactional 메소드를 호출하기 전에 KafkaProducer의 beginTransaction()를 호출한다.
  - 그리고 TransactionSynchronizationManager에 KafkaResourceHolder를 등록한다. (KafkaResourceHolder는 트랜잭션이 시작 된 KafkaProducer를 가지고 있다)
- 트랜잭션 모드일 때, KafkaTemplate은 TransactionSynchronizationManager에 등록된 KafkaResourceHolder의 KafkaProducer를 사용한다.

> **Note**
> 2개 이상의 PlatformTransactionManager이 빈으로 등록된 경우에는 @Transactional의 `transactionManager` 속성을 지정해서 KafkaTransactionManager를 사용해서 트랜잭션을 생성하도록 해야한다.


##### KafkaTemplate Local Transactions
- KafkaTemplate의 executeInTransaction 메소드를 사용
- KafkaTransactionManger를 사용하지 않고 KafkaTemplate만을 사용해서 Kafka Transaction을 생성할 수 있다.
- You can use the KafkaTemplate to execute a series of operations within a local transaction.
- The argument in the callback is the template itself (this). If the callback exits normally, the transaction is committed; if an exception is thrown, the transaction is rolled-back.

```java    
    public void sample3(ProducerRecord<Long, String>... records) {
        kafkaTemplate.executeInTransaction(t -> {
            for (ProducerRecord<Long, String> record : records) {
                t.send(record);
            }
            return true;
        });
    }
```
> **Note**
> - 콜백 메소드의 인자는 KafkaTemplate(this)  이다.
> - KafkaTemplate의 executeInTransaction은 KafkaTransactionManager과는 무관하게 새로운 Kafka Transaction을 생성한다. 

##### @Transactional and KafkaTemplate Local Transaction
``` java
    @Transactional
    public void sample4(ProducerRecord<Long, String> record1, ProducerRecord<Long, String> record2) {
        kafkaTemplate.send(record1);
        kafkaTemplate.executeInTransaction(t -> {
            t.send(record2);
            return true;
        });
    }
``` 

![Alt text]({{ site.url }}/assets/img/posts/spring-kafka-transaction/KafkaTransactionManager2.png)

> **Note**
> KafkaTransactionManager에 의해서 Transaction이 시작된다. 
> 결과적으로 record1과 record2는 서로 다른 트랜잭션에 의해 토픽으로 보낸진다.

### Transactional Listener Container
- @KafkaListener 메소드를 실행하기 전에 Kafka Transaction을 시작하고 @KafkaListener 메소드가 Transaction 범위 내에서 실행되도록 할 수 있다.
- MessageListenerContainer에 KafkaTransactionManager를 설정할 수 있으며, 설정이 된 경우에는 @KafkaListener 메소드가 트랜잭션 범위 내에서 실행이 된다.
- @KafkaListener 메소드가 정상적으로 실행이 되고 종료가 된 경우에는 KafkaProducer의 sendOffsetsToTransaction() 메소드를 사용해서 offsets를 commit한다. 그리고 offsets를 commit하고 난 후에 Transaction을 commit한다.
- @KafkaListener 메소드에서 예외가 발생하는 경우 트랜잭션은 롤백된다. 그리고 Consumer의 position이 조정되어 트랜잭션이 롤백된 record를 다음 poll에서 다시 읽어드린다.

> **Note**
> - 배치로 처리하는 경우 Consumer.poll를 통해 가져온 모든 ConsumerRecord들이 하나의 트랜잭션 범위 내에서 실행이 되고, 그렇지 않은 경우에는 각각의 ConsumerRecord가 하나의 트랜잭션 범위 내에서 실행이 된다.
> - 예를 들어 Consumer.poll 메소드가 ConsumerRecord를 100개 리턴한다고 가정해보면, 배치인 경우에는 트랜잭션이 하나만 생성되고, 100개의 ConsumerRecord를 하나의 트랜잭션에서 처리하게 된다. 배치가 아닌 경우에는 각각의 ConsumerRecord 당 하나의 트랜잭션을 생성하기 때문에 100개의 트랜잭션이 생성이 된다.
> - 테스트 해 본 결과 ConsumerRecord 하나당 하나의 트랜잭션을 생성하면 성능이 많이 떨어진다. 따라서 Batch로 처리하는게 성능상 유리하다.

#### Code Example

``` java
    @Bean("kafkaListenerContainerFactory")
    public ConcurrentKafkaListenerContainerFactory<?, ?> containerFactory(ConcurrentKafkaListenerContainerFactoryConfigurer configurer,
                                                                          ConsumerFactory kafkaConsumerFactory,
                                                                          KafkaTransactionManager kafkaTransactionManager) {
        ConcurrentKafkaListenerContainerFactory<Object, Object> factory = new ConcurrentKafkaListenerContainerFactory<>();
        configurer.configure(factory, kafkaConsumerFactory);
        factory.getContainerProperties().setTransactionManager(kafkaTransactionManager);
        return factory;
    }
```

#### Example
- 배치인 경우
  - ConsumerRecord 각각 하나당 하나의 트랜잭션이 아니라 poll 호출당 하나의 트랜잭션을 생성한다.
  - 만약에 poll 호출을 통해서 ConsumerRecord를 100개 가져온다 하더라도 트랜잭션은 하나만 생성하며 100개의 ConsumerRecord를 하나의 트랜잭션에서 처리하게 된다.

![Alt text]({{ site.url }}/assets/img/posts/spring-kafka-transaction/batch.png)

#### Transaction Synchronization
- DB Transaction과 Kafka Transaction을 묶을 수는 없을까?
- ChainedKafkaTransactionManager를 사용하면 된다. ChainedKafkaTransactionManager는 ChainedTransactionManager의 하위 클래스이며 딱 하나의 KafkaTransactionManager를 가지고 있어야 한다.
- ChainedKafkaTransactionManager는 KafkaAwareTransactionManager 인터페이스를 구현하고 있다. 따라서 MessageListenerContainer에 ChainedKafkaTransactionManager를 설정하면, 
트랜잭션을 커밋하기 전에 sendOffsetsToTransaction 메소드를 이용해서 offset를 커밋한다.
- ChainedKafkaTransactionManager는 지정된 PlatformTransactionManager의 트랜잭션을 차례로 시작하고, 커밋을 역순으로 해준다.
- @KafkaListener 메소드를 호출하기 전에, DB Transaction, Kafka Transaction을  시작하고, @KafkaListener가 정상적으로 종료되면 Kafka Transaction, DB Transaction을 commit 할 수 있다.

![Alt text]({{ site.url }}/assets/img/posts/spring-kafka-transaction/chained.png)

> **Note**
> - Two Phase Commit을 지원하는게 아니라서 이미 다른 트랜잭션이 커밋된 상황에서 하나의 트랜잭션이 롤백 됐다고 해서 이미 커밋된 것들이 다시 롤백되지는 않는다. 따라서 가장 위험한 요소를 최초로 커밋/롤백 시도하도록 해야 한다.

