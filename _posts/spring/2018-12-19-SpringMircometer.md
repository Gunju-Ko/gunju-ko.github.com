---
layout: post
title: "Spring Boot Actuator - 마이크로미터 지원" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring]
---

# Spring Boot Actuator의 마이크로미터 지원

스프링부트 액츄에이터는 마이크로미터를 위한 의존성 관리 및 자동 설정을 제공한다. 이 글은 [SpringBoot 공식문서](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-metrics-meter) 를 번역한 글이다. 자세하고 정확한 내용은 공식문서를 참고하길 바란다. (오역이 있을수 있다)

## 1 Getting Started

스프링부트는 클래스패스에 따라 MeterRegistry 빈을 자동으로 구성해준다. 클래스패스에  `micrometer-registry-{system}` 런타임 의존성을 추가하면, 스프링부트는 해당 시스템의 MeterRegistry를 구성한다.

대부분의 레지스트리들은 공통된 기능을 제공한다. 예를 들어, 마이크로미터 레지스트리 구현이 클래스패스에 있더라도 해당 레지스트리를 비활성화 할 수 있는 기능을 제공한다. influx를 비활성화하려면 아래와 같이 하면 된다.

``` yaml
management:
  metrics:
    export:
      influx:
        enabled: false
```

스프링부트는 자동으로 설정된 MeterRegistry를 Metrics 클래스의 public static 변수인 CompositeMeterRegistry globalRegistry에 추가한다.

MeterRegistryCustomizer 빈을 등록하면 스프링부트에 의해 자동으로 등록된 MeterRegistry를 추가로 커스터마이징할 수 있다. 예를 들어 공통적으로 사용되는 태그를 추가할 수 있다. 

``` java
@Bean
MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
	return registry -> registry.config().commonTags("region", "us-east-1");
}
```

또는 아래와 같이 구체적인 타입을 명시해서 MeterRegistryCustomizer 구현체를 빈으로 등록해도 된다.

``` java
@Bean
MeterRegistryCustomizer<GraphiteMeterRegistry> graphiteMetricsNamingConvention() {
	return registry -> registry.config().namingConvention(MY_CUSTOM_CONVENTION);
}
```

사용자가 직접 메트릭 정보를 추가하고 싶으면 아래와 같이 MeterRegistry를 DI 받아서 메트릭을 등록하면 된다.

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

스프링부트는 자동으로 메트릭을 수집해준다. 예를 들어 스프링부트는 Datasource, Tomcat 메트릭을 수집하는 Meter를 자동으로 생성한다. Datasource의 경우 DataSourcePoolMetrics 클래스를 사용해서 다양한 Meter를 생성한다.

## 2. 지원되는 모니터링 시스템

아래와 같은 모니터링 시스템이 지원된다.

- AppOptics
- Atlas
- Datadog
- Dynatrace
- Elastic
- Ganglia
- Graphite
- Humio
- Influx
- JMX
- KairosDB
- New Relic
- Prometheus
- SignalFx
- Simple (in-memory)
- StatsD
- Wavefront

이 중에서 프로메테우스, Influx만 정리하도록 하겠다. 나머지 모니터링 시스템에 대해서는 공식 문서를 참고하길 바란다.

#### Influx

Influx 서버의 주소는 아래와 같이 설정하면 된다.

``` yaml
management:
  metrics:
    export:
      influx.uri: http://influx.example.com:8086
```

#### 프로메테우스

프로메테우스는 각 인스턴스에서 메트릭정보를 직접 가져온다(poll 방식) 따라서 각 인스턴스는 메트릭정보를 프로메테우스에게 제공하기 위한 엔드포인트를 제공 해야한다. 스프링부트는 `/actuator/prometheus` 액츄에이터 엔드포인트를 제공하여 적절한 형식의 프로메테우스 스크래핑을 제공한다.

> 디폴트 설정은 프로메테우스 액츄에이터 엔드포인트를 노출하지 않는다. 따라서 해당 엔드포인트를 노출시키도록 설정해줘야한다. 자세한 사항은 [Exposing Endpoints](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-endpoints-exposing-endpoints) 를 참고하길 바란다.

다음 예제는 prometheus.yml에 추가할 scrape_config 설정 예제이다.

``` yaml
scrape_configs:
  - job_name: 'spring'
	metrics_path: '/actuator/prometheus'
	static_configs:
	  - targets: ['HOST:PORT']
```

## 3. 지원되는 메트릭

스프링부트는 다음과 같은 핵심 메트릭을 등록한다.

- JVM Metrics, report utilization of
  - 다양한 메모리 및 버퍼풀
  - Garbage Collection 관련 통계
  - 스레드 수
  - 로드 또는 로드되지 않은 클래스 수
- CPU 메트릭
- File Descriptor 메트릭
- 카프카 컨슈머 메트릭
- Log4j2 메트릭
- Logback 메트릭
- Spring Integration 메트릭
- 톰캣 메트릭
- 가동시간 메트릭

#### Spring MVC 메트릭

`management.metrics.web.server.auto-time-requests` 설정이 true이면 모든 요청에 대해 메트릭을 수집한다. 반면에 설정이 false이면, 컨트롤러 메소드에 @Timed 애노테이션을 추가한 경우에만 메트릭을 수집한다.

