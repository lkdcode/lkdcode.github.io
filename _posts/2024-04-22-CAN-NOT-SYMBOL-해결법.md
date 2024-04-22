---
title: \`...can not symbol\` 컨테이너가 클래스를 못 찾아서 실행이 안 된다면?
image:
  path: /images/symbol/cannotsymbol.png
  thumbnail: /images/symbol/rebuild.png
categories:
  - solution
tags:
  - can not symbol
last_modified_at: 2024-04-22T09:07:52-05:00
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