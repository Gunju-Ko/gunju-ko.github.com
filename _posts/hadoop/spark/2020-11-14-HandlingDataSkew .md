---
layout: post
title: "Handling data skew in apache spark" 
author: Gunju Ko
categories: [spark]
cover:  "/assets/instacode.png"
---

# Handling Data Skew in Apache Spark

이 글인 [Handling Data Skew In Apache Spark](https://itnext.io/handling-data-skew-in-apache-spark-9f56343e58e8) 글을 정리한 글입니다.

![그림](https://miro.medium.com/max/1400/1*SICi8EJBHIpWzeQvBb1Jog.png)

## Introduction

* 병렬 시스템에서 가장 잘 알려진 문제 중 하나는 data skewness 이다.
* Apache Spark에선 join, groupBy, orderBy와 같은 데이터 파티션을 변경하는 트랜스포메이션에 의해 data skewness가 발생한다.
* data skewness를 해결하는 여러가지 방법이 존재한다.

## Broadcast join

data skeness를 해결하는 방법 중 하나는 Broadcast 조인을 사용하는것이다. 

#### Broadcast join

* 작은 DataFrame을 클러스터의 전체 워커 노드에 복제한다.
* 각 워커는 독립적으로 조인 작업을 수행할 수 있다.
* Job이 고르게 분산되기 때문에 대부분의 경우 성능이 좋다.
* 아래처럼 힌트를 통해 broadcast 조인을 수행할 수 있다. 혹은 데이터프레임의 크기가 `spark.sql.autoBroadcastJoinThreshold` 보다 작은 경우엔, 스파크가 자동으로 broadcast 조인을 수행한다.

``` scala
df
 .join(broadcast(geoDataDf), exprGeo, “left”)
```

* 한계
  * 테이블의 크기가 8GB 이상이거나, 로우의 개수가 512000000 이상인 경우엔 브로드캐스트 될 수 없다.
  * 높은 메모리 사용 : 만약에 100개의 익스큐터가 실행되고, 브로드캐스트 되는 데이터프레임의 크기가 1GB인 경우엔 총 100GB의 추가적인 메모리가 사용된다. 또한 브로드캐스트되는 테이블의 크기가 증가하면 메모리가 부족하게 되어 예외가 발생할 수 있다. 이런 경우엔 Spark의 익스큐터, 드라이버 메모리 크기 설정을 변경해줘야한다.
  * 성능 : 브로드캐스트 연산시 모든 익스큐터는에 테이블을 복제한다. 따라서 테이블의 크기가 커지거나 익스큐터 수가 증가하게 되면 브로드캐스트 비용도 증가하게 된다.

#### Sort Merge Join 

* 2단계로 일어난다.
  * 가장 먼저 조인키를 이용해서 정렬을 한다.
  * 파티션에 있는 정렬된 데이터를 머지한다. 조인키에 따라 같은 값을 가진 로우를 조인한다.
* 노드간 데이터 공유시 네트워크 비용이 발생한다.
* Spark 2.3에서 Merge-Sort 조인이 기본 조인 알고리즘으로 사용된다.
* 조인키에 대해 동일한 값을 갖는 모든 행이 동일한 파티션에 저장되어야 한다. 만약 조인키에 대해 동일한 값을 갖는 행이 동일한 파티션에 위치하지 않은 경우 셔플이 일어날 수 있다.

* 단점
  * 만약 조인 컬럼의 값이 대부분 같은 경우 레코드가 고르게 분산되지 않을 수 있으며(data skeness), 이는 성능 저하로 이어진다.

> Broadcast 조인 말고도 `Key Salting` 방식을 이용해서 data skeness 문제를 해결할 수 있다. 관련해서는 아래 출처를 참고하길 바란다.

## Summary

* data skew를 해결하는 방법은 데이터의 크기, 다양성, 클러스터 설정 등과 같은 많은 요소에 따라 달라진다. 따라서 해결을 위한 간단한 방법은 없다. 데이터와 코드에 따라 적절한 방법을 선택해서 해결해야 한다. 
* 결정을 내리는데 도움이 되는 사실은 아래와 같다.
  * Scalability : 브로드캐스트 조인은 확장성이 좋지 않다. 브로드캐스트 되는 테이블의 크기 제한이 있다.
  * Reliability : 브로드캐스트 되는 테이블의 사이즈가 증가하거나 변화되는 경우엔 신뢰할 수 없다. 테이블의 크기가 증가하면 OOM이 발생할 수 있다. 반대로 테이블의 크기가 감소하면 클러스터의 리소스를 불필요하게 너무 많이 사용하게 된다.
  * Performance : 브로드캐스트 조인이 적절하게 사용된다면 매우 좋은 성능을 보인다.
  * Maintainability : 브로드캐스트 조인의 경우 추가적인 코드가 필요하지 않으므로 유지보수가 쉽다. 조인시 `broadcast` 힌트만 추가해주면 된다.

### 출처

* https://itnext.io/handling-data-skew-in-apache-spark-9f56343e58e8
* https://medium.com/datakaresolutions/optimize-spark-sql-joins-c81b4e3ed7da
