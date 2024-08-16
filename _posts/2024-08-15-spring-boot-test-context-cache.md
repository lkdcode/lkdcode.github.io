---
date: 2024-08-15 15:40:11
layout: post
title: "Spring-Boot-Test Context Cache"
subtitle: 
description: 
image: '/images/spring-boot-test/spring-boot-test-back.png'
optimized_image: '/images/spring-boot-test/spring-boot-test-back.png'
category: spring-test-context-cache
tags:
author:
paginate: false
---

테스트를 하다보면 여러 테스트들이 실행될 때마다 가끔씩 컨텍스트를 재로드한 후 테스트를 진행하는 경우가 있다. 기본적으로 설정이 같으면 재활용한다. 하지만 설정이 다르면 새로운 컨텍스트를 불러오게 되는데 이게 비용이다. 어떻게하면 이 비용을 줄일 수 있는지 알아보자.  

[🔗공식문서](https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/ctx-management/caching.html)

공식 문서를 참고하면 구성 매개변수 조합과 어디서 가져오는지 알 수 있다. 컨텍스트를 구성하는 매개변수 조합에 의해 고유하게 식별되는데 이 매개변수가 다를 때 새로 띄운다고 보면 된다.
1번째 테스트는 A,B,C 를 설정했고 2번째 테스트는 A,B 를 설정했다면 컨텍스트를 새로 띄운다. (A와 B가 같더라도) 설정을 동일하게 가져가면 하나의 컨텍스트를 모든 테스트에서 재활용할 수 있다.  

# logging  

콘솔에 찍히는 SPRING 로고를 봐도 되지만 로그를 통해서도 볼 수 있다. 아래와 같이 테스트 환경에서 사용할 yaml 을 설정해 준다.  

```yml
# application-test.yml
logging:
  level:
    org.springframework.test.context.cache: TRACE
```

![image-center]({{ '/images/context-cache/context-cache1.png' | absolute_url }}){: .align-center}

실제 콘솔에 찍힌 캐싱 정보인데 간단한 설명은 다음과 같다.  

- size: 현재 캐싱된 인스턴스 수  
- maxSize:  캐싱 가능 최대 수  
- parentContext: 부모 컨텍스트 수 (컨텍스트를 계층 구조로 로드할 수 있음)  
- hitCount: 캐시 조회 성공 횟수  
- missCount: 캐시 조회 실패 횟수  
- failureCount: 로딩 실패 횟수  

최초 띄울 때 missCount 가 1이 되면서 캐싱하기 때문에 많은 테스트들을 수행할 때 컨텍스트를 한 개만 사용했다고 볼 수 있다.  

# 컨텍스트를 로드하는 어노테이션  

특별한 어노테이션이 없다면 단위 테스트는 로드하지 않는다.  
@SpringBootTest, @WebMvcTest, @DataJpaTest, @JdbcTest, @RestClientTest 등 컨텍스트를 로드하는 어노테이션이다. 해당 테스트에서 컨텍스트를 재사용하고 싶다면 동일한 구성 매개변수를 사용하여 설정된 컨텍스트를 사용해야 한다.  

# Mock 도 새로운 컨텍스트를 띄운다.  

모킹하는 객체들은 컨텍스트 입장에서 새로운 설정이다. 고로 재로드하게 되는데 외부 API 연동 등 필요에 따라 모킹하는 객체들을 사용할텐데 해당 클래스들을 일괄 관리하면 컨텍스트를 재활용할 수 있다. 테스트에서 사용할 설정 클래스에 등록할 수 있다. 예를들어 2개의 Api 를 모킹해야한다면 아래와 같이 TestConfiguration 을 작성할 수 있다.  

```java
@TestConfiguration
public class MockBeanConfig {
    @Bean
    @Primary
    public LkdCodeApi mockLkdCodeApi() {
        return BDDMockito.mock(LkdCodeApi.class);
    }
  
    @Bean
    @Primary
    public AnotherCodeApi mockAnotherCodeApi() {
        return BDDMockito.mock(AnotherCodeApi.class);
    }
}
```

실제 클래스 때문에 우선순위를 부여해 빈에 등록해주면 된다.  
행위를 지정해주는 클래스도 필수다. (빈에만 등록한 상황에선 껍데기뿐이기에)  

```java
public abstract class BaseLkdCodeMockBeanInitializeSupport {

    @Autowired
    protected LkdCodeApi lkdCodeApi;
    @Autowired
    protected AnotherCodeApi anotherCodeApi;

    @BeforeEach
    void init() {
      BDDMockito.reset(lkdCodeApi);
      BDDMockito.willDoNothing().given(lkdCodeApi).someMethod(any());

      BDDMockito.reset(anotherCodeApi);
      BDDMockito.given(anotherCodeApi.someMethod(any())).willReturn("lkdCode");
    }
}
```

