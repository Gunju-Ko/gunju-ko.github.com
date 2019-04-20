---
layout: post
title: "Spring Kafka Retry" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka, spring-kafka]
---

# Spring Kafka - Retrying

## Retrying Deliveries

Spring Kafka에는 Retry 기능이 존재한다. Retry 기능을 사용하면 listener 메소드는 `RetryingMessageListenerAdapter`를 통해서 호출이 된다. Retry 기능을 사용하려면 `RetryTemplate`과 `RecoveryCallback<Void>`(callback은 설정하지 않아도 된다)를 container factory에 설정하면 된다. `RetryTemplate`과 `RecoveryCallback<Void>`에 대해서 더 자세히 알고 싶다면 [spring-retry](https://github.com/spring-projects/spring-retry)를 참고하자.
만약에 recovery callback을 등록하지 않으면, retry 실패시 예외는 컨테이너에게 전달된다. 이 경우에는 `ErrorHandler`가 호출될 것이다. (ErrorHandler가 설정된 경우)

`RetryContext`는 `RecoveryCallback`으로 전달된다. 이 때 `RetryContext`의 내용은 listener의 타입에 따라 달라진다. `RetryContext`는 항상 `record` 속성을 가지고 있다. 이 때의 `record`는 처리에 실패한 record이다. listener의 타입이 acknowledging이거나 consumer aware이라면 추가적으로 `acknowledgement` 혹은 `consumer` 속성을 가지게 된다. 편의를 위해서 `RetryingAcknowledgingMessageListenerAdapter`는 속성에 대한 key를 상수로 제공한다.
* record 속성에 대한 key : `CONTEXT_RECORD`
* acknowledgement 속성에 대한 key : `CONTEXT_ACKNOWLEDGMENT`
* consumer 속성에 대한 key : `CONTEXT_CONSUMER`

Batch listener에 대해서는 retry adapter가 제공되지 않는다. 그 이유는 framework가 어느 여러개의 레코드들 중에서 어느 부분에서 에러가 발생했는지 알기 어렵기 때문이다. batch listener를 사용하면서 retry 기능을 사용하고 싶은 경우에는 `RetryTemplate`를 listener 메소드에서 직접 사용해야 한다.

아래의 코드는 `RetryingMessageListenerAdapter`의 onMessage 메소드 부분이다. listener의 메소드를 RetryTemplate의 execute의 콜백 메소드 안에서 호출하고 있다. (사실 여기서 delegate 객체의 타입은 `RecordMessagingMessageListenerAdapter` 이다. 그리고 `RecordMessagingMessageListenerAdapter`의 onMessage() 메소드에서 listener 메소드를 호출한다)

``` java
@Override
public void onMessage(final ConsumerRecord<K, V> record, final Acknowledgment acknowledgment,
		final Consumer<?, ?> consumer) {
	RetryState retryState = null;
	if (this.stateful) {
		retryState = new DefaultRetryState(record.topic() + "-" + record.partition() + "-" + record.offset());
	}
	getRetryTemplate().execute(context -> {
				context.setAttribute(CONTEXT_RECORD, record);
				switch (RetryingMessageListenerAdapter.this.delegateType) {
					case ACKNOWLEDGING_CONSUMER_AWARE:
						context.setAttribute(CONTEXT_ACKNOWLEDGMENT, acknowledgment);
						context.setAttribute(CONTEXT_CONSUMER, consumer);
						RetryingMessageListenerAdapter.this.delegate.onMessage(record, acknowledgment, consumer);
						break;
					case ACKNOWLEDGING:
						context.setAttribute(CONTEXT_ACKNOWLEDGMENT, acknowledgment);
						RetryingMessageListenerAdapter.this.delegate.onMessage(record, acknowledgment);
						break;
					case CONSUMER_AWARE:
						context.setAttribute(CONTEXT_CONSUMER, consumer);
						RetryingMessageListenerAdapter.this.delegate.onMessage(record, consumer);
						break;
					case SIMPLE:
						RetryingMessageListenerAdapter.this.delegate.onMessage(record);
				}
				return null;
			},
	
```

## Stateful Retry
Retry를 하는 경우 Consumer Thread가 중지될 수 있다. (`BackOffPolicy`를 사용하는 경우) 또한 retry를 하는 도중에 `Consumer.poll()` 메소드는 호출이 되지 않는다. Kafka는 Consumer의 상태를 결정하기 위해 2개의 속성을 사용한다. (`session.timeout.ms`, `max.poll.interval.ms`)
* `session.timeout.ms` :  Consumer가 살아있는지의 여부를 판단하기 위해서 사용되는 timeout 값이다. Consumer는 주기적으로 heartbeat를 브로커에게 보내 자신이 살아있음을 알린다. 만약 session.timeout.ms 시간초과가 만료되기 전에 heartbeats를 받지 못하면, 브로커는 해당 Consumer를 group에서 제거한다. 그리고 rebalance를 시작한다. 이 값은 `group.min.session.timeout.ms`값과 `group.max.session.timeout.ms` 허용 범위 내에 있어야 한다. 
  * default : 10000 
* `heartbeat.interval.ms` : Heartbeat를 보내는 주기이다. Heartbeat는 Consumer의 세션이 살아있음을 유지하는데 사용이 된다. 이 값은 `session.timeout.ms` 값보다는 반드시 작아야하며 일반적으로 `session.timeout.ms`값의 1/3 이하로 설정한다. 
  * default :  3000
  * `0.10.1.0` version부터는 heartbeat를 백그라운드 스레드에서 보낸다. 따라서 Consumer는 heartbeat에 영향을 크게 받지 않는다.
* `max.poll.interval.ms` : 이 속성값보다 더 긴 시간동안 `poll()` 메소드가 호출되지 않으면, Consumer가 죽었다고 생각되어 할당된 Partition이 revoke되며 rebalance가 이루어진다. 
  * default : 5분 

`poll()`를 통해서 가져온 record들을 처리하는데 너무 많은 시간이 걸려서 `max.poll.interval.ms`값을 초과하게 되면 할당된 파티션이 revoked되며 retry를 사용하는 경우 이런 문제가 빈번히 발생할 수 있다.
이 경우엔 Stateful Retry를 사용하면 된다. Stateful Retry는 `SeekToCurrentErrorHandler`와 함께 사용이 된다. 이 경우에 예외가 발생하면 예외는 컨테이너에게 전달되며, `SeekToCurrentErrorHandler`는 Consumer의 Position을 조정해서 다음 poll() 메소드에서 예외가 발생한 Record부터 다시 읽게 한다. 이 기능을 사용하면 `max.poll.interval.ms`값을 초과해서 partition이 revoke되는 것을 방지할 수 있다. 물론 각각의 record를 처리하는데, 너무 많은 시간이 걸려 `max.poll.interval.ms` 시간안에 다음 poll()를 호출하지 못한다면 여전히 partiton이 revoke된다. 그러나 적어도 retry로 인해서 poll()을 호출하지 못하는 문제는 막을 수 있다. 
Stateful Retry를 사용하려면 `RetryingMessageListenerAdapter`의 생성자를 통해서 stateful 값을 true로 세팅하면 된다. 혹은 container factory의 `statefulRetry`값을 `true`로 세팅하면 된다.

아래의 코드를 보면 Stateful Retry인 경우에 RetryState 객체를 생성해서 execute 메소드의 3번째 인자로 전달하고 있다. (Stateful Retry가 아닌 경우에는 null을 3번째 인자로 전달한다)

``` java
@Override
public void onMessage(final ConsumerRecord<K, V> record, final Acknowledgment acknowledgment,
		final Consumer<?, ?> consumer) {
	RetryState retryState = null;
	if (this.stateful) {
		retryState = new DefaultRetryState(record.topic() + "-" + record.partition() + "-" + record.offset());
	}
	getRetryTemplate().execute(context -> {
				context.setAttribute(CONTEXT_RECORD, record);
				switch (RetryingMessageListenerAdapter.this.delegateType) {
					// skip
				}
				return null;
			},
			getRecoveryCallback(), retryState);
}
```

아래의 코드는 `RetryTemplate`의 open 메소드인데, 2번째 인자로는 onMessage 메소드에서 생성한 RetryState 객체가 전달된다. RetryState가 null인 경우에는 doOpenInternal 메소드를 호출해서 새로운 RetryContext를 생성해서 리턴한다.
 RetryState가 null이 아닌 경우에는 cache에서 해당 state에 대한 RetryContext가 존재하는지 확인한다. cache에 state에 해당하는 RetryContext가 존재하면 그 RetryContext를 리턴하고 존재하지 않으면 새로운 RetryContext를 리턴한다. 이런 식으로 stateful retry를 사용하는 경우에는 cache에 RetryContext를 저장해놓는다.

``` java
/**
 * Delegate to the {@link RetryPolicy} having checked in the cache for an existing
 * value if the state is not null.
 *
 * @param state a {@link RetryState}
 * @param retryPolicy a {@link RetryPolicy} to delegate the context creation
 *
 * @return a retry context, either a new one or the one used last time the same state
 * was encountered
 */
protected RetryContext open(RetryPolicy retryPolicy, RetryState state) {

    if (state == null) {
    	return doOpenInternal(retryPolicy);
    }

    Object key = state.getKey();
    if (state.isForceRefresh()) {
    	return doOpenInternal(retryPolicy, state);
    }

    // If there is no cache hit we can avoid the possible expense of the
    // cache re-hydration.
    if (!this.retryContextCache.containsKey(key)) {
    	// The cache is only used if there is a failure.
    		return doOpenInternal(retryPolicy, state);
    	}

    	RetryContext context = this.retryContextCache.get(key);
    	if (context == null) {
    		if (this.retryContextCache.containsKey(key)) {
    			throw new RetryException(
    					"Inconsistent state for failed item: no history found. "
    							+ "Consider whether equals() or hashCode() for the item might be inconsistent, "
    							+ "or if you need to supply a better ItemKeyGenerator");
    		}
    		// The cache could have been expired in between calls to
    		// containsKey(), so we have to live with this:
    		return doOpenInternal(retryPolicy, state);
    	}

    	// Start with a clean slate for state that others may be inspecting
    	context.removeAttribute(RetryContext.CLOSED);
    	context.removeAttribute(RetryContext.EXHAUSTED);
    	context.removeAttribute(RetryContext.RECOVERED);
    	return context;

}
```

아래의 코드는 `retryContextCache`에 RetryContext를 저장하는 코드이다. retryCount가 1보다 큰데, 현재 cache에 RetryContext가 저장되어있지 않은 경우 예외가 발생한다. `registerThrowable()` 메소드는 `retryCallback.doWithRetry(context)`에서 예외가 발생한 경우에 호출이 된다.

``` java
protected void registerThrowable(RetryPolicy retryPolicy, RetryState state,
		RetryContext context, Throwable e) {
	retryPolicy.registerThrowable(context, e);
	registerContext(context, state);
}

private void registerContext(RetryContext context, RetryState state) {
	if (state != null) {
		Object key = state.getKey();
		if (key != null) {
			if (context.getRetryCount() > 1
					&& !this.retryContextCache.containsKey(key)) {
				throw new RetryException(
						"Inconsistent state for failed item key: cache key has changed. "
								+ "Consider whether equals() or hashCode() for the key might be inconsistent, "
								+ "or if you need to supply a better key");
			}
			this.retryContextCache.put(key, context);
		}
	}
}
```

Stateful Retry의 경우에는 retryCallback 실행 중 예외가 발생하는 경우 해당 예외를 throw한다. (재시도를 할 수 있음에도 불구하고 재시도를 하지 않고 예외를 전파한다)
따라서 retryCallback을 실행하는 도중에 예외가 발생하는 경우 해당 예외가 KafkaMessageListenerContainer에게 까지 전파가 된다. 만약에 SeekToErrorHandler를 등록했다면 에러가 발생한 레코드의 오프셋으로 Consumer의 포지션을 변경하여 예외가 발생한 레코드를 다시 읽게 할 것이다. 

Stateful retry이 아닌 경우에는 Retry가 가능할 때까지 재시도를 하고, 재시도를 했음해도 불구하고 실패한 경우에는 recoveryCallback을 호출한다. 반면에 Statefult Retry는 재시도가 가능함에도 불구하고 예외를 던진다.

이런 구현 덕분에 Retry로 인해 poll이 호출되지 않는 문제를 막을 수 있다. 하나 주의해야 할 점은 RecoveryCallback에서 예외를 아주 잘 처리해야 한다는 점이다. 만약 RecoveryCallback에서 예외가 발생하면 예외가 다시 컨테이너에게 전달된다. 그러면 다시 컨슈머의 포지션의 값을 변경될 것이다. 이 경우에는 무한루프에 빠질 수 있으므로 조심해야 한다. 다행히도 이 경우에는 RetryContextCache에 실패한 레코드에 대한 RetryContext가 남아있기 때문에 재시도는 하지 않고 다시 RecoveryCallback이 실행된다.


## 추가사항
* RecoveryCallback에서 예외가 발생하는 경우 Container에게 예외가 전파될 것인가? 
  * 테스트 해 본 결과 예외가 Container에게 전파된다.
  * 따라서 RecoveryCallback에서 예외가 발생하는 경우에는 ErrorHandler가 호출된다.
* stateful Retry를 사용하는 경우 errorHandler를 `SeekToCurrentErrorHandler`로 따로 설정해주어야 하는가?
  * 따로 설정을 해주어야 한다. stateful retry를 한다 하더라도 errorHandler가 자동으로 `SeekToCurrentErrorHandler`로 설정되는게 아니다.
* stateful retry의 경우에도 Recovery Callback이 호출되는가?
  * 마찬가지로 retry가 전부 실패하는 경우에 호출이 된다. 

## Code Sample

[spring-kafka-sample](https://github.com/Gunju-Ko/spring-kafka-sample) 해당 repository에서 retry 관련 코드를 보실 수 있습니다.
