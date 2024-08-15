---
date: 2024-08-14 01:15:41
layout: post
title: "Spring-Boot-Test Fixture Monkey 코드로 보기"
subtitle: Fixture Monkey 를 사용하기 위한 코드를 알아보자.
description: 데이터를 표현하는 클래스들을 쉽게 검증하는 (귀차니즘을 줄여주는) 방법이 있다.
image: '/images/fixture-monkey/fixture-monkey-back.png'
optimized_image: '/images/fixture-monkey/fixture-monkey-back.png'
category: Fixture Monkey
tags:
  - fixture-monkey
  - test-code
author: lkdcode
paginate: false
---

# 설정 코드  

테스트 코드를 작성할 땐 무작위의 값을 가지고 있는 인스턴스가 필요하다. 무작위 값이 필요한 테스트에서 전역적으로 사용하기 위해 아래의 설정을 작성해 준다. `FixtureMonkey` 사용을 위한 클래스이다.  

인스턴스 생성을 도와주는 여러 Introspector 들을 추가한다. 필드,생성자,빌더 외에도 상황에 맞게 다양한 Introsepctor 를 추가할 수 있다.  
jakarta.validation 사용을 위해 plugin 을 추가한다. (당연히 이외에도 다른 플러그인을 사용가능)  
생성하는 값에 대해 기본적으로 null 값을 허용하지 않는다.  

해당 클래스는 인스턴스 생성이 불필요하므로 아래와 같이 작성할 수 있다.  

```java
public abstract class BaseFixtureMonkeySupport {  
  
    protected final FixtureMonkey sut = FixtureMonkey.builder()  
        .objectIntrospector(new FailoverIntrospector(  
            List.of(  
                FieldReflectionArbitraryIntrospector.INSTANCE,  
                ConstructorPropertiesArbitraryIntrospector.INSTANCE,  
                BuilderArbitraryIntrospector.INSTANCE  
            )))  
        .plugin(new JakartaValidationPlugin())  
        .defaultNotNull(true)  
        .build();  
}
```

# 테스트 코드에서 사용하기  

값 객체를 테스트하는 것은 꽤 귀찮은 작업이 된다. 보통 잘못된 매개변수로 인한 생성은 실패하는지 올바른 값으로 생성되었는지 등 반복적이고 단순할텐데 이런 요소들은 값 객체 테스트에서 공통적으로 보여진다. 이 부분을 추상화해서 조금 더 단순화해볼 수 있다. 테스트 코드를 캡슐화하는 것은 꽤나 신중한 선택이 필요하다. 어쩌면 여러 사람들이 다른 해석을 할 수도 있고 방대한 테스트 코드에서는 코드를 드러내는 것이 더 잘 읽힐 수 있다. 하지만 반복적인 작업을 단순화해서 얻는 이점도 있다.  

# 시나리오  

값 객체의 검증 시나리오는 공통적으로 3개가 있다고 가정한다.  

1. 생성을 성공해 Exception 이 발생하지 않는다.  
2. 생성했을 때 Null 이 아니어야 한다.  
3. 유효하지 않은 매개변수로의 생성은 Exception 이 발생한다.  

값 객체 생성은 jakarta.vaildation 기반으로 생성하고 검증한다.  

데이터를 표현하는 클래스 예시.  

```java
public record SomeValue(
    @NotNull
    @Size(max = 3_000)
    String value
) {

    public SomeValue(String value) {
        this.description = description;
        ValueArgumentValidator.validate(this);
    }  
}
```

값 객체를 공통적으로 생성해주고 테스트할 클래스 예시.  

```java
public abstract class BaseValueObjectSupport extends BaseFixtureMonkeySupport {
  
    protected final <T> T assertValidArgumentIsNotNull(final Class<T> clazz) {
        final T instance = sut.giveMeOne(clazz);
        assertThat(instance).isNotNull();
  
        return instance;
    }
  
    protected final void assertAllNotThrow(final ThrowableAssert.ThrowingCallable... executables) {
        assertAll(Stream.of(executables)
            .map(newInstance -> 
                () -> assertThatCode(newInstance).doesNotThrowAnyException()));
    }
  
    protected final void assertThatInvalidArgumentThrow(final ThrowableAssert.ThrowingCallable invalidArgument) {
        assertThatThrownBy(invalidArgument)
            .isInstanceOf(LkdCodeException.class)
            .hasMessage("올바르지 않은 매개변수입니다.");
    }  
}
```

`assertValidArgumentIsNotNull()` 메서드는 값 객체의 인스턴스를 얻고 null 이 아님을 검증하고 반환한다. 이때 jakarta.validation 을 기반으로 생성하게 되며 생성과 동시에 유효성 검사를 진행하게 된다.  
`assertAllNotThrow()` 메서드는 단순히 `assertAll()` 메서드를 내부에 두어 똑같이 동작하게 만들었는데 이는 import 를 대신해주고 해당 클래스를 통해 모든 값 객체 테스트를 수행하기 위함이다.  
마지막으로 `assertThatInvalidArgumentThrow()` 메서드는 잘못된 매개변수로 생성을 하게 되면 발생할 익셉션 & 메시지를 정의한다. 로그 처리 등 다양한 검증을 추가할 수 있지만 제외했다.   

