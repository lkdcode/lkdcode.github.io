---
title: Spring-Security-6 다중 필터
image:
  path: /images/multi-filter/multi-filter.png
  thumbnail: /images/multi-filter/multi-filter.png
categories:
  - spring-security-6
  - filter
tags:
  - multi-filter
  - spring-security-6
  - httpSecurity
last_modified_at: 2024-05-06T18:07:52-05:00
---

spring-security-6 framework 에서 제공하는 filter 를 정책에 맞게 다양하게 다루는 방법을 소개한다. 기존에 방식과 달라진 방법을 소개하고 다중 필터를 적용하고 결과를 확인한다.

# HttpSecurity

spring-security-6 이전 버전에서는 security filter 를 설정하기 위해서 `WebSecurityConfigurerAdapter` 를 상속하는 설정 클래스를 구현했는데 이는 deprecated 되었다.(사용자가 컴포넌트 기반 보안 구성으로 전환하도록 권장하기 때문.)

만약 기존에 아래와 같이 설정을 했다면,
```java
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((authz) -> authz
                .anyRequest().authenticated()
            )
            .httpBasic(withDefaults());
    }
}
```

앞으로 아래와 같이 설정하기를 권장한다.

```java
@Configuration
public class SecurityConfiguration {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((authz) -> authz
                .anyRequest().authenticated()
            )
            .httpBasic(withDefaults());
        return http.build();
    }
}
```

참고: <a href="https://spring.io/blog/2022/02/21/spring-security-without-the-websecurityconfigureradapter" target="_blank">Spring Security without the WebSecurityConfigurerAdapter</a>


# 시나리오

여러 도메인들이 존재하고 RESTApi 로 클라이언트의 요청을 받아 수행한다 가정한다. 모두에게 허용되는 앤드-포인트가 있고 인증이 필요한 앤드-포인트가 있을 때, 각 도메인마다 다른 filter 를 적용시키는 방법을 소개한다.

# 예시 도메인 Api

두 개의 도메인이 RESTApi 를 가지고 있다. `Apple` 과 `Banana` 도메인이 존재하고 문자열을 리턴하는 RestController 가 있다고 가정한다. 

아래는 Apple 도메인과 Banana 도메인의 RESTApi 앤드-포인트다.

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/apple")
public class AppleApi {

    @GetMapping("/get")
    public String getAppleGetApi() {
        System.out.println("AppleApi.getAppleApi");
        return "I'm Apple api";
    }

    @GetMapping("/secret")
    public String getAppleSecretApi() {
        System.out.println("AppleApi.getAppleSecretApi");
        return "Invalid";
    }
}
```

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/banana")
public class BananaApi {

    @GetMapping("/get")
    public String getBananaGetApi() {
        System.out.println("BananaApi.getBananaApi");
        return "I'm Banana Api";
    }

    @GetMapping("/secret")
    public String getBananaSecretApi() {
        System.out.println("BananaApi.getBananaSecretApi");
        return "Invalid";
    }
}
```

위의 Api 네이밍으로 유추할 수 있듯이 접근 권한은 아래와 같다.
- `/apple/get` : 모든 접근이 허용된다.
- `/apple/secret` : 인증된 접근만 허용된다.
- `/banana/get` : 모든 접근이 허용된다.
- `/banana/secret` : 인증된 접근만 허용된다.

# 공통 설정 HttpSecurity Filter

각 도메인에 알맞는 필터들을 구현하기에 앞서 공통적으로 설정되는 부분을 따로 추출해내어 관리할 수 있다. `AbstractHttpConfigurer` 클래스를 통해 HttpSecurity 설정을 유연하고 모듈화된 방식으로 사용할 수 있다. 

`AbstractHttpConfigurer` 를 상속받아 공통 설정 클래스를 구현한다.

```java
@Component
@RequiredArgsConstructor
public class BaseSecurity extends AbstractHttpConfigurer<BaseSecurity, HttpSecurity> {
    private final BaseAccessDeniedHandler baseAccessDeniedHandler;
    private final BaseAuthenticationEntryPoint baseAuthenticationEntryPoint;

    private boolean flag;

    @Override
    public void init(HttpSecurity http) throws Exception {
        if (flag) {
            http
                    .csrf(AbstractHttpConfigurer::disable)
                    .cors(AbstractHttpConfigurer::disable)
                    .httpBasic(AbstractHttpConfigurer::disable)

                    .headers(h -> h.frameOptions(HeadersConfigurer.FrameOptionsConfig::disable))
                    .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))

                    .exceptionHandling(exceptionHandlingCustomizer -> exceptionHandlingCustomizer
                            .accessDeniedHandler(baseAccessDeniedHandler)
                            .authenticationEntryPoint(baseAuthenticationEntryPoint))
            ;
        }
    }

    public void active() {
        this.flag = true;
    }
}
```

