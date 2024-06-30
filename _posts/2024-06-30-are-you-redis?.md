---
date: 2024-06-30 08:42:13
layout: post
title: "are you Redis?"
subtitle: Remote Dictionary Server
description: in-memory 의 캐시템! 레디스가 무엇이고 어떤 녀석인지 간략하게 알아본다.
image: '/images/are-you-redis/are-you-redis-thumbnail.png'
optimized_image: '/images/are-you-redis/are-you-redis-thumbnail.png'
category: redis
tags:
  - redis
  - cache
  - spring cache
author: lkdcode
paginate: false
---

database, cache, message broker 등으로 사용하는 in-memory 데이터 베이스 서버로 여러 자료 구조를 제공하며 널리 사용되고 있다. 기본 개요 및 개념들을 살펴본다.

# 메모리 계층 구조
![image-center]({{ '/images/are-you-redis/are-you-redis1.png' | absolute_url }}){: .align-center}
`register`: 가장 빠른 메모리, CPU 내부에 있으며, 매우 작은 용량. 연산에 직접적으로 사용  
`CPU Cache`: Register 다음으로 빠름. CPU 내부 또는 매우 가까운 곳에 위치. 자주 사용하는 데이터 저장  
`Main memory(DRAM)`: 일반적으로 말하는 RAM. 용량이 크고, 속도는 빠르지만, 캐시보다는 느림, 실행 중인 프로그램과 데이터를 저장  
`storage(SSD,HDD)`: 가장 느린 저장 장치, 매우 큰 용량, 장기 데이터 저장  

레디스는 주로 Main Memory(DRAM)를 사용하는 데이터베이스로, 매우 빠른 속도를 제공한다. storage 보다 데이터 용량에 대한 한계가 있으므로 cache 로 사용한다. (Database 보다 더 빠른 Memory 에 <ins>*더 자주 접근하고*, *덜 자주 바뀌는*</ins> 데이터를 저장하여 사용하자) 동작 방식은 메모리 계층 구조와 밀접한 연관이 있다.
1. 메모리 기반 데이터 저장: 전통적인 디스크 기반 데이터베이스보다 빠름.
2. 데이터 영속성: RDB, AOF 방식을 사용

# Redis 가 지원하는 데이터 유형들
![image-center]({{ '/images/are-you-redis/are-you-redis2.png' | absolute_url }}){: .align-center}
Redis 는 기본적으로 single-threade  
Redis 자료 구조는 Atomic Critical Section 에 대한 동기화 제공  
Redis 는 서로 다른 Transaction Read/Write 를 동기화  

Single Thread 서버 이므로 O(n) 의 시간 복잡도는 주의하여 사용한다.  
메모리 파편화, 가상 메모리등의 이해도가 필요하다.

# 동작 방식
![image-center]({{ '/images/are-you-redis/are-you-redis3.png' | absolute_url }}){: .align-center}

`Packet`: 클라이언트가 보내는 명령어. 패킷 형태로 서버에 도착한다.
`ProcessInputBuffer`: 수신된 패킷이 저장되고 명령어가 처리되기 전까지 보관한다.
`ProccessCommand`: 입력 버퍼에 저장된 패킷으로 실제 명령어가 처리된다. (single thread)

# 메모리 파편화
## 초기 상태
아무 데이터가 추가되지 않은 상태이다. (null null ~)
![image-center]({{ '/images/are-you-redis/are-you-redis4.png' | absolute_url }}){: .align-center}

## 데이터가 추가된 상태
데이터의 '해제' 없이 오직 추가만 된 상태이다. 잘 정렬되어 있다.
![image-center]({{ '/images/are-you-redis/are-you-redis5.png' | absolute_url }}){: .align-center}

