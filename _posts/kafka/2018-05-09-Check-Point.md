---
layout: post
title: "Check Point" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka, kafka-stream]
---

# Checkpoint

## CheckPoint
Kafka Streams 어플리케이션이 State Store를 초기화 할 때, 체크포인트 파일이 존재하는지 확인한다. 체크포인트 파일은 State Store에 플러쉬되고 change-log 토픽에 쓰여진 가장 최신의 records에 대한 오프셋들을 저장하고 있다. 만약에 체크포인트 파일이 존재하고 특정 change-log 파티션에 대한 체크 포인트가 존재하면, 그 State Store의 복구는 체크포인트 파일에 존재하는 오프셋부터 시작이 된다. 만약에 체크포인트가 존재하지 않으면 복구는 가장 처음 오프셋부터 시작된다. 

## Writing a checkpoint (EOS)
체크포인트 파일에 write하는 행위는 ProcessorStateManager#checkpoint(final Map<TopicPartition, Long> ackedOffsets) 메소드를 통해서 이루어진다. 보통 checkpoint 메소드는 Task를 commit할 때 호출이 된다. 

``` java
// StreamTask#commit(final boolean startNewTransaction)
new Runnable() {
    @Override
    public void run() {
        flushState();
        if (!eosEnabled) {
            stateMgr.checkpoint(recordCollectorOffsets());
        }
        commitOffsets(startNewTransaction);
    }
}
```


위의 코드는 StreamTask#commit 메소드의 일부분이다. 코드를 보면 알겠지만 exactly once semantic인 경우에는 checkpoint 메소드를 호출하지 않는다. 따라서 eos인 경우에는 active task가 commit 되는 순간마다 체크 포인트를 기록하지 않는다.

``` java
// StandByTask#commit()
@Override
public void commit() {
    log.trace("Committing");
    flushAndCheckpointState();
    updateOffsetLimits();
}

private void flushAndCheckpointState() {
    stateMgr.flush();
    stateMgr.checkpoint(Collections.<TopicPartition, Long>emptyMap());
}
```

위의 메소드는 StandByTask#commit() 메소드이다. StandByTask의 경우 commit을 하면 항상 checkpoint에 오프셋을 기록한다. 이 때 exactly once semantic 여부는 중요하지 않다. 즉 StandByTask#commit 메소드가 호출될 때마다 checkpoint를 기록한다.

마지막으로는 ProcessorStateManager#close(final Map<TopicPartition, Long> ackedOffsets) 메소드이다. 이 메소드는 어플리케이션이 종료될 때 호출이 된다. 더 정확히 말하자면 StreamTask#close 혹은 StandybyTask#close가 호출 될 때이다. StreamTask#close가 호출이 되면 changelog 토픽에 전송된 데이터의 마지막 오프셋을 체크포인트 파일에 기록한다.

``` java

// ProcessorStateManager#close

public void close(final Map<TopicPartition, Long> ackedOffsets) throws ProcessorStateException {
    ProcessorStateException firstException = null;
    // attempting to close the stores, just in case they
    // are not closed by a ProcessorNode yet
    if (!stores.isEmpty()) {
        log.debug("Closing its state manager and all the registered state stores");
        for (final StateStore store : stores.values()) {
            log.debug("Closing storage engine {}", store.name());
            try {
                store.close();
            } catch (final Exception e) {
                if (firstException == null) {
                    firstException = new ProcessorStateException(String.format("%sFailed to close state store %s", logPrefix, store.name()), e);
                }
                log.error("Failed to close state store {}: ", store.name(), e);
            }
        }

        if (ackedOffsets != null) {
	        // 체크포인트 기록
            checkpoint(ackedOffsets);
        }
        stores.clear();
    }

    if (firstException != null) {
        throw firstException;
    }
}
```

마지막으로 아래의 코드는 ProcessorStateManager의 생성자 부분이다. 아래의 코드를 보면 알 수 있지만, checkpoint 파일을 읽고 그 결과를 checkpointableOffsets에 저장한다. 그리고 exactly_once_semantic인 경우에는 checkpoint 파일을 삭제하고 checkpoint 필드를 널로 한다.

