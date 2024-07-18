---
date: 2024-07-16 11:20:08
layout: post
title: "Spring Event 개요"
subtitle: Event 로 의존성을 끊어보자!
description: 각각의 클래스들이 얽히고 섥힐 때 추가 기능은 의존성을 더욱 복잡하게 만든다. 이러한 의존성을 이벤트로 해소할 수 있다.
image: '/images/spring-event/개요/spring-event-back.png'
optimized_image: '/images/spring-event/개요/spring-event-back.png'
category: event
tags: 
  - event
  - spring event
author: lkdcode
paginate: false
---

클라이언트의 요구 사항을 비즈니스 로직으로 풀어낼 때 연쇄되는 요청들이 있을 수 있다. 가령 폴더 삭제 요청을 보낼 때 폴더 하위에 있는 파일들을 삭제한다던가 말이다. 폴더보다 더 상위 개념으로 묶인다면 삭제 요청은 연쇄적으로 작용한다.

# 이벤트를 사용하지 않는다면
삭제 시나리오를 이어서 설명한다. 이벤트를 사용하지 않는 다면 아래 그림과 같이 클라이언트의 삭제 요청에 대해 의존성을 가진채 로직을 수행하게 된다.  

![image-center]({{ '/images/spring-event/개요/se0-1.png' | absolute_url }}){: .align-center}

코드를 요약해보자면 아래와 같다. `A-Service` 는 아래처럼 의존성을 가지고 있는 점이 중요하다.  

```java
public class A-Service {
	/* 어떤 서비스들이 있는지 알고 있다. */
    // B-Service.delete();
    // C-Service.delete();
}
```

여기에 새로운 요구사항이 추가되는데 D-Service 도 삭제를 함께하는 것이다.  

![image-center]({{ '/images/spring-event/개요/se0-2.png' | absolute_url }}){: .align-center}

아래와 같이 A-Service 는 D-Service 에 대해서도 의존성을 가지게 될 것이다.  

```java
public class A-Service {
    /*...*/
    // D-Service.delete();
}
```

# 이벤트를 사용한다면?  

![image-center]({{ '/images/spring-event/개요/se0-3.png' | absolute_url }}){: .align-center}

위의 시나리오처럼 `A-Service` 는 클라이언트의 삭제 요청을 받게 되는데 다른 점이 있다면 직접 다른 서비스들을 호출해서 로직을 처리하는 것이 아니라 `삭제 이벤트`를 발행하면 된다. 이벤트는 다른 서비스들과 의존성이 없다.  

```java
public class A-Service {
    /* 삭제 이벤트 발행! (어떤 서비스들이 있는지 모름) */
}
```

![image-center]({{ '/images/spring-event/개요/se0-4.png' | absolute_url }}){: .align-center}

기존과 같은 요청을 처리하기 위해서는 `B-Service` 와 `C-Service` 는 이벤트를 받을 준비만 하면 된다. 이벤트를 수신하여 올바르게 처리하면 된다.  

Spring-Event 를 사용하기 위해서는 위와 같이 이벤트를 발행하는 발행자, 이벤트를 수신하는 수신자, 그리고 이벤트 그 자체가 필요하다.  

# 구현하기

## 이벤트  
![image-center]({{ '/images/spring-event/개요/se0-5.png' | absolute_url }}){: .align-center}

이벤트를 발행하고 이벤트를 수신한다. 이벤트는 하나의 상자라고도 볼 수 있다. 발행자와 수신자는 이 이벤트 상자를 통해 작업을 수행하게 된다.  

![image-center]({{ '/images/spring-event/개요/se0-6.png' | absolute_url }}){: .align-center}

Spring Application 에서 이벤트를 나타내는 추상 클래스이다. 이벤트를 발행하거나 수신할 때 필요한 객체로 이벤트 기반 프로그래밍에서 특정 이벤트가 발생했을 때 이를 감지하고 처리하기 위해 사용된다.  

직렬화에 사용되는 고유 ID, 이벤트가 발생한 시간을 기록하는 필드, 이벤트의 출처(발생원) 객체 등을 나타내는 값들로 이루어져 있다.  

Spring Application 에서 이벤트를 사용하고 싶다면 해당 클래스를 상속해 구현하면 된다.  
아래 코드는 예시이다. 이벤트를 발행하고 수신하는 쪽에서 `String message` 를 받아 작업을 수행하게 된다.  