위와 같이 `AbstractHttpConfigurer` 를 상속받은 클래스는 여러 HttpSecurity 구성에 재활용될 수 있다. 어떤 설정들이 있는지 하나씩 살펴보자.

- `BaseAccessDeniedHandler`: `AccessDeniedHandler` 를 상속받은 클래스로, 인증된 사용자가 권한이 없는 리소스에 접근하려 할 때 활성화되며 `AuthenticationException` 을 처리한다.
- `BaseAuthenticationEntryPoint`: `AuthenticationEntryPoint` 를 상속받은 클래스로, 인증되지 않은 사용자가 보호된 리소스에 접근하려고 할 때 활성화되며 `AccessDeniedException` 을 처리한다.

- `.csrf(AbstractHttpConfigurer::disable)`: CSRF (Cross-Site Request Forgery) 보호 기능 비활성화한다. RESTApi 서버라면 session session 기반 인증과는 다르게 stateless하기 때문에 비활성화 한다.
- `.cors(AbstractHttpConfigurer::disable)`: CORS 정책을 비활성화한다. 추후에 재정의하는 필터 클래스를 따로 만들어 명시적으로 관리하는 것이 용이하다.
- `.httpBasic(AbstractHttpConfigurer::disable)`: HTTP 기본 인증을 비활성화한다. 
- `.headers(h -> h.frameOptions(HeadersConfigurer.FrameOptionsConfig::disable))`: `Clickjacking` 공격 방지 기능을 비활성화 한다. 브라우저에서 h2-console 을 사용하기 위한 옵션으로 로컬 개발 환경이 h2-console 을 사용하지 않는다면 해당 설정은 없어도 무방하다.
- `.sessionManagement(s -> s.sessionCreationPolicy(STATELESS))`: 세션 관리 정책을 `STATELESS` 로 설정한다. 주로 토큰 기반 인증을 사용하는 RESTApi 서버에서 해당 옵션을 사용한다. 각 요청은 자체 인증 정보를 포함해야 하며, 서버는 상태를 유지하지 않는다.

- `active()`: 해당 메서드를 통해 기본 설정 클래스를 적용 여부를 선택할 수 있다.

## BaseAccessDeniedHandler

인증된 사용자가 권한이 없는 리소스에 접근하려 할 때 활성화되며 `AuthenticationException` 을 처리한다. 정책에 따라 알맞게 구현하면 된다.

```java
@Component
@RequiredArgsConstructor
public class BaseAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(
            HttpServletRequest request,
            HttpServletResponse response,
            AccessDeniedException accessDeniedException) {
        // logic...
        System.out.println("BaseAccessDeniedHandler.handle");
    }
}
```

## BaseAuthenticationEntryPoint

인증되지 않은 사용자가 보호된 리소스에 접근하려고 할 때 활성화되며 `AccessDeniedException` 을 처리한다. 마찬가지로 정책에 따라 알맞게 구현하면 된다.

```java
@Component
@RequiredArgsConstructor
public class BaseAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(
            HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException) {
        // logic...
        System.out.println("BaseAuthenticationEntryPoint.commence");
    }
}
```

HttpSecurity 에 공통으로 적용될 설정들은 이외에도 정책 등에 따라 자유롭게 변경할 수 있다.

# SecurityFilterChain

해당 프로젝트에 적용할 HttpSecurity 설정을 통해 필터 교체, 선/후행 될 필터 정의, 인증 주체 설정 등 자유롭게 필터 설정을 할 수 있다. 본 글에서는 허용된 앤드-포인트 혹은 인증이 필요한 앤드-포인트만 다루어 본다. 

## Apple Domain SecurityFilterChain

어떤 앤드-포인트를 타겟팅해서 필터링을 할 것인가? 허용된 앤드-포인트와 인증이 필요한 앤드-포인트는 무엇인가? 를 정의해서 크게 3가지 설정을 설명한다.

Apple Domain 에 RESTApi 요청에 대해 수행할 필터를 정의한다.