``` java
@RestController
@Timed <1>
public class MyController {

	@GetMapping("/api/people")
	@Timed(extraTags = { "region", "us-east-1" }) <2>
	@Timed(value = "all.people", longTask = true) <3>
	public List<Person> listPeople() { ... }

}
```

- <1> : @Timed 애노테이션을 클래스에 붙이는 경우 모든 메소드에 대해 시간을 측정한다.
- <2> : 메소드에 @Timed 애노테이션을 추가해서 메소드별로 활성화 시킬수도 있다. 클래스에 @Timed 애노테이션이 있는 경우 필요하지 않지만, 타이머를 해당 메소드에 대해서만 커스터마이징하고 싶은 경우에 사용할 수 있다.
- <3> : @Timed 애노테이션의 longTask를 true로 설정하는 경우, LongTaskTimer를 사용해서 메트릭을 측정한다. 

기본적으로, 메트릭은 이름에 `http.server.requests`라는 prefix가 붙는다. prefix는 `management.metrics.web.server.requests-metric-name` 속성을 통해서 커스터마이징 할 수 있다.

또한 Spring MVC 관련 메트릭에는 다음과 같은 정보가 태그로 지정된다.

- exception : 요청을 처리하는 도중에 발생한 예외 클래스 이름
- method : 요청 Method 
- outcome : 요청 결과는 Response의 상태코드를 기반으로 한다.
  - 1XX : INFORMATIONAL
  - 2XX : SUCCESS
  - 3XX : REDIRECTION
  - 4XX : CLIENT_ERROR
  - 5XX : SERVER_ERROR
- status : HTTP 응답의 상태코드
- uri : 요청 URI 템플릿 (예를 들어, `api/person/{id}`)

> 스프링은 WebFlux 컨트롤러에 의해 처리되는 요청에 대한 메트릭도 수집한다. 메트릭 이름과 공통 태그는 Spring MVC 메트릭과 같다. 만약 태그를 커스터마이징 하고 싶으면 WebFluxTagsProvider 타입의 빈을 등록하면 된다.

#### HttpClient 메트릭

스프링부트는 RestTemplate, WebClient에 대한 메트릭도 수집해준다. 그러기위해서는 자동설정된 builder를 주입받고, 주입받은 builder를 통해 객체를 생성해야 한다. 

- RestTemplateBuilder : RestTemplate 생성시 사용
- WebClient.Builder : WebClient 생성시 사용

혹은 customizer를 수동으로 적용해도 된다. (MetricsRestTemplateCustomizer 혹은 MetricsWebClientCustomizer)

디폴트로 메트릭은 `http.client.requests` 라는 prefix가 붙는다. prefix는 `management.metrics.web.client.requests-metric-name` 속성을 통해 커스터마이징할 수 있다. 또한 아래와 같은 태그가 붙는다.

- method : 요청 메소드
- uri : 요청 URI 템플릿 
- status : HTTP 응답 상태 코드
- clientName : URI의 호스트 부분

만약 태그를 커스터마이징 하고 싶으면 RestTemplateExchangeTagsProvider 또는 WebClientExchangeTagsProvider 타입의 빈을 등록해주면 된다. RestTemplateExchangeTags, WebClientExchangeTags 클래스에는 태그 생성을 위한 편리한 static 메소드가 있다.

#### Cache 메트릭

사용가능한 모든 캐시에 대해서 `cache`라는 prefix가 붙은 이름으로 메트릭 정보를 수집한다. 다양한 캐시에 대해 표준화된 메트릭 정보를 수집하며, 구현체에 따라 추가적인 메트릭 정보를 수집하는 경우도 있다.

다음과 같은 캐시 라이브러리들이 지원된다.

- Caffeine
- EhCache 2
- Hazelcast
- Any compliant JCache (JSR-107) implementation

캐시 관련 메트릭은 캐시 이름, CacheManager 빈의 이름이 태그로 지정된다.

> 시작시점에 사용할 수 있는 캐시의 경우에 한해서 메트릭 정보가 Registry에 등록된다. 시작 단계 이후에 또는 프로그래밍 방식으로 작성된 캐시의 경우 명시적인 등록이 필요하다. `CacheMetricsRegistrar` 빈을 사용하면 이를 쉽게 할 수 있다.

다음과 같은 정보가 수집된다.

- cache.size : 캐시의 엔트리 개수이다. 정확한 값이 아닌 대략적인 값이 가능성이 높다.
- cache.gets : 캐시에서 Miss된 횟수
  - tag : "result=miss"
- cache.gets : 캐시에서 hit된 횟수
  - "result=hit"
- cache.puts : 캐시에 새롭게 추가된 항목수
- cache.evictions : 캐시에서 제거된 엔트리 수

#### Datasource 메트릭

jdbc라는 이름의 메트릭으로 DataSource 오브젝트를 계측한다. DataSource의 현재 active 커넥션 수, 최대 커넥션 수, 최소 커넥션 수을 계측한다. 각 메트릭은 jdbc라는 접두어가 붙은 이름을 가진다.

