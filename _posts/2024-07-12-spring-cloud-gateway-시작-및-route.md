---
date: 2024-07-12 13:03:24
layout: post
title: "spring-cloud-gateway 시작 및 Route"
subtitle: 대문은 위치를 안내해주지. 클래스로 커스터마이징까지!
description: gateway 가 받은 요청은 각각의 micro-server 에 전달된다. 어떻게 설정할 수 있을까?
image: '/images/spring-cloud-gateway/개요/scg-thumb2.png'
optimized_image: '/images/spring-cloud-gateway/개요/scg-thumb2.png'
category: spring-cloud-gateway
tags: 
  - spring
  - gateway
  - spring cloud gateway
author: lkdcode
paginate: false
---

![image-center]({{ '/images/spring-cloud-gateway/시작 및 route/scg2-1.png' | absolute_url }}){: .align-center}
# 의존성
- gradle

```shell
implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
```

- 그 외 (선택)

```shell
# build.gradle
implementation 'org.springframework.boot:spring-boot-starter-webflux'
implementation 'io.netty:netty-resolver-dns-native-macos:4.1.68.Final:osx-aarch_64'

dependencyManagement {  
    imports {  
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"  
    }  
}
```

- `spring-cloud-starter` 를 포함하였지만, 게이트웨이를 비활성화 시키고 싶다면

```shell
spring.cloud.gateway.enabled=false
```

# 간략 요약
![image-center]({{ '/images/spring-cloud-gateway/시작 및 route/scg2-2.png' | absolute_url }}){: .align-center}

## spring webFlux
Spring5 에서 도입된 '리액티브 웹 프레임워크' 비동기 및 논블로킹 방식으로 웹 애플리케이션을 개발할 수 있도록 설계되었다.
1. 리액티브 프로그래밍: 데이터 스트림과 이벤트를 비동기적으로 처리하는 프로그래밍
2. 논블로킹 I/O: 요청과 응답을 처리할 때, 작업이 완료될 때까지 '쓰레드'를 블로킹하지 않는다. (대신 이벤트 루프를 사용)
3. 스케일링: 적은 리소스로도 높은 동시성을 처리할 수 있다.

## netty
비동기 이벤트 드리븐 네트워크 애플리케이션 프레임워크이다.

1. 비동기 이벤트 드리븐: 모든 I/O 작업은 비동기적으로 수행.
2. 논블로킹 I/O
3. 확장성이 유연하다.


spring-cloud-gateway 는 netty 와 webFlux 로 구성되어있다.

# Flow

![image-center]({{ '/images/spring-cloud-gateway/시작 및 route/scg2-3.png' | absolute_url }}){: .align-center}

기존의 동기 시스템에선 요청에 대한 응답을 하나의 쓰레드가 책임지면서 수행하게 된다.

![image-center]({{ '/images/spring-cloud-gateway/시작 및 route/scg2-4.png' | absolute_url }}){: .align-center}

하지만 spring-cloud-gateway 는 event loop 를 도입해 논블로킹 방식으로 일을 수행하게 되며 위의 그림은 간략한 묘사다. 콜백 이후 일손이 쉬고 있는 쓰레드가 다시 작업을 수행한다.

# Route Predicate Factories
![image-center]({{ '/images/spring-cloud-gateway/시작 및 route/scg2-9.png' | absolute_url }}){: .align-center}

요청에 대해 다양한 조건을 기반으로 라우트를 구성해주는 역할을 수행한다. 함수형 인터페이스의 Predicate 로 각 라우트가 매핑이 되는지 판단한다. Java 클래스 형태로 구현되며, 각각의 Predicate Factory 는 특정한 조건을 표현한다. Spring Cloud Gateway 에서 제공해주는 Factory 들을 사용할 수 있고 필요하다면 사용자가 커스터마이징 하여 사용할 수 있다.  

구현체 클래스는 아래와 같다.  
![image-center]({{ '/images/spring-cloud-gateway/시작 및 route/scg2-2-1.png' | absolute_url }}){: .align-center}

