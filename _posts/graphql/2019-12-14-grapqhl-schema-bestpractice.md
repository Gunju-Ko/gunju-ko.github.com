---
layout: post
title: "Graphql - Best practice for schema design" 
author: Gunju Ko
categories: [graphql]
cover:  "/assets/instacode.png"
---

> 이 글은 [graphql best practice for design](https://graphqlmastery.com/blog/graphql-best-practices-for-graphql-schema-design) 내용을 정리한 글입니다. 
> 저작권에 문제가 있는 경우 "gunjuko92@gmail.com"으로 연락주시면 감사하겠습니다.

# Graphql best practice for Graphql Schema design

출처 
* https://graphqlmastery.com/blog/graphql-best-practices-for-graphql-schema-design

### 1. mutation에서는 input 오브젝트 타입을 사용해라

* mutation에서는 하나의 변수만 사용하는게 좋다.

``` graphql
type Planet implements Node {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime
  name: String
  description: String
  planetType: PlanetTypeEnum
}
input CreatePlanetInput {
  name: String!
  galaxyId: ID!
  description: String
}
type Mutation {
  createPlanet(input: CreatePlanetInput!): Planet!
}
```



* 필수 인자의 경우 non-null modifier(!)를 사용해라

### 2. mutation에서 결과 객체를 리턴해라

* mutation의 결과로 변경된 객체를 리턴하는게 좋다.
  * mutation에서 리턴된 결과로 프론트엔드의 상태를 적절히 업데이트할 수 있다.

``` graphql
type Mutation {
  createPlanet(input: CreatePlanetInput!): Planet!
  updatePlanet(input: UpdatePlanetInput!): Planet!
}
```

### 3. 리스트의 경우 Pagination을 해라

* Pagination은 보안적인 이유로도 매우 중요하며, 서버에서 가져올 레코드의 수를 제한하고 싶을때 필요하다.

``` graphql
type PageInfo {
  endCursor: String
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
}
type ConstellationConnection {
  nodes: [Constellation]
  pageInfo: PageInfo!
  totalCount: Int!
}
```

* 위와 같은 pagination을 cursor based pagination이라고 한다.
* 누군가가 데이터베이스에 엄청난 수의 레코드를 한번에 쿼리할 수 있는데, Pagination는 이를 막아준다.

### 4. nested object를 사용해라

``` graphql
type Planet implements Node {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime
  name: String
  description: String
  planetType: PlanetTypeEnum
  galaxyId: ID!
}
```

* 위처럼 `galaxyId`를 사용하는것보단 아래와 같이 `Galaxy` 타입을 재사용하는게 좋다. `galaxyId` 를 사용하는 경우 `Planet` 의 `Galaxy` 정보를 가져오기 위해 쿼리를 2번 해야한다. 반면에 아래와 같이 `Galaxy` 타입을 사용하면 한번의 쿼리로 `Planet` 의 `Galaxy` 정보를 가져올 수 있다.

``` 
type Planet implements Node {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime
  name: String
  description: String
  planetType: PlanetTypeEnum
  galaxy: Galaxy!
}
```

* nested object를 사용하면 한번의 쿼리로 여러개의 정보를 가져올 수 있다. 또한 [data loader](https://github.com/graphql/dataloader) 를  사용해서 캐시나 배치처리를 할 수 있다.

### 5. Interface를 사용해라

* 인터페이스나 유니온을 사용하면 추상화를 통해 스키마의 복잡성을 줄이고 단순화 할 수 있다. 

### 6. 향후 스키마 변경을 고려해라

* 향후 스키마 변경을 고려해서 설계를 해라. 
* 복잡한 앱을 구현하는 경우 Graphql 스키마 generator를 맹목적으로 사용하지 말아라
* 스키마 디자인에 대해 많이 고민하고 프론트 엔드 요구사항에 맞추는것이 좋다.

### 7. 스키마에 일관된 이름을 사용해라

* 일관된 네이밍 규칙을 가져라
* input 타입의 경우 다음과 같은 네이밍 규칙을 갖는것도 좋다.
  * \<action>\<type>Input
  * \<action>은 Create, Update 또는 Deleted가 올 수 있다.
* Pagination도 일관된 패턴 및 네이밍으로 제공하는게 좋다.
* enum 같은 경우 enum값을 대문자로 하는게 좋다.

