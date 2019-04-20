---
layout: post
title: "ProductionExceptionHandler" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka]
---

# ProductionExceptionHandler

ProductionExceptionHandler를 사용하면 브로커에 메시지를 보낼때 예외가 발생하는 경우 이를 처리할 수 있다. 기본 구현체는 DefaultProductionExceptionHandler로 항상 FAIL을 리턴한다. FAIL을 리턴하는 경우에는 Stream이 셧다운된다.

ProductionExceptionHandler의 handle 메소드에서는 FAIL 또는 CONTINUE를 리턴할 수 있다. 보통은 어떤 Exception이 발생했느냐에 따라서 리턴되는 값이 결정된다. FAIL을 리턴하는 경우 Stream은 셧다운된다. 반면에 CONTINUE를 리턴하는 경우에는 예외를 무시하고 처리를 계속 진행한다. 

ExceptionHandler에서 RecordTooLargeException를 무시하고 싶은 경우에는 아래와 같이 구현하면 된다.

``` java
import java.util.Properties;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.common.errors.RecordTooLargeException;
import org.apache.kafka.streams.errors.ProductionExceptionHandler;
import org.apache.kafka.streams.errors.ProductionExceptionHandler.ProductionExceptionHandlerResponse;

class IgnoreRecordTooLargeHandler implements ProductionExceptionHandler {
    public void configure(Map&lt;String, Object&gt; config) {}

    public ProductionExceptionHandlerResponse handle(final ProducerRecord&lt;byte[], byte[]&gt; record,
                                                     final Exception exception) {
        if (exception instanceof RecordTooLargeException) {
            return ProductionExceptionHandlerResponse.CONTINUE;
        } else {
            return ProductionExceptionHandlerResponse.FAIL;
        }
    }
}

Properties settings = new Properties();

// other various kafka streams settings, e.g. bootstrap servers, application id, etc

settings.put(StreamsConfig.DEFAULT_PRODUCTION_EXCEPTION_HANDLER_CLASS_CONFIG,
             IgnoreRecordTooLargeHandler.class);

```


>- Note
> * KafkaProducer.send는 비동기로 실행이 된다. KafkaProducer.send 메소드는 레코드를 레코드 버퍼에 저장한 뒤에 바로 리턴되며, 레코드 버퍼에 저장된 레코드들은 별도의 스레드에서 카프카로 전송된다. ProductionExceptionHandler는 비동기 실행 중 발생하는 예외를 처리하기 위해서 사용된다. 즉 버퍼에 저장된 레코드들을 카프카로 전송할 때 발생하는 예외를 처리하기 위해서 사용된다. 레코드를 레코드 버퍼에 저장하는 과정에서 발생하는 예외는 ProductionExceptionHandler로 처리할 수는 없다.

### Fatal exception

브로커에 메시지를 보낼때 발생한 예외가 아래와 같은 경우에는 ProductionExceptionHandler에서 처리되지 않으며, Streams를 셧다운시킨다. 

* ProducerFencedException
* AuthenticationException
* AuthorizationException
* SecurityDisabledException
* InvalidTopicException
* UnknownServerException
* SerializationException
* OffsetMetadataTooLarge
* IllegalStateException

