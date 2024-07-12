---
date: 2024-07-12 13:22:16
layout: post
title: "spring-cloud-gateway Filter"
subtitle: 요청에서 응답까지 필터를 세워보자
description: gateway 의 필터를 활용해 다양한 필터들을 활용해보자
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

# Filter

![image-center]({{ '/images/spring-cloud-gateway/filter/scg3-1.png' | absolute_url }}){: .align-center}

요청에 대해 'Filter' 들을 세울 수 있다. 기본적으로 Spring-Cloud-Gateway 에서 제공해주는 유용한 'Filter'들이 많다. 공식 문서를 통해 여러 'Filter'들을 확인할 수 있는데, 커스터마이징한 필터를 구현하지 않고 기본 제공되는 'Filter'만으로 충분할 수 있다. 하지만, 충분하지 않다면 만들어 사용할 수도 있다.

# 필터의 종류
- Global Filter: 모든 라우트에 전역적으로 동작한다.
- Gateway Filter: 특정 라우트에만 동작한다.
- Abstract Gateway Filter: 특정 라우트에만 동작한다. 설정 파라미터를 받을 수 있다.
- Http Headers Filter: HTTP 헤더를 조작하는 데 사용한다.

4개의 필터들을 조합해 사용할 수 있다. 전역적으로 동작하는 필터를 세울 지 특정 라우트에만 동작하는 필터를 세울지 또 설정을 통해 다양한 동작을 수행하도록 구현할 수 있다.

# pre filter, post filter

![image-center]({{ '/images/spring-cloud-gateway/filter/scg3-2.png' | absolute_url }}){: .align-center}

필터는 기본적으로 요청에 대한 전/후 작업을 수행하도록 구현할 수 있다. 전/후를 나누는 기준은 라우팅 하는 micro service 에 요청을 보내기 전/후와 같다. micro service(api) 에 요청을 보내기 전에 실행되는 필터를 'pre' 필터라 하며 micro service(api) 에서의 응답을 받고 난 후 게이트웨이에서 실행되는 필터를 'post' 필터라 한다. 'Filter'는 Global filter, Gatewayfilter, AbstractGatewayFilterFactory, HttpHeadersFilters 가 있다. 각 필터들을 알아본다.  

## Global Filter

사용 여부와 상관없이 모든 라우트에 대해 전역적으로 동작한다. 주요 사용 사례는 로깅, 인증 및 권한 부여, 트래픽 모니터링 및 제한, 공통 헤더 추가 등이 있다.  

```java
package org.springframework.cloud.gateway.filter;  
  
import org.springframework.web.server.ServerWebExchange;  
import reactor.core.publisher.Mono;  
  
public interface GlobalFilter { /*...*/ }
```

구현을 위해 `GlobalFilter` 를 상속하여 구현하면 된다.  

```java
@Override  
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {  
    // pre filter..
    return chain.filter(exchange)  
        .then(Mono.fromRunnable(() -> {  
                // post filter..
            }  
        ));  
}
```

`ServerWebExchange`: 현재 HTTP 요청 및 응답과 관련된 상태와 데이터를 포함하는 객체이다.(WebFlux)  
`GatewayFilterChain`: 필터 체인을 위한 객체  

### Ordered  

여러 개의 'GlobalFilter' 가 있을 때 동작 순서를 정의할 수 있다.  

```java
package org.springframework.core;  
  
public interface Ordered { /*..*/ }
```

해당 인터페이스를 상속해 구현하면 된다. `int getOrder();` 메서드를 구현할 때 순서를 명시하면 된다. 숫자가 낮을수록 우선적으로 필터가 수행된다. 음수도 허용하지만 관례상 0 이상의 숫자로만 구성한다.  

구현 예시  

```java
public class MyCustomGlobalFilter implements GlobalFilter, Ordered {
	// ...
	@Override  
	public int getOrder() {  
	    return 0; // 낮을수록 우선 순위
	}
}
```

