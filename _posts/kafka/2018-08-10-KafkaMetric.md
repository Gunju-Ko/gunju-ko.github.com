---
layout: post
title: "Kafka Metric" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [kafka]
---

# Kafka Metric reporter

브로커 그리고 클라이언트는 내부 메트릭 정보를 리포트한다. 메트릭 정보를 수집해서 모니터링 하고자 할 경우 카프카에서 제공하는 MetricReporter를 사용하면 된다.

Producer, Consumer, Kafka Streams는 "metric.reporters"라는 속성을 가지고 있다. 이 속성으로 MetricReporter 구현체를 지정해주면 된다. MetricReporter 구현체는 새로운 메트릭이 생성이 되면 새로운 메트릭 정보를 metricChange(KafkaMetric metric) 메소드를 통해 전달받는다. MetricReporter로 JmxReporter는 기본적으로 항상 추가된다. 

## MetricsReporter 인터페이스

``` java
/**
 * A plugin interface to allow things to listen as new metrics are created so they can be reported.
 * <p>
 * Implement {@link org.apache.kafka.common.ClusterResourceListener} to receive cluster metadata once it's available. Please see the class documentation for ClusterResourceListener for more information.
 */
public interface MetricsReporter extends Configurable {

    /**
     * This is called when the reporter is first registered to initially register all existing metrics
     * @param metrics All currently existing metrics
     */
    public void init(List<KafkaMetric> metrics);

    /**
     * This is called whenever a metric is updated or added
     * @param metric
     */
    public void metricChange(KafkaMetric metric);

    /**
     * This is called whenever a metric is removed
     * @param metric
     */
    public void metricRemoval(KafkaMetric metric);

    /**
     * Called when the metrics repository is closed.
     */
    public void close();

}
```

* init : MetricReporter가 Metrics에 등록될 때  호출된다. 만약 현재 KafkaMetric정보가 Metrics 객체에 존재한다면, 모든 KafkaMetrics 정보를 MetricReporter에게 전달한다. 
* metricChanged : 새로운 메트릭 정보가 등록될 때 이 메소드를 통해 새로운 메트릭 정보를 전달받는다.
* removeMetric : 메트릭 정보가 제거될 때 제거된 메트릭 정보를 이 메소드를 통해 전달받는다.
* close : Metrics의 close 메소드가 호출된 경우, Metrics는 등록된 모든 MetricReporter의 close 메소드를 호출한다.

## 관련 속성

* metric.reporters : MetricsReporter 구현체 리스트
* metrics.num.samples : 메트릭 계산을 위해서 유지되는 샘플 수
* metrics.sample.window.ms : 메트릭 샘플이 계산되는 time window

"metrics.num.samples" 속성과 "metrics.sample.window.ms" 속성이 모든 메트릭에 영향을 주는건 아니다. 예를 들어 "records-consumed-total" 메트릭의 경우 컨슘한 총 레코드의 수인데, 이 메트릭인 단순한 누적값이기 때문에 앞에 두 속성에 영향을 받지 않는다. 두 속성에 영향 받는 메트릭은 주로 평균, 최대, 최소, 합 등 특정 구간에 대한 메트릭인 경우이다. 

예를 들어 metrics.num.samples가 2이고, metric.sample.window.ms가 5000이라고 해보자. 그러면 메트릭 계산을 위해서 유지되는 샘플 수는 2개이며, 샘플 하나의 time window는 5000이 된다. 

예를 들어 1초 동안 발생한 메시지 수의 평균을 구하는 메트릭을 추가했다고 가정해보자. 그러면 아래와 같은 방식으로 메트릭이 기록된다.

1. 처음에는 샘플이 하나 추가된다. 그리고 5초 동안의 메시지 수는 샘플1에 기록된다. 샘플1에는 eventCount와 value가 있다. 여기서 value는 메시지 수이다. (eventCount는 메트릭을 업데이트한 횟수이다)
2. "1"에서 5초가 지난뒤에 메시지 수를 기록한다면, 샘플 2가 추가된다. 그리고 앞으로 5초 동안에 발생한 메시지 수는 샘플 2에 기록된다.
3. "2"에서 5초가 지난뒤에 메시지 수를 기록한다면, 샘플 3를 추가하려한다. 하지만 metrics.num.samples가 2이기 때문에 샘플을 추가할 수 없다. 이 때는 샘플1을 재사용한다. 샘플 1은 초기화되고 앞으로 5초동안 발생한 메시지 수는 샘플 1에 기록된다.

그리고 특정 시점에 메트릭 정보를 조회한다면, 다음과 같은 방법으로 메트릭을 계산한다.

1. 모든 샘플에 기록된 모든 메시지 수를 더한다. 즉 샘플 1과 샘플 2의 메시지 수를 더한다. 단 메트릭을 조회하는 시점에 "metrics.num.samples * metric.sample.window.ms" 시간이 지난 샘플은 현재로서는 유효하지 않다고 판단하여 샘플을 리셋시킨다. 예제에서는 메트릭을 조회하는 시점에서 10초가 지난 샘플은 리셋시킨다. 
2.  "모든 샘플의 메시지 수 합 / (현재시간 - 가장 오랜된 샘플의 시작시간)" 가 1초동안 발생한 메시지 수의 평균이 된다.

![Alt text]({{ site.url }}/assets/img/posts/kafka-metric/metrics-1.png)