## 데이터 할당 & 해제 상태
메모리 '할당'과 '해제' 과정에서 사용 가능한 메모리 공간이 비효율적으로 분할된 상태이다. Redis 와 같은 메모리 기반 데이터 베이스에서도 발생 가능하며, 연속된 큰 메모리 공간이 부족해지고 작은 빈 공간들이 흩어져 있는 상태가 된다. 실제 사용 메모리보다 더 큰 메모리를 사용 중이라 인식된다.
![image-center]({{ '/images/are-you-redis/are-you-redis6.png' | absolute_url }}){: .align-center}

## 해결 방법
`메모리 최적화 설정`: Redis 설정에서 `activedefrag` 옵션을 활성화하여 자동으로 메모리 파편화를 줄일 수 있다.  
`메모리 재할당`: 데이터 구조를 최적화하여 메모리 재할당을 최소화하고, 파편화를 줄일 수 있다.  

# 가상 메모리(Virtual Memory)와 스왑(Swap)
Redis 가 물리적 메모리를 효율적으로 사용하는 것을 돕고 메모리 부족 문제를 해결하는데 사용된다.  

## 가상 메모리(Virtual Memory)
물리적 메모리와 디스크 공간을 결합하여 사용하는 메모리 관리 기술  

## 스왑(Swap)
RAM이 부족할 때 사용하지 않는 데이터 블록을 디스크의 특정 영역에 임시로 저장하는 과정이다. 자주 접근하지 않는 데이터를 디스크로 스왑하고 스왑된 데이터에 접근해야 할 때 RAM 으로 다시 가져온다. 최신 버전(2.4 이후)는 해당 기능을 사용하지 않는다. 대신 LRU(Least Recently Used) 정책 등을 통해 자주 사용되지 않는 키를 삭제하는 방식을 사용한다.

# Redis 의 백업 AOF, RDB

## AOF(Append Only File)
각 쓰기 명령을 로그 파일에 기록하는 방식, 모든 명령을 디스크에 순차적으로 저장한다.  
1. 명령 기록: 모든 쓰기 명령을 `appendonly.aof` 파일에 기록한다.  
2. 파일 동기화: 설정에 따라 주기적으로 파일을 디스크에 동기화한다.  
3. 복구 시 재실행: 데이터베이스 복구 시 AOF 파일의 모든 명령을 재실행하여 데이터를 복구한다.  

장/단점
1. 데이터 손실 최소화  
2. 실시간 업데이트  
3. 큰 파일 크기  
4. 느린 복구 속도  

![image-center]({{ '/images/are-you-redis/are-you-redis7.png' | absolute_url }}){: .align-center}

## RDB(Redis Database Backup)
특정 시점의 스냅샷으로 저장하는 방식, 일정한 간격으로 데이터를 덤프하여 디스크에 저장한다.  

1. 스냅샷 생성: 주기적으로 데이터의 전체 스냅샷을 생성하여 디스크에 저장한다.  
2. 파일 저장: RDB 파일 형식으로 저장되며, 이 파일을 사용하여 데이터베이스를 복구할 수 있다.  

장/단점
1. 빠른 복구
2. 낮은 성능 영향  
3. 데이터 손실 가능성

![image-center]({{ '/images/are-you-redis/are-you-redis8.png' | absolute_url }}){: .align-center}

<table>
  <thead>
    <tr>
      <th></th>
      <th>AOF</th>
      <th>RDB</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>저장 형식</td>
      <td>명령어 로그</td>
      <td>데이터 스냅샷</td>
    </tr>
    <tr>
      <td>데이터 일관성</td>
      <td>높음</td>
      <td>낮음</td>
    </tr>
    <tr>
      <td>파일 크기</td>
      <td>큼</td>
      <td>적음</td>
    </tr>
    <tr>
      <td>디스크 I/O 부하</td>
      <td>높음</td>
      <td>낮음</td>
    </tr>
    <tr>
      <td>복구 속도</td>
      <td>느림</td>
      <td>빠름</td>
    </tr>
  </tbody>
</table>