```java
public class MyCustomEvent extends ApplicationEvent {
    private final String message;

    public MyCustomEvent(Object source, String message) {
        super(source);
        this.message = message;
    }

    public String getMessage() {
        return message;
    }
}
```

## 이벤트 발행자

![image-center]({{ '/images/spring-event/개요/se0-7.png' | absolute_url }}){: .align-center}

이벤트를 직접 발행하는 역할을 수행한다. 어떤 이벤트를 발행할 것이지는 `publisher` 로 부터 시작된다. 이벤트를 발행할 때 위의 정의한 이벤트 객체를 send 하게 된다.  

![image-center]({{ '/images/spring-event/개요/se0-8.png' | absolute_url }}){: .align-center}

이벤트 발행자는 위의 인터페이스를 통해 이벤트를 발행하게 된다. 직접 `implements ApplicationEventPublisher` 하지 않아도 발행할 수 있다. 스프링 컨텍스트 내부적으로 구현하는 빈을 제공하기 때문이다. (ApplicationEventPublisher -> AbstractApplicationContext ->  ApplicationEventMulticaster -> SimpleApplicationEventMulticaster)  

직접 `implements ApplicationEventPublisher` 하는 경우  

```java
import org.springframework.context.ApplicationEvent;
import org.springframework.context.ApplicationEventPublisher;

public class CustomEventPublisher implements ApplicationEventPublisher {
    @Override
    public void publishEvent(Object event) {
        // Custom event publishing logic
        System.out.println("Publishing event: " + event);
    }
}
```

주입받아 사용하는 경우  

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Component;

@Service
@RequiredArgsConstructor
public class MyEventPublisher {
    private final ApplicationEventPublisher applicationEventPublisher;

    public void publishCustomEvent(final String message) {
        System.out.println("Publishing custom event. ");
        MyCustomEvent customEvent = new MyCustomEvent(this, message);
        applicationEventPublisher.publishEvent(customEvent);
    }
}
```

특별한 이유가 없다면 주입받아서 사용하는 것이 일반적이고 권장되는 방법이다.  

## 이벤트 수신자
![image-center]({{ '/images/spring-event/개요/se0-9.png' | absolute_url }}){: .align-center}

퍼블리셔가 발행한 이벤트를 수신해 작업을 하는 역할을 수행한다. 간단하게 어노테이션으로 해당 이벤트 객체를 수신할 수 있다.  

![image-center]({{ '/images/spring-event/개요/se0-10.png' | absolute_url }}){: .align-center}

어노테이션외에도 `ApplicationListener` 를 확장해 구현할 수도 있지만 일반적으로 권장되지 않는다. (어노테이션이 더 간편하고 쉽기 때문)  

해당 어노테이션의 속성을 활용해 이벤트를 받을 수 있다.  
- value, classes: 처리할 이벤트 타입을 지정한다.  
- condition: SpEL(Spring Expression Language) 를 사용해 조건부로 이벤트 처리를 결정한다.  
- id: 리스너에게 고유한 식별자를 부여한다.  

```java
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;
import org.springframework.beans.factory.annotation.Value;

@Component
public class MyCustomEventListener {

    // 기본 이벤트 리스너: 특정 이벤트 타입을 지정
    @EventListener(MyCustomEvent.class)
    public void handleMyCustomEvent(MyCustomEvent event) {
        System.out.println("Received MyCustomEvent with message: " + event.getMessage());
    }

    // value() 속성을 사용하여 이벤트 타입 지정
    @EventListener(value = MyCustomEvent.class)
    public void handleEventWithValue(MyCustomEvent event) {
        System.out.println("Received MyCustomEvent with value: " + event.getMessage());
    }

    // condition() 속성을 사용하여 조건부 이벤트 처리
    @EventListener(condition = "#event.message == 'SpecialMessage'")
    public void handleEventWithCondition(MyCustomEvent event) {
        System.out.println("Received special MyCustomEvent with message: " + event.getMessage());
    }

    // id() 속성을 사용하여 이벤트 리스너에 고유 식별자 지정
    @EventListener(id = "customEventListener")
    public void handleEventWithId(MyCustomEvent event) {
        System.out.println("Received MyCustomEvent with id: " + event.getMessage());
    }
}
```