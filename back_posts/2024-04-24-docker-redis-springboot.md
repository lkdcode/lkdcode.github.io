---
date: 2024-04-24 09:07:52
layout: post
title: Docker + Redis + SpringBoot 연동하기
subtitle: 쉽고 간단하게 띄워보자
description: 도커는 사용하기는 쉽다. 자세히는 어렵고... Redis 도 마찬가지! 우선 쉽게 띄워보도록 하자
image: '/images/dockerRedisSpringBoot/dockerRedisSpringBoot.png'
optimized_image: /images/dockerRedisSpringBoot/dockerRedisSpringBoot.png
category: springboot
tags:
  - docker
  - redis
  - springboot
author: lkdcode
---


`Redis` 를 직접 설치해 사용할 수도 있지만, `Docker` 를 통해 간편하게 `Redis` 환경을 구축할 수 있다. `SpringBootApplication` 에서 사용할 `Redis` 를 `Docker` 를 통해 구축해보자.

# Docker?
소프트웨어를 `컨테이너` 라는 격리된 환경에서 실행할 수 있게 해주는 도구이다. 이를 통해 애플리케이션을 더 쉽게 개발, 배포, 실행할 수 있으며 주요 구성 요소에는 `Docker Image` 와 `Docker Container` 가 있다.

웹 애플리케이션을 개발하면서 데이터베이스 관리 도구를 통해 DB 를 생성하고 연결하여 작업을 할 것이다. `Docker` 를 사용하면 쉽고 빠르게 데이터 베이스 서버를 띄울 수 있다! 특히 로컬 개발 환경 및 테스트용 데이터 베이스를 구축하기 좋다. (\* `Testcontainers` 도 테스트 환경을 위와 비슷하게 제공한다.)

## Docker Image?
애플리케이션을 실행하는데 필요한 모든 요소를 포함하고 있는 읽기 전용 템플릿이다. 이미지는 컨테이너를 생성하는 기본 단위로 사용된다. 예를 들어 Java 로 개발된 웹 애플리케이션의 Docker 이미지는 컴파일러, 코드, 외부 라이브러리 등을 포함할 수 있다. 이미지는 불변의 상태이므로 한 번 생성되면 수정 할 수 없다. 기존 이미지를 기반으로 새로운 이미지를 생성하거나 새로 만들어야 한다.

## Docker Container?
`Docker Image` 를 실행한 인스턴스를 뜻한다. 이미지가 소프트웨어의 정적인 부분이라면 컨테이너는 실행 중인 동적인 부분이다. 컨테이너는 이미지를 바탕으로 생성되며, 이미지가 제공하는 소프트웨어, 라이브러리, 코드 등을 사용해 실제로 애플리케이션을 실행한다. 컨테이너는 격리된 환경에서 실행되기 때문에 다른 컨테이너나 호스트 시스템의 환경과 상호 간섭 없이 독립적으로 작동한다. 다양한 애플리케이션이 같은 물리적 또는 가상의 서버에서 충돌 없이 실행될 수 있도록 도와준다.

요약해서, 직접 설치하지 않아도 서버를 띄울 수 있다!~~(물론 내 자원..)~~

<br/>

---

# Redis?
고성능의 `Key&Value` 저장소로 널리 사용되는 오픈 소스 인-메모리 데이터 구조 서버이다. 이 데이터 베이스는 주로 데이터를 메모리 내에 저장하기 때문에 디스크 기반 데이터베이스보다 훨씬 빠르게 데이터를 읽고 쓸 수 있다. `Redis` 는 단순한 `Key&Value` 저장 기능을 넘어서 리스트, 집합, 해시 등 다양한 데이터 구조를 지원한다. (\* 소멸 시간을 지정할 수 있다.)

소멸시간과 `Key&Value` 특성을 활용해 데이터들을 관리할 때 주로 사용한다. 소멸시간을 설정한다면 데이터 베이스에서 자동으로 삭제된다. 인증&인가 처리에 자주 사용되는 `Jwt` 를 예로, 로그아웃 요청 시 해당 `Jwt` 를 재사용할 수 없도록 `Redis` 에 저장(가령 블랙리스트라던지)한다. 소멸시간이 끝날 때까지 해당 토큰은 유효하지 않은 토큰으로 판단할 수 있다. 이메일로 인증 코드를 발송해 회원 가입 절차를 진행하는 경우도 많은데 이때 유효한 입력 시간을 `Redis` 로 관리하는 등 소멸 시간과 `Key&Value` 특성 등을 활용해 임시 데이터, 캐싱, 세션 관리 등 다양하게 활용할 수 있다.

