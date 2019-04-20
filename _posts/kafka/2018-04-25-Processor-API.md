---
layout: post
title: "Processor API" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka, kafka-stream]
---

# Processor API

본 글은 [Processor API](https://kafka.apache.org/11/documentation/streams/developer-guide/processor-api.html)를 번역하고 정리한 글이다. 따라서 더 자세한 내용이나 원본 글을 보고 싶은 경우에는 공식 홈페이지를 참고하길 바란다.
Processor API를 사용하면 커스텀 프로세서를 정의하고 연결할 수 있으며 State Store와 상호작용할 수도 있다. Processor API를 통해서 임의의 스트림 프로세서를 정의할 수 있으며 이 프로세서를 관련된 State Store와 연관시켜 Processor Topology를 구성할 수 있다.
Processor API는 Stateless, Stateful 오퍼레이션을 구현하기 위해 사용될 수 있다. Stateful 오퍼레이션은 State Store를 사용해서 구현할 수 있다. 

> Tip
> * DSL과 Processor API를 함께 사용할 수 있다. DSL의 편리함과 Processor API의 유연함을 결합할 수 있다. 더 자세한 내용은 [ Applying processors and transformers ](https://docs.confluent.io/current/streams/developer-guide/dsl-api.html#streams-developer-guide-dsl-process)를 참고해라

## Defining a Stream Processor

스트림 프로세서는 Processor Topology 상에서 하나의 노드로 하나의 처리 단계이다. Processor API를 사용하면 임의의 스트림 프로세서를 정의할 수 있으며 프로세서를 연관된 State Store와 연결할 수 있다. 스트림 프로세서는 Processor 인터페이스를 구현함으로써 정의할 수 있다. Processor 인터페이스는 process() API 메소드를 제공한다. process() 메소드는 각 record 별로 호출된다. 
Processor 인터페이스는 init() 메소드도 가지고 있다. init() 메소드는 Kafka Streams 라이브러리에 의해 호출되며 Task를 생성하는 단계에서 호출이 된다. Processor 객체는 필요한 초기화 작업이 init 메소드 안에서 해야한다. init() 메소드는 인자를 통해서 ProcessorContext 객체를 전달받는다. ProcessorContext를 통해 현재 처리중인 record의 메타데이터를 가져올 수 잇다. 또한 ProcessorContext#forward 메소드를 통해서 다운 스트림으로 새로운 record를 전달할 수 있다.

아래는 Processor 인터페이스 코드이다.

``` java
public interface Processor<K, V> {

    void init(ProcessorContext context);

    void process(K key, V value);

    @Deprecated
    void punctuate(long timestamp);

    void close();
}
```

ProcessorContext의 schedule() 메소드는 일정한 주기로 특정 메소드를 실행시킬 때 사용한다. 아래의 코드는 ProcessorContext의 schedule 메소드이다. 

``` java
    *
     * @param interval the time interval between punctuations
     * @param type one of: {@link PunctuationType#STREAM_TIME}, {@link PunctuationType#WALL_CLOCK_TIME}
     * @param callback a function consuming timestamps representing the current stream or system time
     * @return a handle allowing cancellation of the punctuation schedule established by this method
     */
    Cancellable schedule(long interval, PunctuationType type, Punctuator callback);

```

아래의 코드는 Punctuator 인터페이스이다.
``` java
public interface Punctuator {

    /**
     * Perform the scheduled periodic operation.
     *
     * @param timestamp when the operation is being called, depending on {@link PunctuationType}
     */
    void punctuate(long timestamp);
}

```
ProcessorContext#schedule()  메소드는 파라미터로 Punctuator 콜백 인터페이스를 받는다. Punctuator의 punctuate 메소드는 PunctuationType을 기반으로 하여 주기적으로 호출된다. PunctuationTyp은 스케쥴링을 위해서 어떤 시간 개념을 사용할 지를 결정한다. 시간 개념은 2가지가 존재한다.

* stream-time (default) : stream-time은 이벤트 시간을 사용한다. 만약 stream-time을 사용한다면 punctuate() 메소드는 순전히 Record에 의해서 트리거된다. 왜냐하면 stream-time은 입력 Record의 타임스탬프 값으로 결정되기 때문이다. input record가 들어오지 않으면, stream-time은 진행되지 않으며 따라서 punctuate()는 호출되지 않는다. stream-time를 기반으로 스케쥴 한다면 Record를 처리하는데 걸리는 시간과 관계없이, Record의 타임스탬프를 기반으로 해서 punctuate() 메소드를 트리거한다.
* wall-clock-time : punctuate() 메소드가 실제 시간으로 트리거 된다. 

> 주의 
> * Stream-time이 향상되기 위해서는 모든 Input 토픽의 파티션이 사용 가능한 새로운 Record를 가지고 있어야한다. 만약에 하나의 파티션이라도 사용 가능한 새로운 Record를 가지고 있지 않으면, Stream-time은 향상되지 않으며 따라서 punctuate()는 트리거되지 않는다. (PunctuationType.STREAM_TIME를 사용하는 경우) 이는 설정된 timestamp extractor와는 무관하다.

아래의 예제는 word count 샘플 코드이다.

``` java
public class WordCountProcessor implements Processor<String, String> {

  private ProcessorContext context;
  private KeyValueStore<String, Long> kvStore;

  @Override
  @SuppressWarnings("unchecked")
  public void init(ProcessorContext context) {
      // keep the processor context locally because we need it in punctuate() and commit()
      this.context = context;

      // retrieve the key-value store named "Counts"
      kvStore = (KeyValueStore) context.getStateStore("Counts");

      // schedule a punctuate() method every 1000 milliseconds based on stream-time
      this.context.schedule(1000, PunctuationType.STREAM_TIME, (timestamp) -> {
          KeyValueIterator<String, Long> iter = this.kvStore.all();
          while (iter.hasNext()) {
              KeyValue<String, Long> entry = iter.next();
              context.forward(entry.key, entry.value.toString());
          }
          iter.close();

          // commit the current processing progress
          context.commit();
      });
  }

  @Override
  public void punctuate(long timestamp) {
      // this method is deprecated and should not be used anymore
  }

  @Override
  public void close() {
      // close the key-value store
      kvStore.close();
  }

}
```

* init() : 1초마다 punctuate() 메소드가 실행되도록 스케쥴링 한다. 그리고 "Counts"라는 로컬 State Store를 찾는다. punctuate() 메소드에서는 로컬 State Store를 순회한다. 그리고 단어의 개수를 다운스트림 Processor에게 전달한다. 마지막으로 현재 스트림의 상태를 커밋한다.
* process() : 레코드의 value를 단어로 쪼갠다. 그리고 단어의 개수를 State Store에 업데이트한다. (여기서는 생략되었고 뒤에서 설명한다)

> Note
> * Stateful processing with state stores: WordCountProcessor는 process() 메소드를 통해서 Record를 받는다. 그리고 처리 상태를 저장하기 위해서 State Store를 사용할 수 있다. 더 자세한 내용은 아래에서 설명하도록 하겠다.

## State Stores

Stateful한 `Processor` 혹은 `Transformer`를 구현하기 위해서는, 1개 이상의 State Store를 제공해야한다. (Stateless Processor 혹은 transformer는 State Store가 필요하지 않다) State Store는 최근에 받은 input records를 저장히기 위해서 사용되기도 하며, input records의 중복을 제거하기 위해서도 사용된다. State Store의 또 다른 특징은 외부 어플리케이션이 State Store에 쿼리할 수 있다는 것이다. 

## Defining and Creating a State Store
사용할 수 있는 Store 타입 중 하나를 사용하거나 직접 Store 타입을 구현할 수 있다.  일반적으로는 Stores 팩토리 클래스에 있는 Store의 타입을 사용한다. 

Kafka Streams를 사용할 때, 보통은 코드를 통해 State Store를 직접적으로 생성하지 않는다. 보통은 StoreBuilder를 통해서 State Store를 생성한다. StoreBuilder는 Kafka Streams에서 State Store를 생성할 때 사용된다.

다음과 같은 Store 타입을 사용할 수 있다.

### KeyValueStore (Rocks DB)
* KeyValueStore<K, V> 
* Rocks DB Storage Engine, and Fault-tolerant by default
* 대부분의 경우는 해당 Store 타입을 사용하면 된다.
* 데이터를 로컬 디스크에 저장한다.
* 관리되는 로컬 state의 크기가 어플리케이션의 메모리의 크기(힙 메모리 크기)보다 더 커질 수 있다. 하지만 대부분의 경우 로컬 디스크 크기보다 커지진 않는다.
* RocksDB 설정에 대한 설명은 [Rocks DB](https://docs.confluent.io/current/streams/developer-guide/config-streams.html#streams-developer-guide-rocksdb-config)를 참고해라

더 자세한 내용은 [PersistentKeyValueStore](https://docs.confluent.io/current/streams/javadocs/org/apache/kafka/streams/state/Stores.PersistentKeyValueFactory.html)를 참고해라

``` java
// Creating a persistent key-value store:
// here, we create a `KeyValueStore<String, Long>` named "persistent-counts".
import org.apache.kafka.streams.processor.StateStoreSupplier;
import org.apache.kafka.streams.state.Stores;

// Note: The `Stores` factory returns a supplier for the state store,
// because that's what you typically need to pass as API parameter.
StateStoreSupplier countStoreSupplier =
  Stores.create("persistent-counts")
    .withKeys(Serdes.String())
    .withValues(Serdes.Long())
    .persistent()
    .build();
```

### KeyValueStore(Inmemory)
* Enable Fault-tolerant by default
* 데이터를 메모리에 저장한다.
* 관리되는 로컬 State의 총 사이즈가 어플리케이션의 힙 메모리 사이즈를 넘어선 안된다.
* 어플리케이션이 로컬 디스크를 사용하지 못하는 환경에서 실행되거나 할 때 유용하다.

``` java
// Creating an in-memory key-value store:
// here, we create a `KeyValueStore<String, Long>` named "inmemory-counts".
import org.apache.kafka.streams.processor.StateStoreSupplier;
import org.apache.kafka.streams.state.Stores;

// Note: The `Stores` factory returns a supplier for the state store,
// because that's what you typically need to pass as API parameter.
StateStoreSupplier countStoreSupplier =
  Stores.create("inmemory-counts")
    .withKeys(Serdes.String())
    .withValues(Serdes.Long())
    .inMemory()
    .build();
```

### Fault-tolerant State Stores
State Store를 fault-tolerant하게 만들고, 데이터 손실 없이 State Store을 마이그래이션 하기 위해서는 State Store가 지속적으로 카프카 토픽에 백업되어야 한다. 이 토픽을 changelog topic 혹은 changelog라고 한다. 만약에 어플리케이션이 비정상적으로 종료된 경우 State Store와 어플리케이션의 State는 Changelog Topic을 통해서 완전히 복구 될 수 있다. 기본적으로 KeyValue Store는 Fault-tolerant하다. 따라서 KeyValueStore는 changelog topic에 백업된다. change log 토픽은 compacted 토픽인데 이는 토픽의 데이터가 무한히 증가하는 것을 방지하기 위해서이다. 또한 Change log 토픽을 이용해서 State Store가 복구되는 시간을 최소한으로 하기 위해서이다.

유사하게 persistent window stores도 fault-tolerant하다. persistent window stores는 토픽에 백업되며 해당 토픽은 compaction과 deletion을 모두 사용한다. changelog 토픽에 보내지는 메시지 키의 구조 때문에, compaction과 deletion이 조합이 필요하다. Window Store의 경우 메시지의 키는 복합키로 일반 키 + window 타임스탬프 구조이다. 이러한 유형의 복합키의 경우 compaction만으로는 changelog 토픽의 사이즈가 무한정 늘어나는 것을 막을 수 없다. deletion이 활성화된 경우 만료된 window는 로그 세그먼트가 만료됨에 따라 카프카의 로그 클러너에 의해서 삭제된다. 기본 보관 설정은 Windows#maintainMs() + 1이다. StreamsConfig에 StreamsConfig.WINDOW_STORE_CHANGE_LOG_ADDITIONAL_RETENTION_MS_CONFIG 설정을 추가함으로서 기본 설정을 오버라이딩할 수 있다.

만약에 StateStore로부터 Iterator를 오픈한 경우엔 작업을 완료한 후에 Iterator의 close() 메소드를 호출해야 한다. 그래야만 리소스가 정상적으로 반환되며, Iterator를 제대로 close() 하지 않으면 OOM 에러가 발생할 수 있다.

### Enable or Disable Fault Tolerance of State Stores (Store Changelogs)
fault-tolerant를 사용할지 여부는 결정할 수 있다. 사용 여부는 enableLogging() 그리고 disableLogging() 메소드를 사용해서 설정한다. 필요한 경우 연관된 토픽 설정을 세밀하게 설정할 수 있다. 

아래의 예제는 fault tolerance를 disable하는 예제이다.

``` java
StoreBuilder<KeyValueStore<String, Long>> countStoreSupplier = Stores.keyValueStoreBuilder(
  Stores.persistentKeyValueStore("Counts"),
    Serdes.String(),
    Serdes.Long())
  .withLoggingDisabled(); // disable backing up the store to a changelog topic
```

> 주의
> * 만약에 changelog가 disable된 경우 State Store는 더 이상 fault-tolerant하지 않는다. 그리고 standy replicas도 가질 수 없다.

아래의 예는 fault tolerance를 enable하고, 추가적인 changelog-topic 설정을 하는 예제이다. 어떠한 log 설정을 추가할 수 있다. 알수없는 설정은 무시된다.

``` java
Map<String, String> changelogConfig = new HashMap();
// override min.insync.replicas
changelogConfig.put("min.insyc.replicas", "1")

StoreBuilder<KeyValueStore<String, Long>> countStoreSupplier = Stores.keyValueStoreBuilder(
  Stores.persistentKeyValueStore("Counts"),
    Serdes.String(),
    Serdes.Long())
  .withLoggingEnabled(changlogConfig); // enable changelogging, with custom changelog settings
```

### Custom State Store 구현
보통 제공되는 State Store 타입을 사용하지만 StateStore를 직접 구현할 수도 있다. 직접 구현하기 위해서 구현해야하는 인터페이스는 StateStore 인터페이스이다. Kafka Streams에는 StateStore를 상속하고 있는 인터페이스도 있다. 대표적으로 KeyValueStore가 있다.
또한 StateStore를 위한 팩토리 클래스를 제공해야 한다. 팩토리 클래스는 StateStoreSupplier 인터페이스를 구현하면 된다. Kafka Streams는 StateStoreSupplier 인터페이스를 통해서 StateStore 객체를 생성한다.

### Connecting Processors and State Stores
Processor와 State Store를 정의한 후에, Processor와 StateStore를 연결함으로써 Processor Topology를 생성할 수 있다. Processor와 StateStores는 Topology 객체를 사용해서 연결할 수 있다. 그리고 특정 카프카 토픽으로부터 input 데이터 스트림을 생성하는 Source Processor를 추가할 수 있다. 또한 Sink processor를 추가할 수 있다.

아래는 이에 대한 구현이다.

``` java
Topology builder = new Topology();

// add the source processor node that takes Kafka topic "source-topic" as input
builder.addSource("Source", "source-topic")

    // add the WordCountProcessor node which takes the source processor as its upstream processor
    .addProcessor("Process", () -> new WordCountProcessor(), "Source")

    // add the count store associated with the WordCountProcessor processor
    .addStateStore(countStoreBuilder, "Process")

    // add the sink processor node that takes Kafka topic "sink-topic" as output
    // and the WordCountProcessor node as its upstream processor
    .addSink("Sink", "sink-topic", "Process");
```

위의 예제에 대한 간단히 설명하면 
* addSource 메소드를 통해서 "Source"라는 이름의 source processor가 topology에 추가되었다. 해당 processor는 "source-topic" 토픽의 데이터를 읽어드린다.
* addProcessor 메소드를 통해서 "Process"라는 이름의 Processor를 추가한다. 추가된 Processor는 "Source" Processor의 다운스트림 Processor로 추가된다.
* countStoreBuilder를 사용해서 미리 정의한 KeyValue State Store가 생성되며 "Process" 프로세서와 연관관계가 생긴다.
* addSink 메소드를 통해서 sink processor가 추가된다. "Process" Processor가 UpStream Processor가 되며, 해당 Processor의 결과를 "sink-topic"이라는 카프카 토픽에 보낸다.

위의 Topology를 간략하게 표현하면 아래와 같다

> Topology
> *  SOURCE -> PROCESS -> SINK

Source Processor가 새로운 Record를 카프카 토픽으로부터 가져오면 이를 Process Processor에게 전달한다. 그리고 WordCountProceesor#process() 메소드가 레코드를 처리하기 위해서 실행된다. WordCountProcessor#punctuate()  메소드 안에서 Context#forward 메소드가 호출될 때마다 aggregate된 key-value값이 Sink Processor에게 보내진다. Sink Processor는 최종적으로 그 결과를 "sink-topic"으로 전송한다. WordCountProcessor 구현에서 Key Value Store에 접근할 때 사용하는 store 이름에 주의해야 한다. "Counts"라는 이름으로 Key Value Store를 접근하지 않는다면 State Store를 찾을수 없다는 예외가 발생한다. 또한 만약 Topology 예제 코드에서 State Store가 Processor에 연결되어 있지 않은 경우에도 Processor에서 해당 State Store를 접근할 수 없다는 예외가 발생한다. 
