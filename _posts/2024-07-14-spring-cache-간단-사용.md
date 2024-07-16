---
date: 2024-07-14 01:03:08
layout: post
title: "Spring Cache 간단 사용 (AOP)"
subtitle: 주요 어노테이션들의 속성을 알아보자.
description: 어노테이션의 속성만 알아도 꽤나 요긴하게 사용할 수 있다. Spring Cache 를 사용하기 위한 어노테이션들을 알아본다. 그리고 주의할 점은 AOP 를 알아야 하는 것!
image: '/images/cache/cache-back.png'
optimized_image: '/images/cache/cache-back.png'
category: cache
tags: 
  - cache
  - spring cache
  - AOP
author: lkdcode
paginate: false
---

# Spring-Cache?
다양한 캐시 제공자(Redis, Caffeine, EHCache 등)를 지원하는 추상화 계층이며 캐시 구현체와 독립적으로 캐시를 사용하고 관리할 수 있다. 대표적인 어노테이션으로는 @Cacheable, @CachePut, @CacheEvict, @Caching 가 있다.

## `@Cacheable`
메서드 호출 결과를 캐시에 저장하고 이후 해당 메서드 호출 시 캐시된 결과를 반환하도록 한다. 여러 속성들이 어떤 기능을 제공하는지 알아본다.  
- value: 캐시 이름을 "itemsCache" 로 지정한다.  

```java
@Cacheable(value = "itemsCache")
public Item getItem(Long id) { return findItemById(id); }
```

- key: SpEL 을 사용하여 캐시 키를 '#id' 로 설정한다. (매개변수 Long id)  

```java
@Cacheable(value = "itemsCache", key = "#id")
public Item getItemWithKey(Long id) { return findItemById(id); }
```

- keyGenerator: 키 생성을 위해 커스텀 키를 사용한다.  

```java
@Cacheable(value = "itemsCache", keyGenerator = "customKeyGenerator")
public Item getItemWithCustomKeyGenerator(Long id) {
	return findItemById(id);
}
```

- condition: 특정 조건이 'true' 일 때만 캐싱한다.  

```java
@Cacheable(value = "itemsCache", condition = "#id > 10")
public Item getItemWithCondition(Long id) {
	return findItemById(id);
}
```

- unless: 특정 조건이 'true' 일 때는 캐싱하지 않는다.  

```java
@Cacheable(value = "itemsCache", unless = "#result.price >= 1000")
public Item getItemWithUnless(Long id) { return findItemById(id); }
```

- sync: 'true' 인 경우 동일한 키로 동시에 여러 쓰레드가 접근할 때 동기화한다.  

```java
@Cacheable(value = "itemsCache", key = "#id", sync = true)
public Item getItemWithSync(Long id) { return findItemById(id); }
```
- cacheManager: 특정 캐시 매니저를 사용하도록 지정한다.  
- cacheResolver: 특정 캐시 리졸버를 사용하도록 지정한다.  

## `@CachePut`
메서드의 결과를 캐싱하지만 항상 메서드를 실행한다. 업데이트가 필요한 경우 유용하다. `@CachePut` 어노테이션의 속성은 `@Cacheable`의 속성과 같다.  

## `@CacheEvict`
캐시에서 데이터를 제거하는데 사용된다. `@CachePut`, `@Cacheable` 의 속성과 같으며 다른 속성들을 알아본다.  

- allEntries: 'true' 인 경우 캐시의 모든 항목을 제거한다.  

```java
@CacheEvict(value = "items", allEntries = true)
public void removeAllItems() { /*..*/ }
```

- beforeInvocation: 'true' 인 경우 메서드 실행 전에 캐시를 제거한다.  

```java
@CacheEvict(value = "items", key = "#id", beforeInvocation = true)
public void removeItemByIdBeforeInvocation(Long id) { /**/ }
```

## `@Caching`  
위의 여러 캐시 관련 어노테이션을 조합해서 사용할 수 있는 어노테이션이다.  

```java
@Caching(
    evict = { @CacheEvict(value = "items", key = "#item.id") },
    put = { @CachePut(value = "items", key = "#item.id") }
)
public Item saveItem(Item item) {
    return itemRepository.save(item);
}
```

# cache solution  
spring-cache 를 구현하는 여러 솔루션들이 있지만 대표적인 캐시를 소개한다. `Caffeine` 과 `Redis`가 있다. 간략한 개요만 알아본다.  

| 특징          | Caffeine                | Redis                     |
| ----------- | ----------------------- | ------------------------- |
| **구조**      | 로컬 캐시                   | 분산 캐시                     |
| **성능**      | 매우 빠름                   | 네트워크 지연으로 인해 약간의 오버헤드 존재  |
| **데이터 지속성** | JVM 종료 시 데이터 소멸         | 지속성 옵션(AOF, 스냅샷) 제공       |
| **확장성**     | 제한적 (단일 JVM 내)          | 높은 확장성 (클러스터링 지원)         |
| **데이터 구조**  | 간단한 키-값 저장              | 다양한 데이터 구조 (리스트, 셋, 해시 등) |
| **고가용성**    | 지원 안 함                  | 레플리케이션, 페일오버, 클러스터링 지원    |
| **복잡한 연산**  | 제한적                     | Lua 스크립트를 통한 복잡한 연산 가능    |
| **사용 사례**   | 단일 서버 애플리케이션, 짧은 수명 데이터 | 분산 애플리케이션, 세션 관리, 실시간 분석  |


