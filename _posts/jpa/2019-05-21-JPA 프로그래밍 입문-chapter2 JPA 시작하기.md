---
layout: post
title: "JPA 프로그래밍 입문 - Chapter02 시작하기"
author: Gunju Ko
categories: [jpa]
cover:  "/assets/instacode.png"
---

이 글은 "JPA 프로그래밍 입문 (최범균 저)" 책 내용을 정리한 글입니다.

> 만약 저작권 관련 문제가 있다면 "gunjuko92@gmail.com"로 메일을 보내주시면, 바로 삭제하도록 하겠습니다.

# JPA 프로그래밍 입문 - JPA 시작하기

이 글은 "JPA 프로그래밍 입문 (최범균 저)" 책의 내용을 정리한 글입니다.

### 모델 클래스와 매핑 설정

* @Entity는 해당 클래스가 JPA의 엔티티임을 의미한다.
  * 엔티티는 DB 테이블에 매핑되는 기본 단위이다.
  * @Table 애노테이션을 사용해서 어떤 테이블과 매핑되는지 설정한다.
* JPA의 엔티티에는 식별자가 있다. @Id 애노테이션은 엔티티를 식별할 때 사용할 프로퍼티를 지정할 때 사용한다. 대부분 엔티티 클래스의 식별자는 DB 테이블의 주요키에 매핑한다.
* java.util.Date 타입을 매핑할 때는 @Timestamp 애노테이션을 사용한다.
  * TemporalType.TIMESTAMP : java.sql.Timestamp로 값을 읽어온 뒤 자바 클래스로 변환
* 쿼리를 생성할 때, 테이블에 읽어온 데이터로부터 자바 객체를 생성할 때 매핑정보를 이용한다.
  * 엔티티 클래스의 인자가 없는 기본 생성자를 이용해서 엔티티 객체를 생성
  * @Column 애노테이션 정보를 이용해서 칼럼 값을 필드에 할당 (@Column 애노테이션이 없는 경우는 칼럼명과 필드명을 비교해서 할당)

### JPA 설정

* JPA는 기본적으로 클래스패스에 있는 META-INF/persistence.xml 파일을 설정 파일로 사용한다.
* JPA는 영속 단위 별로 엔티티 클래스를 관리한다. 영속 단위는 \<persistence-unit> 태그로 추가한다.
  * 한 개 이상의 영속 단위를 설정할 수 있으나 보통 하나의 애플리케이션은 하나의 영속단위를 가짐
  * 영속 단위는 JPA가 영속성을 관리할 단위이다. 영속 단위별로 매핑 대상, DB 연결 설정 등을 관리한다.
* JPA는 로컬 트랜잭션과 JTA 기반 글로벌 트랜잭션을 지원한다. 로컬 트랜잭션은 자바 Connection을 이용해서 트랜잭션을 처리한다.
* \<class> 태그로 영속 단위에서 관리할 엔티티 클래스를 지정한다.
* hibernate.dialect 속성은 하이버네이트가 쿼리를 생성할 때 사용할 Dialect 종류를 지정한다.
  * 이 값을 올바르게 지정해야 하이버네이트가 DB 종류에 맞는 쿼리를 생성한다.

### 영속 컨텍스트와 영속 객체 개요

* 엔티티를 영속 컨텍스트(persistence context)로 관리한다. 영속 컨텍스트는 JPA가 관리하는 엔티티 객체 집합이다.
* 영속 컨텍스트에 속한 엔티티 객체를 DB에 반영한다.
* 영속 컨텍스트에 보관된 객체를 영속 객체(persistent context)라고 부른다.
* 영속 컨텍스트는 세션 단위로 생긴다. 즉 세션 생성 시점(EntityManager)에 영속 컨텍스트가 생성되고 세션 종료 시점에 컨텍스트가 사라진다.
* 애플리케이션은 영속 컨텍스트에 직접 접근할 수 없다. 대신 EntityManager를 통해서 영속 컨텍스트와 관련된 작업을 수행한다. 
  * EntityManager를 생성
  * 트랜잭션 시작 : JPA는 트랜잭션 범위에서 DB 변경을 처리하도록 제한하고 있기 때문에 트랜잭션을 먼저 시작해야 새로운 데이터를 추가하거나 기존 데이터를 변경할 수 있다. 수정 기능이 없고 조회만 하는 경우는 트랜잭션이 필요가 없다.
  * EntityManager를 통해 영속 컨텍스트에 객체를 추가하거나 조회
  * 트랜잭션 커밋
  * EntityManager를 닫는다.

> 일반적인 데이터베이스 프로그래밍과 절차가 유사하다. 대신 EntityManager와 JPA가 제공하는 트랜잭션을 사용하는 것이 다르다.

* EntityManager는 EntityManagerFactory를 생성하는 팩토리다. EntityManagerFactory는 영속 단위별로 생성한다. 애플리케이션 초기화 과정에서 한번만 생성하면 된다. 그리고 애플리케이션 종료 시점에 close() 메소드를 호출해서 팩토리를 종료한다. 이 시점에 커넥션 풀과 같은 자원을 반환한다.
* JPA는 트랜잭션을 종료할 때 영속 컨텍스트에 존재하는 영속 객체의 값이 변경되었는지를 검사한다. 만약 값이 바뀌었다면 변경된 값을 DB에 반영하기 위해 update 쿼리를 실행한다. 이를 하이버네이트에서는 *더티 체킹*이라고 부른다.
* JPA는 SQL과 유사한 JPQL을 제공한다. JPQL은 매핑 설정을 담은 클래스를 이용해서 쿼리를 작성한다. 
  * EntityManger.createQuery() 메소드를 이용해서 쿼리 객체를 구한다.
  * 클래스의 매핑 정보를 사용해서 JPQL을 알맞은 SQL로 변환해서 실행하고 SQL 실행 결과로부터 필요한 객체를 생성한다.

### 정리

* JPA를 사용하면 개발자가 직접 작성해야할 쿼리 수가 많이 줄어든다. 이는 개발자가 애플리케이션 로직을 구현하는데 집중할 수 있도록 해준다.
* JPA가 쿼리를 직접 대신 생성하기 때문에 높은 성능이 요구되는 SQL 쿼리가 필요한 기능은 JPA의 쿼리 생성 기능이 오히려 문제를 유발할 수 있다.

