---
layout: post
title: "JPA 프로그래밍 입문 - Chapter05 EntityManger와 영속 컨텍스트"
author: Gunju Ko
categories: [jpa]
cover:  "/assets/instacode.png"
---

이 글은 "JPA 프로그래밍 입문 (최범균 저)" 책 내용을 정리한 글입니다.

> 만약 저작권 관련 문제가 있다면 "gunjuko92@gmail.com"로 메일을 보내주시면, 바로 삭제하도록 하겠습니다.

# JPA 프로그래밍 입문 - chapter 05

## 1. EntityManager와 영속 컨텍스트

* EntityManager#find()로 읽어온 객체는 영속 객체이다. 영속 객체는 DB에 보관된 데이터에 매핑되는 메모리상의 객체를 의미한다.
* EntityManager#save()를 이용해서 새로운 객체를 추가하면 해당 객체는 영속 객체가 된다.
* EntityManager는 영속 객체를 관리할 때 영속 컨텍스트라는 집합을 사용한다.
  * 일종의 메모리 저장소로서 EntityManager가 관리할 엔티티 객체를 보관한다.
  * 영속 객체를 영속 컨텍스트에 보관한다.
  * EntityManager는 트랜잭션 커밋 시점에 영속 컨텍스트에 보관된 영속 객체의 변경 내역을 추적해서 DB에 반영한다. (insert, delete, update)
  * JPA는 영속 컨텍스트에 보관한 엔티티를 구분할 때 식별자를 이용한다. 즉 영속 컨텍스트는 (엔티티 타입 + 식별자)를 키로 사용하고 엔티티를 값으로 사용하는 데이터 구조를 갖는다.
  * 하이버네이트는 맵을 사용해서 영속 컨텍스트를 구현하고 있다.

### 1.1 영속 컨텍스트와 캐시

* EntityManager 입장에서 영속 컨텍스트는 동일 식별자를 갖는 엔티티에 대한 캐시 역할을 한다.
* 캐시는 영속 컨텍스트와 관련되어 있으므로 EntityManager 객체를 종료하기 전까지만 유효하다.
* 서로 다른 EntityManager 객체는 캐시를 공유하지 않는다.

## 2. EntityManager 종류

#### 애플리케이션 관리 EntityManager

* 애플리케이션에서 직접 EntityManager를 생성하고 종료한다.
* 다음의 두가지를 코드에서 직접 수행한다.
  * EntityManagerFactory 생성과 종료
  * EntityManagerFactory를 이용해서 EntityManager를 생성하고 종료 처리
* EntityManager#close()를 호출해서 EntityManager를 반드시 종료시켜야한다. 그렇지 않으면 자원 누수와 같은 문제가 발생할 수 있다.

#### 컨테이너 관리 EntityManager

* JEE 컨테이너에서 EntityManagerFactory, EntityManager의 라이프 사이클을 관리한다.
* 애플리케이션 코드는 컨테이너가 제공하는 EntityManager를 사용해서 필요한 기능만 구현하면 된다.
* 컨테이너 관리 EntityManager는 @PersistenceContext 애노테이션을 사용하여 구할 수 있다. JEE 컨테이너는 @PersistenceContext 애노테이션이 적용된 필드에 컨테이너가 관리하는 EntityManager 객체를 주입한다.
* JEE 컨테이너는 @Transactional 애노테이션이 적용된 메소드를 트랜잭션 범위에서 실행하는데, @PersistenceContext를 이용해서 주입받은 EntityManager는 JEE가 관리하는 트랜잭션에 참여한다.
  * 애플리케이션 코드에서 트랜잭션을 직접 관리하지 않는다.
* EntityManager 생성과 종료를 컨테이너가 관리하기 때문에 애플리케이션 코드에서 close()를 실행하면 안된다.
  * JPA 스펙에 따르면 컨테이너 관리 EntityManager에 대해 close()0를 호출하는 경우 IllegalArgumentException이 발생하게 되어 있다. 이로 인해 트랜잭션이 롤백 될 수 있다.

> 대부분 스프링 프레임워크가 제공하는 JPA 연동 기능을 사용한다. 스프링 컨테이너도 JEE 컨테이너처럼 @PersistenceContext 애노테이션을 사용해서 EntityManager를 주입하는 기능을 제공한다. 이 기능을 사용하면 스프링이 관리하는 트랜잭션에 연동된 EntityManager를 사용할 수 있으므로 코드에서 직접 트랜잭션을 관리하지 않아도 된다.

## 3. 트랜잭션 타입

JPA는 자원 로컬(Resource Local) 트랜잭션 타입과 JTA 타입의 두 가지 트랜잭션 타입을 지원한다.

#### 자원 로컬(Resource Local) 트랜잭션 타입

* JPA가 제공하는 EntityTransaction을 이용하는 방식이다.
* persistence.xml 파일에 영속 단위의 transaction-type 속성값을 RESOURCE_LOCAL로 지정하면 된다.
* EntityManager#getTransaction() 메소드를 호출하면 트랜잭션을 관리하는 EntityTransaction을 리턴한다.
  * EntityTransaction#begin() : 트랜잭션 시작
  * EntityTransaction#commit() : 트랜잭션 커밋
  * EntityTransaction#rollback() : 트랜잭션 롤백
* EntityManager는 트랜잭션 커밋 시점에 변경 내역을 가지고 수정 쿼리를 실행한다. 따라서 트랜잭션 없이 엔티티 객체를 수정하는 경우 변경 내역이 DB에 반영되지 않는다.

## 4. EntityManager의 영속 컨텍스트 전파

* 보통 서비스는 트랜잭션을 관리하는 주체가 된다. 즉, 서비스 메서드의 시작 시점에 트랜잭션을 시작하고 서비스 메서드의 종료 시점에 트랜잭션을 커밋한다.
  * 고민해야 할 문제는 트랜잭션과 EntityManager의 전파에 대한 것이다.
* 서비스 메소드에서 생성한 EntityManager 객체를 Repository에 전달해줘야 한다. 그래서 서비스 메소드와 서비스 메소드에서 호출한 각 메소드를 한 트랜잭션으로 묶어서 실행해야 한다.
* EntityManager를 전달하는 가장 쉬운 방법은 메서드에 인자로 전달하는 것이다.
* 보통은 ThreadLocal을 사용해서 EntityManager와 트랜잭션을 전파한다.
  * ThreadLocal은 쓰레드 단위로 객체를 공유할 때 사용한다.

### 4.2 컨테이너 관리 EntityManager의 전파

* 컨테이너 관리 EntityManager는 컨테이너가 알아서 EntityManager를 전파해준다.
  * @PersistenceContext 애노테이션을 사용하면 현재 트랜잭션에 참여하는 EntityManager를 구할 수 있다.
  * 컨테이너 관리 EntityManager는 항상 JTA 트랜잭션 타입을 사용해야 한다.
  * @PersistenceContext로 구한 EntityManager는 JTA를 이용한 글로벌 트랜잭션에 참여한다.
* 스프링도 @PersistenceContext 애노테이션을 지원한다. 스프링은 애플리케이션 관리 EntityManager에 대한 @PersistenceContext도 지원하고 스프링이 제공하는 트랜잭션 범위에 묶인다.

