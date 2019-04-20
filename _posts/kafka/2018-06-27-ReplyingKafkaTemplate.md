---
layout: post
title: "Replying KafkaTemplate" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka, spring-kafka]
---

# Replying KafkaTemplate

ReplyingKafkaTemplate은 KafkaTemplate의 하위 클래스이다. ReplyingKafkaTemplate는 요청 / 응답 semantics을 제공한다. ReplyingKafkaTemplate는 sendAndReceive(ProducerRecord<K, V> record); 메소드를 추가로 가진다.

``` java
RequestReplyFuture<K, V, R> sendAndReceive(ProducerRecord<K, V> record);
```

리턴값은 RequestReplyFuture이다. RequestReplyFuture는 ListenableFuture의 구현체이다. ListenableFuture는 비동기적으로 결과 혹은 예외가 채워진다. 또한 RequestReplyFuture는 sendFuture라는 속성을 가진다. sendFuture는 KafkaTemplate.send()의 결과이다. 따라서 sendFuture를 사용해서 send 연산의 결과를 확인할 수 있다.

아래는 RequestReplyFuture 클래스의 코드이다.

``` java
public class RequestReplyFuture<K, V, R> extends SettableListenableFuture<ConsumerRecord<K, R>> {

	private volatile ListenableFuture<SendResult<K, V>> sendFuture;

	public RequestReplyFuture() {
		super();
	}

	protected void setSendFuture(ListenableFuture<SendResult<K, V>> sendFuture) {
		this.sendFuture = sendFuture;
	}

	public ListenableFuture<SendResult<K, V>> getSendFuture() {
		return this.sendFuture;
	}

}
```

아래는 간단한 샘플 예제이다.

``` java
@SpringBootApplication
public class KRequestingApplication {

    public static void main(String[] args) {
        SpringApplication.run(KRequestingApplication.class, args).close();
    }

    @Bean
    public ApplicationRunner runner(ReplyingKafkaTemplate<String, String, String> template) {
        return args -> {
            ProducerRecord<String, String> record = new ProducerRecord<>("kRequests", "foo");
            record.headers().add(new RecordHeader(KafkaHeaders.REPLY_TOPIC, "kReplies".getBytes()));
            RequestReplyFuture<String, String, String> replyFuture = template.sendAndReceive(record);
            SendResult<String, String> sendResult = replyFuture.getSendFuture().get();
            System.out.println("Sent ok: " + sendResult.getRecordMetadata());
            ConsumerRecord<String, String> consumerRecord = replyFuture.get();
            System.out.println("Return value: " + consumerRecord.value());
        };
    }

    @Bean
    public ReplyingKafkaTemplate<String, String, String> kafkaTemplate(
            ProducerFactory<String, String> pf,
            KafkaMessageListenerContainer<String, String> replyContainer) {
        return new ReplyingKafkaTemplate<>(pf, replyContainer);
    }

    @Bean
    public KafkaMessageListenerContainer<String, String> replyContainer(
            ConsumerFactory<String, String> cf) {
        ContainerProperties containerProperties = new ContainerProperties("kReplies");
        return new KafkaMessageListenerContainer<>(cf, containerProperties);
    }

    @Bean
    public NewTopic kRequests() {
        return new NewTopic("kRequests", 10, (short) 2);
    }

    @Bean
    public NewTopic kReplies() {
        return new NewTopic("kReplies", 10, (short) 2);
    }

}
```

ProducerRecord에는 사용자 코드에 의해서 REPLY_TOPIC 헤더가 세팅된다. 뿐만 아니라 ReplyKafkaTemplate은 `KafkaHeaders.CORRELATION_ID` 헤더를 세팅한다. 이 헤더는 ProducerRecord를 구분하기 위함이며 기본 구현은 UUID로 이 헤더값을 채운다. 컨슈머쪽 코드에서도 `KafkaHeaders.CORRELATION_ID` 헤더를 이용해서 특정 레코드에 대한 응답임을 알려야한다. 

아래는 @KafkaListener를 이용해서 응답하는 코드이다.