# spring application 에 spring-cache 적용하기

- 의존성 추가  


```shell
implementation 'org.springframework.boot:spring-boot-starter-cache'
```


- 설정 클래스  

spring cache 는 다양한 캐싱 솔루션을 통합하여 일관된 캐싱 추상화를 제공한다. 때문에 어떤 솔루션을 사용할지 유저가 자유롭게 선택이 가능하다. 아래의 설정은 `Redis` 를 활용한 설정이며 어노테이션과 함께 동작한다.  

```java
@EnableCaching  
@Configuration  
public class CacheConfig {  
  
    @Bean  
    public RedisCacheConfiguration defaultCacheConfig() {  
        return RedisCacheConfiguration  
            .defaultCacheConfig()  
            .entryTtl(Duration.ofHours(1))  
            .disableCachingNullValues()  
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))  
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));  
    }  
  
    @Bean  
    public RedisCacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {  
        final Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();  
  
        return RedisCacheManager  
            .builder(redisConnectionFactory)  
            .cacheDefaults(defaultCacheConfig())  
            .withInitialCacheConfigurations(cacheConfigurations)  
            .build();  
    }  
}
```

`defaultCacheConfig()` 메서드를 통해 `RedisCacheConfiguration` 의 기본 설정을 할당한다. TTL 기본 시간은 1시간이고 기본적으로 null 값을 캐싱하지 않는다.  

`redisCacheManager` 를 통해 캐싱을 관리할 매니저를 설정한다. 기본 캐시 설정을 `defaultCacheConfig()` 메서드로 할당하고 초기 캐시 설정을 `Map<String, RedisCacheConfiguration> cacheConfigurations` 으로 되어있는 설정값을 가져온다.  

캐시에 따라 선택적으로 TTL 및 설정을 수정할 수 있다. 아래의 코드는 `hello-world` 라는 캐시에 대해 TTL(5분) 을 설정하는 코드이다. Map으로 구현한 `cacheConfigurations` 에 추가해주면 된다.  

```java
cacheConfigurations.put("hello-world", defaultCacheConfig().entryTtl(Duration.ofMinutes(10)));
```

# Spring-Cache & AOP
위에서 설명한 Cache 관련 어노테이션을 이용해 캐싱 처리를 수행하면 된다. 캐싱 기능을 구현할 때 스프링은 AOP(Aspect-Oriented-Programming) 를 사용하여 기능을 구현한다. Spring AOP 의 프록시 매커니즘 때문인데 해당 빈의 타겟이 되는 메서드 호출을 가로채어 AOP 어드바이스를 적용한다.  

일반적으로 클라이언트 코드가 `.xxCacheMethod();` 를 호출하게 되면 아래와 같이 AOP 가 동작하게 된다.  

```java
Client -> Proxy (AOP) -> Actual Service.xxCacheMethod();
```

하지만 `this.xxCacheMethod();` 로 호출하게 되면 현재 객체를 참조하므로 프록시를 우회한다. 고로 AOP 어드바이스가 적용되지 않는다.  

```java
Client -> Actual Service(this.xxCacheMethod();)
```

때문에 아래와 같은 코드가 있다면 우리는 캐싱 처리를 올바르게 수행할 것으로 기대하겠지만 AOP 프록시를 우회하므로 실제로 캐싱처리가 되지 않는다. 예를 들어 아래와 같은 코드가 있을 때 `this.getItem();` 메서드는 캐싱이 적용되지 않는다.  

```java
@Service
@RequiredArgsConstructor
public class ExampleService {
	private final Repository repository;

    @Cacheable(value = "itemsCache", key = "#id")
    public Item getItem(Long id) {
        return repository.findItemById(id);
    }

    public void updateItem(Long id) {
        this.getItem(id); // 캐싱이 적용되지 않는다.
        /*...*/
    }
}
```

다른 클라이언트 코드가 호출하거나,  

```java
@Service
@RequiredArgsConstructor
public class OtherService {
    private final ExampleService exampleService;

    public void someMethod(final Long id) {  
        exampleService.getItem(id);
        /*...*/
    }
}
```

혹은 이너 클래스로도 풀어낼 수 있다. 방법이 어찌됐건 AOP 프록시를 우회하지 않도록 하여 캐싱 로직을 올바르게 풀어내는 것이다.  

```java
@Service
@RequiredArgsConstructor
public class ExampleService {
    private final InnerService innerService;

    public Item getItem(Long id) {
        return innerService.findItemById(id);
    }

    public void updateItem(Long id) {
        innerService.getItem(id); // 캐싱이 적용된다.
    }

    @Service
    @RequiredArgsConstructor
    public static class InnerService {
        private final Repository repository;

        @Cacheable(value = "itemsCache", key = "#id")
        public Item getItem(Long id) {
            return repository.findItemById(id);
        }
    }
}
```