---
layout: post
title: "KTable, Global KTable" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka, kafka-stream]
---

# KStream, KTable, GlobalKTable

## KStream

> Note
> 오직 Kafka Streams DSL만이 KStream 개념이 가지고 있다.

KStream은 레코드 스트림의 추상화이다. 
레코드 스트림안에 있는 모든 레코드들은 INSERT로 해석된다. KStream에서는 레코드를 추가하는 것만 가능하다. 삭제나 업데이트 개념이 없다. 레코드의 키가 같다 하더라도 같은 키를 가진 기존행을 대체할 수는 없다. 

예를 들어 다음 두 개의 데이터 레코드가 스트림으로 보내지고 있다고 가정해보자.

```
("alice", 1) --> ("alice", 3)
```

어플리케이션이 사용자 당 값을 합한다면 alice에 대해 4를 반환합니다. 왜냐하면 두 번째 데이터 레코드가 이전 레코드의 업데이트로 간주되지 않기 때문이다. KStream의 동작을 아래 KTable의 동작과 비교해보길 바란다.

## KTable
> Note
> 오직 Kafka Streams DSL만 KTable 개념이 있다.

KTable은 changelog 스트림의 추상화이다. 여기서 각 데이터 record는 업데이트를 나타낸다. 더 정확하게 말하면, record의 value는 동일한 키에 대한 마지막 값의 "UPDATE"로 해석된다. 만약에 키가 아직 존재하지 않는면 업데이트는 INSERT로 간주된다. changelog 스트림에서 데이터 record는 UPSERT(INSERT / UPDATE)로 해석된다. 왜냐하면 동일한 키에 대한 기존 행을 덮어쓰기 때문이다. 또한 널값은 특별하게 해석된다. value가 널인 record는 해당 키에 대한 DELETE 혹은 삭제 표시를 나타낸다.

> Note
> Effects of Kafka’s log compaction : KStream과 KTable에 대한 또 다른 생각은 다음과 같다. 만약에 카프카 토픽에 KTable을 저장한다면, 저장 공간을 절약하기 위해 카프카의 log compaction 기능을 활성화 하고 싶을 것이다. 그러나 KStream의 경우 log compaction을 사용하는 것이 안전하지 않을 수 있다. log compaction이 동일한 키의 이전 record를 제거하면, 데이터의 의미가 깨질 수 있다. 따라서 log compaction은 KTable에 대해서만 완벽하게 안전하다.

또한 KTable은 키을 사용하여 데이터 Record의 현재 값을 조회할 수 있다. 테이블 조회 기능은 [join operation](https://docs.confluent.io/current/streams/concepts.html#streams-concepts-joins)과 [Interactive Queries](https://docs.confluent.io/current/streams/concepts.html#streams-concepts-interactive-queries)를 통해서 할 수 있다.


## GlobalKTable
> Note
> 오직 Kafka Streams DSL만 GlobalKTable 개념이 있다.

KTable과 같이 GlobalKTabble은 changelog 스트림의 추상화이다. 여기서 각 데이터 record는 업데이트를 나타낸다.  GlobalKTable과 KTable은 채워지는 데이터가 다르다. 각 GlobalKTable은 카프카 토픽의 데이터을 읽는다. 간단한 예를 들어 설명을 하면 input 토픽이 5개의 파티션을 가지고 있다고 하자. 그리고 해당 토픽을 테이블로 읽는 5개의 어플리케이션을 실행시킨다고 하자.
* Input 토픽을 KTable을 이용해서 읽는 경우 , 각 어플리케이션의 로컬 KTable 객체는 5개의 파티션 중에서 1개의 파티션 데이터로만 채워질 것이다.
* Input 토픽을 GlobalKTable을 이용해서 읽는 경우, 각 어플리케이션의 로컬 GlobalKTable 객체는 해당 토픽의 모든 파티션의 데이터로 채워질 것이다.

GlobalKTable은 키를 사용해서 record의 현재값을 조회하는게 가능하다. 테이블 조회 기능은 [join operations](https://docs.confluent.io/current/streams/concepts.html#streams-concepts-joins)와 [interactive Queries](https://docs.confluent.io/current/streams/concepts.html#streams-concepts-interactive-queries)를 통해 사용할 수 있다.

GlobalKTable의 장점은 아래와 같다.
* 보다 편리하고 효율적으로 join할 수 있다. GlobalKTable을 사용하면 star join을 할 수 있으며 외래키 조회를 지원한다. (즉 Record의 키로 조회하는 것 뿐만 아니라 Record의 value에 있는 데이터를 사용해서 조회할 수도 있다) 또한 여러 조인을 chaining하는 경우 더욱 효율적이며 GlobalKTable을 조인할 때는 input 데이터가 [co-partitioned](https://docs.confluent.io/current/streams/developer-guide/dsl-api.html#streams-developer-guide-dsl-joins-co-partitioning)일 필요가 없다.
* 실행중인 모든 어플리케이션의 정보를 브로드 캐스팅하는데 사용할 수 있다.

GlobalKTable의 단점은 아래와 같다.
* 모든 토픽의 데이터를 읽어오기 때문에 KTable에 비해서 로컬 스토리지를 더 많이 사용한다.
* 모든 토픽의 데이터를 읽어오기 떄문에 KTable에 비해서 네트워크 부하 및 카프카 브로커 부하가 증가한다.

### 출처
* https://docs.confluent.io/current/streams/concepts.html#globalktable%20312