``` java
@SpringBootApplication
public class KReplyingApplication {

    public static void main(String[] args) {
        SpringApplication.run(KReplyingApplication.class, args);
    }

    @KafkaListener(id="server", topics = "kRequests")
    @SendTo // use default replyTo expression
    public String listen(String in) {
        System.out.println("Server received: " + in);
        return in.toUpperCase();
    }

    @Bean
    public NewTopic kRequests() {
        return new NewTopic("kRequests", 10, (short) 2);
    }

    @Bean // not required if Jackson is on the classpath
    public MessagingMessageConverter simpleMapperConverter() {
        MessagingMessageConverter messagingMessageConverter = new MessagingMessageConverter();
        messagingMessageConverter.setHeaderMapper(new SimpleKafkaHeaderMapper());
        return messagingMessageConverter;
    }

}
```

위 처럼 @KafkaListener와 @SendTo를 사용하면 아주 쉽게 응답을 보낼 수 있다. @SendTo에 value 속성을 세팅하지 않은 경우에는 KafkaHeaders.REPLY_TOPIC의 값이 replyTopic이 된다. 또한 replyTopic에 응답을 보내기전에 ConsumerRecord의 CORRELATION_ID 헤더값을 꺼내서 응답 ProducerRecord 헤더에 넣는다. 이를 통해 특정 레코드에 대한 응답임을 알 수 있게 된다. 

> 중요
> * 여러 클라이언트 인스턴스를 실행시킬 경우 각 인스턴스마다 서로 다른 전용 reply topic을 가지고 있어야 한다. 아니면 클라이언트 인스턴스마다 컨슈머의 group.id를 다르게 세팅해야한다. 또 다른 대안책으로는 KafkaHeaders.REPLY_PARTITION 헤더를 사용하는 것이다. KafkaHeaders.REPLY_PARTITION를 통해서 각 인스턴스마다 전용 reply topic 파티션을 지정해야한다. 응답 서버쪽에서는 KafkaHeaders.REPLY_PARTITION 헤더와 KafkaHeaders.REPLY_TOPIC 헤더를 사용해서 올바른 토픽 파티션으로 응답을 보내야한다. @KafkaListener와 @SendTo를 사용하면 이를 쉽게 할 수 있다. KafkaHeader.REPLY_PARTITION을 사용하는 경우 reply 컨테이너는 카프카 그룹 management 기능을 사용하면 안되고 고정된 파티션을 읽도록 설정해야 한다. (이는 ContainerProperties 생성자 파라미터의 TopicPartitionInitialOffset를 통해 가능하다)

아래는 참고 사항이다.

> DefaultKafkaHeaderMapper
> *  DefaultKafkaHeaderMapper를 사용하려면 Jackson이 클래스 패스에 존재해야 한다. 만약에 존재하지 않는다면 MessageConverter가 HeaderMapper를 가지지 않는다. 이런 경우에는 위에 예제와 같이 SimpleKafkaHeaderMapper를 사용하도록 MessagingMessageConverter를 구성해야 한다.

## 코드 분석

위에서 ReplyingKafkaTemplate는 sendAndReceive라는 메소드를 추가적으로 구현하고 있다고 했다. sendAndReceive 코드는 아래와 같다.

#### sendAndReceive

``` java
@Override
public RequestReplyFuture<K, V, R> sendAndReceive(ProducerRecord<K, V> record) {
	Assert.state(this.running, "Template has not been start()ed"); // NOSONAR (sync)
	// 1
	CorrelationKey correlationId = createCorrelationId(record);
	record.headers().add(new RecordHeader(KafkaHeaders.CORRELATION_ID, correlationId.getCorrelationId()));
	// skip
	
	// 2
	TemplateRequestReplyFuture<K, V, R> future = new TemplateRequestReplyFuture<>();
	this.futures.put(correlationId, future);
	try {
		// 3
		future.setSendFuture(send(record));
	}
	catch (Exception e) {
		this.futures.remove(correlationId);
		throw new KafkaException("Send failed", e);
	}
	// 4
	this.scheduler.schedule(() -> {
		RequestReplyFuture<K, V, R> removed = this.futures.remove(correlationId);
		if (removed != null) {
			if (this.logger.isWarnEnabled()) {
				this.logger.warn("Reply timed out for: " + record + " with correlationId: " + correlationId);
			}
			removed.setException(new KafkaException("Reply timed out"));
		}
	}, Instant.now().plusMillis(this.replyTimeout));
	return future;
}

/**
 * Subclasses can override this to generate custom correlation ids.
 * The default implementation is a 16 byte representation of a UUID.
 * @param record the record.
 * @return the key.
 */
protected CorrelationKey createCorrelationId(ProducerRecord<K, V> record) {
	UUID uuid = UUID.randomUUID();
	byte[] bytes = new byte[16];
	ByteBuffer bb = ByteBuffer.wrap(bytes);
	bb.putLong(uuid.getMostSignificantBits());
	bb.putLong(uuid.getLeastSignificantBits());
	return new CorrelationKey(bytes);
}
```