예를 들어 위와 같은 경우에 메트릭 값은 (1000 + 400) / 700 = 200이 된다. 즉 1초 동안 발생한 메시지 수의 평균이 200이 된다. (샘플 1과 샘플 2 모두 메트릭을 조회하는 시점에 10초가 지나지 않았기 때문에 유효하다)

> 사실 메트릭 정보를 계산하는 것은 위에 설명보다는 좀 더 복잡하다. 하지만 개인적으론 대부분의 경우 위와 같이 계산 될 것으로 예상된다.



## 메트릭 정보

브로커 그리고 클라이언트는 많은 메트릭 정보를 리포트한다. 아래는 그 중 일부만을 설명한다. 

* fetch-size-avg : 요청 당 가져오는 평균 바이트 수
* fetch-rate : 초당 fetch 요청을 보내는 수
* records-consumed-rate : 초당 컨슘한 레코드의 평균 값
* records-consumed-total : 컨슘한 총 레코드 수

KakaMetrics 정보들은 이미 계산된 메트릭 정보이다. 즉 이미 평균을 계산했거나, 최대, 최소값을 계산한 메트릭이 된다. 따라서 카프카 메트릭 정보를 Micrometer를 이용해서 외부 시스템으로 리포팅하고 있다면 이 점을 유의해야 한다. 카프카는 정말 다양한 메트릭 정보를 리포팅한다. 따라서 자세한 것은 공식문서를 참고하길 바란다.

## Metrics

Sensor, KafkaMetric, MetricReporter를 가지고 있는 클래스이다.

* Sensor : Sensor는 measurement를 기록할 때 사용하는 클래스이다. 또한 Sensor는 measurement와 관련된 메트릭을 가지고 있다. 예를 들어 Sensor가 메시지의 크기를 기록할 수 있으며, 센서에 의해 기록된 메시지 크기의 평균, 최대, 또는 기타 통계 메트릭을 센서에 추가할 수 있다. 그리고 Sensor에 추가된 메트릭 정보는 Metrics 클래스에 추가된다. (그리고 Metrics는 추가된 메트릭을 MetricReporter에게 알린다)

Metrics 클래스는 보통 아래와 같은 방식으로 사용된다.

``` java

Metrics metrics = new Metrics(); // this is the global repository of metrics and sensors

// create sensor
Sensor sensor = metrics.sensor(&quot;message-sizes&quot;);
MetricName metricName = new MetricName(&quot;message-size-avg&quot;, &quot;producer-metrics&quot;);
sensor.add(metricName, new Avg());
metricName = new MetricName(&quot;message-size-max&quot;, &quot;producer-metrics&quot;);
sensor.add(metricName, new Max());

```

* Sensor sensor(String name) : 이 메소드를 통해 Sensor 객체를 생성하고, 생성된 Sensor 객체는 Metrics에 등록된다. 만약에 주어진 이름의 Sensor가 이미 존재한다면 Sensor 객체를 생성하지 않고 존재하는 Sensor 객체를 리턴한다.
* add(MetricName metricName, MeasurableStat stat) : Sensor에 새로운 메트릭을 등록한다. 등록된 메트릭은 Metrics에도 추가 된다. Metrics는 추가된 메트릭을 MetricReporter에게 알려준다. 

Fetcher, KafkaProducer 등은 Metrics 클래스를 통해서 Sensor 객체를 생성하고, Measurement를 기록한다. Fetcher가 Metrics#sensor 메소드를 통해서 Sensor 객체를 생성하면, Sensor 객체는 Metrics에 등록된다. 또한 Sensor에 메트릭을 추가할 때마다 메트릭이 Metrics에도 등록된다. Metrics 클래스에서 특정 메트릭의 현재 값을 알고 싶다면, KafkaMetric#metricValue 메소드를 호출하면 된다. 또한 Metrics에 등록된 모든 메트릭 정보를 가져오고 싶다면, Metrics#metrics() 메소드를 호출하면 된다. Fetcher는 Sensor#record 메소드를 호출해서 Measurement를 기록한다. 그리고 이 때 메트릭이 업데이트 된다.

아래 코드는 Fetcher에서 Sensor recordsFetched를 생성하는 코드이다. recordsFetched는 컨슘한 레코드의 수를 기록하는 Sensor이다. 

``` java
this.recordsFetched = metrics.sensor("records-fetched");
this.recordsFetched.add(metrics.metricInstance(metricsRegistry.recordsPerRequestAvg), new Avg());
this.recordsFetched.add(new Meter(metrics.metricInstance(metricsRegistry.recordsConsumedRate),
        metrics.metricInstance(metricsRegistry.recordsConsumedTotal)));
```

아래 코드는 Fetcher가 Sensor에 메트릭을 업데이트하는 부분이다. sensor의 records 메소드를 통해서 메트릭을 업데이트한다.

``` java
public void record(TopicPartition partition, int bytes, int records) {
    
    // skip 
    if (this.unrecordedPartitions.isEmpty()) {
        this.sensors.bytesFetched.record(this.fetchMetrics.fetchBytes);
        this.sensors.recordsFetched.record(this.fetchMetrics.fetchRecords);
        // skip
    }
}
```


## 관련 문서

* [Monitoring Kafka](https://docs.confluent.io/current/kafka/monitoring.html#fetch-metrics)