## Key & Value
`Redis` 에서는 기본적으로 `Key` 와 `Value` 가 1:1로 맵핑되어 쌍으로 저장된다. `Key` 는 고유 식별자이며 해당 데이터 베이스에서 유일해야 한다. 같은 `Key` 를 두번 입력하면 덮어쓰기가 된다. `Value` 는 여러 형태의 데이터 구조를 가질 수 있으며 문자열, 리스트, 집합, 해시 등이 있다.

<br/>

---

# Docker 설치하기

<a href="https://www.docker.com/get-started/" target="_blank">__Docker (설치 Link)__</a> 해당 사이트에서 다운을 받아도 되고 맥 유저라면 <a href="https://brew.sh/" target="_blank">__homebrew (설치 Link)__</a> 로 간편하게 설치가 가능하다. 설치 후 `Redis` 이미지를 다운 받고 컨테이너로 띄울 수 있다. `homebrew` 가 설치된 맥북에서 아래의 명령어를 통해 다운로드 할 수 있다.
```bash
$ brew install --cask docker
```

![image-center]({{ '/images/dockerRedisSpringBoot/dockerDesktop.png' | absolute_url }}){: .align-center}

설치가 완료 됐다! 이제 도커를 통해 Redis 서버를 띄워보자

<br/>

---

# Redis 이미지를 컨테이너에 띄우기

이미지를 검색해서 다운로드 받고 컨테이너를 띄운다. 터미널과 Docker DeskTop 을 이용한 두 가지 방법을 설명한다.

# 1. 터미널에서 CLI

# Redis 이미지
애플리케이션 실행을 위한 읽기 전용 템플릿, 컨테이너에 띄울 `Redis` 이미지를 다운로드 해야한다. `CLI` 와 `Docker` 애플리케이션을 이용해 다운로드할 수 있다.

## 1. Redis 이미지 검색
컨테이너에 띄울 이미지를 다운로드 하기전에 검색해볼 수 있는 명령어다. 이미지의 정확한 이름을 몰라도 검색이 가능하다. 이미지 이름이 헷갈려도 검색해서 찾는데 크게 문제가 되지 않는다.

### 1-1. 터미널에서 이미지 검색하기
```bash
# redis 이미지 검색
$ docker search redis

# 이미지 이름이 뭐였더라..
$ docker search redi
$ docker search re
$ docker search red
```

이미지를 검색하면 아래와 같이 조회할 수 있다.
![image-center]({{ '/images/dockerRedisSpringBoot/1-1.png' | absolute_url }}){: .align-center}

## 2. Redis 이미지 다운로드

### 2-1. 터미널에서 이미지 다운로드 하기
검색을 하면 여러 `Redis` 목록이 나오는데 상황에 알맞게 다운로드를 하면 된다. `Officail` 을 다운한다면 아래와 같이 명령어를 입력할 수 있다.
```shell
# redis 이미지 다운로드
$ docker pull redis
```

![image-center]({{ '/images/dockerRedisSpringBoot/2-1.png' | absolute_url }}){: .align-center}

## 3. 다운로드 받은 이미지 목록 보기

### 3-1. 터미널에서 이미지 목록 보기
```shell
# 이미지 목록 조회
$ docker images
```

![image-center]({{ '/images/dockerRedisSpringBoot/3-1.png' | absolute_url }}){: .align-center}

## 4. 컨테이너 띄우기

### 4-1 터미널에서 컨테이너 띄우기

다운로드 받은 이미지를 컨테이너로 띄운다.  
`--name`: 컨테이너의 이름을 설정할 수 있다. (변수명을 지정하는 것과 유사하다.)  
&nbsp;&nbsp;&nbsp;&nbsp;- `<containerName>`: 컨테이너의 이름  
`-p`: 호스트와 컨테이너 간의 포트 매핑을 설정한다.  
&nbsp;&nbsp;&nbsp;&nbsp;- `<portNumber>`: 호스트의 포트 번호  
&nbsp;&nbsp;&nbsp;&nbsp;- `<containerPortNumber>`: 컨테이너의 포트 번호  
`<imageName>`: 실행하고자 하는 이미지의 이름  

