---
date: 2024-08-16 23:11:34
layout: post
title: "Spring-Boot-Test TestContainers 개요"
subtitle: 테스트용 데이터 베이스를 컨테이너를 통해 구축한다. (Feat.Config 클래스처럼 활용!)
description: 테스트를 수행할 때 필요한 서버를 컨테이너로 띄울 수 있다. 자바 코드로 환경을 구성할 수 있다. 해당 글에서는 데이터 베이스를 띄우는 방법을 소개한다.
image: '/images/spring-boot-test/spring-boot-test-back.png'
optimized_image: '/images/spring-boot-test/spring-boot-test-back.png'
category: spring-test
tags: 
  - test
  - test-containers
author: lkdcode
paginate: false
---

테스트를 수행할 때 특정 서버를 띄우거나 데이터 베이스를 연결하는 등 해당 환경에서 테스트를 수행해야할 때가 있다. 예를 들어 데이터 베이스가 필요한 테스트에서 실제 local DB 를 연결하거나 H2 인메모리, docker-compose 등 다양한 방법을 통해 데이터 베이스를 연결하여 테스트를 수행할 수 있다. 각각 장, 단점이 있는데 TestContainers 는 테스트 환경과 실제 환경을 동일하게 가져갈 수 있고 테스트 시작과 종료에 따라 자동으로 컨테이너를 실행하고 종료해준다. TestContainers 를 통해 테스트 환경을 구축하는 방법을 설명한다.  

# Testcontainers

[🔗공식 문서](https://testcontainers.com/)

TestContainers 의 공식 문서를 통해 사용법을 확인할 수 있다. 도커 컨테이너를 자바 코드로 조작할 수 있어 자바 개발자라면(?) 더 익숙할 수도 있다.  

- 의존성 추가하기 (gradle)  

```gradle
testImplementation 'org.junit.jupiter:junit-jupiter:5.10.3'  
testImplementation 'org.testcontainers:junit-jupiter'  
testImplementation 'org.testcontainers:postgresql'  
testImplementation 'org.testcontainers:testcontainers'
```

# PostgreSQL Container

컨테이너를 띄우는 것은 도커를 사용하는 것과 똑같다. 다만 자바코드로 구축할뿐. 실행 순서도 동일하다. 컨테이너로 띄울 이미지를 등록하고 환경 변수들을 설정해준다. 이렇게 띄운 컨테이너를 사용할 수 있는 여러 방법이 있는데 해당 글에서는 config.class 처럼 선택적으로 등록하고 사용할 수 있는 방법을 설명한다.  

## @DynamicPropertySource 를 사용하지 않는다  

공식문서에 있는 일부 코드인데 아래와 같이 @DynamicPropertySource 를 사용해 주입받은 객체를 통해 설정을 등록할 수 있다. 정적 메서드여야 하며 동적으로 설정을 할당할 수 있다. @DynamicPropertySource 어노테이션은 간단하게 사용할 수는 있지만 해당 클래스(컨테이너)를 사용하기 위해 extends 해야한다. 이 방법 대신 config 클래스로 만들어 테스트 환경마다 필요한 컨테이너를 등록해서 사용한다.  

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class AlbumControllerTest {
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("photos.api.base-url", wireMock::baseUrl);
    }
}
```

@DynamicPropertySource 어노테이션을 사용해 컨텍스트가 로드될 때 동적으로 환경 변수들을 설정할 수 있다. 컨텍스트를 로드해야 하므로 로드 후 설정들을 추가해야 한다. @DynamicPropertySource 어노테이션을 설정한 클래스를 두고 extends 해서 사용할텐데 이렇게 되면 여러 컨테이너들을 동시에 띄우는 테스트가 번잡할 수 있다. 해당 글에서는 아래와 같이 특정 컨텍스트에 필요한 컨테이너들을 등록해 사용하는 방법을 설명한다.  

![image-center]({{ '/images/spring-boot-test/test-containers/test-containers-img1.png' | absolute_url }}){: .align-center}

위의 @ContextConfiguration-initializers 에 PostgresContainer, RedisContainer 등 TestContainers 를 이용해 띄운 컨테이너들을 등록해서 사용할 수 있다. 테스트에 필요한 환경에 따라 컨텍스트를 구축할 수 있다.     

# ApplicationContextInitializer  

앞서 @DynamicPropertySource 처럼 컨텍스트가 로드되면서 초기화될 때 필요한 설정들을 ApplicationContextInitializer 로 대체할 수 있다. 해당 인터페이스를 활용해 데이터 베이스에 커넥션할 변수들을 동적으로 설정할 수 있다.  

![image-center]({{ '/images/spring-boot-test/test-containers/test-containers-img2.png' | absolute_url }}){: .align-center}

# PostgreSQL Container

```java
@Testcontainers  
public class PostgresContainerInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {  
    private static final String POSTGRE_IMAGE = "postgres:latest";  
    private static final String POSTGRE_DATABASE_NAME = "lkdcode";  
    private static final String POSTGRE_USER_NAME = "lkdcode";  
    private static final String POSTGRE_PASSWORD = "lkdcode";  
  