```java
@Component
@RequiredArgsConstructor
public class AppleSecurityFilter {
    private static final String APPLE_API_PREFIX = "/apple";
    private static final String API = APPLE_API_PREFIX + "/**";
    private static final String[] ALLOW_LIST = {
            APPLE_API_PREFIX + "/get",
    };
    private static final String[] AUTHENTICATED = {
            APPLE_API_PREFIX + "/secret",
    };

    private final BaseSecurity baseSecurity;

    public SecurityFilterChain doFilterChain(HttpSecurity http) throws Exception {
        return http
                .securityMatcher(API)
                .with(baseSecurity, BaseSecurity::active)

                .authorizeHttpRequests(auth -> auth
                        .requestMatchers(ALLOW_LIST).permitAll()
                        .requestMatchers(AUTHENTICATED).authenticated()
                        .anyRequest().authenticated())

                .build();
    }
}
```

- `.securityMatcher(API)`: 타겟팅할 앤드-포인트를 정의한다. 현재 설정에 따르면 `/apple/**` 에 대한 요청은 `AppleSecurityFilter.doFilterChain();` 가 실행된다.
- `.with(baseSecurity, BaseSecurity::active)`: 위에서 정의한 공통 설정 클래스다. 해당 클래스를 주입 받아 활성화하는 메서드 참조이다.
- `.authorizeHttpRequests();`: 해당 메서드를 통해 어떤 요청에 대해 허용할 것이며, 어떤 요청에 대해 인증이 필요한 것인지 설정할 수 있다. 위의 설정은 `/apple/get` 은 허용하며, `/apple/secret` 은 인증이 필요하다, 그 외에 모든 요청(`/apple/**`)은 인증이 필요하다는 설정이다.

## Banana Domain SecurityFilterChain

Apple Domain 에서 설명과 같다. (앤드-포인트만 다를뿐)

```java
@Component
@RequiredArgsConstructor
public class BananaSecurityFilter {
    private static final String BANANA_API_PREFIX = "/banana";
    private static final String API = BANANA_API_PREFIX + "/**";
    private static final String[] ALLOW_LIST = {
            BANANA_API_PREFIX + "/get",
    };
    private static final String[] AUTHENTICATED = {
            BANANA_API_PREFIX + "/secret",
    };
    private final BaseSecurity baseSecurity;

    public SecurityFilterChain doFilterChain(HttpSecurity http) throws Exception {
        return http
                .securityMatcher(API)
                .with(baseSecurity, BaseSecurity::active)

                .authorizeHttpRequests(auth -> auth
                        .requestMatchers(ALLOW_LIST).permitAll()
                        .requestMatchers(AUTHENTICATED).authenticated()
                        .anyRequest().authenticated())

                .build();
    }
}
```

## 필터의 순서

필터가 동작하는 순서가 매우 중요하다. 여러 필터들 중 하나의 필터가 요청에 대한 처리를 수행하게 되면 다른 필터는 동작하지 않는다. 더 넓은 범위의 요청을 `.securityMatcher(API)` 로 설정한다면 이후에 더 좁은 범위의 요청을 수행하는 필터를 설정한 의미가 없다. 예컨대, `.securityMatcher("/**")` 먼저 수행할 필터의 범위를 와일드 카드 2개를 사용해서 앤드-포인트를 매칭시킨다면 이후에 사용할 `.securityMatcher("/apple")` apple filter 는 수행되지 않는다.

이처럼 순서는 매우 중요하다. 올바르게 필터를 동작하게 하기 위해선 `@Component` 어노테이션을 사용하여 스프링 빈으로 자동 등록하고 순서를 정의할 것이다.

## 필터 실행 순서 정의하기

순서 정의를 위한 클래스를 만든다. 각 필터들을 한데 모아 순서만 설정해주면 된다. `@Order` 어노테이션을 통해 순서를 설정해준다. 기본 설정 값은 `Integer.MAX_VALUE;` 이다.

```java
@Configuration
@RequiredArgsConstructor
public class FilterConfig {
    private final AppleSecurityFilter appleSecurityFilter;
    private final BananaSecurityFilter bananaSecurityFilter;

    private final FinalSecurityFilter finalSecurityFilter;

    @Bean
    @Order(1)
    public SecurityFilterChain appleSecurityFilterChain(HttpSecurity http) throws Exception {
        return appleSecurityFilter.doFilterChain(http);
    }

    @Bean
    @Order(2)
    public SecurityFilterChain bananaSecurityFilterChain(HttpSecurity http) throws Exception {
        return bananaSecurityFilter.doFilterChain(http);
    }

    @Bean
    @Order(3)
    public SecurityFilterChain loginFilterChain(HttpSecurity http) throws Exception {
        return finalSecurityFilter.doFilterChain(http);
    }
}
```