## Route Predicate Factories example
Route Predicate 를 설정하기 위해서는 크게 `class` 와 `yaml` 로 설정하는 방법이 있다. 2가지 방법을 보고 구조에 대한 차이를 확인한다.

#### *Path Route Predicate Factory*: 특정 URL 패턴과 일치하는 요청 라우팅  

`class`  

```java
.route("path_route", r -> r.path("/get")
	.uri("http://example.org"))
```

`yaml`  

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: path_route
        uri: http://example.org
        predicates:
        - Path=/get
```

#### *Method Route Predicate Factory*: 특정 HTTP 메서드(GET,POST 등)에 일치하는 요청 라우팅  

`class`

```java
.route("method_route", r -> r.method("GET")
    .uri("http://example.org"))
```

`yaml`

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: http://example.org
        predicates:
        - Method=GET
```

#### *Header Route Predicate Factory*: 특정 헤더와 일치하는 요청을 라우팅

`class`  

```java
.route("header_route", r -> r.header("X-Request-Id", "123")
    .uri("http://example.org"))
```

`yaml`  

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: http://example.org
        predicates:
        - Header=X-Request-Id, 123
```

#### *Query Route Predicate Factory*: 특정 쿼리 파라미터를 가진 요청 라우팅  

`class`  

```java
.route("query_route", r -> r.query("foo", "bar")
    .uri("http://example.org"))
```

`yaml`  

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: http://example.org
        predicates:
        - Query=foo, bar
```

이외에도 공식 문서에서 제공해주는 여러 필터링들을 확인할 수 있고 상황에 따라 사용자가 원하는 라우팅 조건을 추가할 수 있다.  
커스터마이징을 원하면 `AbstractRoutePredicateFactory` 추상 클래스를 구현하면 된다.  
![image-center]({{ '/images/spring-cloud-gateway/시작 및 route/scg2-6.png' | absolute_url }}){: .align-center}


# class 로 구현하기
`SpringCloud docs` 에 `SpringCloudGateway` 를 설명할 때 아래와 같이 등록하는 방법을 간단히 보여준다.
```java
@SpringBootApplication
public class DemogatewayApplication {
	@Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
		return builder.routes()
			.route("path_route", r -> r.path("/get")
				.uri("http://httpbin.org"))
			.route("host_route", r -> r.host("*.myhost.org")
				.uri("http://httpbin.org"))
			.route("rewrite_route", r -> r.host("*.rewrite.org")
				.filters(f -> f.rewritePath("/foo/(?<segment>.*)", "/${segment}"))
				.uri("http://httpbin.org"))
			.build();
	}
}
```
`SpringCloudGateway docs` 에서는 `yaml` 로 설정하는 방법을 알려준다. `yaml` 로 설정한다면 컴파일러의 도움을 받을 수 없다. 또 위와 같이 하나의 클래스에서 여러 도메인들에 대한 라우팅을 설정하는 것은 `yaml` 과 다를게 없다. 컴파일러의 도움을 받을 수 있으면서 도메인별로 나누어 설정을 하기 위해 `RouteLocator` 를 구현하여 설정한다.  

# RouteLocator

![image-center]({{ '/images/spring-cloud-gateway/시작 및 route/scg2-7.png' | absolute_url }}){: .align-center}
![image-center]({{ '/images/spring-cloud-gateway/시작 및 route/scg2-8.png' | absolute_url }}){: .align-center}  

`RoutePredicateHandlerMapping` 클래스는 라우트를 찾고 매칭된 라우트를 찾으면 해당 라우트에 정의된 필터 체인을 통해 요청을 처리한다. 내부적으로 `RouteLocator` 를 통해 라우트를 찾는다. 스프링 컨테이너를 통해 `RouteLocator` 객체를 등록하면 라우트를 등록할 수 있다.

