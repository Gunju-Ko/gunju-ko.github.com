---
layout: post
title: "Micrometer" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [Monitoring]
---

# Micrometer

## Micrometer란?

마이크로미터는 모니터링 시스템을 위한 측정 클라이언트에 대한 파사드를 제공한다. (SLF4J와 유사하다) 마이크로미터에 의해서 기록된 어플리케이션 메트릭 정보는 주로 모니터링 용도 또는 알람의 용도로 사용된다. 메트릭 정보를 수집하기 위해서 약간의 코드만 추가하면 되도록 설계되어있다. 또한 메트릭 정보의 이식성을 극대화하였다.

마이크로미터는 다음과 같은 모니터링 시스템을 지원한다  : Atlas, Datadog, Graphite, Ganglia, Influx, JMX and Prometheus.

> Facade 패턴
> * 파사드 패턴은 서브 시스템을 감춰주는 상위 수준의 인터페이스를 제공함으로써 코드 중복과 서브 시스템에 대한 직접적인 의존 문제를 해결한다. 파사드 패턴에서 파사드는 서브 시스템에 접근하여 클라이언트가 원하는 데이터를 제공하는 역할을 한다. 따라서 클라이언트는 파사드에만 의존하게 되며, 서브 시스템의 직접적인 의존이 제거된다. 파사드 패턴을 적용하면 클라이언트는 파사드에만 의존하기 때문에, 서브 시스템의 일부가 변경되더라도 그 여파는 파사드로 한정될 가능성이 높다. 또 다른 장점은 파사드를 인터페이스로 정의함으로써 클라이언트의 변경 벗이 서브 시스템 자체를 변경할 수 있다는 것이다.
> * 파사드 패턴을 클래스와 비교해보면, 파사드는 마치 서브 시스템의 상세함을 감춰주는 인터페이스와 유사하다. 파사드를 통해서 서브 시스템의 상세한 구현을 캡슐화하고, 이를 통해 상세한 구현이 변경되더라도 파사드를 사용하는 코드에 주는 영향을 줄일 수 있게 된다.
> * 출처 : 개발자가 반드시 정복해야할 객체지향과 디자인 패턴

## Supported monitoring systems

마이크로 미터는 코어 모듈과 SPI를 포함하고 있다. SPI는 다양한 모니터링 시스템에 대한 구현의 집합이다. (각각을 Registry라고도 한다) 모니러팅 시스템에는 중요한 3가지 특징이 있다. 

#### Dimensionality

일부 모니터링 시스템은 메트릭의 이름에 태그(key-value)를 붙일 수 있도록 지원한다. 이를 Dimensionality라 한다.
반면에 flat한 메트릭 이름만 지원하는 경우도 있다. 이를 hierarchical 이라한다. 마이크로미터는 메트릭을 hierarchical한 시스템에 보낼때, 태그들을 메트릭 이름에 추가해서 보낸다.

* Dimensional : Atlas, Datadog, Datadog StatsD, Influx, Prometheus, SignalFx, Telegraf StatsD, Wavefront
* Hierarchical : Graphite, Ganglia, JMX, Etsy StatsD

#### Rate aggregation

특정 지표에 대한 평균값을 구하는 경우가 있다. 어떤 모니터링 시스템은 어플리케이션에서 평균값을 구해서 평균값으로 보내주기를 기대한다. 반면에 어떤 모니터링 시스템은 어플리케이션이 누적된 값을 보내주기를 기대하며, 모니터링 시스템이 직접 평균값을 구하는 경우도 있다. 

* Client side : Atlas, Datadog, Influx, Graphite, Ganglia, JMX, all StatsD flavors, SignalFx, Wavefront
* Server side : Prometheus 

#### Publishing

일부 모니터링 시스템에서는 메트릭 정보를 어플리케이션으로부터 polling해간다. 반면 어플리케이션이 일정한 간격으로 메트릭 정보를 push해줘야 하는 모니터링 시스템도 있다.

* Client Push : Atlas, Datadog, Graphite, Ganglia, Influx, JMX, all StatsD flavors, SignalFx, Wavefront
* Server polls : Prometheus

마이크로미터는 어떤 Registry를 사용하느냐에 따라, 위와 같은 요구사항을 충족시키기 위해 메트릭 정보를 커스터마이징한다.

## MeterRegistry