```shell
# 명령어
$ docker run --name <containerName> -p <portNumber>:<containerPortNumber> <imageName>

# 사용 예시: `my-redis-container` 라는 이름으로 호스트 포트 번호 6379, 컨테이너 포트 번호는 6379 이며 `redis` 이미지를 실행한다.
$ docker run --name my-redis-container -p 6379:6379 redis
```

![image-center]({{ '/images/dockerRedisSpringBoot/4-1.png' | absolute_url }}){: .align-center}

## 5. 실행 중인 컨테이너 목록 조회하기

### 5-1 터미널에서 실행 중인 컨테이너 목록 조회하기
레디스 이미지로 띄운 컨테이너가 조회되는 것을 확인할 수 있다. 컨테이너 이름은 `my-redis-container` 로 실행되어야 하며 이미지, 포트번호를 확인할 수 있다.

```shell
# 실행 중인 컨테이너 목록 조회
$ docker ps -a
```

![image-center]({{ '/images/dockerRedisSpringBoot/5-1.png' | absolute_url }}){: .align-center}

## 6. 컨테이너 접속하기
컨테이너에 띄워진 `Redis` 서버로 접속할 수 있다. 데이터 베이스 도구, 터미널, Docker DeskTop 등을 활용해 접속할 수 있다. CLI 와 Docker DeskTop 을 이용해 접속한다.

### 6-1 CLI 로 Redis 서버 접속하기
`-it`: 표준 입력을 열어 인터랙티브 모드 활성화 & 가상 터미널 할당
`my-redis-container`: 컨테이너 이름
`bash`: 컨테이너 내에서 실행할 명령, bash 셸을 시작하여 내부에서 명령을 수동으로 입력 가능
```shell
# 실행 중인 Docker 컨테이너에서 명령을 실행하라는 명령어
$ docker exec -it my-redis-container bash

# redis 커맨드 라인 인터페이스(cli) 도구
$ redis-cli
```

아래와 같은 화면이라면 접속이 성공된 것이다.
![image-center]({{ '/images/dockerRedisSpringBoot/6-1.png' | absolute_url }}){: .align-center}

<br/>

---

# 2. Docker DeskTop

터미널에서 CLI 로 조작하는 것은 불편할 수 있다. 직관적인 UI가 더 편할 수 있다. Docker DeskTop 을 이용해 `Redis` 서버를 띄운다.

## 1. Redis 이미지 검색
컨테이너에 띄울 이미지를 다운로드 하기전에 검색해볼 수 있는 명령어다. 이미지의 정확한 이름을 몰라도 검색이 가능하다. 이미지 이름이 헷갈려도 검색해서 찾는데 크게 문제가 되지 않는다.

### 1-2. Docker DeskTop 에서 이미지 검색하기
상단 검색창을 이용해 이미지들을 검색할 수 있다. `Redis` 를 검색하면 된다.

![image-center]({{ '/images/dockerRedisSpringBoot/1-2.png' | absolute_url }}){: .align-center}

<br/>

---

## 2. Redis 이미지 다운로드

### 2-2. Docker DeskTop 에서 이미지 다운로드 하기

![image-center]({{ '/images/dockerRedisSpringBoot/2-2.png' | absolute_url }}){: .align-center}

> Pull 과 Run 의 차이는 다음과 같다.
> - **Docker Pull**: 이미지를 로컬 시스템으로 다운로드만 하고 실행하지는 않는다.
> - **Docker Run**: 이미지를 기반으로 컨테이너를 생성하고 실행한다. 이미지가 로컬에 없을 경우 자동으로 이미지를 다운로드(`pull`)하고, 컨테이너를 시작한다.

<br/>

---

## 3. 다운로드 받은 이미지 목록 보기

### 3-2. Docker DeskTop 에서 이미지 목록 보기
좌측 매뉴에 `Images` 버튼을 누르면 다운로드 받은 이미지 목록들을 확인할 수 있다.
![image-center]({{ '/images/dockerRedisSpringBoot/3-2.png' | absolute_url }}){: .align-center}

<br/>

---

## 4. 컨테이너 띄우기

### 4-2 Docker DeskTop 에서 컨테이너 띄우기

