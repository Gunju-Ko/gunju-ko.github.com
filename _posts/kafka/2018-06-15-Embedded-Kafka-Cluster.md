---
layout: post
title: "Embedded Kafka Cluster" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka]
---

# EmbeddedKafkaCluster

KafkaStreams의 소스 코드를 보면 Integration Test 작성을 위해서 EmbeddedKafkaCluster를 많이 사용하는것을 볼 수 있다. EmbeddedKafkaCluster를 사용하면 마치 로컬에서 Kafka Broker를 실행시키는 것과 같은 효과를 얻을 수 있다. 

``` java
public class StreamIntegrationTest {
    private static final int NUM_BROKERS = 1;

    @ClassRule
    public static final EmbeddedKafkaCluster CLUSTER =
        new EmbeddedKafkaCluster(NUM_BROKERS);

    @Before
    public void before() throws InterruptedException {
        
    }

    @After
    public void whenShuttingDown() throws IOException {
       
    }

    @Test
    public void test() throws Exception {
        
    }
}
```

다음과 같이 EmbeddedKafkaCluster에 @ClassRule 어노테이션을 추가하면 EmbeddedKafkaCluster를 사용해서 Integraion Test를 작성할 수 있다. 생성자를 통해서 브로커의 개수를 설정할 수 있다.

보통 @Before에서 테스트에 사용할 토픽을 생성한다. EmbeddedKafkaCluster#createTopic 메소드를 사용하면 토픽을 생성할 수 있다. 아래는 createTopic 메소드 코드이다. 토픽을 생성할 때 파티션의 개수나 Replication Factor를 정할 수 있다.

``` java
/**
 * Create a Kafka topic with the given parameters.
 *
 * @param topic       The name of the topic.
 * @param partitions  The number of partitions for this topic.
 * @param replication The replication factor for (the partitions of) this topic.
 */
public void createTopic(final String topic, final int partitions, final int replication) throws InterruptedException {
    createTopic(topic, partitions, replication, new Properties());
}
```

또한 EmbeddedKafkaCluster#createTopics 메소드를 사용하면 한 개 이상의 토픽을 생성할 수 있다. 단 이 때는 파티션 개수나 Replication Factor를 정할 수 없고 둘 다 1로 설정이 된다.

``` java
public void createTopics(final String... topics) throws InterruptedException {
    for (final String topic : topics) {
        createTopic(topic, 1, 1, new Properties());
    }
}
```

주의할 점은 KafkaStreams 객체를 생성할 때 사용하는 Properties의 BOOTSTRAP_SERVERS_CONFIG를 EmbeddedKafkaCluster#bootstrapServers() 메소드의 리턴값으로 세팅해야한다는 것이다. 그래야만 KafkaStreams 객체가 EmbeddedKafkaCluster와 상호작용 할 것이다.

아래는 그 예이다. 

``` java
Properties streamsConfiguration = new Properties();
streamsConfiguration
    .put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, CLUSTER.bootstrapServers());

// 생략
kafkaStreams = new KafkaStreams(builder.build(), streamsConfiguration);
```

테스트를 위해서 특정 토픽에 메시지를 보내야할 경우가 있을 수 있다. 그 때는 IntegrationTestUtils#produceKeyValuesSynchronouslyXXXX 메소드를 사용하면 된다. IntegrationTestUtils에는 다양한 메소드가 존재하는데 이 메소드들을 잘 활용하면 Integration Test를 보다 쉽게 할 수 있다. 

아래의 코드는 IntegrationTestUtils.produceKeyValuesSynchronouslyWithTimestamp 메소드를 사용해서 카프카 메시지를 생성하는 코드이다. IntegrationTestUtils에서는 카프카 메시지를 생성하기 위해서 몇가지 메소드를 제공하고 있다. 각 메소드는 설명만 보면 쉽게 이해가 되므로 자세한 설명은 생략하도록 하겠다.

