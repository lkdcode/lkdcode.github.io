---
date: 2024-08-14 00:52:56
layout: post
title: "Spring-Boot-Test Fixture Monkey 개요"
subtitle: Fixture Monkey 를 사용해 귀차니즘과 엣지 케이스를 잡아보자.
description: 테스트 코드를 작성할 때 데이터를 표현하는 클래스들을 생성하기 귀찮을 것이다. 해당 라이브러리를 활용해 테스트 코드를 쉽게 작성해 보자.
image: '/images/fixture-monkey/fixture-monkey-back.png'
optimized_image: '/images/fixture-monkey/fixture-monkey-back.png'
category: Fixture Monkey
tags: 
  - fixture-monkey
  - test-code
author: lkdcode
paginate: false
---

테스트 코드를 작성할 때 데이터를 표현하는 클래스들을 생성하기란 귀찮고 귀찮다. 필드가 조금만 많아져도 생성하기 귀찮고 필드가 없어도 귀찮다. '누가 자동으로 생성해주면 좋겠다' 를 해결해주는 게 `Fixture Monkey` 이다. 무작위, 유효성 검사, 사용자 설정 등을 적용하여 인스턴스 생성이 가능하다.  

# 준비  

- github & 공식 문서  

[🔗naver-fixture-monkey-github](https://github.com/naver/fixture-monkey)  
[🔗naver-fixture-monkey-docs](https://naver.github.io/fixture-monkey/v1-0-0-kor/docs/get-started/requirements/)  

- 의존성  

```gradle
testImplementation 'com.navercorp.fixturemonkey:fixture-monkey-starter:1.0.14'
```

# Fixture Monkey 로 객체를 생성한다면?  

예를 들어 아래와 같은 코드가 있다면, 생성하기 귀찮을 것이다. (빌더가 있다하더라도.)  

```java
@Value
public class Product {
    long id;
    String productName;
    long price;
    List<String> options;
    Instant createdAt;
    ProductType productType;
    Map<Integer, String> merchantInfo;
}
```

만약 `FixtureMonkey` 를 사용한다면 `//when` 부분처럼 뿅하고 얻을 수 있다.  

```java
@Test
void test() {
    // given
    FixtureMonkey fixtureMonkey = FixtureMonkey
    .builder()
    .objectIntrospector(ConstructorPropertiesArbitraryIntrospector.INSTANCE)
    .build();
  
    // when
    Product actual = fixtureMonkey.giveMeOne(Product.class);
  
    // then
    then(actual).isNotNull();
}
```

# 객체 생성 Introspector  

`FixtureMonkey` 는 마법처럼 객체를 생성해주지만 진짜 마법은 아니다. 생성 방법을 정의할 수 있다. 인스턴스 생성 방법에 사용되는 여러 `Introspector` 들을 간단히 살펴본다.  

## BeanArbitraryIntrospector  

객체 생성에 사용하는 기본 `Introspector`. 리플렉션과 setter 메서드를 사용한다.  

```java
FixtureMonkey fixtureMonkey = FixtureMonkey
	.builder()
	.objectIntrospector(BeanArbitraryIntrospector.INSTANCE)
	.build();
```

## ConstructorPropertiesArbitraryIntrospector  

주어진 생성자로 객체를 생성한다. `@ConstructorProperties`가 있거나 없으면 클래스가 레코드 타입이어야한다. 레코드 클래스를 생성할 때 여러 생성자를 가질 경우 `@ConstructorProperties` 주석이 있는 생성자가 우선 선택된다.  

```java
FixtureMonkey fixtureMonkey = FixtureMonkey
	.builder()
	.objectIntrospector(ConstructorPropertiesArbitraryIntrospector.INSTANCE)
	.build();
```

## FieldReflectionArbitraryIntrospector  

리플렉션을 사용해 생성한다. `getter` or `setter` 가 필요하다.  

```java
FixtureMonkey fixtureMonkey = FixtureMonkey
	.builder()
	.objectIntrospector(FieldReflectionArbitraryIntrospector.INSTANCE)
	.build();
```

## BuilderArbitraryIntrospector  

클래스 빌더를 사용해 생성한다.  

```java
FixtureMonkey fixtureMonkey = FixtureMonkey
	.builder()
	.objectIntrospector(BuilderArbitraryIntrospector.INSTANCE)
	.build();
```

## FailoverArbitraryIntrospector  

여러 개의 `Introspector` 를 사용할 수 있다. 설정한 `Introspector` 중 하나가 생성에 실패하더라도 다음 `Introspector` 로 객체 생성을 시도한다.  

```java
FixtureMonkey monkey = FixtureMonkey.builder()
    .objectIntrospector(new FailoverIntrospector(
        List.of(
            FieldReflectionArbitraryIntrospector.INSTANCE,
            ConstructorPropertiesArbitraryIntrospector.INSTANCE,
            BuilderArbitraryIntrospector.INSTANCE
        )))
    .build();
```

# 인스턴스 생성하기  

`.giveMeOne();`: 특정 타입의 인스턴스를 하나 얻을 수 있다.  

```java
Product actual = fixtureMonkey.giveMeOne(Product.class);
```

`.giveMe();`: 특정한 타입으로 고정되고 두 개 이상의 인스턴스가 필요할 때 사용할 수 있다. 크기를 지정해 스트림, 리스트를 생성할 수 있다.  

```java
List<Product> productList = fixtureMonkey.giveMe(Product.class, 3);
List<List<String>> strListList = fixtureMonkey.giveMe(new TypeReference<List<String>>() {}, 3);
```

`.giveMeBuilder();`: 인스턴스를 커스텀할 수 있다. 'productName' 필드를 'lkdcode' 로 고정하고 그외에 필드는 랜덤으로 생성해준다.

```java
Product actual = monkey.giveMeBuilder(clazz)
    .set("productName", "lkdcode")
    .sample();
```

# With Validation  

`FixtureMonkey` 는 무작위 값으로 생성해주기 때문에 의도와 다른 값이 항상 생성된다. `jakarta.validation` 으로 Bean 검사를 추가할 수 있다.  

- 의존성 추가하기  

```java
testImplementation("com.navercorp.fixturemonkey:fixture-monkey-jakarta-validation:1.0.14")
```

- `FixtureMonkey` 에 `JakartaValidationPlugin` 을 추가해준다.  

```java
FixtureMonkey fixtureMonkey = FixtureMonkey.builder()
  .plugin(new JakartaValidationPlugin()) // or new JavaxValidationPlugin()
  .build();
```

위에서 봤던 `Product` 클래스에 유효성 검사 어노테이션을 추가한다.  

```java
@Value  
public class Product {  
    @Min(1)  
    long id;  
  
    @NotBlank  
    String productName;  
  
    @Max(100000)  
    long price;  
  
    @Size(min = 3)  
    List<@NotBlank String> options;  
  
    @Past  
    Instant createdAt;  
}
```

`FixtureMonkey` 가 인스턴스를 생성할 때 유효한 객체로 생성해준다.  

```java
Product actual = fixtureMonkey.giveMeOne(Product.class); // 유효성 검사 Ok
```

# 인스턴스 커스터마이징
`FixtureMonkey` 는 `ArbitraryBuilder` 를 통해 생성된 객체를 커스텀할 수 있다.  
## set()  

setter 다. 키,벨류로 해당 필드의 값을 할당한다. 아래의 코드는 id 라는 필드의 값을 1000으로 할당해 생성한다.  

```java
fixtureMonkey.giveMeBuilder(Product.class).set("id", 1000);
```

## Just  

`set()` 을 사용할 때 `Just` 로 래핑하여 설정할 수 있다.  

```java
Product product = fixture.giveMeBuilder(Product.class)
	.set("options", Values.just(List.of("red", "medium", "adult"))
	.set("options[0]", "blue")
	.sample();
```

## size(), minSize(), maxSize()  

특정 컬렉션의 사이즈를 설정할 수 있다. default 설정은 0~3이다.  

```java
fixtureMonkey.giveMeBuilder(Product.class)
	.size("options", 5); // size:5 

fixtureMonkey.giveMeBuilder(Product.class)
	.size("options", 3, 5); // minSize:3, maxSize:5 

fixtureMonkey.giveMeBuilder(Product.class)
	.minSize("options", 3); // minSize

fixtureMonkey.giveMeBuilder(Product.class)
	.maxSize("options", 5); // maxSize:5
```

## setNull(), setNotNull()  

속성을 `Null` 로 하거나 값이 존재하도록 보장할 수 있다.  

```java
fixtureMonkey.giveMeBuilder(Product.class)
	.setNull("id");

fixtureMonkey.giveMeBuilder(Product.class)
	.setNotNull("id");
```

공식문서를 참고한다면 이외에도 다양한 커스터마이징을 지원하는 것을 확인할 수 있다.  