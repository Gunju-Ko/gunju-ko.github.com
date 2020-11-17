---
layout: post
title: "Understanding pagination : REST, GraphQL, Relay" 
author: Gunju Ko
categories: [graphql]
cover:  "/assets/instacode.png"
---

> 이 글은 [Understanding-pagination-rest-graphql-and-relay](https://www.apollographql.com/blog/understanding-pagination-rest-graphql-and-relay-b10f835549e7/) 에 있는 글을 정리한 글입니다.

# Understanding pagination : REST, GraphQL, Relay

### Pagination : What is it for?

* 노출시킬 데이터가 너무 많은 경우엔 일부 데이터만 노출시키는게 사용자 입장에서 더 좋을 수 있다.
* 조회할 데이터가 너무 많은 경우엔 서버에 부담이 될 수 있다. 클라이언트 입장에서도 한번에 너무 많은 데이터를 조회하는게 성능에 영향을 줄 수 있다.

#### Type of pagination UX

* Numbered pages

![그림](https://wp.apollographql.com/wp-content/uploads/2020/03/1_q8ECZQ1pJfJAlrTvbv_cXw.png)

* Sequential pages

![그림](https://wp.apollographql.com/wp-content/uploads/2020/03/1_9LL9gEk99q4K-167dcxjiw.png)

* Infinite scroll

![그림](https://wp.apollographql.com/wp-content/uploads/2020/03/1_OAj7lrCAsGZqNi_qkjEixw.png)

### Implemetation : Numbered pages

* 숫자가 매겨진 페이지는 아래와 같은 SQL 실행으로 구현할 수 있다.

``` sql
// We want page 3, with a page size of 10, so we should
// load 10 items, starting after item 20
SELECT * FROM posts ORDER BY created_at LIMIT 10 OFFSET 20;
```

* 아래와 같은 count 쿼리를 실행해서 데이터의 총 개수나 총 페이지 수를 알아낼 수 있다.

``` sql
SELECT COUNT(*) FROM posts;
```

* 숫자가 매겨진 페이지는 구현이 쉽고 Rest나 Graphql에도 쉽게 매핑된다. 
* Rest에서는 page 쿼리 파라미터로 조회하고자 하는 페이지 정보를 넘길 수 있다.

``` html
<a href="https://meta.discourse.org/latest.json?page=2" target="_blank" rel="noreferrer noopener">https://meta.discourse.org/latest.json?page=2</a>
```

* Graphql에서는 파라미터를 통해 조회하고자 하는 페이지 정보를 넘긴다.

``` graphql
{
  latest(page: 2) {
    title,
    category {
      name
    }
  }
}
```

#### Drawback of page numbering

* 유저가 페이지를 이동하는 동안 새로운 데이터가 삭제될 수 있다. 이런 상황에서 아래와 같은 이슈가 발생할 수 있다.
  * 항목을 건너뛸 수 있다.
  * 중복해서 데이터를 노출시킬 수 있다. 
* 숫자가 매겨진 페이지는 static한 데이터를 쉽게 처리하는데 유용하지만 dynamic 데이터를 처리하는데는 적절하지 않다. dynamic한 데이터의 경우 페이지의 경계점이 계속해서 변하기 때문이다.

![그림](https://wp.apollographql.com/wp-content/uploads/2020/03/1_MxFDviHiryQTjg8_TAObRw-2048x1194.jpg)

### Cursor-based pagination

* 커서 베이스 페이지네이션에서는 목록에서 시작할 위치를 지정하고 가져올 항목 수를 정한다. 이 경우엔 새로운 아이템이 추가되도 다음에 가져올 데이터 항목에 영향을 미치진 않는다. 시작할 위치를 지정하고 있는 포인터를 커서라고 부른다.
* 커서는 데이터의 일부분으로 일반적으로 ID를 사용한다.
* 새로운 데이터를 가져오기 위해선 아래 파라미터가 필요하다.
  * 시작할 커서
  * 새롭게 가져올 데이터의 개수
* 아래는 커서 베이스 페이지네이션의 간단한 예제이다.

``` html
<a href="https://www.reddit.com/?count=25&after=t3_49i88b" target="_blank" rel="noreferrer noopener">https://www.reddit.com/?count=25&after=t3_49i88b</a>
```

### Implementation : Cursor-style pagination

* 목록에서 마지막으로 본 데이터의 타임스탬프인 after 커서가 있다고 가정하고 그 이후에 25개 항목을 가져오는 쿼리는 아래와 같다.

``` sql
SELECT * FROM posts
WHERE created_at < $after
ORDER BY created_at LIMIT $page_size;
```

* 위와 같이 타임스탬프나 메타데이터를 커서로 이용하는 경우엔 데이터 삭제에가 발생해도 큰 문제가 되진 않는다. 반면에 데이터의 ID를 커서로 이용하는 경우 커서가 가르키고 있는 데이터가 삭제되는 경우 문제가 발생할 수 있다.
* REST에서는 다음과 같이 쿼리 파라미터로 커서 정보와 가져올 데이터 개수를 전달해줄 수 있다. 

``` 
GET example.com/posts?after=153135&count=25
```

* API 응답에는 다음 데이터를 가져올 수 있도록 커서 정보를 포함하고 있어야 한다.

``` json
{
  cursors: {
    after: 23492834
  },
  posts: [ ... ]
}
```

* Graphql 에서도 비슷한 방법을 통해 구현이 가능하다. Graphql도 Rest와 마찬가지로 응답에 다음 커서 정보를 포함해야 한다.

``` graphql
{
  latest(after: 153135, count: 25) {
    cursors {
      after
    },
    posts {
      title,
      category {
        name
      }
    }
  }
}
```

### Relay cursor connections

* Relay Cursor Connection은 Graphql 서버에서의 페이지네이션 구현 스펙이다.
* Relay Cursor Connection에 대한 간단한 예제는 아래와 같다.

``` 
{
  user {
    id
    name
    friends(first: 10, after: "opaqueCursor") {
      edges {
        cursor
        node {
          id
          name
        }
      }
      pageInfo {
        hasNextPage
      }
    }
  }
}
```

* Relay에서는 모든 항목이 커서를 가지고 있다. 따라서 원하는 경우 목록의 중간부터 n개의 데이터를 가져오도록 요청을 보낼수도 있다.
* 데이터는 edge로 감싸져서 리턴이된다. edge는 커서와 데이터를 포함하고 있다.
* Relay cursor connection에서 사용하는 용어는 아래와 같다.
  * connection : 페이징치러된 필드 (위 예제에선 friends가 connection이 된다.)
  * edge : edge는 데이터와 메타데이터를 포함하고 있다. 또한 커서 정보를 포함하고 있다.
  * node : 실제 데이터 부분
  * pageInfo : 가져올 페이지 데이터가 더 있는지 클라이언트에게 알린다. Relay 스펙에서는 데이터의 총 수를 알리지 않는다. 
* 위와 같은 정보를 다 제공하는 경우 Relay 클라이언트는 새로운 아이템을 효과적으로 가져올 수 있다. 
* Replay cursor 스펙은 Graphql에서만 국한되지 않는다.

### So What's the best approach?

* 반드시 커서 기반의 페이지네이션이 필요한건 아니다.
* Twitter, Facebook 같은 앱은 커서 기반의 페이지네이션을 사용하여 훌륭한 사용자 경험을 제공하고 있다.
* 숫자가 매겨진 페이지네이션은 구현이 더 단순하다는 장점이 있다. 커서 기반의 페이지네이션은 성능상에 이점이 있으며 더 좋은 사용자 경험을 제공해줄 수 있다.

### 출처 & 더 읽어볼 거리

* 출처 : [Understanding pagination rest graphql and relay](https://www.apollographql.com/blog/understanding-pagination-rest-graphql-and-relay-b10f835549e7/)
* 더 읽어볼 거리 : [Graphql Cursor Connections Specification](https://relay.dev/graphql/connections.htm)


