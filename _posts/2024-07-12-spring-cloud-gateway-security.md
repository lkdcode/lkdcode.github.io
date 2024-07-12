---
date: 2024-07-12 13:32:14
layout: post
title: "spring-cloud-gateway security"
subtitle: 게이트웨이에 시큐리티 설정하기
description: 도메인별로 시큐리티 설정을 분리할 수 있다. spring-security-6 의 다중필터처럼 적용해보자
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

# Security
![image-center]({{ '/images/spring-cloud-gateway/security/scg4-1.png' | absolute_url }}){: .align-center}

spring-cloud-gateway 의 security 는 가장 앞에 있게 된다. 로그를 통해 확인할 수 있는데 모든 요청에 대해 시큐리티부터 동작하는 것을 확인할 수 있다.  

- 유효한 권한의 요청  
![image-center]({{ '/images/spring-cloud-gateway/security/scg4-2.png' | absolute_url }}){: .align-center}

- 유효하지 않은 권한의 요청  
![image-center]({{ '/images/spring-cloud-gateway/security/scg4-3.png' | absolute_url }}){: .align-center}

# 도메인별 시큐리티 (인증&인가)  
![image-center]({{ '/images/spring-cloud-gateway/security/scg4-4.png' | absolute_url }}){: .align-center}

Route 를 각 도메인마다 분리했던 것처럼 시큐리티의 설정들도 분리하여 적용할 수 있다. 마찬가지로 Bean 으로 등록하여 각각의 도메인에 맞게 시큐리티를 동작시키는 것이다.  

인증 인가를 위해 컨테이너에게 등록할 클래스는 아래와 같다. 아래의 클래스에 각 도메인의 설정들을 등록해주면 된다.  
![image-center]({{ '/images/spring-cloud-gateway/security/scg4-5.png' | absolute_url }}){: .align-center}

gateway security 의 기본 설정 클래스는 아래와 같이 구현할 수 있는데 각 도메인별로 등록할 타입은 `ExchangeProvider` 이다. 시큐리티에 공통적으로 적용될 BaseSecurity 도 있다.  

```java
@Configuration  
@EnableWebFluxSecurity  
@RequiredArgsConstructor  
public class GatewaySecurity {  
    private final BaseSecurity baseSecurity;  
    private final List<ExchangeProvider> authorizeExchangeProvider;  
  
    @Bean  
    public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) throws Exception {  
        baseSecurity.with(http);  
        authorizeExchangeProvider.forEach(exchange -> exchange.with(http));  
  
        return http  
                .securityMatcher(ServerWebExchangeMatchers.pathMatchers("/**"))  
                .authorizeExchange(exchange -> exchange.anyExchange().denyAll())  
                .build()  
                ;  
    }  
}
```

ExchangeProvider 클래스는 ServerHttpSecurity 에 등록할 설정들을 추상화해 제공한다. 각 도메인별로 구현하기만 하면 된다.  

```java
public abstract class ExchangeProvider {  
  
    public final void with(ServerHttpSecurity http) {  
        http.authorizeExchange(exchange -> {  
            exchange.pathMatchers(allowRoutes()).permitAll();  
            exchange.pathMatchers(authenticatedRoutes()).authenticated();  
            authorityRoutes().customize(exchange);  
        });  
    }  
  
    protected abstract Customizer<ServerHttpSecurity.AuthorizeExchangeSpec> authorityRoutes();  
  
    protected abstract String[] allowRoutes();  
  
    protected abstract String[] authenticatedRoutes();  
}
```

예를 들어 `Orders` 라는 도메인의 시큐리티를 이렇게 등록할 수 있다. 허용할 목록과 인증이 필요한 목록 등을 구성할 수 있다. 또 특정 권한이 필요한 요청도 설정할 수 있다.  

```java
@Service  
public class OrdersExchangeProvider extends ExchangeProvider {  
  
    private static final String[] ALLOW_LIST = {"/orders/", "/orders/v3", "/orders/jwt",};  
    private static final String[] AUTHENTICATED = {"/orders/authenticated",};  
  
    @Override  
    protected Customizer<ServerHttpSecurity.AuthorizeExchangeSpec> authorityRoutes() {  
        return auth -> auth.pathMatchers("/orders/admin/**").hasAuthority("ADMIN");  
    }  
  
    @Override  
    protected String[] allowRoutes() {  
        return ALLOW_LIST;  
    }  
  
    @Override  
    protected String[] authenticatedRoutes() {  
        return AUTHENTICATED;  
    }  
}
```

위의 설정은 spring-security-6 의 httpSecurity 설정과 매우 비슷하며 multi-filter 처럼 등록하는 개념이라고(?) 볼 수도 있다.  

# 또 CORS  
![image-center]({{ '/images/spring-cloud-gateway/security/scg4-6.png' | absolute_url }}){: .align-center}

한 가지 주의할 점이 있는데 spring-cloud-gateway 에서 cors 이다. 각각의 마이크로 서버에 security 가 존재할 수 있고 그 안에 cors 관련 설정이 있을 수 있다. 하지만 cors 설정은 중복 설정할 수 없으므로 scg 혹은 micro server 중 한 곳에서만 이루어져야 한다.