Meter는 어플리케이션의 메트릭을 수집하기 위한 인터페이스이다. 마이크로미터에서 Meter는 MeterRegistry에 의해 생성되며, MerterRegistry에 등록된다. 지원되는 모니터링 시스템는 MeterRegistry 구현체가 있다. 

SimpleMeterRegistry는 각 meter의 최신값을 메모리에 저장한다. 그리고 메트릭 정보를 다른 시스템으로 보내진않는다. 만약에 아직 모니터링 시스템을 결정하지 못했다면 우선 SimpleMeterRegistry를 사용하면 된다. 

``` java
MeterRegistry registry = new SimpleMeterRegistry();
```

### CompositeMeterRegistry

여러개의 모니터링 시스템에 메트릭 정보를 보내야하는 경우에는 CompositeMeterRegistry를 사용하면 된다. CompositeMeterRegistry에 여러개의 MeterRegistry를 등록할 수 있다. 그리고 CompositeMeterRegistry는 등록된 MeterRegistry를 통해 여러개의 모니터링 시스템에 메트릭정보를 보낸다.

``` java
CompositeMeterRegistry compositeRegistry = new CompositeMeterRegistry();
SimpleMeterRegistry oneSimpleMeter = new SimpleMeterRegistry();
AtlasMeterRegistry atlasMeterRegistry 
  = new AtlasMeterRegistry(atlasConfig, Clock.SYSTEM);
 
compositeRegistry.add(oneSimpleMeter);
compositeRegistry.add(atlasMeterRegistry);
```

### Global registry

마이크로미터는 static 변수로 글로벌 MeterRegistry를 제공한다. Metrics.globalRegistry를 통해 static 변수에 접근할 수 있으며 Metrics 클래스에는 글로벌 MeterRegistry를 기반으로 Meter를 생성하는 정적 빌더 메소드가 있다. 아래는 Metrics 클래스의 코드이다. 코드를 보면 알 수 있듯이 글로벌 MeterRegistry는 CompositeMeterRegistry 객체이다. 

``` java
public class Metrics {
    public static final CompositeMeterRegistry globalRegistry = new CompositeMeterRegistry();
    private static final More more = new More();

    /**
     * Add a registry to the global composite registry.
     *
     * @param registry Registry to add.
     */
    public static void addRegistry(MeterRegistry registry) {
        globalRegistry.add(registry);
    }

    /**
     * Remove a registry from the global composite registry. Removing a registry does not remove any meters
     * that were added to it by previous participation in the global composite.
     *
     * @param registry Registry to remove.
     */
    public static void removeRegistry(MeterRegistry registry) {
        globalRegistry.remove(registry);
    }

    /**
     * Tracks a monotonically increasing value.
     *
     * @param name The base metric name
     * @param tags Sequence of dimensions for breaking down the name.
     * @return A new or existing counter.
     */
    public static Counter counter(String name, Iterable<Tag> tags) {
        return globalRegistry.counter(name, tags);
    }
}
```

아래는 간단한 샘플 코드이다.

``` java
class MyComponent {
    Counter featureCounter = Metrics.counter("feature", "region", "test"); (1)

    void feature() {
        featureCounter.increment();
    }

    void feature2(String type) {
        Metrics.counter("feature.2", "type", type).increment(); (2)
    }
}

class MyApplication {
    void start() {
        // wire your monitoring system to global static state
        Metrics.addRegistry(new SimpleMeterRegistry()); (3)
    }
}
```

1. 가능하다면 Meter 객체를 멤버 변수로 저장해라. 그러면 Meter 객체를 사용할 때마다 MeterRegistry에서 Meter 객체를 찾을 필요가 없다. 
2. 2번과 같이 메소드의 파라미터를 통해서 Tag값을 받는 경우에는, 메소드를 호출할 때마다 Meter 객체를 찾거나(name과 tag에 해당하는 Meter 객체가 이미 존재하는 경우), Meter 객체를 생성해야 한다. Meter 객체는 name과 tag값으로 MeterRegistry에서 찾는다. 
3. Metrics.counter와 같은 메소드를 통해서 미리 Meter 객체를 생성한 후에, MeterRegistry를 등록해도 괜찮다. 모든 Meter 객체들은 각 MeterRegistry에 추가된다. 

## Meters

마이크로미터는 Meter의 구현체로 다음와 같은 것들을 지원한다. 

* Timer, Counter, Gauge, DistributionSummary, LongTaskTimer, FunctionCounter, FunctionTimer, TimeGauge.