![image-center]({{ '/images/dockerRedisSpringBoot/4-2.1.png' | absolute_url }}){: .align-center}
![image-center]({{ '/images/dockerRedisSpringBoot/4-2.2.png' | absolute_url }}){: .align-center}

CLI 와 같은 설정으로 컨테이너가 생성된다.

<br/>

---

## 5. 실행 중인 컨테이너 목록 조회하기

### 5-2 Docker DeskTop 에서 실행 중인 컨테이너 목록 조회
좌측 `Containers` 매뉴를 통해 현재 실행 중인 컨테이너의 목록과 상태를 확인할 수 있다.
![image-center]({{ '/images/dockerRedisSpringBoot/5-2.png' | absolute_url }}){: .align-center}

<br/>

---

## 6. 컨테이너 접속하기

### 6-2 Docker DeskTop 으로 Redis 서버 접속하기
위와 크게 다르지 않다. 똑같이 터미널로 접근해 `redis-cli` 를 입력하면 된다.
![image-center]({{ '/images/dockerRedisSpringBoot/6-2.1.png' | absolute_url }}){: .align-center}
![image-center]({{ '/images/dockerRedisSpringBoot/6-2.2.png' | absolute_url }}){: .align-center}

도커를 활용해 `Redis` 서버를 구축하고 해당 컨테이너에 접속하는데 성공했다. 아래에 정리한 명령어를 통해 `Redis` 를 다루어보고 스프링 부트 애플리케이션에 연결한다.

<br/>

---

# 자주 사용하는 Redis CLI 정리
## Information
```shell
# redis cli 로 접속
$ redis-cli
```

```shell
# 서버가 실행 중인지 확인, 'PONG' 으로 응답
$ PING
```

```shell
# 서버의 정보
$ INFO
```

```shell
# 런타임 설정 조회, `*` 와일드 카드로 조회 가능
$ CONFIG GET *
```

```shell
# 런타임 설정 변경
$ CONFIG SET
```

```shell
# 현재 데이터 베이스의 모든 키 삭제
$ FLUSHDB
```

```shell
# Redis 서버의 모든 데이터 베이스 모든 키 삭제
$ FLUSHALL
```

## 생성
```shell
# key & value 생성
$ SET <key> <value>

# example
$ SET myKey imvalue
$ SET fruit-apple value-values
```

```shell
# key & value 생성 + 여러개 생성
$ MSET <key1> <value1> <key2> <value2> <key3> <value3>..

# example
$ MSET first-fruit apple second-fruit banana third-fruit tomato
$ MSET fruit apple phone 010-1234-1234
```

```shell
# key & value 생성 + 소멸 시간 추가 (초 단위)
$ SETEX <key> <seconds> <value>

# example
$ SETEX myFirstKey 100 value
```

## 삭제
```shell
# 특정 key 삭제
$ DEL <key>

# example
$ DEL fruit
$ DEL phone
```

## 조회
```shell
# 전체 key 조회
$ KEYS *
```

```shell
# 특정 key 의 value 조회
$ GET <key>

# example
$ GET first-fruit
$ GET second-fruit
```

```shell
# 특정 검색어를 포함한 key 조회 (대,소문자 구분)
$ KEYS *<검색어>*

# example
$ KEYS *my*
```

```shell
# 특정 key 만료 시간 조회 (초 단위)
$ TTL <key>

# example
$ TTL myFirstKey
```

```shell
# 특정 key 만료 시간 조회 (밀리 초 단위)
$ PTTL <key>

# example
$ PTTL myFirstKey
```

<br/>

---

# Spring Boot 와 연결하기
`SpringBootApplication` 을 실행하고 `application.yml` 을 작성한다. 여러 `.yml` 설정 파일을 사용해도 되고 하나의 `application.yml` 만 사용하여도 된다. 본 글에서는 `Redis` 설정 클래스와 `application.yml` 설정만 다룬다.

### 1. application.yml

```yml
spring:  
  data:  
    redis:  
      repositories:  
        enabled: false # Redis 의 저장소 지원을 활성화 하거나 비활성화 한다.
      host: 127.0.0.1  # Redis 서버 연결을 위한 호스트 주소 (localhost 문자열도 가능하다.)
      port: 6379 # Redis 서버 연결을 위한 포트 번호
```

