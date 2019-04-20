---
layout: post
title: "Testing a stream application" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka, kafka-stream]
---

# Testing a Stream Application

Kafka Streams 어플리케이션을 테스트 하기 위해서, 카프카가 test-utils을 제공한다. 이는 dependecy를 추가함으로써 사용할 수 있다. 예를 들어 pom.xml에 다음과 같이 디펜던시를 추가할 수 있다.

```
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams-test-utils</artifactId>
    <version>1.1.0</version>
    <scope>test</scope>
</dependency>
```

test-utils 패키지는 TopologyTestDriver를 제공한다. 해당 클래스는 Topology에 데이터를 흘려보내는데 사용된다. 테스트 드라이버는 input 토픽으로부터 데이터를 읽어와서 Topology를 통해서 데이터를 처리하는 과정을 시뮬레이션한다. 테스트 드라이버를 사용하여 지정된 Processor Topology가 수동으로 흘려보낸 데이터를 정확하게 처리하는지를 확인할 수 있다. 테스트 드라이버는 결과 Record를 캡쳐하고 임베디드된 State Store에 쿼리할 수 있도록 해준다.

``` java
// Processor API
Topology topology = new Topology();
topology.addSource("sourceProcessor", "input-topic");
topology.addProcessor("processor", ..., "sourceProcessor");
topology.addSink("sinkProcessor", "output-topic", "processor");
// or
// using DSL
StreamsBuilder builder = new StreamsBuilder();
builder.stream("input-topic").filter(...).to("output-topic");
Topology topology = builder.build();

// setup test driver
Properties config = new Properties();
config.put(StreamsConfig.APPLICATION_ID_CONFIG, "test");
config.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "dummy:1234");
TopologyTestDriver testDriver = new TopologyTestDriver(topology, config);    
```

TopologyTestDriver는 키와 값의 Type이 byte[]인 ConsumerRecord를 허용한다. 키와 값의 타입이 byte[]인 ConsumerRecord를 생성하는 것이 귀찮을 수 있는데, ConsumerRecordFactory를 사용하면 이를 쉽게 할 수 있다. 아래는 ConsumerRecordFactory의 샘플 코드이다.

``` java
ConsumerRecordFactory<String, Integer> factory = new ConsumerRecordFactory<>("input-topic", new StringSerializer(), new IntegerSerializer());
testDriver.pipe(factory.create("key", 42L));
```

위의 코드를 보면 ConsumerRecordFactory는 제네릭을 통해서 키와 값의 타입을 결정한다. 그리고 생성자를 통해서 Topic이름과, 키와 값에 대한 Serializer 객체를 전달하면 된다. 그리고 factory의 create 메소드의 리턴값은 ConsumerRecord<byte[], byte[]>이다. 

결과를 검증하기 위해서, TopologyTestDriver는 ProducerRecord를 생성한다. 이 때 키와 값의 타입은 byte[]이다. 결과를 좀 더 쉽게 검증하기 위해서 readOutput의 2번째 3번째 인자로 각각 키와 값의 Deserializer 객체를 전달하면 원하는 타입으로 ProducerRecord를 받을 수 있다.

```
ProducerRecord<String, Integer> outputRecord = testDriver.readOutput("output-topic", new StringDeserializer(), new LongDeserializer());
```

결과 검증을 위해서 OutputVerifier를 사용할 수 있다.  OutputVerifier는 헬퍼 메소드를 제공해서 결과 Record의 특정 부분만 비교할 수 있도록 해준다.  예를 들어 Record의 키와 값만 비교하고 결과 Record의 타임스탬프값은 무시할 수 있다. 

```
OutputVerifier.compareKeyValue(outputRecord, "key", 42L); // throws AssertionError if key or value does not match
```

추가적으로 TopologyTestDriver를 통해서 StateStore에 접근할수도 있다. 테스트를 하기전에 StateStore에 접근해서 원하는 값을 넣을 수도 있으며, 테스트 후에 StateStore에 접근해서 원하는 값이 있는지 확인할 수도 있다.

```
KeyValueStore store = testDriver.getKeyValueStore("store-name");
```

그리고 항상 마지막에 TopologyTestDriver의 close() 메소드를 호출해서 모든 리소스가 올바르게 릴리스되도록 해야한다.
 
```
testDriver.close();
```

### Example

다음의 예제는 테스트 드라이버와 헬퍼 클래스를 어떤식으로 사용하는지에 대한 간단한 예제이다. 예제는 Key-Value Store를 사용해서 키에 해당하는 값의 최대값을 구하는 Topology를 생성한다. 처리를 하는 동안 결과는 생성되지 않는다. 단지 Key-Value Store만 업데이트 된다. 결과는 단지 다운스트림으로 보내진다.

