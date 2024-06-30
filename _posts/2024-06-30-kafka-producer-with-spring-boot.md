---
date: 2024-06-30 07:16:23
layout: post
title: "kafka-producer-with-spring-boot"
subtitle: 카프카 프로듀서 기본 개념
description: 카프카에서 메시지(이벤트)를 발송하는 역할을 수행하는 `producer` 에 대해 간략하게 알아본다.
image: '/images/kafka-producer/kafka-producer-thumbnail.png'
optimized_image: '/images/kafka-producer/kafka-producer-thumbnail.png'
category: kafka
tags:
    - kafka
    - producer
    - spring-boot
author: lkdcode
paginate: false
---

# Kafka Producer
![image-center]({{ '/images/kafka-producer/kafka-producer1.png' | absolute_url }}){: .align-center}
1. `Serializer`: 메시지를 byte 배열로 변환
2. `Partitioner`: 어느 토픽의 파티션으로 보낼지 결정
3. `버퍼&배치`: 메시지 모음
4. `sender`: 배치를 전송

Producer 가 메시지를 보내게 되면 `Serializer` 를 통해 메시지를 byte 배열로 변환하고 `Partitioner` 를 통해 어느 토픽의 파티션으로 보낼지 결정한다. 결정된 메시지는 버퍼의 배치로 들어가게 되고 `sender`를 통해 `kafka broker` 에게 메시지를 전달하게 된다. `Send`와 `Sender` 는 각각 별도의 쓰레드로 동작하므로 `Sender` 가 메시지를 보내는 동안 배치로 메시지를 모으게 되고 또 배치가 찼는지, 배치의 갯수와 상관없이 차례대로 브로커에게 메시지를 전송하게 된다.

# 전송 결과
![image-center]({{ '/images/kafka-producer/kafka-producer2.png' | absolute_url }}){: .align-center}

## 전송 결과 확인하지 않고 보내기

```java
producer.send(new ProducerRecord<>("topic","value"));
```
- 전송 실패를 알 수 없다.
- 실패에 대한 별도 처리가 필요 없는 메시지 전송에 사용한다.

## 전송 결과 확인: Future

```java
package java.util.concurrent;

public interface Future<V> {  
  boolean cancel(boolean mayInterruptIfRunning);  
  boolean isCancelled();
  boolean isDone();
  V get() throws InterruptedException, ExecutionException;  
  V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException; }
```

- 사용 예시

```java
Future<RecordMetadata> f = producer.send(new ProducerRecord<>("topic","value"));
try {
  RecrodMetadata meta = f.get(); // blocking
} catch (ExecutionException ex) { ... }
```
- `Future.get();` 해당 메서드는 블로킹 되기에 배치 효과가 떨어진다. (처리량 저하)
- 처리량이 낮아도 되는 경우에만 사용한다.

## 전송 결과 확인: Callback

```java
package org.apache.kafka.clients.producer;

public interface Callback {
  void onCompletion(RecordMetadata metadata, Exception exception);  
}
```

- 사용 예시

```java
producer.send(new ProducerRecord<>("topic", "value"), new Callback() {
  @Override
  public void onCompletion(RecordMetadata metadata, Exception ex) {... }
});
```
- 함수형 인터페이스로 `Future.get();` 메서드와 다르게 논블로킹 방식으로 처리량 저하가 없다. (배치가 쌓이지 않음)

# 전송 보장과 ack
프로듀서가 보낸 메시지가 성공적으로 잘 도착했는지 메시지 전송 확인 방식에 대해 지정할 수 있다. 브로커는 메시지의 복제와 관리(고가용성, 데이터 내구성)를 위해 리더와 팔로워를 사용하는데 팔로워와 리더에 메시지가 잘 저장되었는지 `ack` 를 통해 설정하여 메시지 전송에 대한 보장을 처리할 수 있다.

![image-center]({{ '/images/kafka-producer/kafka-producer3.png' | absolute_url }}){: .align-center}

## ack = 0
- 서버 응답을 기다리지 않음
- 전송 보장도 zero

## ack = 1
- 파티션의 리더에 저장되면 응답을 받음
- 리더 장애시 메시지 유싱 가능

## ack = all (or -1)
- 모든 리플리카에 저장되면 응답 받음
- 브로커 min.insync.replicas 설정에 따라 달라진다

# ack + min.insync.replicas
## min.insync.replicas (브로커 옵션)
- 프로듀서 ack 옵션이 all 일 대 저장에 성공했다고 응답할 수 있는 동기화된 리플리카 최소 갯수
## 예시

### 레플리카 갯수 3, ack = all, min.insync.replicas = 2
![image-center]({{ '/images/kafka-producer/kafka-producer4.png' | absolute_url }}){: .align-center}
- 리더에 저장하고, 팔로워 중 한 개에 저장하면 성공 응답

### 리플리카 갯수 3, ack = all, min.insync.replicas = 1
![image-center]({{ '/images/kafka-producer/kafka-producer5.png' | absolute_url }}){: .align-center}
- 리더에 저장되면 성공 응답
- ack = 1과 동일 (리더 장애시 메시지 유실 가능)

### 리플리카 갯수 3, ack = all, min.insync.replicas = 3
![image-center]({{ '/images/kafka-producer/kafka-producer6.png' | absolute_url }}){: .align-center}
- 리더와 팔로워 2개에 저장되면 성공 응답
- 팔로워 중 한 개라도 장애가 나면 리플리카 부족으로 저장에 실패

# 실패 대응
## 재시도로 대응
- 재시도 가능한 에러는 재시도 처리
- 브로커 응답 타임 아웃, 일시적인 리더 없음 등

재시도 위치
- 프로듀서는 자체적으로 브로커 전송 과정에서 에러가 발생하면 재시도 가능한 에러에 대해 재전송 시도(retries 속성)
- send() 메서드에서 익셉션 발생시 익셉션 타입에 따라 send() 재호출
- 콜백 메서드에서 익셉션을 받으면 타입에 따라 send() 재호출

무한 재시도는 엄격하게 다뤄야 하며 보통의 경우 사용하지 않는다.

## 기록으로 대응
추후 처리를 위해 기록
- 별도 파일, DB 등을 사용해 기록
- 추후에 수동(또는 자동) 보정 작업 진행

기록 위치
- send() 메서드에서 익셉션 발생 시
- send() 메서드에 전달한 콜백에서 익셉션 받는 경우
- send() 메서드가 리턴한 Future의 get() 메서드에서 익셉션 발생 시

# 재시도와 메시지 중복 전송 가능성
- 브로커 응답이 늦게 와서 재시도할 경우 중복 발송 가능
- enable.idempotence 속성으로 중복 가능성을 줄일 수 있음

![image-center]({{ '/images/kafka-producer/kafka-producer7.png' | absolute_url }}){: .align-center}

# 재시도와 순서
- 재시도는 메시지의 순서를 바꿀 수 있음
- `max.in.flight.requests.per.connection`
블로킹없이 한 커넥션에서 전송할 수 있는 최대 전송중인 요청 개수, 이 값이 1보다 크면 재시도 시점에 따라 메시지 순서가 바뀔 수도 있다. 전송 순서가 중요하다면 해당 값을 `1`로 지정

![image-center]({{ '/images/kafka-producer/kafka-producer8.png' | absolute_url }}){: .align-center}