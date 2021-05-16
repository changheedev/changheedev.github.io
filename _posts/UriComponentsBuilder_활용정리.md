## UriComponentsBuilder

-   스프링에서 URI를 생성할 때, 편리하게 할 수 있도록 도와주는 클래스.
-   Spring Web 의존성이 필요하고 `org.spring.framework.web.util` 패키지에 포함되어 있다.

## 사용법

1.  fromXXX() 메소드를 사용하여 UriComponentsBuilder를 생성한다.
2.  path(), queryParam() 등 각 메소드를 이용하여 URI 컴포넌트들을 세팅한다.
3.  빌드하여 UriComponents 인스턴스를 생성한다.

### 기본 사용예시

```java

UriComponents uriComponents = UriComponentsBuilder
      .fromHttpUrl("http://example.com")
      .path("/api/sample")
      .build();

// 또는

UriComponents uriComponents = UriComponentsBuilder
      .fromUriString("http://example.com/api/sample")
      .build();

```

### 인코딩하여 생성

```java

UriComponents uriComponents = UriComponentsBuilder
      .fromUriString("http://example.com/api/sample/한글")
      .build()
      .encode(); 

// http://www.example.com/api/sample/%ED%95%9C%EA%B8%80

```

### Query 추가하기

```java

UriComponents uriComponents = UriComponentsBuilder
    .fromUriString("http://example.com/api/sample")
    .queryParam("page", 1)
    .queryParam("max-contents", 20)
    .build();

// http://www.example.com/api/sample?page=1&max-contents=20

```

### Uri Path 대치시키기

URI 템플릿 스트링으로 만들고 UriComponentsBuilder의 buildAndExpand()를 활용하면 {}로 감싸진 곳에 값을 대치시킬 수 있다.

```java

final Long resourceId = 1000L;

UriComponents uriComponents = UriComponentsBuilder
    .fromUriString("http://example.com/api/sample/{resourceId}")
    .buildAndExpand(resourceId);

// http://www.example.com/api/sample/1000

```