- jdbc.connection.active : 현재 active 상태의 커넥션 수
- jdbc.connection.max : 커넥션 풀의 최대 커넥션 수
- jdbc.connection.min : 커넥션 풀의 최소 커넥션 수

> 스프링부트는 대부분의 데이터소스에 대한 메타데이터를 제공한다. 하지만, 만약 사용하는 데이터소스가 메타데이터를 제공하지 않는 경우엔 DataSourcePoolMetadataProvider 인터페이스의 구현체를 직접 구현하고 빈으로 등록해야 한다. DataSourcePoolMetadataProvidersConfiguration를 참고하길 바란다.

또한 Hikari 고유 메트릭은 hikaricp라는 접두어가 붙는다. 각 메트릭은 pool 이름을 태그로 갖는다. 

#### Tomcat

TomcatMetrics 클래스를 통해 Tomcat 관련 메트릭 정보를 수집한다. 수집하는 메트릭 정보는 아래와 같다.

- global request
- servlet
- cache
- thread pool
  - tomcat.threads.config.max : 설정된 스레드풀 최대 개수
  - tomcat.threads.busy : 현재 요청을 처리중인 스레드 개수
  - tomcat.threads.current : 스레드풀의 스레드 개수 (active thread + idle thread)
- session

> 이 글에서는 설명하진 않았지만 스프링 부트는 아래와 같은 메트릭을 수집해준다. 자세한 내용은 공식문서를 참고하길 바란다.
>
> - Jersey Server Metrics
> - Hibernate Metrics
> - RabbitMQ Metrics

## 4. Registering custom metrics

커스텀 메트릭을 등록하려면, 컴포넌트에서 MeterRegistry 빈을 DI 받아야한다. 

``` java
class Dictionary {

	private final List<String> words = new CopyOnWriteArrayList<>();

	Dictionary(MeterRegistry registry) {
		registry.gaugeCollectionSize("dictionary.size", Tags.empty(), this.words);
	}

	// …

}
```

여러 구성 요소 또는 애플리케이션에서 여러개의 메트릭을 반복적으로 측정하는 경우 MeterBinder를 사용해서 캡슐화할 수 있다. MeterBinder는 대표적으로 DataSourcePoolMetrics가 있는데, DataSourcePoolMetrics은 Datasource의 메트릭을 생성하고, MeterRegistry에 등록한다.

## 5. Customizing individual metrics

특정 Meter 객체를 커스터마이징 해야하는 경우 MeterFilter 인터페이스를 사용하면 된다. 기본적으로 모든 MeterFilter 빈들은 자동적으로 MeterRegistry.Config에 등록된다. 

예를 들어, 이름이 "com.example"로 시작하는 Meter의 "mytag.region" 태그 이름을 "mytag.area"로 변경하고 싶다면 다음과 같이 하면 된다.

``` java
@Bean
public MeterFilter renameRegionTagMeterFilter() {
	return MeterFilter.renameTag("com.example", "mytag.region", "mytag.area");
}
```

#### Commons tags

공통 태그는 모든 Meter에 적용되며 아래와 같이 설정할 수 있다. 

``` yaml
management:
  metrics:
    tags:
      region: us-east-1
      stack: prod
```

위의 예제의 경우 region과 stack 태그를 모든 Meter에 추가한다. 

#### Per-meter properties

속성을 사용해서도 Meter 단위로 커스터마이징을 할 수 있다. Meter 단위 커스터마이징은 주어진 이름을 prefix로 같는 모든 Meter에 적용된다. 예를 들어 아래와 같이 설정하면 example.remote로 시작하는 모든 Meter를 비활성화 한다.

``` yaml
management:
  metrics:
    enable:
      example.remote: false
```

다음과 같은 속성은 Meter 단위로 커스터마이징이 가능하다. (각 속성에 대한 설명은 공식문서를 참고하길 바란다)

- management.metrics.enable
- management.metrics.distribution.percentiles-histogram
- management.metrics.distribution.minimum-expected-value
- management.metrics.distribution.maximum-expected-value
- management.metrics.distribution.percentiles
- management.metrics.distribution.sla

## 6. Metrics endpoint

스프링부트는 애플리케이션에서 수집한 메트릭을 보여주기위한 엔드포인트를 제공한다. 디폴트로 해당 엔트포인드는 노출되지 않으므로 설정을 통해 해당 엔트포인트를 노출시켜줘야 한다. 자세한건 [exposing endpoints](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-endpoints-exposing-endpoints)를 참고하길 바란다. 

`/actuator/metrics`를 호출해보면 모든 Meter의 이름을 볼 수 있다. 만약에 특정 Meter에 대한 정보를 보고 싶으면 `/actuator/metrics/jvm.memory.max` 이런식으로 호출하면 된다. 또한 쿼리 파라미터를 통해 태그 정보를 추가할 수 있다. 태그는 tag=KEY:VALUE 형식으로 추가해야 한다. 예를 들어 다음과 같이 쿼리 파라미터로 태그 정보를 넘겨줄 수 있다. `/actuator/metrics/jvm.memory.max?tag=area:nonheap` 