이 메소드는 아래와 순서로 실행된다.

1. CorrelationKey correlationId 객체를 생성한다. 기본 구현은 UUID로 CorrelationKey 객체를 생성한다. CorrelationKey 객체는 응답 레코드를 구별하기 위해서 사용된다. correlationId는 레코드 헤더에 추가된다. 나중에 이 레코드에 대한 응답을 보낼때는 반드시 correlationId 값을 꺼내서 응답 레코드 헤더에 추가해줘야 한다.
2. TemplateRequestReplyFuture 객체를 생성한 뒤에 ConcurrentMap<CorrelationKey, RequestReplyFuture<K, V, R>> futures에 추가한다. 이 때 key는 correlationId가 된다.
3. 부모 클래스의 send 메소드를 호출한다. 그리고 send 메소드의 호출 결과를 TemplateRequestReplyFuture 객체의 sendFuture 필드에 세팅한다. 따라서 KafkaTemplate.send 결과를 알고 싶으면 TemplateRequestReplyFuture 객체의 sendFuture 필드를 사용하면 된다.
4. TaskScheduler#schedule() 메소드를 사용해서 일정 시간(timeout)이 지난뒤에 콜백 메소드가 실행되도록 한다. 콜백 메소드는 futures 맵에서 correlationId를 키로해서 매핑되는 객체를 제거한다. 만약에 제거된 RequestReplyFuture 객체가 존재한다면, RequestReplyFuture#setException 메소드를 이용해서 응답 시간이 지났음을 세팅한다. 

#### onMessage

ReplyingKafkaTemplate은 전달받은 GenericMessageListenerContainer의 messageListener를 자기자신으로 세팅한다. 그리고 ReplyingKafkaTemplate#start 메소드에서 GenericMessageListenerContainer#start() 메소드를 호출해 messageContainer를 실행한다. 

``` java
@Override
public void onMessage(List<ConsumerRecord<K, R>> data) {
	data.forEach(record -> {
		// 1
		Iterator<Header> iterator = record.headers().iterator();
		CorrelationKey correlationId = null;
		while (correlationId == null && iterator.hasNext()) {
			Header next = iterator.next();
			if (next.key().equals(KafkaHeaders.CORRELATION_ID)) {
				correlationId = new CorrelationKey(next.value());
			}
		}
		if (correlationId == null) {
			// 2
			this.logger.error("No correlationId found in reply: " + record
					+ " - to use request/reply semantics, the responding server must return the correlation id "
					+ " in the '" + KafkaHeaders.CORRELATION_ID + "' header");
		}
		else {
			// 3
			RequestReplyFuture<K, V, R> future = this.futures.remove(correlationId);
			if (future == null) {
				this.logger.error("No pending reply: " + record + " with correlationId: "
						+ correlationId + ", perhaps timed out");
			}
			else {
				if (this.logger.isDebugEnabled()) {
					this.logger.debug("Received: " + record + " with correlationId: " + correlationId);
				}
				future.set(record);
			}
		}
	});
}
```

위의 메소드는 Reply 토픽에서 응답 레코드를 받기 위한 메소드이다. 위 메소드는 GenericMessageListener#onMessage 메소드를 구현한 것이다. 이 메소드는 아래와 같은 순서로 동작한다.

1. 헤더에서 correlationId를 구한다. correlationId는 레코드가 어떤 레코드에 대한 응답인지를 구별하기 위해서 사용된다. 
2. correlationId가 널이면 로깅을 남기고 끝난다.
3. correlationId가 널이 아니라면, correlationId를 키로해서 futures 맵에서 TemplateRequestReplyFuture 객체를 구한다. 만약에 correlationId 키에 해당하는 TemplateRequestReplyFuture 객체가 없다면, 로깅을 남기고 끝난다. (이 경우는 timeout으로 인해서 TemplateRequestReplyFuture#setException이 호출된 경우일 것이다) 만약에 correlationId 키에 해당하는 TemplateRequestReplyFuture 객체가 있다면 TemplateRequestReplyFuture#set 메소드를 이용해서 결과를 세팅한다. 


