---
layout: post
title: "Spring Micrometer Kafka Metric 수집" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring]
---

# Spring Micrometer Kafka Metric 수집

Spring Boot Actuator를 추가하면 Micrometer 관련된 디펜던시를 추가하고, 자동설정(auto-configuration)을 해준다. 스프링부트는 CompositeMeterRegistry를 자동설정하고 클래스패스에서 찾은 MeterRegistry를 CompositeMeterRegistry에 등록한다. 새로운 MeterRegistry을 등록하려면 `micrometer-registry-{system}` 디펜던시를 런타임으로 추가해주면 된다. 또한 스프링부트는 자동설정한 MeterRegistry를 `Metrics` 클래스의 글로벌 스태틱 변수에 등록한다. 아래와 같이 MeterRegistry를 주입받고 Meter를 생성해서 사용하면 된다.

``` java
@Component
public class SampleBean {

    private final Counter counter;

    public SampleBean(MeterRegistry registry) {
        this.counter = registry.counter("received.messages");
    }

    public void handleMessage(String message) {
        this.counter.increment();
        // handle message implementation
    }

}
```

스프링부트는 `MeterBinder` 구현체를 제공해서 다양한 메트릭을 자동을 수집해준다. 

스프링부트는 적용 가능한 경우 아래와 같은 메트릭을 수집한다.

* JVM Metrics
  * 스레드 사용률
  * 가비지 컬렉션 관련 메트릭
  * 다양한 메모리 및 버퍼풀
  * 로드/언로드된 클래스의 수
* CPU Metrics
* File Descriptor Metrics
* Kafka Consumer Metrics
* Log4j2 Metrics
* Logback Metrics
* Uptime Metrics
* Tomcat Metrics
* Spring Integration Metrics
* Spring MVC Metrics
* Spring WebFlux Metrics
* Jersey Server Metrics
* Cache Metrics
* DataSource Metrics
* Hibernate Metrics
* RabbitMQ Metrics

자세한 내용은 [spring boot - production ready metrics meter](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-metrics-meter) 를 참고하길 바란다.

### Kafka Metrics

스프링은 카프카 메트릭을 수집과 관련해서 자동설정을 위한 `KafkaMetricsAutoConfiguration` 클래스를 제공한다. `KafkaMetricsAutoConfiguration` 의 경우 `MBeanServer`가 빈으로 등록된 경우에만 `KafkaConsumerMetrics` 빈을 등록하기 때문에 `spring.jmx.enabled` 를 `true` 로 설정해야만 동작한다.

`KafkaConsumerMetrics` 는 아래와 같은 메트릭을 수집한다.

#### consumer fetch manager metrics

* records-lag : 파티션의 가장 최근 LAG
* records-lag-avg : 파티션의 평균 LAG
* records-lag-max : 해당 window에 최대 LAG
* records-lead : The latest lead of the partition
* records-lead-min : The min lead of the partition.
* records-lead-avg : The average lead of the partition.
* fetch-size-avg : 요청당 가져오는 평균 bytes 수
* records-per-request-avg : 요청당 평균 레코드 수
* bytes-consumed-total : 컨슘한 bytes 총 수
* records-consumed-total : 컨슘한 레코드 총 수
* fetch-total : fetch 요청 총 수
* fetch-latency-avg : fetch 요청의 평균 latency
* fetch-latency-max : fetch 요청의 최대 latency
* fetch-throttle-time-avg : 평균 throttle 시간. quotas가 활성화 된 경우 브로커는 제한을 초과한 컨슈머를 제한하기 위해 fetch 요청을 지연시킨다. 이 메트릭은 fetch 요청에 throttling 시간이 평균적으로 얼마나 더해졌는지를 나타낸다.
* fetch-throttle-time-max : 최대 throttle 시간

#### consumer coordinator-metrics

* assigned-partitions : 컨슘머에 할당된 파티션 수
* commit-rate : 초당 커밋이 호출된 횟수
* join-rate : 초당 조인요청을 보낸 횟수
* sync-rate : 초당 sync 요청을 보낸 횟수. Group synchronization는 rebalance 프로토콜의 두번째이자 마지막 단계로 해당 같이 높으면 컨슈머 그룹이 불안정하다는것을 나타낸다.
* hearbeat-rate : 초당 hearbeat 요청을 보낸 횟수
* commit-latency-avg : 커밋 요청에 소요된 평균 시간
* commit-latency-max : 커밋 요청에 소요된 최대 시간
* join-time-avg : 그룹에 rejoin 하는데 걸린 평균 시간
* join-time-max : 그룹에 rejoin 하는데 걸린 최대 시간. 해당 값이 session timeout보다 크면 안된다.
* heartbeat-response-time-max : heartbeat 요청에 대한 응답을 받는데까지 소요된 최대 시간
* last-hearbeat-seconds-ago : 마지막 heartbeat 이후 지난 시간

#### consumer metrics

* connection-count : active 커넥션 개수
* connection-creation-total : 새롭게 생성된 커넥션 개수
* connection-close-total : close된 커넥션 개수
* io-ratio : I/O 스레드가 I/O 작업을 하는데 소요된 시간의 비율
* io-wait-ratio : I/O 스레드가 wating 하는데 소요된 시간의 비율
* select-total : I/O 계층에서 새로운 I/O를 수행하기 위해 체크한 횟수
* io-time-ns-avg : The average length of time for I/O per select call
* io-wait-time-ns-avg : The average length of time the I/O thread spent waiting for a socket to be ready for reads or writes
* network-io-total : 받은 바이트 총 수
* outgoing-byte-total : 전송된 바이트 총 수
* request-total : 요청의 총 수
* response-total : 응답의 총 수
* io-waittime-total : Time spent on the I/O thread waiting for a socket to be ready for reads or writes.
* iotime-total : Time spent in I/O during select calls.

> 자세한 사항은 [confluent-new-consumer-metrics](https://docs.confluent.io/current/kafka/monitoring.html#new-consumer-metrics), KafkaConsumerMetrics 클래스를 참고