유의할 점은 'pre filter' 가 우선적으로 수행된다면 'post filter' 는 그만큼 후순위로 배정되는 것이다. 그림을 다시 확인하자. 'pre&post filter' 의 동작 순서는 아래와 같다.

![image-center]({{ '/images/spring-cloud-gateway/filter/scg3-2.png' | absolute_url }}){: .align-center}

## GatewayFilter  

특정 라우트에 적용되는 필터로 라우트별로 요청을 처리하는 고유한 로직을 정의하는 데 사용된다. 요청 로깅, 헤더 조작, 요청 유효성 검사 등 특정 라우트에 대해 적용해야 하는 작업을 처리할 때 사용한다. 상대적으로 단순한 필터 처리에 적합하다.  

interface GatewayFilter  

```java
package org.springframework.cloud.gateway.filter;  
  
import org.springframework.cloud.gateway.support.ShortcutConfigurable;  
import org.springframework.web.server.ServerWebExchange;  
import reactor.core.publisher.Mono;  
  
public interface GatewayFilter
	extends ShortcutConfigurable { /*...*/ }
```

구현 예시  

```java
@Component
public class CustomGatewayFilter implements GatewayFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // pre filter...
        System.out.println("Custom Gateway Filter - Pre-processing");

        // 필터 체인에 요청을 전달
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            // post filter...
            System.out.println("Custom Gateway Filter - Post-processing");
        }));
    }
}
```

@Component 를 사용해 스프링 빈에 등록해주어야 한다. 'GlobalFilter' 와 마찬가지로 'Ordered' 인터페이스를 구현하여 우선 순위를 설정할 수 있다.(잘 쓰이지는 않는다.) GatewayFilter는 간단한 필터 구현을 도와준다.  

## AbstractGatewayFilterFactory  

Gateway Filter 를 쉽게 만들 수 있도록 도와주는 추상 클래스이다. 복잡한 필터 로직을 캡슐화하고 재사용 가능한 필터를 쉽게 구현할 수 있다. 구성 가능한 필터를 만들어 다양한 라우트에서 재사용할 때 유용하다. 상대적으로 재사용 가능한, 유연한 필터 처리에 적합하다. 설정 클래스를 두어 GatewayFilter 보다는 더 복잡한 필터를 구현할 수 있다.  

추상 클래스 AbstractGatewayFilterFactory\<C\>   

```java
package org.springframework.cloud.gateway.filter.factory;  
  
import org.springframework.cloud.gateway.support.AbstractConfigurable;  
import org.springframework.context.ApplicationEventPublisher;  
import org.springframework.context.ApplicationEventPublisherAware;  
  
public abstract class AbstractGatewayFilterFactory<C>
	extends AbstractConfigurable<C>
	implements GatewayFilterFactory<C>,
	ApplicationEventPublisherAware { /*...*/ }
```

GatewayFilter 를 생성하는 팩토리 클래스이다. 설정 관련 정보를 이너 클래스로 두어 관리할 수 있으며 각 라우트마다 개별적인 설정을 통해 같은 필터라도 다르게 동작할 수 있다.  

```java
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.stereotype.Component;

@Component
public class CustomFilterFactory extends AbstractGatewayFilterFactory<CustomFilterFactory.Config> {
    public CustomFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            // pre filter..
	        
            // 다음 필터로 요청 전달
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                // post filter..
            }));
        };
    }

    public static class Config {
        // 구성 파라미터
        private String name;
        public String getName() { return name; }
        public void setName(String name) { this.name = name; }
    }
}
```

## HttpHeadersFilters  

HTTP 요청과 응답의 헤더를 조작하는 필터이다. 요청과 응답의 헤더를 동적으로 추가, 수정 및 삭제가 가능하다. 해당 필터를 통해 HTTP 헤더를 유연하게 조작할 수 있다. gateway <-> micro server 간의 필요한 정보들은 hedaer 를 통해 주고 받을 수 있으며 중요한 정보들은 client 로 나가기 전 꼭 삭제해주어야 한다.  

