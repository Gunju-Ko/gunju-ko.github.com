---
layout: post
title: "JPA 프로그래밍 입문 - AttributeConverter"
author: Gunju Ko
categories: [jpa]
cover:  "/assets/instacode.png"
---

# AttributeConverter

> 이 글은 "JPA 프로그래밍 입문 - 최범균저"에 있는 내용을 정리한 글 입니다.

AttributeConverter는 주로 다음과 같은 상황에서 사용된다.

* JPA가 지원하지 않는 타입을 매핑
* 두 개 이상의 속성을 갖는 밸류 타입을 한 개 칼럼에 매핑

### 1. JPA가 지원하지 않는 타입을 매핑

``` java
public interface AttributeConverter<X,Y> {
    public Y convertToDatabaseColumn (X attribute);
    public X convertToEntityAttribute (Y dbData);
}
```

* X : 엔티티의 속성에 대응하는 타입
* Y : DB에 대응하는 타입
* convertToDatabaseColumn : 엔티티의 X 타입 속성을 Y 타입의 DB 데이터로 변환한다. 엔티티 속성을 DB에 반영할 때 사용된다.
* convertToEntityAttribute : Y 타입으로 읽은 DB 데이터를 엔티티의 X 타입의 속성으로 변환한다. 엔티티 조회시 DB에서 읽어온 데이터를 엔티티의 속성에 반영할 떄 사용된다.

아래는 VARCHAR 타입의 테이블 컬럼을 InetAddress 타입 속성으로 사용할 때 필요한 AttributeConverter이다.

```  java
@Convert
public class InetAddressConverter implements AttributeConverter<InetAddress, String> {

    @Override
    public String convertToDatabaseColumn(InetAddress attribute) {
        if (attribute == null) {
            return null;
        }
        return attribute.getHostAddress();
    }

    @Override
    public InetAddress convertToEntityAttribute(String dbData) {
        if (dbData == null || dbData.isEmpty()) {
            return null;
        }
        try {
            return InetAddress.getByName(dbData);
        } catch (UnknownHostException e) {
            throw new RuntimeException("InetAddressConverter fail to convert : " + dbData);
        }
    }
}
```

* @Converter 애노테이션은 해당 클래스가 AttributeConverter를 구현한 클래스임을 지정한다.
  * authApply 속성값을 `true`로 하는 경우 @Convert 애노테이션으로 AttributeConverter를 지정하지 않아도 해당 타입의 속성에 대해 자동으로 적용된다. 다만 자동으로 적용하려면 AttributeConverter를 `persistence.xml`에 등록해야 한다.
* InetAddressConverter를 사용하려면 아래와 같이 @Convert 애노테이션을 사용하면 된다.

``` java
public class AuthLog {
  
  	@Convert(convert = InetAddressConverter.class)
    private InetAddress inetAddress;
}
```

> AttributeConverter를 이용하면 DBCode를 Enum으로 변환하는 작업을 쉽게 할 수 있다. 자세한 내용은 아래 링크를 참고하길 바란다.
>
> * [우아한 형제들 - Legecy DB의 JPA Entity Mapping](https://woowabros.github.io/experience/2019/01/09/enum-converter.html)

### 2.두 개 이상의 속성을 갖는 밸류 타입을 한 개 칼럼에 매핑

``` java
public class Money {
    private Double value;
    private String currency;

    public Money(Double value, String currency) {
        this.value = value;
        this.currency = currency;
    }

    public Double getValue() {
        return value;
    }

    public void setValue(Double value) {
        this.value = value;
    }

    public String getCurrency() {
        return currency;
    }

    public void setCurrency(String currency) {
        this.currency = currency;
    }

    @Override
    public String toString() {
        return value.toString() + currency;
    }
}
```

* Money 타입을 DB에 보관할 때 "1000KRW"이나 "100USD"와 같은 문자열로 저장된다.
* Money를 한 개의 칼럼에 매핑하므로 @Embeddable을 사용할 수 없다.
* Money를 한 개의 컬럼에 매핑하기 위한 AttributeConverter 구현체는 아래와 같다.

``` java
@Converter(autoApply = true)
public class MoneyConverter implements AttributeConverter<Money, String> {

    @Override
    public String convertToDatabaseColumn(Money attribute) {
        if (attribute == null) {
            return null;
        }
        return attribute.toString();
    }

    @Override
    public Money convertToEntityAttribute(String dbData) {
        if (dbData == null) {
            return null;
        } else {
            String value = dbData.substring(0, dbData.length() - 3);
            String currency = dbData.substring(dbData.length() - 3);
            return new Money(Double.valueOf(value), currency);
        }
    }
}
```

* autuApply 설정을 true로 설정했으므로 JPA 프로바이더는 MoneyConverter를 자동으로 적용한다.