실제 값 객체 테스트 사용 예시.  
엣지 케이스를 위해 200회 반복 수행하고 jakarta.validation 에 유효성 검사가 올바르게 적용됐는지 확인한다.  

```java
class SomeValueTest extends BaseValueObjectSupport {
  
    @RepeatedTest(TestEnvironment.VALUE_COUNT)
    void 생성에_성공할_것이다() throws Exception {
        final var someValue = super.assertValidArgumentIsNotNull(SomeValue.class);
        super.assertAllNotThrow(
            () -> Assertions.assertThat(SomeValue.value()).isNotBlank(),
            () -> Assertions.assertThat(SomeValue.value().length()).isLessThanOrEqualTo(3_000)
        }
    }
  
    @ParameterizedTest
    @NullSource
    @ValueSource(strings = {"   ", ""})
    void 올바르지_않은_매개변수는_생성에_실패할_것이다(final String arg) throws Exception {
        super.assertThatInvalidArgumentThrow(() -> new SomeValue(arg));
    }
}
```

TestEnvironment.VALUE_COUNT 는 테스트 코드에서 여러 환경들을 일괄적으로 제어하는 클래스로 예를 들어 아래와 같이 작성할 수 있다.  

```java
public final class TestEnvironment {
    public static final int VALUE_COUNT = 200;
}
```

# ValueArgumentValidator.validate(this);  

비즈니스 로직을 수행하기 위해 각 클래스들마다 메서드들을 이리저리 호출한다. 값 객체가 잘못 생성되었음에도 호출하지 않는 이상 잘 못되었는지 알 수가 없다. 그러므로 인스턴스가 생성되는 순간에 올바른 데이터인지 검증할 필요가 있다. 빠른 실패는 애플리케이션의 리소스를 줄여주며 여러 장점들을 제공하는데 이 역할을 수행할 수 있는 클래스이다.  

값 객체들을 생성하는 시점에 검증을 동작하도록 만들어 이를 해결할 수 있다. jakarta.validation 을 통해 검증하게 된다. 기존에 `class` 로 데이터를 표현했다면 java 14 이후 `record` 를 사용하여 표현할 수도 있다. 일부 책에서 소개하는 추상 클래스를 extends 하는 방식이 더 좋은 코드일 수 있지만 아쉽게도 record 는 추상 클래스를 확장할 수 없어 정적 메서드를 활용해 적용한다. interface 에서 default 메서드나 static 메서드를 둘 수도 있다.  

```java
public final class ValueArgumentValidator {  
    private static final Validator VALIDATOR;
  
    static {
        VALIDATOR = Validation.buildDefaultValidatorFactory()
            .getValidator();
    }
  
    public static <T> void validate(final T vo) {
        final var violations = VALIDATOR.validate(vo);
  
        if (!violations.isEmpty()) {
            final var message = new StringBuilder();

            violations.forEach(v -> message
                .append(v.getMessage())
                .append(System.lineSeparator()));
  
            log.warn("Invalid argument: {}, class: {}", message, vo.getClass());
            throw new LkdCodeException(ResponseCode.INVALID_VALUE_OBJECT);
        }
    }
}
```

위의 클래스를 값 객체마다 추가해주어 생성 시점에 올바른지 검증할 수 있다. 다시 아래의 값 객체를 본다면 해당 검증 클래스를 사용하는 시점이 보일것이다. (class 는 검증 클래스를 추상 클래스로 두어 구현한다.)  

```java
public record SomeValue(
    @NotNull
    @Size(max = 3_000)
    String value
) {

    public SomeValue(String value) {
        this.description = description;
        ValueArgumentValidator.validate(this);
    }  
}
```

# Arbitrary<T\>  

몽키는 Arbitrary 를 사용해 인스턴스를 생성해준다. Arbitrary 를 이용해 필터링된 인스턴스를 얻을 수 있다. jakarta.validation 을 기반으로 올바르게 생성해줄 수도 있지만 간혹 정규식이 적용 안 될때는 Arbitrary 를 이용해 생성해줘야 한다. 정규식 외에도 특정 조건에 해당하는 값들을 생성하기 위해 Arbitrary 를 사용할 수 있다. 문자열의 경우 특정 문자열과 겹치지 않게도 가능하다. Arbitrary 의 여러 메서드들을 활용해 다양한 객체들을 생성할 수 있다.  

- 비밀번호 예시  

```java
public Arbitrary<String> getPasswordArbitrary() {  
    return Arbitraries.strings()
        .withChars("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&")
        .ofMinLength(8)
        .ofMaxLength(20)
        .filter(p -> p.matches("^(?=.*[a-z])(?=.*[A-Z])(?=.*[0-9])(?=.*[!@#$%^&]).{8,20}$"));  
}
```

특정 문자 제외는 `.excludeChars()` 메서드를 사용할 수 있다. 이외에도 다양한 기능을 지원하니 살펴보는 것이 좋다.  

```java
Arbitraries.strings()
    .excludeChars(/* 특정 문자 제외 */)
```

- Long 인스턴스 예시  

```java
public Arbitrary<Long> getLong(){  
    return Arbitraries.longs()  
        .between(10L,100L)  
        .filter(e-> e % 2 ==0);  
}
```

- 사용예시  

```java
monkey
    .giveMeBuilder(Value.class)  
    .set("password", getPasswordArbitrary())  
    .sample();
```