Meter에 따라 수집되는 메트릭 수가 다르다. 각각의 Meter는 이름과 태그로 식별된다. 일반적으로 이름을 중심점으로 사용할 수 있어야한다. 태그는 같은 이름의 메트릭을 좀 더 세부적으로 나누기 위해 사용된다. 

#### Tags

Meter의 식별자는 이름과 태그로 구성된다. 단어를 "."으로 구분하는 네이밍 컨벤션을 지켜야한다. 이는 여러 모니터링 시스템에 대한 메트릭 이름 이식성을 보장한다. 

``` java
Counter counter = registry.counter("page.visitors", "age", "20s");
```

* 이름 : page.visitors
* 태그 : age, 20s

태그는 값에 대한 추론을 위해 메트릭을 분할하는데 사용된다. 위의 경우 20대에 대한 페이지 방문자 수임을 추론할 수 있다. 

대형 시스템의 경우 Registry에 공통 태그를 추가할 수 있다. 

``` java
registry.config().commonTags("region", "ua-east");
```

#### Counter

Counter는 어플리케이션의 지정된 속성에 대한 수를 리포트한다. 빌더 메소드 혹은 MetricRegistry의 헬퍼 메소드를 통해 커스텀 카운터를 생성할 수 있다. 

``` java
Counter counter = Counter
  .builder("instance")
  .description("indicates instance count of the object")
  .tags("dev", "performance")
  .register(registry);
 
counter.increment(2.0);
  
assertTrue(counter.count() == 2);
  
counter.increment(-1);
  
assertTrue(counter.count() == 2);
```

위으 코드를 볼 수 있듯이, 카운트는 증가만 가능하다. 감소는 되지 않는다.

#### Timers

시스템의 이벤트 빈도나 latency를 측정하고자 한다면, Timers를 사용해라. Timer는 적어도 이벤트가 발생한 수와 총 시간을 리포트한다. 

아래는 간단한 예제이다.

``` java
SimpleMeterRegistry registry = new SimpleMeterRegistry();
Timer timer = registry.timer("app.event");
timer.record(() -> {
    try {
        TimeUnit.MILLISECONDS.sleep(1500);
    } catch (InterruptedException ignored) { }
});
 
timer.record(3000, MILLISECONDS);
 
assertTrue(2 == timer.count());
assertTrue(4510 > timer.totalTime(MILLISECONDS) 
  && 4500 <= timer.totalTime(MILLISECONDS));
```

#### Gauge

Gauge는 Meter의 현재 값을 보여준다. 다른 Meter와는 달리 Gauge는 데이터가 관찰된 경우에만 데이터를 리포트한다. Gauge는 캐시 등의 통계를 모니터링할 때 유용하다. 

``` java
SimpleMeterRegistry registry = new SimpleMeterRegistry();
List<String> list = new ArrayList<>(4);
 
Gauge gauge = Gauge
  .builder("cache.size", list, List::size)
  .register(registry);
 
assertTrue(gauge.value() == 0.0);
  
list.add("1");
  
assertTrue(gauge.value() == 1.0);

```

이 외에도 다양한 Meter가 존재한다. Meter에 대한 더 자세한 내용은 공식문서를 참고하도록 해라

## Binders
Micrometer에는 JVM, 캐시, ExecutorService 및 로깅 서비스를 모니터링하는 여러 내장 바인더가 있다. JVM 및 시스템 모니터링의 경우 클래스 로더 메트릭(ClassLoaderMetrics), JVM 메모리 풀 (JvmMemoryMetrics), GC 메트릭(JvmGcMetrics), 스레드 및 CPU 사용량(JvmThreadMetrics, ProcessorMetrics)을 모니터링 할 수 있다. 캐싱 모니터링은 Guava, EhCache, Hazelcast, Caffeine만 지원된다. (GuavaCacheMetrics, EhCache2Metrics, HazelcastCacheMetrics, CaffeineCacheMetrics를 사용) 그리고 로깅 서비스를 모니터링 하기 위해서 LogbackMetrics를 사용할 수 있다.

``` java
new LogbackMetrics().bind(registry);
```

다른 바인더의 사용법도 위와 매우 비슷하며 매우 간단하다. 따라서 자세한 내용은 다루지 않겠다. 

## 출처 
* [마이크로 미터 공식문서](https://micrometer.io/docs)
* [Baeldung](http://www.baeldung.com/micrometer)
* [Spring Boot Mircometer](https://spring.io/blog/2018/03/16/micrometer-spring-boot-2-s-new-application-metrics-collector)
