---
date: 2024-06-28 13:18:32
layout: post
title: "Spring Cloud Gateway 개요"
subtitle: 우리 집 대문
description: MSA 를 구축할 때 대문 역할의 gateway 를 세운다. gateway 를 구현하기 위한 방법 중 spring-cloud-gateway 를 선택지로 사용할 수 있다. 간략한 개요를 설명한다.
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

# Spring Cloud Gateway

![image-center]({{ '/images/spring-cloud-gateway/개요/Spring Cloud Gateway 1.png' | absolute_url }}){: .align-center}

SpringCloudGateway 는 고성능, 비동기 API 게이트웨이를 구축하기 위한 솔루션이다. 비동기 및 리액티브 프로그래밍 모델을 사용하여 높은 성능과 확장성을 제공하며, 마이크로 서비스 아키텍처에서 서비스 간의 요청을 라우팅하고 필터링하여 보안 및 모니터링 기능을 제공한다. Spring Cloud Gateway 는 `Spring Boot`, `Spring WebFlux`, `Project Reactor` 를 기반으로 구축되었다. 따라서 많은 동기식 라이브러리와 패턴이 적용되지 않을 수 있다.

> - Spring Boot: 빠르고 쉽게 Spring 애플리케이션을 구축할 수 있도록 도와주는 프레임 워크
> - Spring WebFlux: 비동기 및 논블로킹 I/O를 지원하는 웹 프레임 워크(리액티브 프로그래밍 모델 제공)
> - Project Reactor: 리액티브 프로그래밍을 위한 라이브러리(비동기 시퀀스와 스트림 처리하는데 사용)

Netty 런타임이 필요하다.
비동기 이벤트 드리븐 네트워크 애플리케이션 프레임 워크로 높은 성능과 확장성을 제공한다.
전통적인 서블릿 컨테이너나 WAR 파일로 빌드된 경우에는 Spring Cloud Gateway 가 작동하지 않는다.

SpringCloudGateway 는 고성능, 비동기 API 게이트웨이를 구축하기 위한 솔루션.
비동기 및 리액티브 프로그래밍 모델을 사용하여 높은 성능과 확장성을 제공하며
마이크로 서비스 아키텍처에서 서비스 간의 요청을 라우팅하고 필터링하여 보안 및 모니터링 기능을 제공한다.

![image-center]({{ '/images/spring-cloud-gateway/개요/Spring Cloud Gateway 2.png' | absolute_url }}){: .align-center}

> - Route: 게이트 웨이의 기본 구성 요소다. Id, 목적지, 여러 개의 Predicate 및 여러 개의 Filter 로 정의된다. 라우트는 전체 Predicate 가 참이면 매치된다.
> - Predicate: Java 의 FuntionalInterface 이다. 입력 타입은 `ServerWebExchange` 이며 HTTP 요청의 헤더나 파라미터와 같은 요소를 기준으로 매칭할 수 있다.
> - Filter: 특정한 팩토리로 구성된 `GatewayFilter` 의 인스턴스이다. 다운 스트림 요청을 보내기 전/후로 요청&응답을 수정할 수 있다.

# 작동 원리
![image-center]({{ '/images/spring-cloud-gateway/개요/Spring Cloud Gateway 3.png' | absolute_url }}){: .align-center}
- ① 클라이언트 요청 처리: Spring Cloud Gateway 에 요청을 보낸다.  
- ② Gateway Handler Mapping: 클라이언트의 요청이 미리 정의된 라우트 중 하나와 일치하는 지 확인한다. 일치하면 `Gateway Web Handler` 로 전달한다.  
- ③ Gateway Web Handler: 해당 요청에 특화된 필터 체인을 통해 실행한다.  
- ④ Filters1, 2..: 필터는 점선을 기준으로 `pre` 와 `post` 로 나누어 실행되는데, 모든 `pre` 필터 로직이 실행된 후 실제 대상 서비스로 전달되며 프록시 요청이 완료된 후에 `post` 필터 로직이 실행된다. 전/후로 나뉘어 실행하게 되어 실제 순서는 그림처럼 1번부터 8번까지 실행된다.  

# 동작 순서
![image-center]({{ '/images/spring-cloud-gateway/개요/Spring Cloud Gateway 4.png' | absolute_url }}){: .align-center}
클라이언트의 요청으로 시작되는 게이트웨이 흐름에서의 필터들은  `Route Predicate Filter` -> `Global Filter` -> `GatewayFilter Factories` -> `HttpHeadersFilters` 순서로 동작한다. 각 필터들은 어떤 역할을 수행할까?

1. Route Predicate Factories: 요청이 특정 라우트와 매칭되는지 결정하는 조건을 정의한다.
2. Global Filters: 모든 요청와 모든 응답에 대해 수행할 로직을 정의한다.
3. GatewayFilter Factories: 요청이나 응답을 수정하는 로직을 정의한다.
4. HttpHeadersFilters: 요청과 응답 헤더를 처리하는 로직을 정의한다.

# Logging
Gateway 서버를 띄우고 로그를 확인한다.
![image-center]({{ '/images/spring-cloud-gateway/개요/Spring Cloud Gateway 5.png' | absolute_url }}){: .align-center}

SpringCloudGateway 에서 `Gateway Handler Mapping` 의 구현체는 `RoutePredicateHandlerMapping.class` 이며 요청을 적절한 라우트로 매핑하는 역할을 수행한다.
 
- `PathRoutePredicateFactory`: 요청 경로(Path)를 기반으로 라우트를 매칭하는 Predicate 를 정의한다.
- `RoutePredicateHandlerMapping`: 매칭된 라우트를 찾으면 해당 라우트에 정의된 필터 체인을 통해 요청을 처리한다.

위의 Gateway 로 요청을 보내게 되면 설정대로 라우팅을 볼 수 있으며 로그를 자세히 본다면 필터&라우팅 수행 정보를 볼 수 있다.
```java
2024-06-28T15:36:11.081+09:00 DEBUG 49110 --- [scg] [     parallel-1] o.s.c.g.h.RoutePredicateHandlerMapping   : Mapping [Exchange: GET http://localhost:8000/orders/v3] to Route{id='orders', uri=http://localhost:8091, order=0, predicate=Paths: [/orders/**], match trailing slash: true, gatewayFilters=[[[RewritePath /orders/(?<path>.*) = '/${path}'], order = 0]], metadata={}}
```
![image-center]({{ '/images/spring-cloud-gateway/개요/Spring Cloud Gateway 6.png' | absolute_url }}){: .align-center}
추가적으로 콘솔을 확인하면 필터들이 어떤 순서로 실행되는지 확인할 수 있다.  
`pre` -> `post` 순서대로 글로벌필터가 동작하고 있다.