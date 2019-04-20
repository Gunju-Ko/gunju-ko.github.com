---
layout: post
title: "Spring Data JPA를 활용한 페이징 처리" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring]
---

# Spring Data JPA를 이용한 페이징 처리

## PagingAndSortingRepository
PagingAndSortingRepository는 CrudRepository를 상속하고 있는 인터페이스이다. PagingAndSortingRepository는 페이징 처리를 위한 메소드를 제공하고 있다.

``` java
public interface PagingAndSortingRepository<T, ID extends Serializable>
  extends CrudRepository<T, ID> {

  Iterable<T> findAll(Sort sort);

  Page<T> findAll(Pageable pageable);
}
```

페이지의 크기가 20인 유저 페이지의 2번째 페이지를 가져오려면 아래와 같은 코드를 작성하면 된다. Pageable 인터페이스의 구현체인 PageRequest를 사용했다. 생성자의 첫번째 인자로는 가져올 페이지, 두번째 인자로는 페이지의 크기를 넘겨주면 된다. 참고로 페이지는 0부터 시작한다.

``` java
PagingAndSortingRepository<User, Long> repository = // … get access to a bean
Page<User> users = repository.findAll(new PageRequest(1, 20));
```

## Special Parameter Handling
JPA는 Pageable, Sort와 같은 특별한 타입의 파라미터를 인식하여 페이징 처리나 정렬을 한다.

``` java
Page<User> findByLastname(String lastname, Pageable pageable);

Slice<User> findByLastname(String lastname, Pageable pageable);

List<User> findByLastname(String lastname, Sort sort);

List<User> findByLastname(String lastname, Pageable pageable);
```

* 첫번째 방법은 Pageable 객체를 쿼리 메소드로 전달하여 쿼리에 페이징을  동적으로 추가할 수 있다. Page를 통해서 사용 가능한 데이터의 총 개수 및 전체 페이지 수를 알수 있다. 이는 카운트 쿼리를 실행함으로써 전체 페이지 수 및 전체 데이터의 개수를 알아낸다. 
* 카운트 쿼리가 많은 비용이 드는 경우에는 Slice를 사용하면 된다. Slice는 다음 Slice가 존재하는지 여부만 알고 있다. 전체 데이터 셋의 크기가 큰 경우에는 Slice를 사용하는게 성능상 유리하다.
* Pageable을 통해서도 정렬을 할 수 있지만, 정렬만 하는 경우에는 그냥 Sort를 사용하는게 좋다.
* 결과를 단순히 List로 받을 수 있다. 이 경우엔 Page 인스턴스를 생성하기 위한 메타데이터가 생성되지 않기 때문에 카운트 쿼리가 실행되지 않는다. 단순히 주어진 범위내의 엔티티를 검색하기 위한 쿼리만 실행된다.

> Note
> * Page를 사용하면 페이지의 총 수를 알아내기 위해서 추가적으로 카운트 쿼리가 실행된다. 기본적으로 카운트 쿼리는 실제로 실행되는 쿼리에서 파생된다.

## Paging Web support
Spring Data Jpa에서는 HandlerMethodArgumentResolver 인터페이스 구현체를 제공함으로써 컨트롤러 메소드의 파라미터로 Sort, Pageable 등을 사용할 수 있도록 한다.

### PageableHandlerMethodArgumentResolver
아래의 예제는 컨트롤러의 메소드 파라미터로 Pageable를 사용한 예이다. PageableHandlerMethodArgumentResolver는 컨트롤러 메소드에 Pageable 타입의 파라미터가 존재하는 경우, 요청 파라미터를 토대로 PageRequest 객체를 생성한다.

``` java
@Controller
@RequestMapping("/users")
class UserController {

  private final UserRepository repository;

  UserController(UserRepository repository) {
    this.repository = repository;
  }

  @RequestMapping
  String showUsers(Model model, Pageable pageable) {
    model.addAttribute("users", repository.findAll(pageable));
    return "users";
  }
}
```

위의 예제 처럼 컨트롤러의 메소드 파라미터로 Pageable 타입이 존재하는 경우, Spring MVC가 요청 파라미터로부터 Pageable 객체를 생성하려고 시도한다. 이 때 사용되는 기본 설정은 아래와 같다.

