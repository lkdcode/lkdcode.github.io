---
date: 2024-04-22 12:07:52
layout: post
title: ...can not symbol 컨테이너가 클래스를 못 찾아서 실행이 안 된다면?
subtitle: 간단하게 해결해보자
description: 무슨 에러인지 찾기 힘들다면.......
image: '/images/symbol/rebuild.png'
optimized_image: '/images/symbol/cannotsymbol.png'
category: error
tags:
  - bug?
  - spring boot application
author: lkdcode
---

Spring Application 을 실행(Run) 하거나 `build` 를 할 때 `...can not symbol` 이라는 에러가 나면서 실행이 안 되는 이유를 모를 때가 있다. 특정 위치에서 안 되거나 패키지 구조가 잘 못 되지 않았음에도 실행이 안 된다면 `build` 를 다시 해보자.

# IntelliJ

![image-center]({{ '/images/symbol/rebuild.png' | absolute_url }}){: .align-center}

# Maven

```bash
# 방법1
$ mvn clean install

# 방법2
$ mvn clean package
```

# Gradle

```bash
# 방법1
$ gradle clean build

# 방법2
$ ./gradlew clean build
```