- `FinalSecurityFilter`: 마지막 수행될 필터로 모든 요청에 대해 인증이 필요하다는 설정이 담겨있다. `.securityMatcher("/**")` 설정과 `.authorizeHttpRequests(auth -> auth.anyRequest().authenticated())` 설정이 담겨있다.

apple filter 를 1번으로 banana filter 를 2번으로 등록했다. final filter 를 3번으로 등록했다.


# Filter 적용 결과 보기

postman 으로 요청을 보낸 후 어떤 필터가 작동되었는지 확인해 본다. 우선 스프링부트 애플리케이션을 실행시키면 SecurityFilterChain 이 등록되는 순서를 확인할 수 있다.

![image-center]({{ '/images/multi-filter/filter-order-console.png' | absolute_url }}){: .align-center}

세부 정보를 콘솔에서 보기위해 `application.yml` 에서 logging 을 설정해준다. security 와 관련된 모든 로깅에 대해 TRACE 수준으로 설정한다.

```yml
logging:
  level:
    org.springframework.security: TRACE
```

## Apple Domain RESTApi

`/apple/get` 요청을 보내게 되면 정상 응답과 함께 문자열을 리턴받으면 된다. 해당 요청에는 `AppleFilter` 가 작동해야 한다. 요청에 대한 응답으로는 postman 에서 확인할 수 있다.

![image-center]({{ '/images/multi-filter/apple-get-postman.png' | absolute_url }}){: .align-center}

어떤 필터가 동작했는지 console 에서 확인할 수 있다. 로깅 수준을 TRACE 로 할 경우 상세한 내역을 console 에서 확인할 수 있다. `Trying to match request against DefaultSecurityFilterChain [RequestMatcher=Or [Mvc [pattern='/apple/**']],` 요청에 대해 일치하는 패턴을 찾고 해당 필터를 수행시킨다.
주황 박스를 통해 여러 필터 체인을 확인할 수 있다. spring-security 에서 충분한 보안을 제공하며 사용자 정의 추가할 수 있다. (추후 자세히 다룰 예정)

![image-center]({{ '/images/multi-filter/apple-get-console.png' | absolute_url }}){: .align-center}

인증이 필요한 요청에 대해서는 `AccessDeniedException` 이 발생하고 `BaseAuthenticationEntryPoint.commence();` 가 수행하게 된다.

## Banana Domain RESTApi

apple api 와 같은 설정을 가지고 있으며 console 에 출력되는 내용으로 필터가 어떻게 수행되는지 알아볼 수 있다. postman 요청 결과는 같으며 console 에 출력된 내용을 살펴본다.

![image-center]({{ '/images/multi-filter/banana-get-console.png' | absolute_url }}){: .align-center}

요청을 보낸 앤드-포인트는 `/banana/get` 이며 해당 요청에 대한 매핑을 위해 필터가 2개 동작했다. 위에서 설정한 방법대로 1번 필터는 `/apple/**` 에서 매칭이 되고 2번 필터는 `/banana/**` 에서 매칭된다. 요청 매칭을 위해 1번 필터부터 차례대로 꺼냈다는 뜻이 된다. `/banana/get` 요청은 `/apple/**` 과 매칭되지 않으므로 두 번째 필터를 꺼냈고 매칭이 된 것이다. 해당 앤드-포인트는 허용된 앤드-포인트이므로 콘솔에 결과가 출력되는 것을 확인할 수 있다.

인증이 필요한 요청을 인증 없이 보낸다면 아래와 같이 console 에 출력된다.

![image-center]({{ '/images/multi-filter/banana-secret-console.png' | absolute_url }}){: .align-center}

앤드-포인트 매칭을 위해 첫 번째 필터부터 꺼내어 확인했으며 결과적으로 두 번째 필터에서 매칭이 되었고 필터링을 수행했다. 마지막에 `AccessDeniedException` 이 발생하고 `BaseAuthenticationEntryPoint.commence();` 를 실행한다.

## final security

apple 과 banana 가 정의하지 않은 앤드-포인트는 finalSecurityFilter 에서 매칭이 된다. (`/**`) 유효하지 않은 요청을 보내고 console 에 출력물을 확인해 보자.

![image-center]({{ '/images/multi-filter/final-console.png' | absolute_url }}){: .align-center}

첫 번째 필터부터 두 번째 필터를 지나 마지막 필터에 매칭이 되어 필터링을 수행하는 것을 확인할 수 있다. 해당 요청은 인증이 필요하므로 `AccessDeniedException` 이 발생하고 `BaseAuthenticationEntryPoint.commence();` 를 실행한다.