``` java
private TopologyTestDriver testDriver;
private KeyValueStore<String, Long> store;

private StringDeserializer stringDeserializer = new StringDeserializer();
private LongDeserializer longDeserializer = new LongDeserializer();
private ConsumerRecordFactory<String, Long> recordFactory = new ConsumerRecordFactory<>(new StringSerializer(), new LongSerializer());

@Before
public void setup() {
    Topology topology = new Topology();
    topology.addSource("sourceProcessor", "input-topic");
    topology.addProcessor("aggregator", new CustomMaxAggregatorSupplier(), "sourceProcessor");
    topology.addStateStore(
        Stores.keyValueStoreBuilder(
            Stores.inMemoryKeyValueStore("aggStore"),
            Serdes.String(),
            Serdes.Long()).withLoggingDisabled(), // need to disable logging to allow store pre-populating
        "aggregator");
    topology.addSink("sinkProcessor", "result-topic", "aggregator");

    // setup test driver
    Properties config = new Properties();
    config.setProperty(StreamsConfig.APPLICATION_ID_CONFIG, "maxAggregation");
    config.setProperty(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "dummy:1234");
    config.setProperty(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
    config.setProperty(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.Long().getClass().getName());
    testDriver = new TopologyTestDriver(topology, config);

    // pre-populate store
    store = testDriver.getKeyValueStore("aggStore");
    store.put("a", 21L);
}

@After
public void tearDown() {
    testDriver.close();
}

@Test
public void shouldFlushStoreForFirstInput() {
    testDriver.pipeInput(recordFactory.create("input-topic", "a", 1L, 9999L));
    OutputVerifier.compareKeyValue(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer), "a", 21L);
    Assert.assertNull(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer));
}

@Test
public void shouldNotUpdateStoreForSmallerValue() {
    testDriver.pipeInput(recordFactory.create("input-topic", "a", 1L, 9999L));
    Assert.assertThat(store.get("a"), equalTo(21L));
    OutputVerifier.compareKeyValue(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer), "a", 21L);
    Assert.assertNull(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer));
}

@Test
public void shouldNotUpdateStoreForLargerValue() {
    testDriver.pipeInput(recordFactory.create("input-topic", "a", 42L, 9999L));
    Assert.assertThat(store.get("a"), equalTo(42L));
    OutputVerifier.compareKeyValue(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer), "a", 42L);
    Assert.assertNull(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer));
}

@Test
public void shouldUpdateStoreForNewKey() {
    testDriver.pipeInput(recordFactory.create("input-topic", "b", 21L, 9999L));
    Assert.assertThat(store.get("b"), equalTo(21L));
    OutputVerifier.compareKeyValue(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer), "a", 21L);
    OutputVerifier.compareKeyValue(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer), "b", 21L);
    Assert.assertNull(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer));
}

@Test
public void shouldPunctuateIfEvenTimeAdvances() {
    testDriver.pipeInput(recordFactory.create("input-topic", "a", 1L, 9999L));
    OutputVerifier.compareKeyValue(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer), "a", 21L);

    testDriver.pipeInput(recordFactory.create("input-topic", "a", 1L, 9999L));
    Assert.assertNull(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer));

    testDriver.pipeInput(recordFactory.create("input-topic", "a", 1L, 10000L));
    OutputVerifier.compareKeyValue(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer), "a", 21L);
    Assert.assertNull(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer));
}

@Test
public void shouldPunctuateIfWallClockTimeAdvances() {
    testDriver.advanceWallClockTime(60000);
    OutputVerifier.compareKeyValue(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer), "a", 21L);
    Assert.assertNull(testDriver.readOutput("result-topic", stringDeserializer, longDeserializer));
}

public class CustomMaxAggregatorSupplier implements ProcessorSupplier<String, Long> {
    @Override
    public Processor<String, Long> get() {
        return new CustomMaxAggregator();
    }
}

public class CustomMaxAggregator implements Processor<String, Long> {
    ProcessorContext context;
    private KeyValueStore<String, Long> store;

    @SuppressWarnings("unchecked")
    @Override
    public void init(ProcessorContext context) {
        this.context = context;
        context.schedule(60000, PunctuationType.WALL_CLOCK_TIME, new Punctuator() {
            @Override
            public void punctuate(long timestamp) {
                flushStore();
            }
        });
        context.schedule(10000, PunctuationType.STREAM_TIME, new Punctuator() {
            @Override
            public void punctuate(long timestamp) {
                flushStore();
            }
        });
        store = (KeyValueStore<String, Long>) context.getStateStore("aggStore");
    }

    @Override
    public void process(String key, Long value) {
        Long oldValue = store.get(key);
        if (oldValue == null || value > oldValue) {
            store.put(key, value);
        }
    }

    private void flushStore() {
        KeyValueIterator<String, Long> it = store.all();
        while (it.hasNext()) {
            KeyValue<String, Long> next = it.next();
            context.forward(next.key, next.value);
        }
    }

    @Override
    public void punctuate(long timestamp) {} // deprecated; not used

    @Override
    public void close() {}
}
```

## 출처
* https://kafka.apache.org/11/documentation/streams/developer-guide/testing.html
