---
date: 2024-06-22 21:40:27
layout: post
title: docker + kafka + spring-boot App 연동하기
subtitle: 카프카 기본 설정, docker-compose 활용하기
description: 카프카 기본 설정, docker-compose 활용하기
image: '/images/dockerKafkaSpringBoot/dockerKafkaSpringBoot3.png'
optimized_image: '/images/dockerKafkaSpringBoot/dockerKafkaSpringBoot3.png'
category: kafka
tags: 
    - kafka
    - docker
    - zookeeper
author: lkdcode
---

docker 를 이용해 카프카와 주키퍼 서버를 띄우고  
스프링 부트 애플리케이션에서 이벤트(메시지)를 발행하고 읽는 과정을 담았다.  
도커 설치 이후부터 설명한다.  

# docker-compose

OS 에 상관없이 개발 환경을 맞출 수 있고 간단한 스크립트로 쉽게 컨테이너를 띄울 수 있다.  

```yaml
version: '3.8'

services:
  zookeeper:
    image: wurstmeister/zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"

  kafka:
    image: wurstmeister/kafka
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: 127.0.0.1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

## 실행 후 docker desktop 에서 확인하기

![image-center]({{ '/images/dockerKafkaSpringBoot/2.png' | absolute_url }}){: .align-center}

정상적으로 실행되었다. cli를 통해 확인할 수 있지만 cli 대신 spring-boot 로 바로 연결한다.

# kafka

카프카의 구성요소를 살짝 알아본다.

![image-center]({{ '/images/dockerKafkaSpringBoot/1.png' | absolute_url }}){: .align-center}

- <strong>zookeeper</strong>: 카프카 클러스터를 관리한다. 카프카 클러스터와 관련된 정보 기록 및 관리를 담당한다.

- <strong>Kafka cluster</strong>: 카프카에서 사용할 메시지 저장소다. 하나의 카프카 클러스터는 여러 개의 브로커로 구성된다. 브로커는 각각의 서버이며 메시지를 나눠서 저장하거나 이중화 처리 장애 대응 등의 역할을 수행한다.

- <strong>Kafka producer</strong>: 카프카 클러스터에 메시지를 보내는 역할을 수행한다.

- <strong>Kafka consumer</strong>: 카프카 클러스터에 메시지를 읽어오는 역할을 수행한다.

# spring-boot

spring-boot application에서 kafka 를 사용하기 위한 설정 방법이다.

## 의존성

```groovy
implementation 'org.springframework.kafka:spring-kafka'
testImplementation 'org.springframework.kafka:spring-kafka-test'
```

spring boot application 에 카프카 의존성을 추가해준다.

## 설정 클래스

### producer config

```java
@EnableKafka
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        final HashMap<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

`"localhost:9092"`: docker 로 띄운 카프카 서버의 호스트와 포트를 지정한다.
`"StringSerializer.class"`: 이벤트(메시지)를 보낼 때 key 와 value 를 각각 String 타입으로 직렬화해서 보낸다. (필요에 따라 다른 데이터 타입이나 json 등으로 직렬화 할 수 있다.)

### consumer config

```java
@EnableKafka
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        final HashMap<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
        final ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}
```

`"localhost:9092"`: 마찬가지로 docker 로 띄운 카프카 서버의 호스트와 포트를 지정한다.
`"StringSerializer.class"`: 이벤트(메시지)를 가져올 때 key 와 value 를 각각 String 타입으로 역직렬화해서 읽어온다. (필요에 따라 다른 데이터 타입이나 json 등으로 역직렬화 할 수 있다.)

# 실행해보기

`SpringBootApp1` 은 port 8100, `SpringBootApp2` 은 port 8200 으로 띄운 후  
`SpringBootApp2:8200` --- 메시지 발행 --> `SpringBootApp1:8100` 소비하도록 해보자.  

## producer code (port:8200)

```java
@GetMapping("/kafka")
public String getOrderWithKafka() {
    kafka.send("order-topic", "order 가 보낸 이벤트!");
    return "getOrderWithKafka";
}
```

`order-topic` 으로 이벤트를 발행하고 문자열을 리턴한다.  

## consumer code (port:8100)

```java
@KafkaListener(topics = "order-topic", groupId = "order-group1")
public String getItemWithKafka(String event) {
    System.out.println("ItemApi.getItemWithKafka: " + event);
    return "getItemWithKafka";
}
```

`order-topic` 으로 이벤트를 구독한다. 해당 이벤트를 받아서 콘솔에 출력한다.

이벤트를 잘 발행하고 잘 소비했다!

![image-center]({{ '/images/dockerKafkaSpringBoot/3.png' | absolute_url }}){: .align-center}