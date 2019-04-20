---
layout: post
title: "Spring Boot Compression" 
author: Gunju Ko
cover:  "/assets/instacode.png" 
categories: [spring, spring-boot]
---

# Spring Boot Compression

### Spring Boot Compression
Spring boot에서는 Http Response Body를 압축할 수 있는 기능을 제공한다. 그 설정은 아래와 같다.

```
server.compression.enabled=false 
server.compression.excluded-user-agents= 
server.compression.mime-types=text/html,text/xml,text/plain,text/css,text/javascript,application/javascript
server.compression.min-response-size=2048
```

* server.compression.enabled (기본값 : false) : 응답 압축을 사용할지 여부
* server.compression.excluded-user-agents (기본값 : 빈 리스트) : 압축에서 제외할 사용자 에이전트 목록
* server.compression.mime-types : 압축해야 하는 MIME 타입 목록
  * 기본 타입 목록에 `application/json` 이 기본 타입 목록에 없다는 점을 주의하자
* server.compression.min-response-size : 압축을 수행할 최소 Content-Length 값

만약에 server.compression.enable를 true로 설정하고 response body의 크기가 2048(기본값)을 넘어서 압축이 되는 경우에는 Response 헤더의 Content-Encoding이 gzip이 된다.

근데 보통 Transfer-Encoding으로 chunked를 많이 사용한다. 하지만 chunked에서는 Content-Length 헤더가 Response에 포함되지 않는다.

그러면 Transfer-Encoding이 chunked인 경우에는 압축이 되지 않는걸까? 결론은 아니다. Transfer-Encoding이 chunked이고 compression이 enabled된 경우에는 mime-types에 해당하는 데이터에 대해서 무조건 압축을 한다.따라서 이 경우에는 min-response-size 설정이 아무런 의미가 없다.

아래의 코드는 Http11Processor#isCompressible 메소드이다. 이 메소드는 Response를 압축할지 여부를 결정한다.

``` java
private boolean isCompressible() {

    // Check if content is not already gzipped
    MessageBytes contentEncodingMB =
        response.getMimeHeaders().getValue("Content-Encoding");

    if ((contentEncodingMB != null)
        && (contentEncodingMB.indexOf("gzip") != -1)) {
        return false;
    }

    // If force mode, always compress (test purposes only)
    if (compressionLevel == 2) {
        return true;
    }

    // Check if sufficient length to trigger the compression
    long contentLength = response.getContentLengthLong();
    if ((contentLength == -1)
        || (contentLength > compressionMinSize)) {
        // Check for compatible MIME-TYPE
        if (compressableMimeTypes != null) {
            return (startsWithStringArray(compressableMimeTypes, response.getContentType()));
        }
    }

    return false;
}
```

코드는 아래와 같은 순서로 실행된다.
* 이미 Content-Encoding이 "gzip"으로 setting된 경우에는 false를 리턴한다.
* compressionLevel이 2인 경우에는 무조건 gzip으로 한다.
  * 0 : off => server.compression.enabled 설정이 false인 경우
  * 1 : on => server.compression.enabled 설정을 true인 경우 
  * 2 : force
* Response의 contentLength값과 compressionMinSize(`server.compression.min-response-size`)를 비교한다.
  * Response의 contentLength 값이 더 큰 경우 : response의 content-type이 압축을 해야하는 타입인지 체크한다. 압축을 해야한다면 true를 리턴하고 그렇지 않으면 false를 리턴한다. 
  * Response의 contentLength 값이 -1인 경우 : response의 content-type이 압축을 해야하는 타입인지 체크한다. 압축을 해야한다면 true를 리턴하고 그렇지 않으면 false를 리턴한다.
  * Response의 contentLength 값이 더 작은 경우 : false를 리턴하고 압축을 하지 않는다. 
  * 압축을 해야하는 타입은 `server.compression.mime-types` 설정에 포함된 타입이다.
   
Transfer-Encoding이 chunked인 경우에는 contentLength가 항상 -1이다. 따라서 content-type이 압축을 해야하는 타입이면 무조건 압축을 한다.

아래는 Apache Tomcat의 공식 문서중에서 Compression 관련된 내용만 뽑은 것이다.
* compressibleMimeType : 압축해야 하는 MIME 타입 목록
* compression : 아래와 같은 값을 가질 수 있다.
  * off : 압축하지 않는다.
  * on : compressionMinSize를 넘고 compressibleMimeType에 해당하는 데이터를 압축한다. Content-Length를 알 수 없는 경우에는 compressibleMimeType만 보고 데이터를 압축한다.
  * force : 모든 데이터를 압축한다. 
  * 기본값은 off이다.
* compressionMinSize : compression이 "on"으로 설정된 경우 Response의 Content-Length가 최소 이 값을 넘어야만 압축을 한다.

공식문서에도 나와있듯이 Content-Length을 알 수 없는 경우에는 압축을 한다고 나와있다. 따라서 Transfer-Encoding이 chunked인 경우에는 compressionMinSize와 관계없이 mimeType만 보고 압축 여부를 결정 한다.

``` java
if (entityBody && (compressionLevel > 0) && sendfileData == null) {
    isCompressible = isCompressible();
    if (isCompressible) {
        useCompression = useCompression();
    }
    // Change content-length to -1 to force chunking
    if (useCompression) {
        response.setContentLength(-1);
    }
}
```

위의 코드를 보면 sendFileData가 널이 아닌 경우엔 isCompressible가 항상 false가 된다는 것을 알 수 있다. isCompressible이 false면 압축이 되지 않는다. sendFile이 뭔지는 잘 모르겠지만 compression보다 더 높은 우선순위를 갖는다고 문서상으로 나와있다. 따라서 sendFile을 사용하는 경우에는 압축이 되지 않는다.

### Transfer-Encoding, Content-Encoding
Transfer-Encoding, Content-Encoding과 관련된 내용은 아래 링크를 참고하길 바란다.

* 콘텐츠 인코딩과 전송 인코딩 / 청크 인코딩 : http://eminentstar.tistory.com/48
* [Transfer-Encoding](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Transfer-Encoding)
* [Content-Encoding](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Content-Encoding)

## 출처
* [Apache Tomcat 8](https://tomcat.apache.org/tomcat-8.0-doc/config/http.html)
