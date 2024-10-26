---
date: 2024-10-26 13:50:47
layout: post
title: "Java 리플렉션 개요"
subtitle: 리플렉션을 사용해 클래스의 필드와 어노테이션 정보 등을 얻을 수 있다.
description: 클래스의 필드와 어노테이션 등 정보를 얻을 수 있는데 이를 통해 데이터 베이스 형상 관리를 작게나마 해보자!
image: '/images/java/reflections.png'
optimized_image: '/images/java/reflections.png'
category: java
tags:
  - java
  - reflections
author: lkdcode
paginate: false
---

# 개요  

MsSQL 데이터 베이스를 사용하면 flyway 로 형상 관리를 할 수가 없다. DB 형상 관리가 필수인 프로젝트에서 형상 관리를 사람이 직접할 수 가 없으며 강제성 부여와 실패시 배포 금지 등 다양한 제약들이 필요하다. flyway 가 정말 안 되는지 다방면으로 여러 시도를 하거나 다른 라이브러리를 찾아서 적용해보거나 직접 구현해보는 방법들이 있을 것이다.  

아래와 같은 이유로 직접 구현하기로 했다.  
- 현 상황에서는 테이블 생성 쿼리(DDL)와 JpaEntity 만 비교하면 되므로 그다지 복잡하지 않다.  
- 이참에 **리플렉션 써보자**  
- 언제 구현해보겠어  
- 작고 소중한 나만의 라이브러리  

큰 뼈대가 되는 시나리오를 수립한다.  

# 시나리오  

flyway 도 결국 sql 을 토대로 버저닝을 하고 데이터 베이스 형상 관리를 도와준다. `DDL.sql` 은 테이블을 생성하는 쿼리 `CREATE TABLE LKDCODE(...)` 라고 가정했을 때,
`DDL.sql` 을 기준으로 `JpaEntity.java` 를 비교해서 유효성 검사를 수행하면 된다. (제약 사항의 누락, 컬럼과 필드의 타입 불일치, 컬럼과 필드의 속성 불일치 등..)  

`DDL.sql` 은 파일을 잘 읽고 문자열을 잘 파싱하면 된다. 주로 문자열 가공에 대한 내용이기에 생략하고 리플렉션을 사용해 `JpaEntity.java` 클래스에 대한 정보들을 어떻게 가져올 수 있는지 설명한다.  

# 나의 도구 Reflections  

자바 표준 라이브러리인 `java.lang.reflect` 를 사용할 수도 있지만 사용 편의성이 떨어지므로 좀 더 나은 서드파티 라이브러리를 사용한다. 해당 라이브러리를 사용하는 목적은 `JpaEntity.java` 인 클래스를 잘 읽기(?) 위함이다.  

|                |      `java.lang.reflect`      |       `org.reflections`        |
| :------------: | :---------------------------: | :----------------------------: |
| 클래스 로딩 및 정보 조회 |   기본적인 클래스 로딩과<br>정보 조회 가능    | 간편한 클래스 패스 스캔 및<br>메타데이터 조회 제공 |
|    애노테이션 처리    |   직접 애노테이션을 조회하고<br>처리해야 함    |   애노테이션을 기반으로 검색 등<br>고수준 처리   |
|    서브타입 검색     |  직접 서브타입을 찾기 위한<br>로직 구현 필요   |     서브타입 검색을 위한<br>메서드 제공      |
|     성능 최적화     |  기본적인 리플렉션 API로<br>직접 최적화 가능  |  내부적으로 캐싱 등을 통해<br>성능 최적화 시도   |
|     사용 편의성     | 저수준 API로 복잡하고<br>코드량이 많을 수 있음 |   고수준 API 로 간결하고<br>사용하기 쉬움    |

깃허브를 참고해 사용법 등을 볼 수 있으며 아래는 gradle 의존성  