## Route Config
`interface`
```java
@FunctionalInterface
public interface RouteLocatorBuilderProvider {  
    void build(RouteLocatorBuilder.Builder routes);  
}
```
`config`
```java
@Configuration  
@RequiredArgsConstructor  
public class RoutesConfig {  
    private final List<RouteLocatorBuilderProvider> routeLocatorBuilderProviders;  
  
    @Bean  
    public RouteLocator routeLocator(RouteLocatorBuilder builder) {  
        final RouteLocatorBuilder.Builder routes = builder.routes();  
        routeLocatorBuilderProviders.forEach(provider -> provider.build(routes));  
        return routes.build();  
    }  
}
```
각 도메인 별로 `RouteLocatorBuilderProvider` 의 구현체가 되어 스프링 컨테이너에 등록을 하고 `RouteLocatorBuilder.Builder` 를 통해 `Filter` 와 `URI` 등 설정을 정의한다.

## Route Domain (micro service)
위의 인터페이스를 구현하여 구현해본다면 아래의 예시처럼 구현할 수 있다.
```java
@Service  
@RequiredArgsConstructor  
public class OrdersRouteLocatorBuilderProvider implements RouteLocatorBuilderProvider {  
    private static final String ORDERS_URI = "http://localhost:8091";  
    private final L2Filter l2Filter;  
  
    @Override  
    public void build(RouteLocatorBuilder.Builder routes) {  
        routes.route("orders1", r -> r.path("/orders/jwt")  
            .filters(f -> f  
                .rewritePath("/orders/(?<path>.*)", "/${path}")  
                .filter(l2Filter.apply(new L2Filter.Config(true, true)))  
            )  
            .uri(ORDERS_URI));  
  
        routes.route("orders2", r -> r.path("/orders/**")  
            .filters(f -> f  
                .rewritePath("/orders/(?<path>.*)", "/${path}")  
            )  
            .uri(ORDERS_URI));  
    }  
}
```

도메인에 라우팅을 설정한다.
- `"orders1", "orders2"`:  라우트의 `id`를 설정한다.
- `r -> r.path(String... patterns)`: 라우트에 매핑될 경로를 설정한다.
- `filters(Function<GatewayFilterSpec, UriSpec> fn)`: `custom filter`, `GatewayFilter Factories` 등 필요한 필터들을 선택적으로 등록할 수 있다.
- `L2Filter`: 사용자가 등록한 커스터마이징 필터다. (다음 챕터에서 다룬다.)

라우트 해야할 앤드-포인트가 많고 적용해야 할 필터가 다르다면 순서를 메서드 단위로 구현하여 api 문서(restdocs, swagger 등)처럼 보이도록 구현할 수도 있다. 주의할 점은 `route-id` <U>중복과 누락</U>된 라우팅 설정이다. 기본적으로 모든 http메서드에 대해 라우트해주지만 http 메서드를 명시해 특정 지을 수 있다. `spring-security`의 필터처럼 라우트 순서가 중요하다. 위의 route path 매핑 순서가 바뀐다면 `"/orders/**"` 에 모든 경로가 포함되어 `"/orders/jwt"` 는 라우트가 실행되지 않는다. 또 `yaml` 등을 통해 local 환경의 URI, cloud 환경의 pods URI 등 설정이 가능하다.
```java
//..example
@Override  
public void build(RouteLocatorBuilder.Builder routes) {  
    this.routes = routes;
    apiUsersV1CompanyFindEmail();
    apiUsersV1CompanyLogout();
    /*...*/
}

private void apiUsersV1CompanyFindEmail() {  
    routes.route(ROUTE_ID_PREFIX + "-email-post", r -> r.path(apiUsersV1CompanyFindEmailPath)  
            .and().method(POST)  
            .uri(COMPANY_API));  
}  
  
private void apiUsersV1CompanyLogout() {  
    routes.route(ROUTE_ID_PREFIX + "-logout-post", r -> r.path(apiUsersV1CompanyLogoutPath)  
            .and().method(POST)  
            .filters(f -> f  
                .filter(jwtRemovePostFilter.apply((Void) null)))  
            .uri(COMPANY_API));  
}
```

위의 설정 일부를 `yaml` 로 구현한다면 `RewritePath`는 아래와 같이 작성할 수 있다. 컴파일러의 도움을 받긴 힘들다.
```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: rewritepath_route
        uri: https://example.org
        predicates:
        - Path=/red/**
        filters:
        - RewritePath=/red/?(?<segment>.*), /$\{segment}
```