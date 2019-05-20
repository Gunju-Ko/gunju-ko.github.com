---
layout: post
title: "JPA 프로그래밍 입문 - Chapter03 엔티티"
author: Gunju Ko
categories: [jpa]
cover:  "/assets/instacode.png"
---

이 글은 "JPA 프로그래밍 입문 (최범균 저)" 책 내용을 정리한 글입니다.

> 만약 저작권 관련 문제가 있다면 "gunjuko92@gmail.com"로 메일을 보내주시면, 바로 삭제하도록 하겠습니다.

## 엔티티

JPA에서 엔티티는 영속성을 가진 객체로 가장 중요한 타입이다. JPA의 엔티티는 DB 테이블에 보관할 대상이 된다.

JPA는 두가지 방법으로 엔티티를 설정한다.

* @Entity 애노테이션 사용
* XML 매핑 설정 사용

@Entity 애노테이션을 주로 사용한다.

### 1. @Entity 애노테이션과 @Table 애노테이션

#### @Entity

* EntityManager는 @Entity 애노테이션을 적용한 클래스를 이용해서 SQL을 작성할 때 클래스 이름을 테이블 이름으로 사용

#### @Table

* 클래스 이름과 테이블 이름이 다른 경우에 사용 (name 속성으로 지정)
* 쿼리를 생성할 때 name 속성을 사용
* catalog 속성 : 테이블의 카탈로그를 지정
* schema 속성 : 테이블의 스키마를 지정

### 2. @Id 애노테이션

* 엔티티는 식별자를 갖는다. JPA는 식별자를 설정하기 위해 @Id 애노테이션을 사용한다.
* @Id 애노테이션을 필드에 적용하면 모든 필드가 매핑 대상이 된다.
* @Id 애노테이션을 자바빈 방식의 프로퍼티 getter 메소드에 적용하면 모든 자바빈 메소드 형식의 프로퍼티가 매핑 대상이 된다.
* @Id 애노테이션이 적용된 필드값은 EntityManager#find() 메소드에서 엔티티 객체를 찾을 때 식별자로 사용된다.
* 보통 DB 테이블의 주요키 칼럼에 매핑할 대상에 @Id 애노테이션을 적용한다.

### 3. @Basic 애노테이션과 지원 타입

* @Basic 애노테이션은 보통 생략한다.
* JPA는 영속 대상 필드가 int, long, String과 같은 기본 타입일 때 @Basic 애노테이션을 사용한다.
  * int, long, double
  * Integer, Long, Double
  * BigInteger, BigDecimal
  * String
  * Calendar
  * Timestamp
  * enum 타입
  * byte[], Byte[], char[], Character[]
* 날짜와 시간 타입은 @Temporal 애노테이션과 함께 사용한다. @Temporal 애노테이션을 적용한 필드나 프로퍼티가 어떤 SQL 타입에 매핑되느냐에 따라 알맞은 TemporalType을 값으로 설정한다.
  * TemporalType.DATE : java.sql.Date에 해당한다. 
  * TemporalType.TIME : java.sql.Time에 해당한다.
  * TemporalType.TIMESTAMP : java.sql.Timestamp에 해당한다.

> 하이버네이트 5.2의 경우 LocalDateTime을 지원한다.

* 열거 타입에 대한 매핑은 @Enumerated 애노테이션을 사용한다.
  * EnumType.STRING : 매핑된 칼럼이 열거 타입의 상수 이름을 값으로 가질때 사용
  * EnumType.ORDINAL(기본값) : 매핑된 칼럼이 열거 타입의 상수 이름 대신 인덱스를 저장하는 경우 사용
  * EnumType.STRING을 사용할 것을 권장함 (유지보수가 편함)
* @Column 애노테이션과 이름 지정
  * 필드 / 프로퍼티의 이름과 테이블의 칼럼 이름이 다를때 사용한다.
  * insertable 속성 : false로 지정하는 경우 엔티티 객체를 DB에 저장할 때 insert 쿼리해서 해당값을 제외한다.
  * updatable 속성 : false로 지정하는 경우 update 쿼리의 수정 대상에서 제외된다.
  * 읽기 전용 필드를 지정할 수 있다. (insertable, updatable을 모두 false로 지정)
* @Access 애노테이션은 필드 접근 방식에서 특정 영속 대상에 대해서만 프로퍼티 접근 방식을 사용해야 하는 경우나 그 반대의 경우에 사용한다.
  * AccessType.PROPERTY : 프로퍼티를 통해 접근 (getter / setter 이용)
  * AccessType.FIELD : 필드를 통해 접근
* 필드 접근 타입을 사용하는데 영속 대상이 아닌 필드가 존재하는 경우 transient 키워드를 사용해서 영속 대상에서 제외시킬 수 있다. 혹은 @Transient 애노테이션을 사용해도 된다.

### 4. 엔티티 클래스의 제약 조건