``` java
private void produceMessages(final long timestamp) throws Exception {
    IntegrationTestUtils.produceKeyValuesSynchronouslyWithTimestamp(
        topic,
        Arrays.asList(
            new KeyValue<>(1, "A"),
            new KeyValue<>(2, "B"),
            new KeyValue<>(3, "C"),
            new KeyValue<>(4, "D"),
            new KeyValue<>(5, "E")),
        TestUtils.producerConfig(
            CLUSTER.bootstrapServers(),
            IntegerSerializer.class,
            StringSerializer.class,
            new Properties()),
        timestamp);
}
```

만약에 KafkaStreams의 토폴로지안에서 메시지 처리 결과를 특정 토픽으로 보내고 있다면, 처리 결과를 확인하기 위해서 결과 토픽을 확인해야 하는 경우가 있을수 있다. 
다행히도 IntegrationTestUtils는 특정 토픽의 메시지를 읽어 올 수 있는 메소드를 제공하고 있다. 많이 사용하는 메소드는 waitUntilMinValuesRecordsReceived 메소드이다. 

``` java
/**
 * Wait until enough data (value records) has been consumed.
 *
 * @param consumerConfig     Kafka Consumer configuration
 * @param topic              Topic to consume from
 * @param expectedNumRecords Minimum number of expected records
 * @param waitTime           Upper bound in waiting time in milliseconds
 * @return All the records consumed, or null if no records are consumed
 * @throws AssertionError       if the given wait time elapses
 */
public static <K, V> List<KeyValue<K, V>> waitUntilMinValuesRecordsReceived(final Properties consumerConfig,
                                                            final String topic,
                                                            final int expectedNumRecords,
                                                            final long waitTime) throws InterruptedException 
```

* consumerConfig : KafkaConsumer 설정
* topic : 메시지를 가져올 토픽
* expectedNumRecords : 예상 레코드의 최소 수 (최소한 이 값 이상의 레코드를 읽어와야 메소드가 리턴된다)
* waitTime : 최대 대기 시간

``` java
private <K, V> List<KeyValue<K, V>> receiveMessages(final int numMessages) throws InterruptedException {
    final Properties consumerProperties = new Properties();
    consumerProperties.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, CLUSTER.bootstrapServers());
    consumerProperties.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "test");
    consumerProperties.setProperty(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
    // 생략
    
    return IntegrationTestUtils.waitUntilMinKeyValueRecordsReceived(
            consumerProperties,
            outputTopic,
            numMessages,
            60 * 1000);
}
```

위의 코드는 waitUntilMinValuesRecordsReceived 메소드를 사용해서 특정 토픽의 레코드를 읽는 간단한 코드이다. 위의 코드는 numMessages 수만큼의 레코드를 읽을 때까지 최대 60초간 대기한다. 

waitUntilMinValuesRecordsReceived 메소드 외에도 다양한 메소드를 통해서 카프카 메시지를 읽어올 수 있다. 더 자세한건 IntegrationTestUtils 클래스의 코드를 참고하길 바란다.

특정 조건이 일어날 때까지 테스트가 종료되지 않게 하거나 아니면 다음 작업을 해야하지 말아야 할 경우에는 TestUtils.waitForCondition 메소드를 사용하면 좋다. 예를 들어 KafkaStreams가 시작되고 난 뒤에 토픽에 메시지를 보내야한다면 아래와 같이 코딩하면 된다. (아래 코드만으로 이해가 되지 않으면 sample code를 참고하길 바란다)

``` java
    @Test
    public void embeddedKafkaTest() throws Exception {
        TestUtils.waitForCondition(() -> wordCountStream.isRunning(), 10 * 1000, "Steams never started");

        produceMessages();

        // do something
    }
```

## 사용 후기
Kafka에서는 테스트를 위한 클래스들을 많이 만들어놨다. 이 클래스들을 잘 활용하면 KafkaStream, Topology 등을 테스트 할 수 있다. 직접 써보니 너무나도 유용했고 사용법도 쉽기 때문에 금방 따라할 수 있을 것이다. 

## Sample Code
아래 링크에 가면 샘플 코드를 볼 수 있다. 

* https://github.com/Gunju-Ko/KafkaStream-ProcessorAPI