reset 을 추가해야 하는 이유는 다른 테스트에서 행위를 다시 지정할 수 있다. 가령 성공/실패 테스트를 위해 모킹한 클래스가 'true' or 'false' 를 리턴해야 한다. 만약 TestConfiguration 에서 default 값으로 'true' 를 등록했다면 실패 테스트에서 'false' 로 재정의 하게 될 것이다. 재정의 후 테스트 코드가 실패할 수 있기 때문에 스텁 기록들을 모두 제거해주고 초기화해주어야 한다. `.reset()` 메서드를 통해 초기화 작업을 해주어야 한다.  
위의 코드를 상속해 사용할 수 있다.  

# @ContextConfiguration 에 등록

@ContextConfiguration 에 설정을 일괄적으로 등록해 컨텍스트를 재활용할 수 있다. 아래의 예시는 @SpringBootTest 클래스를 하나 만들어 @SpringBootTest 가 필요한 테스트에서 재활용할 수 있다.  

```java
@ContextConfiguration(classes = {MockBeanConfig.class})
@ExtendWith({SpringExtension.class})
@SpringBootTest
public abstract class BaseSpringBootSupport {
}
```

이런식으로 추상 클래스를 하나 만들어 상속해 사용할 수 있으며 다른 어노테이션도 Context 설정 매개변수만 같다면 얼마든지 재활용이 가능하다.  

```java
@ContextConfiguration(classes = {MockBeanConfig.class})
@RestClientTest
public abstract class BaseRestClientTestConfig {
}
```

# @ExtendsWith  

JUnit 5 에서 확장을 등록하기 위해 사용하는 어노테이션이다. 테스트 실행할 때 여러 단계들이 있는데 해당 단계에서 추가적인 동작을 수행하게 할 수 있다. 콜백 인터페이스를 통해 수행할 수 있다. SpringExtension.class 외에도 커스터마이징한 확장을 통해 수행하게 테스트 전/후 등 추가적인 동작을 수행할 때 사용할 수 있다. 해당 인터페이스를 확장하게 되면 @ExtendsWith 에 추가할 수 있다.  

![image-center]({{ '/images/context-cache/context-cache2.png' | absolute_url }}){: .align-center}

아래 외에도 자주 사용되는 인터페이스가 있지만 대표적으로 몇 가지만 설명한다.  

- **BeforeAllCallback**  
    - @BeforeAll 기능 수행  
- **BeforeEachCallback**  
    - @BeforeEach 기능 수행  
- **AfterAllCallback**  
    - @AfterAll 기능 수행  
- **AfterEachCallback**  
    - @AfterEach 기능 수행  
- **TestInstancePostProcessor**  
    - 테스트 인스턴스가 생성된 후 호출. 주로 의존성 주입 등의 작업을 수행  
- **BeforeTestExecutionCallback**  
    - 각 테스트 메서드가 실행되기 직전에 호출. `BeforeEachCallback` 이후에 호출  
- **AfterTestExecutionCallback**  
    - 각 테스트 메서드가 실행된 직후에 호출. `AfterEachCallback` 이전에 호출  
- **ParameterResolver**  
    - 테스트 메서드의 매개변수를 동적으로 제공  

예를 들어 아래와 같은 클래스를 작성한 후 @ExtendsWith 로 사용할 수 있다.  

```java
public class LkdCodeTestExtension implements BeforeAllCallback, AfterAllCallback,
    BeforeEachCallback, AfterEachCallback, TestInstancePostProcessor,
    BeforeTestExecutionCallback, AfterTestExecutionCallback, ParameterResolver {

    @Override
    public void beforeAll(ExtensionContext context) throws Exception {
        System.out.println("Before all tests");
    }

    @Override
    public void afterAll(ExtensionContext context) throws Exception {
        System.out.println("After all tests");
    }

    @Override
    public void beforeEach(ExtensionContext context) throws Exception {
        System.out.println("Before each test");
    }

    @Override
    public void afterEach(ExtensionContext context) throws Exception {
        System.out.println("After each test");
    }

    @Override
    public void postProcessTestInstance(Object testInstance, ExtensionContext context) throws Exception {
        System.out.println("Post process test instance");
    }

    @Override
    public void beforeTestExecution(ExtensionContext context) throws Exception {
        System.out.println("Before test execution");
    }

    @Override
    public void afterTestExecution(ExtensionContext context) throws Exception {
        System.out.println("After test execution");
    }

    @Override
    public boolean supportsParameter(ParameterContext parameterContext, ExtensionContext extensionContext)
        throws ParameterResolutionException {
        return false;
    }

    @Override
    public Object resolveParameter(ParameterContext parameterContext, ExtensionContext extensionContext)
        throws ParameterResolutionException {
        return null;
    }
}

```

아래의 사용 예시로 사용할 수 있고 여러 설정들을 가지고 있는 추상 클래스를 들어 상속해 사용해도 된다.  

```java
@ExtendWith(LkdCodeTestExtension.class)
public class MyTest {

    @Test
    void testExample() {
        System.out.println("lkdcode test");
    }
}
```