* 기본 생성자가 있어야 한다.
* 엔티티는 클래스여야 한다. 
* 엔티티 클래스는 final이면 안된다. 지연 로딩과 같은 기능을 제공하기 위해 엔티티 클래스를 상속받은 프록시 객체를 사용하기 때문
* 엔티티의 메소드나 영속 대상 필드도 final이면 안된다.
* JPA 프로바이더가 사용하는 캐시 구현 기술이 Serializable 인터페이스를 요구하는 경우 엔티티 클래스가 Serializable 인터페이스를 상속해야한다.

### EntityManager 기본 기능

* public \<T> T find(Class\<T> entityClass, Object primaryKey)
  * entityClass : 엔티티 타입
  * primaryKey : 식별자 (@Id 애노테이션으로 매핑한 영속 대상의 값을 사용)
  * 존재하지 않는 경우 null을 리턴
* public \<T> T getReference(Class\<T> entityClass, Object primaryKey
  * 프록시 객체를 리턴하며, getReference() 메소드 호출시에는 쿼리를 실행하지 않고 최초로 데이터에 접근할 때 쿼리를 실행. 이 때 데이터가 존재하지 않는 경우 EntityNotFoundException이 발생
  * 엔티티 클래스가 final이면 클래스 상속을 이용한 프록시 객체를 생성할 수 없다. 이 경우 하이버네이트는 getReference() 메소드에서 프록시를 리턴하는 대신 바로 select 쿼리를 실행한다. 이 때 데이터가 존재하지 않으면 EntityNotFoundException이 발생한다.
* 프록시 객체가 데이터를 가져오기 위해 쿼리를 실행할 때 EntityManager 세션이 유효한 범위여야 한다. 세션이 종료된 후에 프록시 객체에 데이터를 처음 사용하는 경우 쿼리를 실행하기 위한 커넥션을 구할 수 없기 때문에 LazyInitializationException이 발생한다.
* public void persist(Object entity)
  * 트랜잭션 범위 내에서 실행해야 한다.
  * insert 쿼리를 실행하는 시점은 엔티티 클래스의 식별자를 생성하는 규칙에 따라 달라진다.
    * 직접 식별자를 설정하는 경우 : 트랜잭션 커밋하는 시점에 insert 쿼리가 실행된다.
    * @GeneratedValue(strategy = IDENTITY) : persist()를 실행하는 시점에 insert 쿼리가 실행된다. IDENTITY는 데이터베이스의 식별 칼럼을 사용해서 엔티티의 식별자를 생성한다는 것을 의미한다. (MySQL의 경우 auto increment 칼럼에 해당) persist() 메소드를 실행하는 시점에 식별자 생성을 위해 insert 쿼리를 실행한다. JPA는 쿼리 실행 후 생성된 식별자를 엔티티에 반영한다.
    *  @GeneratedValue(strategy = SEQUENCE) : 트랜잭션을 커밋하는 시점에 insert 쿼리가 실행된다. 시퀀스를 사용하는 경우 시퀀스만 사용해서 식별자를 생성할 수 있으므로 persist() 시점에 insert 쿼리를 실행하지 않고 시퀀스 관련 쿼리만 실행
    * @GeneratedValue(stategy = TABLE) : 트랜잭션 커밋 시점에 insert 쿼리가 실행된다. persist() 시점에는 식별자를 생성하기 위한 쿼리만 실행된다.
* public void remove(Object entity) 
  * 엔티티를 제거할 때 사용한다.
  * 트랜잭션 커밋 시점에 delete 쿼리를 실행한다.
* 엔티티 수정
  * 트랜잭션 범위에서 엔티티 객체의 상태가 변경되면 이를 트랜잭션 커밋 시점에 반영한다. JPA는 영속 컨텍스트에 속한 객체의 변경 여부를 추적하며, 트랜잭션 커밋 시점에 변경된 객체가 존재하면 알맞은 update 쿼리를 실행해서 변경 내역을 DB에 반영한다.

> 영속 컨텍스트는 세션 범위의 엔티티 객체를 관리하고, 각 엔티티 객체를 구분할 때 식별자를 사용한다. persist() 메소드를 실행하면 엔티티 객체를 영속 컨텍스트에 보관하는데 새로 추가한 엔티티 객체와 구분하기 위해 식별자가 필요하다. 그런 이유로 JPA는 persist()를 이용해서 엔티티 객체를 영속 컨텍스트에 추가할 때 식별자 생성기를 이용해서 식별자를 생성한다.

### 5. 식별자 생성 방식

* 직접 할당 방식 : 별도의 생성자 생성 규칙이 존재하는 경우에 적합하다.

#### 식별 칼럼 방식

JPA가 식별 칼럼을 이용해서 식별자를 생성하는 경우 (MySQL의 auto_increment)

* @GeneratedValue(strategy = IDENTITY)
* persist() 메소드 호출시 즉시 insert 문이 실행된다. 이는 식별자를 생성하기 위함이다.

#### 시퀀스 사용 방식

DB 시퀀스를 이용하는 경우

* @GeneratedValue(strategy = SEQUENCE)
  * generator : 식별자 생성기의 name을 지정한다.
* @SequenceGenerator : 시퀀스 기반의 식별자 생성기를 설정
  * name : 시퀀스 생성기의 이름을 지정한다. @GeneratedValue에서 이 이름을 사용한다.
  * sequenceName : 식별자를 생성할 때 사용할 시퀀스 이름을 지정
  * allocationSize : 시퀀스에서 읽어온 값을 기준으로 몇 개의 식별자를 생성할 지 결정한다. 
* persist() 시점에 insert 쿼리를 실행하지 않고 시퀀스 관련 쿼리만 실행함

``` java
@Entity
@Table(name = "hotel_review")
public class Review {
    @Id
    @SequenceGenerator(
      name = "review_seq_gen",
      sequenceName = "hotel_review_seq",
      allocationSize = 1
    )
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator="review_seq_gen")
    private Long id;
```

> allocationSize를 1로 설정해야 하는 이유
>
> 예를 들어 allocationSize를 50(기본값)으로 설정해서 사용하는 경우 다음과 같은 방식으로 식별자를 생성한다.
>
> * 최초에 DB 시퀀스에서 값을 구한다. 이 값을 식별자 구간의 시작 값으로 사용한다. 처음 값은 1이기 때문에 시작 값은 1이다.
> * DB 시퀀스에서 한 번 더 값을 구하고 값은 2이다. 식별자 구간의 끝 값은 2이다.
> * 엔티티를 추가할 때 식별자 1, 2를 차례로 사용한다.
> * 식별자 구간 끝에 도달했으로 시퀀스에서 다음 값인 3을 구한다. 3을 식별자 구간의 끝 값으로 사용하고 여기서 49(50 -  1)를 뺀 -46을 시작 값으로 사용한다.
> * 문제는 "-46 ~ 3"을 식별자 구간으로 사용하기 때문에 1, 2 식별자를 사용하는 경우 충돌이 발생한다.
>
> 이런 문제를 해결하려면 DB 시퀀스의 증가 크기를 allocationSize와 동일하게 잡으면 된다. 하지만 식별자를 메모리에서 증가시키기 때문에 두 개 이상의 JVM을 사용하면 식별자 값이 순차적으로 증가하지 않는 문제가 발생하게 된다. 애초에 이런 문제가 발생하지 않도록 하려면 매번 시퀀스에서 값을 읽어오도록 allocationSize를 1로 설정해야한다.

#### 테이블 사용방식

* 모든 DB에서 동일한 방식으로 적용 가능
* 테이블은 2개의 칼럼으로 구성됨
  * 식별자를 구분할 때 사용할 주요키 칼럼
  * 식별자로 사용할 숫자를 보관할 칼럼
* @TableGenerator 애노테이션을 이용해서 테이블 생성기를 설정해야 한다.
  * name : 테이블 생성기의 이름을 지정한다. @GeneratedValue의 generator 속성값으로 사용한다.
  * table : 식별자를 생성할 때 사용할 테이블을 지정한다.
  * pkColumnName : 주요키 칼럼을 지정한다.
  * pkColumnValue : 이 테이블 생성기가 주요키 칼럼에 사용할 값을 지정한다. 각 엔티티마다 다른 값을 사용해야 한다. 엔티티 클래스의 이름을 사용하면 편리하다.
  * valueColumnName : 생성할 식별자를 갖는 칼럼을 지정한다.
  * initialValue : 식별자의 초기값을 지정한다. 식별자 생성용 테이블에 해당 레코드가 없을 때 이 값을 기준으로 다음 식별자를 생성한다.
  * allocationSize : 식별자의 할당 크기를 지정한다. 이 값을 1보다 크면 메모리에서 이 값만큼 식별자를 증가시키므로 다중 JVM 노드에서 운영한다면 이 값을 1로 지정해야 한다.
* persist()를 실행하는 시점에는 식별자를 생성하기 위한 쿼리만 실행하고, 트랜잭션이 커밋하는 시점에 insert 쿼리를 실행한다.

``` java
@Entity
public class City {
	@Id
  @TableGenerator(name = "idgen", table="id_gen", pkColumnName="entity", pkColumnValue="city", valueColumnName="nextid", initialValue=0. allocationSize=1)
  @GeneratedValue(generator="idgen")
  private Long id;

```

> 식별자 생성용 테이블에서 식별자를 조회할 때 for update를 사용한다. 이는 테이블에 대해 행 단위로 동시 접근을 막기 위한 선점 잠금을 수행한 것이다. 하이버네이트는 여러 쓰레드가 식별자 생성을 요청할 때 발생할 수 있는 식별자 충돌 문제를 해결하기 위해 선점 잠금을 수행한다.