[🐈‍⬛ Github](https://github.com/ronmamo/reflections)  

```yaml
# gradle
implementation 'org.reflections:reflections:0.10.2'
```


의존성을 추가하는 것만으로 준비는 마쳤고 아래의 로직 순서대로 구현한다.  
1. `DDL.sql` 에 작성된 쿼리를 잘 읽고 잘 파싱한다. *(생략)*  
2. 경로를 지정해 `JpaEntity.java` 클래스들을 전부 읽어온다.  
3. 필요한 필드 및 어노테이션 등에 작성된 값들을 추출한다.  
4. `DDL.sql` 과 `JpaEntity.java` 를 비교한다.  

# 경로를 지정해 JpaEntity.java 읽어오기  

지정된 패키지 안에 있는 모든 `JpaEntity.java` 를 읽어온다. 타겟팅 할 요소는 `jakarta.persistence.@Table` 어노테이션이며 해당 어노테이션이 적용된 클래스들을 동적으로 수집한다.  

```java
import jakarta.persistence.Table;  
import org.reflections.Reflections;  
import org.reflections.Store;  
import org.reflections.util.QueryFunction;  
  
import java.util.Set;  
  
import static org.reflections.scanners.Scanners.TypesAnnotated;

final class JpaEntityReader {  
    private static final String PACKAGE_NAME = "lkdcode.entity";  
    private static final QueryFunction<Store, Class<?>> TARGET = TypesAnnotated.with(Table.class).asClass();  
    private static final Reflections REFLECTIONS = new org.reflections.Reflections(PACKAGE_NAME, TypesAnnotated);  
  
    Set<Class<?>> getJpaEntities() {  
        return Collections.unmodifiableSet(REFLECTIONS.get(TARGET));
    }
}
```

- `String PACKAGE_NAME`: 스캐닝할 패키지  
- `TypesAnnotated.with(Table.class).asClass()`: 리플렉션의 스캐너를 통해 `jakarta.persistence.@Table` 어노테이션이 붙어있는 모든 클래스들을 가져온다.  
	- `org.reflections.util.QueryFunction`: 결과 추출을 위한 함수형 인터페이스  
	- `org.reflections.Store`: 스캔된 메타데이터를 저장하는데 사용  
	- `java.lang.Class`: 읽어온 클래스(클래스의 메타 데이터를 나타내는 객체)  
- `Reflections REFLECTIONS`: 리플렉션을 통해 스캔된 메타데이터를 관리하고 조회할 수 있는 객체  
- `getJpaEntities()`: 수정 불가 뷰로 리턴한다.  

리턴 타입이 Set 인데 만약 순서 보장이 필요하다면 아래와 같이 적용할 수 있다.  
`@Table` 어노테이션의 `name` 속성을 이용해 정렬할 수 있다.  

```java
private List<Class<?>> getJpaEntityList() {  
    return reader.getJpaEntities().stream()  
        .sorted((v1, v2) -> {  
            final var v1TableName = getNameToUpperCase(v1);  
            final var v2TableName = getNameToUpperCase(v2);  
  
            return v1TableName.compareTo(v2TableName);  
        })  
        .toList();  
}  
  
private static String getNameToUpperCase(final Class<?> v) {  
    return v.getAnnotation(Table.class).name().toUpperCase();  
}
```

## Example `LkdCodeJpaEntity.java`  

예시 `JpaEntity` 클래스이다.  
`JpaEntity` 에서 복합키를 사용하는 경우이며 중첩 클래스로 구현되어있다고 가정한다.  

복합키도 아니고 중첩클래스가 아니라면 고유키 비교 검사를 위해 `jakarta.persistence.@Id` 어노테이션을 타겟팅하면 된다.  

```java
import lkdcode.validator.ValueArgumentValidator;  
import jakarta.persistence.*;  
import jakarta.validation.constraints.NotNull;  
import jakarta.validation.constraints.Size;  
import lombok.Builder;  
import lombok.EqualsAndHashCode;  
import lombok.Getter;  
import lombok.NoArgsConstructor;  
  
import java.io.Serializable;  
import java.time.LocalDateTime;  
  
import static lombok.AccessLevel.PROTECTED;  
  
@Getter  
@Entity  
@Table(name = "LKDCODE")  
@NoArgsConstructor(access = PROTECTED)  
public class LKDCodeJpaEntity {  
  
    @Getter  
    @Embeddable
    @EqualsAndHashCode
    @NoArgsConstructor(access = PROTECTED)  
    public static class LKDCodeId implements Serializable {  
  
        @NotNull  
        @Size(max = 20)  
        @Column(name = "FIRST_KEY", nullable = false, length = 20)  
        private String firstKey;  
  
        @NotNull  
        @Size(max = 20)  
        @Column(name = "SECOND_KEY", nullable = false, length = 20)  
        private String secondKey;  
  
        @Builder  
        public LKDCodeId(String firstKey, String secondKey) {  
            this.firstKey = firstKey;  
            this.secondKey = secondKey;  
            ValueArgumentValidator.validate(this);  
        }  
    }  
  
    @NotNull  
    @EmbeddedId  
    private LKDCodeJpaEntity.LKDCodeId id;  
  
    @NotNull  
    @Size(max = 20)  
    @Column(name = "NAME", nullable = false, length = 20)  
    private String name;  
  
    @Column(name = "RATE")  
    private Double rate;  
  
    @NotNull  
    @Column(name = "LKD_TIME", nullable = false, columnDefinition = "datetime")  
    private LocalDateTime lkdTime;  
  
    @Builder  
    public LKDCodeJpaEntity(LKDCodeId id, String name, Double rate, LocalDateTime lkdTime) {  
        this.id = id;  
        this.name = name;  
        this.rate = rate;  
        this.lkdTime = lkdTime;  
        ValueArgumentValidator.validate(this);  
    }  
}
```

**`ValueArgumentValidator.validate(this);`**: 인스턴스 생성 유효성 검사   

값 객체를 생성할 때 유효성 검사를 바로 수행해주면 이점들이 많다. 리소스를 아낀다던가, 빠른 추적이 가능하다던가 등, 이를 위한 검증 클래스이다. `jakarta.validation` 를 사용해 Java Bean Validation Api 기반으로 유효성 검사를 수행한다. 상속을 사용해도 되지만 `java.16 record` 때문에 정적 클래스로 사용한다.  

```java
@Slf4j  
public final class ValueArgumentValidator {  
    private static final Validator validator;  
  
    static {  
        validator = Validation.buildDefaultValidatorFactory()  
            .getValidator();  
    }  
  
    public static <T> void validate(final T vo) throws LkdCodeException {  
        final var violations = validator.validate(vo);  
  
        if (!violations.isEmpty()) {  
            final var message = new StringBuilder();  
  
            violations.forEach(v -> message  
                .append(System.lineSeparator())  
                .append(" - ")  
                .append(v.getPropertyPath().toString())  
                .append(" : ")  
                .append(v.getMessage())  
            );  
  
            log.info("[   ArgumentValidator.validate   ]  Class-Path: {} \n Invalid fields: {}", vo.getClass(), message);  
            throw new LkdCodeException(INVALID_VALUE);  
        }  
    }  
}
```


`DDL.sql` 과 `LkdCodeJpaEntity.java` 에 명시되어 있는 어노테이션들의 메타정보들을 비교해 유효성 검사를 수행할 것이다.  

위의 `LkdCodeJpaEntity.java` 는 복합키로써 `@EmbeddedId` 와 `@Embeddable` 가 있으며 중첩 클래스도 존재한다. 중첩 클래스의 필드 정보를 얻는다.  

```java
public List<Field> extractFieldList(final Class<?> clazz) {  
    return Stream.concat(  
            Arrays.stream(clazz.getDeclaredClasses())  
                .filter(e -> e.isAnnotationPresent(Embeddable.class))  
                .flatMap(e -> Arrays.stream(e.getDeclaredFields())),  
  
            Arrays.stream(clazz.getDeclaredFields()))  
        .toList();
}
```

**`Arrays.stream(clazz.getDeclaredClasses())`**: `Class<?> clazz` 의 private 접근 제한자들을 포함한 모든 클래스들을 배열로 얻고 stream()으로 돌려준다.  

**`.filter(e -> e.isAnnotationPresent(Embeddable.class))`**: 를 통해 `@Embeddable` 어노테이션 Predicate 를 적용하고  

**`.flatMap(e -> Arrays.stream(e.getDeclaredFields()))`**: private 접근 제한자를 포함해 모든 필드를 얻은 후 flatMap 을 통해 하나의 스트림으로 평탄화 한다.  

**`Arrays.stream(clazz.getDeclaredFields()))`**: private 접근 제한자를 포함한 모든 필드를 얻는다.  

이후에 `Stream.concat` 으로 하나의 `List` 로 만들어준다.  

`Field` 는 클래스의 개별 필드에 대한 메타 정보이며 위의 중첩 클래스(복합키용클래스)를 포함한 모든 필드를 얻었다. 필요에 따라 Predicate 를 적용해 원하는 필드들을 뽑아낼 수 있다.  

예를 들어 참조 관계를 얻기 위해 `@ManyToOne` `@OneToMany` 등 적용할 수 있으며
필드뿐만 아니라 어노테이션도 얻을 수 있다.  

```java
/* - Field 추출하기 - */
public List<Field> extractForeignColumns(final Class<?> clazz) {  
    return Stream.concat(  
            Arrays.stream(clazz.getDeclaredFields())  
                .filter(annotationCondition(JoinColumn.class)),  
  
            Arrays.stream(clazz.getDeclaredFields())  
                .filter(annotationCondition(JoinColumns.class))  
        )  
        .distinct()  
        .toList();  
}

/* - JoinColumn 어노테이션 추출하기 - */
public List<JoinColumn> extractJoinColumns(final Class<?> clazz) {  
    return Stream.concat(  
            Arrays.stream(clazz.getDeclaredFields())  
                .filter(annotationCondition(JoinColumn.class))  
                .map(e -> e.getAnnotation(JoinColumn.class)),  
  
            Arrays.stream(clazz.getDeclaredFields())  
                .filter(annotationCondition(JoinColumns.class))  
                .flatMap(e -> Arrays.stream(e.getAnnotation(JoinColumns.class).value())))  
        .toList();  
}

/* - 공통 Predicate - */
private static Predicate<Field> annotationCondition(final Class<? extends Annotation> annotation) {  
    return e -> e.isAnnotationPresent(annotation);  
}
```


# DDL.sql 과 JpaEntity.java 비교하기  

`LkdCodeJpaEntity.java`에서 `jakarta.persistence.@Column` 들을 모두 추출했다고 가정하고 유효성 검사를 진행해보자. `DDL.sql` 도 잘 읽고 잘 파싱했다고 가정한다.  

```java
package jakarta.persistence;  
  
import java.lang.annotation.ElementType;  
import java.lang.annotation.Retention;  
import java.lang.annotation.RetentionPolicy;  
import java.lang.annotation.Target;  
  
@Target({ElementType.METHOD, ElementType.FIELD})  
@Retention(RetentionPolicy.RUNTIME)  
public @interface Column {  
    String name() default "";  
  
    boolean unique() default false;  
  
    boolean nullable() default true;  
  
    boolean insertable() default true;  
  
    boolean updatable() default true;  
  
    String columnDefinition() default "";  
  
    String table() default "";  
  
    int length() default 255;  
  
    int precision() default 0;  
  
    int scale() default 0;  
}
```

예를 들어 `DDL.sql` 을 통해 특정 컬럼의 `NOT NULL` 제약을 얻었다고 가정해보자면
`@Column.nullable()` 속성과 비교해주면 된다.  

```java
public void valid(String query, Column column, Field field) {  
    if (query.equals("NOT NULL")) {  
        if (column.nullable()) {  
            throw new LkdCodeException("FieldName: " + field.getName() + ", @Column.nullable 이 누락됐습니다.");  
        }
  
        if (!field.isAnnotationPresent(NotNull.class)) {  
            throw new LkdCodeException("FieldName: " + field.getName() + ", @NotNull.class 가 누락됐습니다.");  
        }
    }  
}
```

`String query` 는 `DDL.sql` 을 파싱해서 얻은 컬럼의 NULL 에 제약 사항이다. "NOT NULL" or "NULL" 이렇게 2개만 존재한다고 가정한다. (enum 으로 관리하던 뭘하던 ok)  

**`if (column.nullable()) {...}`**: `@Column` 의 속성 중 하나인 `nullable()` 의 값을 얻어 비교한다. 이 때 `default 값`을 주의해야 한다.  

**`if (!field.isAnnotationPresent(NotNull.class)) {...}`**: `Field` 를 통해 `jakarta.validation.constraints.@NotNull` 어노테이션이 적용됐는지 검사한다. 생성자에서 유효성 검사할 때 사용하던 어노테이션과 같다.  

`DDL.sql`에 작성된 특정 컬럼에 대한 Null 제약 유효성 검사를 위해 `LkdCodeJpaEntity.java` 의 `Field` 와 `@Column` 어노테이션을 사용했다.  

`@Column.columnDefinition()` 속성을 통해 데이터베이스 수준에서 컬럼의 타입, 제약 조건, 기본값 등을 명시적으로 정의할 수 있다.  

`LkdCodeJpaEntity.java` 에서 `LocalDateTime lkdTime` 의 `columnDefinition = "datetime"` 으로 명시되어 있는데 이 또한 비교가 가능하다.  

`DDL.sql` 에서 특정 컬럼의 타입이 "DATETIME" 이라면 아래와 같이 비교할 수 있다. 컬럼의 타입, 속성 등을 검증한다.  

```java
public void valid(String query, Column column, Field field) {  
    if (query.equalsIgnoreCase("DATETIME")) {  
        if (!column.columnDefinition().equals("datetime")) {  
            throw new LkdCodeException("FieldName: " + field.getName() + ", @Column columnDefinition 속성이 누락됐습니다.");  
        }  
  
        if (field.getType() != LocalDateTime.class) {  
            throw new LkdCodeException("FieldName: " + field.getName() + ", JpaEntity 의 필드 타입이 LocalDateTime.class 가 아닙니다.");  
        }  
    }  
}
```

**`if (!column.columnDefinition().equals("datetime")) {...}`**: `@Column.columnDefinition()` 속성 값을 비교한다.  

**`if (field.getType() != LocalDateTime.class) {...}`**: 필드의 타입을 비교한다.  

이외에도 `@Table.name` 속성을 통해 테이블 이름이 중복되었는지  
`LkdCodeJpaEntity.java` 클래스는 있지만 `DDL.sql` 이 없다던지(혹은 반대) 등
다양한 유효성 검사를 수행할 수 있다.  

리플렉션을 사용해 특정 패키지안에 특정 조건의 클래스들을 수집하였고 검사에 필요한 어노테이션, 필드 등의 속성 값들을 가져와 유효성 검사를 진행했다.  