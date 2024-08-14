---
date: 2024-07-17 13:43:23
layout: post
title: "Spring Boot RestClient 개요"
subtitle: Spring Boot 에서 HTTP 요청 보내기
description: Spring Boot 에서 쉽고 간단하게 HTTP 요청을 보낼 수 있다. 스프링6 이상에서 사용이 가능하다. 간단하게 알아본다!
image: '/images/restClient/개요/RestClientBack.png'
optimized_image: '/images/restClient/개요/RestClientBack.png'
category: RestClient
tags: 
  - RestClient
  - spring boot
  - API
author: lkdcode
paginate: false
---

Spring-Boot-Application 에서 외부 API 요청이 필요한 경우에 사용할 수 있다. 모던하고 직관적인 API 를 제공해 쉽게 HTTP 요청을 보낼 수 있다.  [🔗RestClient-docs](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)  

1. 동기식 클라이언트: 요청은 동기식이다.  
2. 유연한 API: 직관적이고 읽기 쉬운 방식으로 HTTP 요청을 작성할 수 있다.  
3. 다양한 HTTP 라이브러리 지원  
4. 메시지 변환: JSON, XML 과 같은 형식을 자바 객체로 변환할 수 있다.  
5. 쓰레드 안전성: 한번 만들어진 RestClient 는 여러 쓰레드에서 안전하게 사용할 수 있다.  



# 사용해보기  
별다른 의존성 없이 스프링 6(부트3이상)이상이면 사용이 가능하다. 아래와 같이 바로 인스턴스를 얻을 수 있다. 다양한 설정들을 적용하여 생성해주는 `.builder()` 도 제공한다.  

```java
RestClient defaultClient = RestClient.create();

RestClient customRestClient = RestClient.builder()
        .baseUrl("https://api.example.com")  // 기본 URL 설정
        .defaultHeader("Authorization", "Bearer token")  // 기본 헤더 설정
        .build();

RestClient customClient = RestClient.builder()
  .requestFactory(new HttpComponentsClientHttpRequestFactory())
  .messageConverters(converters -> converters.add(new MyCustomMessageConverter()))
  .baseUrl("https://example.com")
  .defaultUriVariables(Map.of("variable", "foo"))
  .defaultHeader("My-Header", "Foo")
  .requestInterceptor(myCustomInterceptor)
  .requestInitializer(myCustomInitializer)
  .build();
```

## 요청 보내기  
RestClient 의 인스턴스를 얻은 후 `.uri()` 설정을 해주고 `.retrieve()` 메서드를 통해 요청을 바로 보낼 수 있다. 요청 이후 `.body()` 메서드를 통해 클래스를 넣어주면 본문을 다양한 타입으로 변환해준다.   

```java
String result = restClient.get() 
    .uri("https://example.com") 
    .retrieve() 
    .body(String.class); 

System.out.println(result); 
```

요청 uri 를 설정할 때 변수를 사용할 수도 있다.  

```java
int id = 42;
restClient.get()
	.uri("https://example.com/orders/{id}", id) ....
```

헤더를 설정할 땐 `.header()` 메서드를 통해 헤더를 추가할 수 있다.  

```java
restClient.get()
.uri("")
.header("hello", "world") ....
```

## content-type
`RestClient` 이후에 메서드 호출로 여러개를 보면 자연어처럼 읽혀서 설정이 쉬운 걸 느끼게 된다. JSON 외에도 `MediaType` 의 여러 타입들이 가능하다.  

```java
Pet pet = ...
ResponseEntity<Void> response = restClient.post()
  .uri("https://petclinic.example.com/pets/new")
  .contentType(APPLICATION_JSON)
  .body(pet)
  .retrieve()
  .toBodilessEntity();
```

## 에러 핸들링  

RestClient 는 `retrieve()` 메서드 호출 이후 결과에 대한 에러 핸들링을 제공한다. `.onStatus()` 메서드를 통해 HttpStatusCode 로 핸들링 하도록 제공해준다. `.onStatus()` 메서드는 `Predicate<HttpStatusCode>` 와 `ErrorHandler` 를 매개변수로 받는다.   

![image-center]({{ '/images/restClient/개요/rc0-1.png' | absolute_url }}){: .align-center}

구현 예시 코드  
```java
String result = restClient.get()
  .uri("https://example.com/this-url-does-not-exist")
  .retrieve()
  .onStatus(HttpStatusCode::is4xxClientError, (request, response) -> {
      throw new MyCustomRuntimeException(response.getStatusCode(), response.getHeaders())
  })
  .body(String.class);
```

## 고급 시나리오 핸들링  

`.exchange()` 메서드를 활용해 요청과 응답에 대해 직접 접근하여 더 많은 제어를 필요로할 때 사용한다. (`.onStatus()` + `.body()` 는 상태 코드를 기반으로 특정 처리)  

```java
Pet result = restClient.get()
  .uri("https://petclinic.example.com/pets/{id}", id)
  .accept(APPLICATION_JSON)
  .exchange((request, response) -> {
    if (response.getStatusCode().is4xxClientError()) {
      throw new MyCustomRuntimeException(response.getStatusCode(), response.getHeaders());
    }
    else {
      Pet pet = convertResponse(response);
      return pet;
    }
  });
```

## Delete, body..?

> DELETE 요청에 대한 본문(body) 데이터 포함 가능한가?  
> 공식 사양은 금지하지는 않는데.. 권장되지는 않는 것 같다.(~~찜찜~~)  
> [🔗RFC 7231](https://www.rfc-editor.org/rfc/rfc7231#section-4.3.5)  

`.delete();` 메서드를 호출하면 요청 본문을 작성할 수 없다. 대신 `.method()` 를 호출해 `.body()` 를 호출하도록 할 수 있다. (RestClient.create().delete()~~.body();~~ 이렇게 안 됨)  

```java
RestClient.create()  
    .method(HttpMethod.DELETE)  
    .uri(uri)  
    .contentType(MediaType.APPLICATION_JSON)  
    .body() ...
```
