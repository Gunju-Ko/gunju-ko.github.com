---
layout: post
title: "How to manage the size of state stores using tombstones?" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka, kafka-stream]
---

# How to manage the size of state stores using tombstones?

이 글은 Kafka Stream Processor API 또는 DSL 사용방법에 대한 일반적인 패턴을 설명한다. 본 글은 [Kafka Stream Usage Pattern](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Stream+Usage+Patterns#KafkaStreamUsagePatterns-Howtomanagethesizeofstatestoresusingtombstones)를 참고하길 바란다. 

## How to manage the size of state stores using tombstones?

Aggregation을 사용하는 어플리케이션은 State Store에서 더 이상 필요하지 않은 레코드를 제거함으로써 리소스를 보다 효율적으로 사용할 수 있다. Kafka Streams는 tombstone 레코드를 사용해서 레코드를 State Store에서 제거할 수 있다. tombstone 레코드는 키가 널이 아니며, value가 널인 레코드를 말한다. Kafka Streams는 tombstone 레코드를 받은 경우, 레코드 키에 해당하는 데이터를 State Store에서 삭제한다.

tombstone 레코드를 사용하는 예제는 아래에서 찾아볼 수 있다. 중요한점은 키가 널인 레코드는 내부적으로 버려진다는 것이다. 그리고 aggregation에서는 키가 널인 레코드가 도착하지 않는다. aggregation 코드에서는 State Store에서 레코드를 언제 지울지에 대한 로직이 포함되어 있어야 한다. 만약 레코드를 삭제하고 싶으면 null를 리턴하면 된다.

아래의 예제는 reduce를 사용하여 State Store를 삭제하는 예제이다. 항공사가 고객의 비행 단계를 추적하려고 한다. 단계는 총 4단계로 이루어져있으며 예약 완료, 탑승 완료, 착륙, 그리고 비행 후 설문 조사 상태가 있다. 고객이 비행 후 설문 조사를 완료했다면, 항공사는 더 이상 고객를 추적할 필요가 없다. 그전까지는 항공사는 고객이 어떤 단계에 있는지 파악하고 고객 데이터에 대해 다양한 집계를 수행하고자 한다. 이는 다음과 같은 Topology를 통해서 수행할 수 있다.

``` java
// customer flight statuses
final String BOOKED = "booked";
final String BOARDED = "boarded";
final String LANDED = "landed";
final String COMPLETED_FLIGHT_SURVEY = "survey";
 
// topics
final String SOURCE_TOPIC = "someInputTopic";
final String STORE_NAME = "someStateStore";
 
// topology
KStreamBuilder builder = new KStreamBuilder();
KStream<String, String> stream = builder.stream(SOURCE_TOPIC);
 
KTable<String, String> customerFlightStatus = stream
    .groupByKey()
    .reduce(
        new Reducer<String>() {
            @Override
            public String apply(String value1, String value2) {
                if (value2.equals(COMPLETED_FLIGHT_SURVEY)) {
                    // we no longer need to keep track of this customer since
                    // they completed the flight survey. Create a tombstone
                    return null;
                }
                // keeping this simple for brevity
                return value2;
            }
        }, STORE_NAME);
```

고객이 비행 후 설문 조사를 완료한 후에만 Reducer에서 널을 리턴한다. tombstone는 널을 리턴함으로써 생성된다. 그리고 레코드는 State Store에서 즉시 삭제된다. 다음 테스트를 통해서 확인할 수 있다.

``` java
final String customerFlightNumber = "customer123_dl90210";
File stateDir = createStateDir(); // implementation omitted
KStreamTestDriver driver = new KStreamTestDriver(builder, stateDir, Serdes.String(), Serdes.String());
driver.setTime(0L);
 
// get a reference to the state store
KeyValueStore<String, String> store = (KeyValueStore<String, String>) driver.context().getStateStore(STORE_NAME);
 
// the customer has booked their flight
driver.process(SOURCE_TOPIC, customerFlightNumber, BOOKED);
store.flush();
assertEquals(BOOKED, store.get(customerFlightNumber));
 
// the customer has boarded their flight
driver.process(SOURCE_TOPIC, customerFlightNumber, BOARDED);
store.flush();
assertEquals(BOARDED, store.get(customerFlightNumber));
 
// the customer's flight has landed
driver.process(SOURCE_TOPIC, customerFlightNumber, LANDED);
store.flush();
assertEquals(LANDED, store.get(customerFlightNumber));
 
// the customer has filled out the post-flight survey, so we no longer need to track them
// in the state store. make sure the key was deleted
driver.process(SOURCE_TOPIC, customerFlightNumber, COMPLETED_FLIGHT_SURVEY);
store.flush();
assertEquals(null, store.get(customerFlightNumber));
```

tombstones이 다운스트림으로 어떻게 전달되는지는 중요하다. tombstones이 서브 토폴로지에 전달되는지에 대한 여부는 서브 토폴로지가 입력 데이터를 스트리밍하기 위해 KTable을 사용하느냐 또는 KStream을 사용하는냐에 달려있다. 아래의 코드는 이러한 차이점을 보여준다.

``` java
// tombstones are visible in a KStream and must be filtered out
KStream<String, String> subtopologyStream = customerFlightStatus
        .toStream()
        .filter((key, value) -> {
            if (value == null) {
                // tombstone! skip this
                return false;
            }
            // other filtering conditions...
            return true;
        });
 
// tombstone forwarding is different in KTables. The filter below is not evaluated for a tombstone
KTable<String, String> subtopologyKtable = customerFlightStatus
        .filter((key, value) -> {
            // the tombstone never makes it here. no need to check for null
 
            // other filtering conditions...
            return true;
        });
```
 
 KTable<String, String> customerFlightStatus의 결과를 KStream으로 변환해서 받은 경우에는 tombstones이 포함된다. 따라서 위에 예제와 같이 filter를 통해서 tombstones를 필터링하는게 일반적이다. 그리나 KTable (changelog stream)의 경우 tombstones는 filter, mapValues 등과 같은 연산에 전달되지 않는다.
 
## 출처 

* https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Stream+Usage+Patterns#KafkaStreamUsagePatterns-Howtomanagethesizeofstatestoresusingtombstones