Pageable 객체를 생성하기 위해서 사용되는 요청 파라미터
* page :  가져올 페이지 (기본값 : 0)
* size : 페이지의 크기 (기본값 : 20)
* sort :  정렬 기준으로 사용할 속성으로 기본적으로 오름차순으로 한다. 정렬 기준 속성이 2개 이상인 경우에는 sort 파라미터를 2개 이상 넣어주면된다. 예) sort=firstname&sort=lastname,asc.

Pageable 파라미터에 @PageableDefault 어노테이션을 사용하면 디폴트 값을 변경할 수 있다.

커스터마이징을 해야하는 경우는 PageableHandlerMethodArgumentResolverCustomizer  또는 SortHandlerMethodArgumentResolverCustomizer 인터페이스를 구현하는 빈을 등록하면 된다. customize() 메소드를 통해서 설정을 변경할 수 있다. 아래는 간단한 예제이다.

``` java
@Bean 
SortHandlerMethodArgumentResolverCustomizer sortCustomizer() {
    return s -> s.setPropertyDelimiter("<-->");
}
```

기본 MethodArgumentResolver의 설정을 변경하는 것으로 충분하지 않다면, SpringDataWebConfiguration를 상속해서 pageableResolver() 또는 sortResolver() 메소드를 오버라이딩 하면된다. 

하나의 요쳥으로 두 개 이상의 Pageable 또는 Sort 객체를 생성하는 경우에, @Qualifier 어노테이션을 사용해서 하나씩 구별할 수 있다. 그리고 요청 파라미터는 ${qualifier}_. 접두사가 있어야한다. 아래와 같은 경우 foo_page, bar_page 등의 파라미터가 있어야한다.

``` java
String showUsers(Model model,
      @Qualifier("foo") Pageable first,
      @Qualifier("bar") Pageable second) { … }
```

### Hypermedia support for Pageables
Spring HATEOAS에서 PagedResources 클래스를 제공한다. PagedResources 클래스는 Page 객체의 메타데이터와 내용 뿐만 아니라 이전 페이지나 다음 페이지에 대한 링크를 포함하고 있다. 

Page 객체를 PagedResources로 변환하는 작업은 ResourceAssembler 인터페이스의 구현체인 PagedResourcesAssembler를 통해서 할 수 있다. 아래는 간단한 예제이다.

``` java
@Controller
class PersonController {

  @Autowired PersonRepository repository;

  @RequestMapping(value = "/persons", method = RequestMethod.GET)
  HttpEntity<PagedResources<Person>> persons(Pageable pageable, PagedResourcesAssembler assembler) {
    Page<Person> persons = repository.findAll(pageable);
    return new ResponseEntity<>(assembler.toResources(persons), HttpStatus.OK);
  }
}
```

toResources() 메소드를 호출하면 아래와 같은 일이 발생한다.
* Page 객체의 내용은 PagedResources 객체의 내용이 된다.
* PageMetadata 객체가 Page 객체의 정보를 기반으로 생성되고 PagedResources 객체에 포함된다.
* Page의 상태에 따라서 PagedResources의 prev, next 링크가 생성된다. 링크는 다음 또는 이전 페이지에 대한 URL이며 페이징과 관련된 파라미터를 포함하고 있다.

아래의 간단한 예를 들어보자면, 만약 데이터베이스에 30명의 유저정보가 있다고 가정해보자. 만약 유저 정보를 가져오는  `GET : http://localhost:8080/persons` 요청을 보냈다면 아래와 같은 응답이 올 것이다.

``` java
{ "links" : [ { "rel" : "next",
                "href" : "http://localhost:8080/persons?page=1&size=20 }
  ],
  "content" : [
     … // 20 Person instances rendered here
  ],
  "pageMetadata" : {
    "size" : 20,
    "totalElements" : 30,
    "totalPages" : 2,
    "number" : 0
  }
}
```

assembler가 올바른 Link가 생성된 것을 볼 수 있다. 기본적으로 Link는 현재 호출된 컨트롤러 메소드를 기반으로 생성이 된다. 

## 출처
공식문서 : https://docs.spring.io/spring-data/jpa/docs/2.0.6.RELEASE/reference/html/
더 읽을거리 : https://dzone.com/articles/pagination-and-sorting-with-spring-data-jpa
