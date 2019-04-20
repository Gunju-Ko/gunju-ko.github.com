---
layout: post
title: "Working with SQL Databases" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring, spring-boot]
---

# Working with SQL Databases

스프링은 SQL 데이터베이스에 대한 여러가지 지원을 제공한다. JdbcTemplate를 사용해서 직접 JDBC에 접근할 수 있고, 하이버네이트와 같은 ORM에 대한 지원도 제공한다. [Spring Data](https://projects.spring.io/spring-data/)는 추가 기능을 제공한다. 인터페이스 정의를 통해 Repository를 구현할 수 있으며, 인터페이스에 정의된 메소드 이름을 통해서 쿼리를 생성한다. 

## Configure a DataSource

자바의 Datasource 인터페이스는 데이터베이스 커넥션에 대한 표준 메소드를 제공한다. 일반적으로 DataSource는 URL과 인증 정보을 사용해서 데이터베이스와 커넥션을 맺는다.

### Embedded Database Support

개발과정에서 in-memory 데이터베이스를 사용하면 굉장히 편하다. 물론 in-memory 데이터베이스는 데이터를 영구히 저장하지 않는다. 어플리케이션 시작 시점에 데이터베이스에 데이터를 넣는 작업이 필요할 수 있으며, 어플리케이션 종료시 데이터가 전부 날아간다는 사실을 꼭 알아야한다.

스프링 부트는 H2, HSQL, Derby 데이터베이스를 자동설정해준다. 이러한 데이터베이스는 URL 설정을 하지 않아도되며, 사용할 데이터베이스에 대한 build dependency만 추가해주면 된다.

> in memory 데이터베이스를 테스트용도로 사용할 수 있다. 사용하는 ApplicationContext 수와 관계없이 전체 테스트에서 하나의 데이터베이스가 재사용된다. 각 컨텍스트 별로 별도의 임베디드 데이터베이스를 사용하고 싶다면, `spring.datasource.generate-unique-name` 설정을 'true'로 해야한다.

예를 들어, 일반적인 POM 종속성은 다음과 같다.

``` 
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
	<groupId>org.hsqldb</groupId>
	<artifactId>hsqldb</artifactId>
	<scope>runtime</scope>
</dependency>
```

* 임베디드 데이터베이스 자동 설정을 위해서는 `spring-jdbc`에 대한 의존성이 필요하다. 이 예에서는 `spring-boot-starter-data-jpa`에 대한 의존성을 추가함으로써 `spring-jdbc` 에 대한 의존성을 추가했다.
* 어떤 이유로든 임베디드 데이터베이스에 URL 설정을 추가했다면, 데이터베이스에 자동 종료가 비활성화 되었는지 확인해라. H2를 사용하는 경우 `DB_CLOSE_ON_EXIT=FALSE ` 설정을 URL에 추가해라. 데이터베이스 자동 종료를 비활성화 함으로써, 스프링부트가 데이터베이스 종료를 제어할 수 있도록 해야한다. 

### Connection to a Production Database

상용 환경에서는 DataSource를 Pooling 해야한다. 스프링부트에서는 다음과 같은 알고리즘을 사용해서 어떤 커넥션풀을 사용할지 결정한다. 

* 기본적으로는 HikariCP를 사용한다. HikariCP를 사용할 수 있다면, HikariCP를 사용한다.
* 그렇지 않으면 Tomcat pooling DataSource를 사용한다. (Tomcat pooling DataSource를 사용할 수 있는 경우)
* HikariCP, Tomcat pooling DataSource 모두 사용이 불가능한경우, Commons DBCP2를 사용한다. (Commons DBCP2를 사용할 수 있는 경우)

> 스프링부트 2.0에서는 spring-boot-starter-jdbc만 추가하면, HikariCP에 대한 의존성이 자동으로 추가된다. 

위 알고리즘을 무시하고 싶은 경우에는 `spring.datasource.type` 프로퍼티를 통해 사용할 커넥션풀을 지정해주면 된다. 또한 자동설정을 사용하지 않고 수동으로 Datasource를 설정할 수도 있다. 수동으로 Datasource 빈을 등록한 경우, 자동 설정은 일어나지 않는다.

Datasource 설정은 `spring.datasource. *` 프로퍼티를 통해 할 수 있다. 예를 들어 application.properties에 아래와 같이 Datasource 설정을 추가할 수 있다.

``` 
spring.datasource.url=jdbc:mysql://localhost/test
spring.datasource.username=dbuser
spring.datasource.password=dbpass
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

* spring.datasource.url 프로퍼티를 통해 데이터베이스 URL를 설정할 수 있다. 이 프로퍼티를 설정하지 않으면 임베디드 데이터베이스를 자동 설정하여 사용한다. 
* 스프링부트는 `url`로부터 데이터베이스의 driverClassName을 추론한다. 따라서 `driver-class-name` 설정은 대부분의 경우 명시하지 않아도 된다.
* DataSource를 생성하기 위해서는, Driver 클래스가 사용가능해야 한다. 예를 들어 `spring.datasource.driver-class-name=com.mysql.jdbc.Driver`와 같이 설정했다면, 해당 클래스를 로드할 수 있어야한다. 

 
 DataSourceProperties 클래스를 통해서 사용가능한 프로퍼티를 확인할 수 있다. 이러한 프로퍼티는 실제 구현과 관계없이 동작하는 표준 옵션이다. 또한 각각의 prefix를 사용하여, 실제 구현에 대한 세부적인 설정을 할 수 있다. 예를 들어 `spring.datasource.hikari.*` 프로퍼티를 사용해서 HikariCP에 대한 설정을 할 수 있다. 자세한건 공식문서를 참고해라
  
## JPA and “Spring Data”

JPA는 객체를 관계형 데이터베이스에 "매핑"할 수있는 표준 기술이다. `spring-boot-starter-data-jpa`는 POM은 다음과 같은 주요 디펜던시를 제공한다. 

* hibernate : 대표적인 JPA 구현체
* spring data jpa : JPA 기반의 repository를 쉽게 구현할 수 있도록 해준다.
* spring orms

### Entity Classes

일반적으로, JPA Entity 클래스를 "persistence.xml" 파일에 지정해야한다. 하지만 스프링 부트에서는 "persistence.xml" 파일이 필요없다. 대신에 "Entity Scanning"이 사용된다. 기본적으로 메인 클래스(@SpringBootApplication 또는 @EnableAutoConfiguration 어노테이션이 붙은 클래스) 아래의 모든 패키지가 스캔된다.

@EntityScan 어노테이션을 통해서 스캐닝 위치를 커스터마이즈 할 수 있다. 

``` java
@Configuration
@EnableAutoConfiguration
@EntityScan(basePackageClasses=City.class)
public class Application {

	//...

}
```
 
### Spring Data JPA Repositories

Spring Data JPA repository는 데이터에 접근하기 위한 인터페이스이다. 인터페이스 메소드 이름을 통해, JPA 쿼리가 자동으로 생성된다. 복잡한 쿼리를 위해서는 @Query 어노테이션을 사용하면 된다.

자동 설정을 사용하는 경우, 메인 클래스 아래의 모든 패키지를 스캔해서 repository를 찾는다. 
자세한 내용은 [Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)를 참고하길 바란다.

### Creating and Dropping JPA Databases

기본적으로 JPA 데이터베이스는 임베디드 데이터베이스를 사용하는 경우에만 자동으로 생성된다. spring.jpa.* 프로퍼티를 사용해서 JPA를 설정할 수 있다. 예를 들어, 테이블을 작성하고 삭제하려면 application.properties에 다음 행을 추가해라

```
spring.jpa.hibernate.ddl-auto=create-drop
```

> Hibernate는 테이블 생성 및 제거를 위한 `hibernate.hbm2ddl.auto` 라는 프로퍼티를 가지고 있다. `spring.jpa.properties.*` 프로퍼티를 통해서 hibernate 관련 네이티브 프로퍼티를 설정할 수 있다. 아래는 Hibernate를 위한 JPA 프로퍼티 설정의 예이다. 아래 예는 Hibernate Entity Manager를 설정할 때 `hibernate.globally_quoted_identifiers` 속성을 true로 세팅한다. 

``` 
spring.jpa.properties.hibernate.globally_quoted_identifiers=true
```

기본적으로 DDL은 ApplicationContext가 시작할 때까지 연기된다. `spring.jpa.generate-ddl`  프로퍼티가 존재하지만, ddl-auto 설정이 더 세밀하기 때문에, 잘 사용되지 않는다.

### Open EntityManager in View

웹 어플리케이션을 실행하고 있다면, 스프링 부트는 "Open EntityManager in View" 패턴을 적용하기 위해, 기본적으로 OpenEntityManagerInViewInterceptor를 등록한다. "Open EntityManager in View" 패턴은 lazy loading을 허용한다. 이런 동작을 원하지 않는다면, `spring.jpa.open-in-view`를 false로 세팅해야 한다.

### Database Initialization

#### Initialize a Database Using JPA

JPA는 DDL 생성을 위한 기능이 있으며, 시작 할 때 실행되도록 설정 할 수 있다. 이것은 두 가지 프로퍼티를 통해 설정된다.

* spring.jpa.generate-ddl (boolean) : 벤더에 독립적이다. 
* spring.jpa.hibernate.ddl-auto (enum) : Hibernate인 경우에만 사용할 수 있으며 좀 더 세부적인 설정이 가능하다. 

#### Initialize a Database Using Hibernate

`spring.jpa.hibernate.ddl-auto` 프로퍼티를 설정할 수 있으며, 표준 Hibernate 프로퍼티 값은 none, validate, update, create 및 create-drop 이다. 스프링부트는 임베디드 데이터베이스(hsqldb, h2 및 derby) 사용여부에 따라 기본값을 결정한다. schema manager가 감지되지 않으면, create-drop를 기본값으로 사용한다. 반면에 schema manager가 감지되면 none이 기본값으로 사용된다. 

in-memory 데이터베이스를 사용하다가, 실제 데이터베이스를 사용하는 경우는 주의해야한다. 

> org.hibernate.SQL를 사용가능하게 함으로써 스키마 생성을 출력할 수 있다. 디버그 모드를 활성화하면 자동으로 수행된다.

또한 클래스 패스의 루트에있는 "import.sql"이라는 이름의 파일은 Hibernate가 직접 스키마를 생성 할 경우 (즉, ddl-auto 프로퍼티가 create 또는 create-drop으로 설정된 경우) 어플리케이션이 시작할 때 실행된다.

개발 초기 단계나 테스트시에는 이 기능이 유용하겠지만, 상용 환경에서는 사용하지 않도록 주의를 기울여여한다. 또한 이 기능은 hibernate의 기능이며 스프링과는 무관하다.

#### Initialize a Database
 
 Spring Boot는 자동으로 DataSource의 스키마 (DDL 스크립트)를 생성하고 초기화한다 (DML 스크립트). 표준 루트 클래스 경로 위치에서 SQL을로드합니다. (DDL : schema.sql, DML : data.sql) 또한 스프링부트는 `schema-${platform}.sql`, `data-${platform}.sql` 파일을 처리한다. 여기서 `${platform}`는 `spring.datasource.platform` 프로퍼티의 값이다. 이는 데이터베이스 별로 스크립트를 작성할 수 있도록 해준다. 예를 들어, `spring.datasource.platform` 프로퍼터를 데이터베이스 벤더 이름 (hsqldb, h2, oracle, mysql, postgresql 등)으로 설정할 수 있다.

> 스프링부트는 임베디드 데이터베이스의 스키마를 자동으로 생성한다. 이 동작은 `spring.datasource.initialization-mode` 프로퍼티를 사용하여 커스터마이징 할 수 있다. 예를 들어, 유형에 관계없이 DataSource를 항상 초기화하려면 `spring.datasource.initialization-mode` 프로퍼티를 `always`로 설정하면 된다.

기본적으로 Spring Boot는 Spring JDBC initializer의 fail-fast 기능을 사용한다. 즉, 스크립트에서 예외가 발생하면 어플리케이션이 시작되지 않는다. `spring.datasource.continue-on-error` 프로퍼티를 설정하여 해당 동작을 조정할 수 있다.
 
 > JPA 기반의 앱에서는 Hibernate가 스키마를 생성하거나 schema.sql을 사용할 수 있도록 선택할 수 있다. 하지만 둘 다 할 수는 없다. 따라서 schema.sql을 사용하는 경우 spring.jpa.hibernate.ddl-auto를 비활성화 해야한다.

## Using H2’s Web Console

H2 데이터베이스는 스프링 부트가 자동 구성 할 수있는 브라우저 기반 콘솔을 제공한다. 다음 조건이 충족되면 콘솔이 자동으로 구성된다.

* 서블릿 기반 웹 애플리케이션을 개발 중이다.
* `com.h2database:h2`가 클래스패스에 존재한다.
* 당신은 스프링 부트의 개발자 도구를 사용하고 있다.

Spring의 개발자 도구를 사용하고 있지 않지만, H2의 콘솔을 사용하려는 경우 `spring.h2.console.enabled` 프로퍼티를 true로 세팅하면 된다. H2 콘솔은 개발 중에만 사용하는게 좋다. production에서 `spring.h2.console.enabled`가 true로 설정되지 않도록 주의해야한다.

## Appendix - 관련 속성

```
# DATASOURCE (DataSourceAutoConfiguration & DataSourceProperties)
spring.datasource.continue-on-error=false # Whether to stop if an error occurs while initializing the database.
spring.datasource.data= # Data (DML) script resource references.
spring.datasource.data-username= # Username of the database to execute DML scripts (if different).
spring.datasource.data-password= # Password of the database to execute DML scripts (if different).
spring.datasource.dbcp2.*= # Commons DBCP2 specific settings
spring.datasource.driver-class-name= # Fully qualified name of the JDBC driver. Auto-detected based on the URL by default.
spring.datasource.generate-unique-name=false # Whether to generate a random datasource name.
spring.datasource.hikari.*= # Hikari specific settings
spring.datasource.initialization-mode=embedded # Initialize the datasource with available DDL and DML scripts.
spring.datasource.jmx-enabled=false # Whether to enable JMX support (if provided by the underlying pool).
spring.datasource.jndi-name= # JNDI location of the datasource. Class, url, username & password are ignored when set.
spring.datasource.name= # Name of the datasource. Default to "testdb" when using an embedded database.
spring.datasource.password= # Login password of the database.
spring.datasource.platform=all # Platform to use in the DDL or DML scripts (such as schema-${platform}.sql or data-${platform}.sql).
spring.datasource.schema= # Schema (DDL) script resource references.
spring.datasource.schema-username= # Username of the database to execute DDL scripts (if different).
spring.datasource.schema-password= # Password of the database to execute DDL scripts (if different).
spring.datasource.separator=; # Statement separator in SQL initialization scripts.
spring.datasource.sql-script-encoding= # SQL scripts encoding.
spring.datasource.tomcat.*= # Tomcat datasource specific settings
spring.datasource.type= # Fully qualified name of the connection pool implementation to use. By default, it is auto-detected from the classpath.
spring.datasource.url= # JDBC URL of the database.
spring.datasource.username= # Login username of the database.
spring.datasource.xa.data-source-class-name= # XA datasource fully qualified name.
spring.datasource.xa.properties= # Properties to pass to the XA data source.


# JPA (JpaBaseConfiguration, HibernateJpaAutoConfiguration)
spring.data.jpa.repositories.enabled=true # Whether to enable JPA repositories.
spring.jpa.database= # Target database to operate on, auto-detected by default. Can be alternatively set using the "databasePlatform" property.
spring.jpa.database-platform= # Name of the target database to operate on, auto-detected by default. Can be alternatively set using the "Database" enum.
spring.jpa.generate-ddl=false # Whether to initialize the schema on startup.
spring.jpa.hibernate.ddl-auto= # DDL mode. This is actually a shortcut for the "hibernate.hbm2ddl.auto" property. Defaults to "create-drop" when using an embedded database and no schema manager was detected. Otherwise, defaults to "none".
spring.jpa.hibernate.naming.implicit-strategy= # Fully qualified name of the implicit naming strategy.
spring.jpa.hibernate.naming.physical-strategy= # Fully qualified name of the physical naming strategy.
spring.jpa.hibernate.use-new-id-generator-mappings= # Whether to use Hibernate's newer IdentifierGenerator for AUTO, TABLE and SEQUENCE.
spring.jpa.mapping-resources= # Mapping resources (equivalent to "mapping-file" entries in persistence.xml).
spring.jpa.open-in-view=true # Register OpenEntityManagerInViewInterceptor. Binds a JPA EntityManager to the thread for the entire processing of the request.
spring.jpa.properties.*= # Additional native properties to set on the JPA provider.
spring.jpa.show-sql=false # Whether to enable logging of SQL statements.

```

## More Reading

* [how to configure a datasource](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#howto-configure-a-datasource)
* [spring data jpa](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
* [h2]()