    @Container  
    public static final PostgreSQLContainer<?> POSTGRE_SQL_CONTAINER;  
  
    static {  
        POSTGRE_SQL_CONTAINER = new PostgreSQLContainer<>(POSTGRE_IMAGE)  
            .withDatabaseName(POSTGRE_DATABASE_NAME)  
            .withUsername(POSTGRE_USER_NAME)  
            .withPassword(POSTGRE_PASSWORD);  
        POSTGRE_SQL_CONTAINER.start();  
    }  
  
    @Override  
    public void initialize(ConfigurableApplicationContext applicationContext) {  
        final var environment = applicationContext.getEnvironment();  
        final Map<String, Object> testContainersProperties = new HashMap<>();  
        testContainersProperties.put("spring.datasource.url", POSTGRE_SQL_CONTAINER.getJdbcUrl());  
        testContainersProperties.put("spring.datasource.username", POSTGRE_SQL_CONTAINER.getUsername());  
        testContainersProperties.put("spring.datasource.password", POSTGRE_SQL_CONTAINER.getPassword());  
        environment.getPropertySources().addFirst(new MapPropertySource("lkdcode-postgres", testContainersProperties));  
    }  
}
```

도커 이미지와 데이터 베이스의 설정들을 지정한다. Testcontainer 에서 제공하는 @Container 를 통해 테스트의 생명주기동안 관리되도록 한다. `initialize()` 메서드 의 가장 아래부분 "lkdcode-postgres" 가 설정할 속성의 키인데 중복되면 덮어쓰기 등의 문제가 발생할 수 있다.  

# Redis Container

```java
@Testcontainers  
public class RedisContainerInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {  
    private static final String REDIS_IMAGE = "redis:latest";  
    private static final int REDIS_PORT = 6379;  
    private static final String REDIS_PASSWORD = "lkdcode";  
  
    @Container  
    public static final GenericContainer<?> REDIS_CONTAINER;  
  
    static {  
        REDIS_CONTAINER = new GenericContainer<>(REDIS_IMAGE)  
            .withExposedPorts(REDIS_PORT)  
            .withEnv("REDIS_PASSWORD", REDIS_PASSWORD);  
        REDIS_CONTAINER.start();  
    }  
  
    @Override  
    public void initialize(ConfigurableApplicationContext applicationContext) {  
        final var environment = applicationContext.getEnvironment();  
        final Map<String, Object> testContainersProperties = new HashMap<>();  
        testContainersProperties.put("spring.data.redis.host", REDIS_CONTAINER.getHost());  
        testContainersProperties.put("spring.data.redis.port", REDIS_CONTAINER.getMappedPort(REDIS_PORT));  
        testContainersProperties.put("spring.data.redis.password", REDIS_PASSWORD);  
        environment.getPropertySources().addFirst(new MapPropertySource("lkdcode-redis", testContainersProperties));  
    }  
}
```

이미지를 등록하고 포트를 연결하는 등 크게 다르지 않다. 클라우드 환경에서 컨테이너 외부에 접속하거나 내부 네트워크에서 보안이 필요하다면 위의 `.withEnv()` 메서드를 통해 비밀번호도 설정해줄 수 있다.  
위와 같이 작성한다면 @ContextConfiguration-initializers 에 등록이 가능하다.  

위와 같이 작성된 클래스들은 컨텍스트에 등록해 사용할 수 있다. 아래 사진과 같이 필요한 테스트마다 다른 컨테이너들을 등록해 사용할 수 있다.  

![image-center]({{ '/images/spring-boot-test/test-containers/test-containers-img1.png' | absolute_url }}){: .align-center}

추가로 DB 형상관리를 위해 flyway 를 사용한다면 sql 을 올바르게 작성했는지 쉽게 알 수 있다. 매 테스트마다 새로운 컨테이너를 띄우고 종료하기 때문에 히스토리를 따로 건들지 않아도 된다.  