---
layout: single
title: "멀티 모듈 구성 상황과 적용"
categories: project
tag: [주차의상상은현실이된다, 멀티모듈]
typora-root-url: ../
toc: true
author_profile: false
---

“주차의 상상은 현실이된다” 프로젝트를 진행하면서 멀티 모듈로 리팩토링한 과정을 기록하고자 한다.

## 상황

멀티 모듈을 구성하게 된 상황: ‘주차의 상상은 현실이 된다’ 프로젝트에서 주차장의 데이터를 가져올 때, 스케줄러를 통해서 가져오고 있다. 주차장의 경우, 현재 서울시와 부산시의 공영 주차장 데이터를 가져오고 있지만 추가로 다른 시나 다른 데이터도 필요하다. 주차장 API를 추가할 때마다 실행되고 있는 서비스를 중지하고 배포하기보다는 기존 모듈과 스케줄러 모듈을 분리해서 스케줄러에 다른 API를 추가하더라도 기존 서비스는 동작하도록 하는게 우리 서비스의 가용성을 높이는 것이라 판단했다.

## 모듈 구성

<img src="/images/2024-04-12/image-20240412201325584.png" alt="image-20240412201325584" style="zoom:33%;" />

기존 프로젝트에서 스케줄러 모듈을 나눈다면 위 그림처럼 된다. 스케줄러는 기존 APP 모듈을 의존하며 서비스가 동작하고, APP 모듈은 API 서버로 동작한다. 이렇게 했을 때 외부 API(다른시의 주차장 데이터)를 추가하면 스케줄러 모듈만 다시 배포하면 되지만, API 변경이 일어났을 때는 APP 모듈을 배포하고 스케줄러도 다시 배포되어야한다.

이 방식을 개선하고자 생각한 방법은 다음과 같다.

<img src="/images/2024-04-12/image-20240412201517574.png" alt="image-20240412201517574" style="zoom:33%;" />

실행 단위인 app-api, app-scheduler로 나누고 공통적으로 사용되는 도메인(도메인 및 인프라) 모듈을 분리하는 것이다.
이렇게 분리했을 때는 다음과 같은 장점이 있었다.

1. **변경 사항에 따른 배포**
   API, 스케줄러가 변경되었을 때 해당 모듈만 다시 배포하면 된다.

2. **의존성 분리**
   모듈 별로 의존성을 둘 수 있기 때문에 각 모듈에 필요한 의존성만 알면 된다.
   ex) 도메인 모듈에서 불필요한 Swaggere등과 같은 의존성을 없앨 수 있다.

3. **책임 분리**

   비즈니스 로직의 경우 도메인 모듈에 요청, 응답에 관련한건 API 모듈에 나누기가 쉬워지는 거 같다.
   
   

## 모듈 분리하는 방법

인텔리제이를 기준으로 루트 디렉토리에 오른쪽 마우스로 클릭해서 New -> Module을 클릭하면 모듈을 생성할 수 있다.
(Cmd + N, M 으로도 가능)

![image-20240412222023902](/images/2024-04-12/image-20240412222023902.png)

이후, 필요한 모듈을 생성하고 `settings.gradle` 을 보면 다음과 같다.
```groovy
rootProject.name = 'parking'
include 'domain'
include 'app-api'
include 'app-scheduler'
```

- 생성한 domain, app-api, app-scheduler 모듈이 include 되어 있는걸 확인할 수 있다.

이후, `build.gradle` 을 수정해주면 된다.

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

repositories {
    mavenCentral()
}

bootJar.enabled = false

subprojects {
    repositories {
        mavenCentral()
    }

    group = 'com.parkingcomestrue'
    version = '0.0.1-SNAPSHOT'

    apply plugin: 'java'
    apply plugin: 'java-library'
    apply plugin: 'org.springframework.boot'
    apply plugin: 'io.spring.dependency-management'


    dependencies {
        compileOnly 'org.projectlombok:lombok'
        annotationProcessor 'org.projectlombok:lombok'

        runtimeOnly 'com.h2database:h2'
        runtimeOnly 'com.mysql:mysql-connector-j'

        implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
        implementation 'org.springframework.boot:spring-boot-starter-web'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
    }

    test {
        useJUnitPlatform()
    }
}
```

- `subprojects` 구문에서 하위에 공통으로 사용할 의존성을 넣어주면 된다..

API 모듈이나 스케줄러 모듈의 경우 도메인 모듈을 의존하기 때문에 다음과 같이 `dependencies` 에 의존성을 추가해줘야 한다.
```groovy
dependencies {
    implementation project(':domain')
  	... 이하 필요한 의존성 추가
}
```



### TestFixture

추가로, 우리 프로젝트에서 테스트시 Fixture를 사용하는데 공통으로 Fixture를 사용하기 위해서는 다음과 같은 작업이 필요하다.

TestFixture를 정의한 모듈의 `build.gradle`에 다음과 같이 플러그인을 추가해준다.
```groovy
plugins {
    id 'java-library'
    id 'java-test-fixtures'
    id 'maven-publish'
}

```

Fixture에서 사용하는 의존성 같은 경우는 다음과 같이 `testFixturesImplementation` 으로 추가해줘야한다.

```groovy
dependencies {
    testFixturesImplementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    testFixturesImplementation group: 'org.hibernate.orm', name: 'hibernate-spatial', version: '6.3.1.Final'
}
```

- 우리 프로젝트 같은 경우는 Repository와 Point를 사용하고 있어서 위와 같이 추가해주었다.

이렇게 추가해주면 다음과 같이 모듈의 src 경로에 `testFixtures` 를 생성할 수 있다.

![image-20240413015255140](/images/2024-04-12/image-20240413015255140.png)

이후, `Fixture` 객체를 `testFixtures` 하위에 생성해준다.

![image-20240413015818403](/images/2024-04-12/image-20240413015818403.png)

그리고 `Fixture`를 사용하는 모듈의 `build.gradle` 에 다음과 같이 의존성을 추가해주면 된다.

```groovy
dependencies {
    testImplementation(testFixtures(project(":domain")))
}
```

- project부분에는 `testFixture` 를 정의한 모듈명을 써주면 된다. (우리 프로젝트에서는 `domain`)

---

## 마치며

이번에는 멀티 모듈을 구성한 상황과 적용방법을 기록하였고

멀티 모듈을 구성하면서 수정한 CI CD 부분은 다음 포스팅에 작성할 예정이다.