``` java
public ProcessorStateManager(final TaskId taskId,
                             final Collection<TopicPartition> sources,
                             final boolean isStandby,
                             final StateDirectory stateDirectory,
                             final Map<String, String> storeToChangelogTopic,
                             final ChangelogReader changelogReader,
                             final boolean eosEnabled,
                             final LogContext logContext) throws IOException {
    super(stateDirectory.directoryForTask(taskId));

	// 생략...

    // load the checkpoint information
    checkpointableOffsets.putAll(checkpoint.read());

    if (eosEnabled) {
        // delete the checkpoint file after finish loading its stored offsets
        checkpoint.delete();
        checkpoint = null;
    }
}
```

## 정리
코드를 종합해서 정리해봤을 경우에 eos를 사용하면 아래와 같이 동작할 것으로 예상된다.
* EOS인 경우에만, ProcessorStateManager를 생성하는 순간에 체크포인트 파일을 삭제한다.  Task를 생성하는 순간에 ProcessorStateManager를 생성하며, ProcessorStateManager는 해당 Task의 체크포인트 파일을 삭제한다. 물론 EOS인 경우에만 해당한다.
* EOS인 경우에는 StreamTask는 커밋되는 순간마다 체크포인트를 기록하지 않는다. EOS가 아니라면 StreamTask.commit를 호출하는 순간마다 체크포인트를 기록한다.
* StandyTask는 커밋되는 순간마다 체크포인트를 기록한다.
* StreamTask, StandyTask가 close되는 순간에 체크포인트를 기록한다. 좀 더 정확히 말하면 Task.close(true)인 경우, 즉 clean한 close인 경우에만 체크포인트를 기록한다. 따라서 비정상적으로 어플리케이션이 종료된 경우에는 체크포인트 파일이 없다. 결과적으로 비정상적으로 어플리케이션이 종료되고 난 뒤에 재시작한다면, 체크포인트 파일이 없기 때문에 StateStore를 처음부터 복구하려고 할 것이다.

## More
위에서 보면 EOS인 경우에 ProcessorStateManager를 생성하는 순간에 체크포인트 파일을 삭제한다. 또한 EOS인 경우에는 StreamTask는 커밋되는 순간마다 체크포인트를 기록하지 않는다. 이렇게 코딩을 한 이유가 궁금했고 문서를 찾아보았다. [Uncleanly shutting down a task](https://cwiki.apache.org/confluence/display/KAFKA/KIP-129%3A+Streams+Exactly-Once+Semantics)를 보면 그 이유를 할 수 있다. 아래에서 간단하게 설명을 하도록 하겠다.

### Uncleanly shutting down a task
로컬 StateStore에 적용된 업데이트는 Kafka Transaction이 롤백이 되더라도 롤백할 수 없다. 따라서 트랜잭션이 롤백된 경우엔 task를 재시작하는 할 때 로컬 State Store를 다시 사용하지 않고 changelog 토픽으로부터 State Store를 다시 복구시킨다. Kafka Transaction이 롤백되는 경우에는 Task가 비정상적으로 종료되었음을 나타내고, chagelog 토픽으로부터 State Store를 복원한다. 이 때 changelog 토픽에서 트랜잭션이 commit된 레코드만 복구를 하며, abort된 레코드는 복구가 되지 않는다.

## 출처
* [WIKI - Add State Store Checkpoint Interval Configuration](https://cwiki.apache.org/confluence/display/KAFKA/KIP-116%3A+Add+State+Store+Checkpoint+Interval+Configuration)
* [WIKI - Exactly Once Semantics](https://cwiki.apache.org/confluence/display/KAFKA/KIP-129%3A+Streams+Exactly-Once+Semantics)
* [Kafka Streams Exactly Once Design](https://docs.google.com/document/d/1pGZ8xtOOyGwDYgH5vA6h19zOMMaduFK1DAB8_gBYA2c/edit#)

