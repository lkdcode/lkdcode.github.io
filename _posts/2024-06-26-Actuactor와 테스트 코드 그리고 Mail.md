---
date: 2024-06-26 08:35:18
layout: post
title: "Actuator 와 테스트 코드 그리고 Mail"
subtitle: 잘 되던 테스트 코드 Actuator 의존성 추가 이후 실패!
description: Actuator 은 애플리케이션의 모든 정보를 알고 있다. 어떻게?
image: '/images/throubleShooting.png'
optimized_image: '/images/throubleShooting.png'
category: TROUBLESHOOTING
tags: TROUBLESHOOTING
author: lkdcode
paginate: true
---

# Actuator 의존성 추가.. 그리고 실패하는 테스트 코드
잘 실행되던 테스트 코드가 actuator 의존성을 추가하고 나서 실패한다.
```yaml
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

스프링 컨테이너가 제대로 안 띄워지고 아래의 로그를 확인할 수 있는데 Acuator 의존성을 제거하면 테스트 코드는 잘 통과된다.
```java
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'mvcHandlerMappingIntrospector' defined in class path resource... //...생략
```

비즈니스 로직 중 SMTP 서버를 통해 메일을 보내는 로직이 있다.
```yaml
implementation 'org.springframework.boot:spring-boot-starter-mail'
```

# Actuator 그리고 Mail health
Actuator 가 애플리케이션 컨텍스트를 로드할 때 Mail 관련 빈을 생성하지 않도록 해야 한다.
이를 통해 Mail 설정이 올바르게 되지 않거나 필요한 리소스가 누락되었을 때 빈을 생성하지 않도록 한다.

<ins>...가 아니라</ins> <strong>왜 `MailHealthIndicator` 빈이 제대로 생성되지 않을까?</strong> 메일을 보내는 설정과 관련해서 오류가 있는지 찾아보는 게 우선이다.🫨

의존성을 끊고 테스트를 통과하는게 우선이라면 테스트 코드에서 실행할 yml 에 다음과 같이 추가하면 된다. 메일 외에도 Actuator 로 방해받고 싶지 않다면 false 로 급하게 통과시킬 수 있다.
```yaml
management:  
  endpoints:  
    enabled-by-default: false # 기본적으로 모든 Actuator 앤드포인트 비활성화  
    web:  
      exposure:  
        include: none # 웹을 통한 모든 Actuator 앤드포인트 비활성화  
  health:  
    mail:  
      enabled: false # Mail 헬스 체크 비활성화  
    enabled: false # 모든 헬스 체크 비활성화  
  info:  
    enabled: false # 애플리케이션 정보 앤드포인트 비활성화
```
# MailHealthIndicator
Spring Boot Actuator 에서 제공하는 Healthindicator 의 구현체 중 하나로 메일 서버의 상태를 모니터링하는 역할을 수행한다. 실제 메일 서버와 연결하여 상태를 확인하기 때문에 테스트 관련해서 실패할 수도 있다.
- 실제 메일 서버와의 연결을 성공하면 정상적으로 동작한다고 판단하고 Health 상태를 UP 으로 설정한다.
- 실제 메일 서버와의 연결을 실패하면 비정상적으로 동작한다고 판단하고 Health 상태를 DOWN 으로 설정한다.

위를 토대로 `MailHealthIndicator`의 간단한 단위 테스트 코드다.
```java
@SpringBootTest  
public class MailHealthIndicatorTest {  
    @Autowired  
    private MailHealthIndicator mailHealthIndicator;  
  
    @Test  
    void mailHealth() {  
        final Health health = mailHealthIndicator.health();
        Assertions.assertThat(health.getStatus()).isEqualTo(Status.UP);  
    }  
}
```

위의 테스트 코드는 잘 통과과 되었으므로 아래의 SMTP 관련 yml 은 문제가 없다.
```yaml
spring:  
  mail:  
    host: # 호스트
    port: 465 # (일반적으로 SSL을 사용하는 SMTP 서버의 포트)
    username: # 인증할 사용자 이름
    password: # 인증할 비밀번호
    default-encoding: UTF-8 # 기본 인코딩
    properties:  
      mail:  
        smtp:  
          auth: true # SMTP 서버에 로그인 인증을 시도하는가?
          ssl:  
            enable: true # SSL 을 사용하여 이메일을 전송하는가?
            trust: # 호스트, 특정 호스트의 SSL 인증서를 신뢰하는가?
          timeout: 15000 # 서버 연결 타임아웃
          quitwait: false # SMTP 서버가 연결을 종료할 때까지 기다릴 것인가?
```

## 그럼 무엇이 문제였을까?
테스트 코드에 결함이 하나 있었는데 `JavaMailSender` 를 mock 객체로 주입 받고 있었던 것이다.
```java
@MockBean  
protected JavaMailSender javaMailSender;
```

Actuator 는 실제 메일 서버에 접근하여 health 정보를 가져오는데 목객체로 주입을 받으니 실제 서버와 연결할 수 없어 테스트 코드가 실패했던 것이다. restdocs 작성을 위한 테스트에서는 공통 코드들을 하나의 클래스에서 관리하고 상속하여 구현하는데 JavaMailSender 를 목객체로 주입을 받으니 해당 클래스가 테스트에 실패했던 것이다. 빠른 개발도 좋지만 시간이 더 걸리더라도 올바른 구현을 해야한다. 오히려 시간이 더 소모되기에.. 임시 방편으로 해결했던 Mail 관련 헬스 체크를 모두 꺼준 default 설정으로도 테스트 코드를 통과할 수 있다.