```java
package org.springframework.cloud.gateway.filter.headers;  
  
import java.util.List;  
import org.springframework.http.HttpHeaders;  
import org.springframework.web.server.ServerWebExchange;  
  
public interface HttpHeadersFilter { /*...*/ }
```

구현 예시  

```java
@Component
public class CustomResponseHeadersFilter
	implements HttpHeadersFilter {

    @Override
    public HttpHeaders filter(HttpHeaders input, ServerWebExchange exchange) {
        HttpHeaders modifiedHeaders = new HttpHeaders();
        modifiedHeaders.putAll(input);
        modifiedHeaders.add("X-Custom-Response-Header", "CustomValue");
        return modifiedHeaders;
    }

    @Override
    public boolean supports(Type type) {
        return type == Type.RESPONSE;
    }
}
```

`boolean supports();`  메서드의 매개변수인 'Type'을 통해 'REQUEST' or 'RESPONSE' 중 선택적으로 헤더를 조작할 수 있다. 'REQUEST' 는 게이트웨이에서 마이크로 서버에 보내는 request 를 뜻한다.  

## AbstractGatewayFilterFactory-Config.class  

설정 클래스를 통해 동적으로 사용할 수 있는데 각 라우트마다 조금씩 다른 필터를 구현해야 할 때 이 config 클래스를 통해 유연하게 대처할 수 있다.  

예를 들어 같은 로직을 수행하는 필터가 있고 선택적으로 pre, post, pre+post 로 동작하길 원한다면 Config 클래스를 두어 선택적으로 유연한 필터를 설정할 수 있다. 필터는 1개만 구현하면 된다.  

```java
@Component  
public class MyCustomFilter extends AbstractGatewayFilterFactory<MyCustomFilter.Config> {  
  
    @Override  
    public GatewayFilter apply(Config config) {  
        return (exchange, chain) -> {  
            if (config.isPre()) {  
                // pre..  
            }  
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {  
                if (config.isPost()) {  
                    // post..  
                }  
            }));  
        };  
    }
  
	@Data
	@Builder
	@AllArgsConstructor
	public static class Config {  
	    private boolean pre;  
	    private boolean post;  
	}
}
```

class에서 사용한다면 아래와 같이 설정할 수 있다. pre, post 의 값을 각 라우트마다 설정해 하나의 필터를 두고서 여러 동작을 수행하게 할 수 있다.  

![image-center]({{ '/images/spring-cloud-gateway/filter/scg3-3.png' | absolute_url }}){: .align-center}
  
yaml로 한다면 아래와 같이 작성할 수 있다.  

```shell
spring:
  cloud:
    gateway:
      routes:
      - id: a_route
        uri: http://example.org/a
        filters:
        - name: MyCustomFilter
          args:
            pre: true
            post: true
		predicates: 
		- Path=/a
            
      - id: b_route
        uri: http://example.org/b
        filters:
        - name: MyCustomFilter
          args:
            pre: true
            post: false
		predicates: 
		- Path=/b

      - id: c_route
        uri: http://example.org/c
        filters:
        - name: MyCustomFilter
          args:
            pre: false
            post: true
		predicates: 
		- Path=/c
```

# class vs yaml

![image-center]({{ '/images/spring-cloud-gateway/filter/scg3-4.png' | absolute_url }}){: .align-center}

클래스로 작성하던 yaml 로 작성하던 올바르게 잘 동작하면 문제 없다. 다만 어떤 방식이 더 나은지는 놓여진 상황에 맞게 선택하면 될 뿐이다. 컴파일러의 도움과 하나의 클래스에서 모든 설정을 하는 것은 yaml 가 다를게 없다고 판단해 분리했던 것뿐이다.