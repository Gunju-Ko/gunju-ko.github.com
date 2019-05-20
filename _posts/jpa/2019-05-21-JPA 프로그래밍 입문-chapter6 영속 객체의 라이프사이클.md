---
layout: post
title: "JPA 프로그래밍 입문 - Chapter06 영속 객체의 라이프 사이클"
author: Gunju Ko
categories: [jpa]
cover:  "/assets/instacode.png"
---

이 글은 "JPA 프로그래밍 입문 (최범균 저)" 책 내용을 정리한 글입니다.

> 만약 저작권 관련 문제가 있다면 "gunjuko92@gmail.com"로 메일을 보내주시면, 바로 삭제하도록 하겠습니다.

# 영속 객체의 라이프사이클

영속 컨텍스트와 연관된 객체를 영속 객체라 부른다. 영속 객체는 영속 컨텍스트와 연관 상태에 따라 아래와 같이 관리됨(managed), 분리됨(detached), 삭제됨(removed) 상태를 갖는다.

![그림](http://lh6.ggpht.com/_NNjxeW9ewEc/TLSCVTaUc4I/AAAAAAAAKbg/JCxadzVSpz8/tmp9D36_thumb1_thumb3.jpg?imgmax=800)

* managed/persistent : 영속 컨텍스트를 통해서 관리되는 상태의 영속 객체. 트랜잭션 범위 안에서 변경하면 트랜잭션 커밋 시점에 변경 내역을 DB에 반영한다.
* detatched : 분리된 상태의 영속 객체는 연관된 영속 컨텍스트가 없다. 따라서 변경 내역이 추적되지 않으므로, 필드를 변경해도 변경 내역이 DB에 반영되지 않는다.
* removed : EntityManager#remove 메소드에 관리 상태의 객체를 전달하면 삭제됨 상태가 된다. 삭제됨 상태가 되면 트랜잭션 커밋 시점에 해당 데이터를 DB에서 삭제한다.

### EntityManager#persist()와 관리 상태 객체

* EntityManager#persist()를 이용해서 영속 컨텍스트에 엔티티 객체를 추가하면, 해당 엔티티 객체는 관리 상태가 된다.
* 영속 컨텍스트는 엔티티 객체를 관리할 때 식별자를 기준으로 각 엔티티를 관리한다.
* EntityManager#persist()를 할 때 주의할 점은 트랜잭션 범위에서 실행하지 않으면 엔티티를 DB에 추가하는 insert 쿼리를 실행하지 않는다는 점이다.
* EntityManager#persist()로 저장한 객체는 관리 상태이므로 변경 내역을 추적한다. 즉 persist() 이후에 엔티티 객체의 상태를 변경하면 트랜잭션을 커밋할 때 변경 내역을 함께 DB에 반영한다. 
* EntityManager#persist()로 저장한 객체는 캐시(영속 컨텍스트)에 보관된다.

### EntityManager#find()와 관리 상태 객체

* EntityManager#find()로 구한 객체도 영속 컨텍스트에 보관되어 관리 상태가 된다. 관리 상태의 영속 객체는 트랜잭션 범위에서 상태가 바뀌면 트랜잭션을 커밋하는 시점에 변경하기 위한 update 쿼리를 실행한다.
* 트랜잭션을 사용하지 않으면 EntityManager를 닫기 전에 객체를 변경해도 변경 내역이 DB에 반영되지 않는다.

### 분리 상태 객체

* 영속 컨텍스트에 보관된 영속 객체는 EntityManager가 종료되면 분리 상태가 된다.
* 분리 상태가 되면 트랜잭션 범위에서 객체의 상태를 변경해도 DB에 반영하지 않는다. 
* EntityManager#detach() 메서드를 사용하면 관리 상태의 객체를 강제로 분리 상태로 만들 수 있다.

### EntityManager#merge()로 분리 상태를 관리 상태로 바꾸기

* EntityManager#merge() 메소드를 사용하면 분리상태의 엔티티를 다시 관리 상태로 만들 수 있다.
* 관리 상태가 되면 트랜잭션 커밋 시점에 변경 내역을 DB에 반영한다.

> EntityManager는 더티 체크를 위해 엔티티 객체의 초기값을 카피해놓는다. 만약 분리된 객체를 merge() 메소드로 다시 영속 객체로 만든 경우, EntityManager는 엔티티 객체의 초기값을 DB에서 읽어(select)온다.

### 삭제 상태 객체

* 관리 상태 영속 객체를 EntityManager#remove() 메서드에 전달하면 삭제 상태로 바뀐다. 트랜잭션 커밋 시점에 DELETE 쿼리를 실행해서 삭제 상태에 해당하는 해당 데이터를 DB에서 삭제한다.
* remove()로 삭제 상태로 바뀐 엔티티를 다시 mege()에 전달하면 익셉션이 발생한다. 

> 삭제 상태의 엔티티라 하더라도 EntityManager 종료 이후 다른 EntityManager의 merge()에 전달할 수 있다. 이 경우 삭제 상태의 엔티티가 관리 상태가 된다. 관리 상태에 해당하는 데이터가 DB에 없으므로 트랜잭션 커밋 시점에 insert 쿼리를 실해해서 DB에 추가한다.

