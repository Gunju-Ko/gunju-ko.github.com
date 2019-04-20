---
layout: post
title: "DeserializationExceptionHandler" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka, kafka-stream]
---

## DeserializationExceptionHandler

DeserializationExceptionHandler를 사용하면 레코드를 deserialize할 때 발생하는 예외를 처리할 수 있다. DeserializationExceptionHandler 인터페이스의 구현체는 레코드와 발생한 예외를 참고해서 FAIL 혹은 CONTINUE를 리턴해야 한다.

-  FAIL을 리턴하는 경우 : Streams이 shutdown 되어야하는 경우
- CONTINUE을 리턴하는 경우 : Streams이 예외를 무시하고 처리를 계속해야하는 경우 

기본 구현 클래스는 LogAndFailExceptionHandler이다. 그리고 Kafka Streams는 아래와 같은 ExceptionHandler를 제공한다. 

- LogAndContinueExaceptionHandler : deserialization 예외를 로깅한다. 그리고 CONTINUE를 리턴한다. 따라서 KafkaStream은 deserialize 할 때 예외가 발생한 레코드는 스킵하고, 다음 레코드를 계속해서 처리한다. 
- LogAndFailExceptionHandler : deserialization 예외를 로깅한다. 그리고 FAIL를 리턴한다. 따라서 KafkaStream은 shutdown되어 레코드 처리를 중단한다.

필요에 따라 DeserializationExceptionHandler 인터페이스 구현체를 직접 구현할 수도 있다. 커스터마이징 된 ExceptionHandler 구현 예제가 궁금한다면 [Failure and exception handling](https://docs.confluent.io/current/streams/faq.html#streams-faq-failure-handling-deserialization-errors)에서 확인하길 바란다.

아래는 DeserializationExceptionHandler 인터페이스이다.

``` java
public interface DeserializationExceptionHandler extends Configurable {
    /**
     * Inspect a record and the exception received.
     * @param context processor context
     * @param record record that failed deserialization
     * @param exception the actual exception
     */
    DeserializationHandlerResponse handle(final ProcessorContext context,
                                          final ConsumerRecord<byte[], byte[]> record,
                                          final Exception exception);

    /**
     * Enumeration that describes the response from the exception handler.
     */
    enum DeserializationHandlerResponse {
        /* continue with processing */
        CONTINUE(0, "CONTINUE"),
        /* fail the processing and stop */
        FAIL(1, "FAIL");

        /** an english description of the api--this is for debugging and can change */
        public final String name;

        /** the permanent and immutable id of an API--this can't change ever */
        public final int id;

        DeserializationHandlerResponse(final int id, final String name) {
            this.id = id;
            this.name = name;
        }
    }

}
```

### default.deserialization.exception.handler

KafkaStream에서 사용할 DeserializationExceptionHandler 구현체는 "default.deserialization.exception.handler" 설정을 통해 설정할 수 있다. 다음은 간단한 설정 예제이다. 아래의 코드는 LogAndContinueExaceptionHandler를 DeserializationExceptionHandler로 사용한다.

``` java
public StreamsConfig getStreamsConfig(EdaProperties edaProperties, StreamProperties streamProperties) {
    Map<String, Object> props = new HashMap<>();
    
    // configure
    props.put(StreamsConfig.DEFAULT_DESERIALIZATION_EXCEPTION_HANDLER_CLASS_CONFIG,
              LogAndContinueExceptionHandler.class);
    ...
        
    return new StreamsConfig(props);
}
```

### RecordDeserializer

Task는 처리해야할 레코드를 RecordQueue에 넣는다. RecordQueue는 레코드를 큐에 넣기 전에 RecordDeserialzier를 사용해서 deserialize한다. 아래는 RecordDeserializer의 deserialize 메소드이다. 

``` java
ConsumerRecord<Object, Object> deserialize(final ProcessorContext processorContext,
                                           final ConsumerRecord<byte[], byte[]> rawRecord) {

    try {
        return new ConsumerRecord<>(
            rawRecord.topic(),
            rawRecord.partition(),
            rawRecord.offset(),
            rawRecord.timestamp(),
            TimestampType.CREATE_TIME,
            rawRecord.checksum(),
            rawRecord.serializedKeySize(),
            rawRecord.serializedValueSize(),
            sourceNode.deserializeKey(rawRecord.topic(), rawRecord.headers(), rawRecord.key()),
            sourceNode.deserializeValue(rawRecord.topic(), rawRecord.headers(), rawRecord.value()));
    } catch (final Exception deserializationException) {
        final DeserializationExceptionHandler.DeserializationHandlerResponse response;
        try {
            response = deserializationExceptionHandler.handle(processorContext, rawRecord, deserializationException);
        } catch (final Exception fatalUserException) {
            log.error("Deserialization error callback failed after deserialization error for record {}",
                      rawRecord,
                      deserializationException);
            throw new StreamsException("Fatal user code error in deserialization error callback", fatalUserException);
        }

        if (response == DeserializationExceptionHandler.DeserializationHandlerResponse.FAIL) {
            throw new StreamsException("Deserialization exception handler is set to fail upon" +
                " a deserialization error. If you would rather have the streaming pipeline" +
                " continue after a deserialization error, please set the " +
                DEFAULT_DESERIALIZATION_EXCEPTION_HANDLER_CLASS_CONFIG + " appropriately.",
                deserializationException);
        } else {
            sourceNode.nodeMetrics.sourceNodeSkippedDueToDeserializationError.record();
        }
    }
    return null;
 }
```

위의 코드를 보면 알 수 있듯이 deserialize시 예외가 발생하면 DeserializationExceptionHandler#handle 메소드를 호출한다. 그리고 DeserializationExceptionHandler#handle 메소드가 FAIL을 리턴하는 경우 StreamsException이 발생한다. 그리고 이 경우엔 Streams가 shutdown된다. DeserializationExceptionHandler#handle 메소드가 CONTINUE를 리턴하는 경우는 널을 리턴한다. RecordQueue는 RecordDeserializer#deserialize의 리턴값이 널인 경우에는 해당 레코드를 스킵한다.