### 2. Redis 설정 클래스
`Redis` 서버에 연결 접속 정보를 가지고 있는 클래스다.

```java
import org.springframework.boot.context.properties.ConfigurationProperties;  
import org.springframework.boot.context.properties.EnableConfigurationProperties;  
import org.springframework.context.annotation.Configuration;

@Configuration  
@EnableConfigurationProperties({RedisProperties.RedisInfo.class})  
public class RedisProperties {  
    @ConfigurationProperties(prefix = "spring.data.redis")  
    public record RedisInfo(
            String host,  
            Integer port  
    ) {  
    }}
```
<br/>

`Redis template` 를 설정하는 클래스다.
```java
import lombok.RequiredArgsConstructor;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.data.redis.connection.RedisConnectionFactory;  
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;  
import org.springframework.data.redis.core.RedisTemplate;  
import org.springframework.data.redis.serializer.StringRedisSerializer;  
  
@Configuration  
@RequiredArgsConstructor  
public class RedisConfiguration {  
  
    private final RedisProperties.RedisInfo redisInfo;  
  
    @Bean  
    public RedisConnectionFactory redisConnectionFactory() {  
        return new LettuceConnectionFactory(redisInfo.host(), redisInfo.port());  
    }  
  
    @Bean  
    public RedisTemplate<String, Object> redisTemplate() {  
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();  
        redisTemplate.setConnectionFactory(redisConnectionFactory());  
  
        // 일반적인 key:value의 경우 시리얼라이저  
        redisTemplate.setKeySerializer(new StringRedisSerializer());  
        redisTemplate.setValueSerializer(new StringRedisSerializer());  
  
        // Hash를 사용할 경우 시리얼라이저  
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());  
        redisTemplate.setHashValueSerializer(new StringRedisSerializer());  
  
        // 모든 경우  
        redisTemplate.setDefaultSerializer(new StringRedisSerializer());  
  
        return redisTemplate;  
    }  
}
```

<br/>

---

## 간단한 Api 로 테스트하기!
특정 api 로 요청을 보내면 해당 요청에 담긴 정보를 Redis 에 저장한다. (jwt 의 블랙리스트 구현을 간소화)  
request 와 바로 저장하는 로직만으로 구성된 api 에 요청을 보내고 정상적으로 Redis 에 저장이 되는지 확인한다.

<br/>

### RequestDTO 클래스
HTTP 요청시 토큰을 바디에 담아서 보내지는 않지만 간소화한 requestDTO 다. 토큰을 담아서 보낸다고 생각하면 된다.

```java
import lombok.Data;  
  
@Data  
public class RedisRequest {  
    private String token;  
}
```

<br/>

### Api 클래스
요청을 받기위한 클래스이다. 단순히 바디에 담긴 값을 꺼내 redis 에 저장한다. 소멸 시간도 함께 설정한다.  
소멸 시간 단위는 millisecond 이다.

```java
import lombok.RequiredArgsConstructor;  
import org.springframework.data.redis.core.StringRedisTemplate;  
import org.springframework.web.bind.annotation.PostMapping;  
import org.springframework.web.bind.annotation.RequestBody;  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.RestController;  
  
import java.time.Duration;  
  
@RestController  
@RequestMapping("/redis")  
@RequiredArgsConstructor  
public class RedisApi {  
    private final StringRedisTemplate redis;  
  
    @PostMapping  
    public void getRedis(  
            @RequestBody final RedisRequest request  
    ) {  
        redis.opsForValue()  
                .set(request.getToken(), "BLACK_LIST", Duration.ofMillis(5000000L));  
    }  
}
```

<br/>

### 요청 보내기!
요청을 보내기전에 redis-cli  로 현재 저장된 `key` 값들을 조회한다. 현재 저장된 값은 없다.
![image-center]({{ '/images/dockerRedisSpringBoot/7-1.png' | absolute_url }}){: .align-center}

<br/>

`postman` 으로 요청을 보낸다.
![image-center]({{ '/images/dockerRedisSpringBoot/7-2.png' | absolute_url }}){: .align-center}

<br/>

`$ keys *` 목록 조회, `$ get key` 특정 키 조회, `$ ttl key` 특정 키 소멸 시간 조회에 모두 성공했다.
![image-center]({{ '/images/dockerRedisSpringBoot/7-3.png' | absolute_url }}){: .align-center}