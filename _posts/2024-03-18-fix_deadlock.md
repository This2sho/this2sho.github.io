---
layout: single
title: "커낵션 풀사이즈 설정과 데드락 해결기"
categories: project
tag: [이돈이면, 데드락, 커낵션풀]
typora-root-url: ../
toc: true
author_profile: false
---

“이돈이면” 프로젝트를 진행하면서 발생한 데드락 해결 과정을 기록하고자 한다.

## 상황

서비스에 적절한 커넥션 풀 사이즈를 찾기위해 부하 테스트를 진행하고 있었다. 

그러던 중 댓글 작성 API에서 커낵션 풀 사이즈가 5일 때 예외가 발생하는 걸 발견했다.

![3-1](/images/2024-03-18-3/3-1.png)

- 테스트에서 사용한 값
  - Number of Thread(users): 112 (현재 사용자 수 * 2)
  - Ramp-up period (seconds): 10(한 화면에 10초 정도 머문다고 가정)

위 사진에서 에러를 보면 88.39% 거의 모든 요청이 실패한 걸 알 수 있다.

------

## 문제 파악

서비스에서 게시글에 댓글이 달렸을 경우, 게시글 작성자에게 댓글이 달렸다는 알림이 간다.

![3-2](/images/2024-03-18-3/3-2.png)

위의 로그를 보면 5개의 커낵션이 특정 요청들에 의해 점유되고 있는걸 볼 수 있다.

서버에서 발생한 예외를 확인해보면 다음과 같다.

```java
java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30000ms.
	at com.zaxxer.hikari.pool.HikariPool.createTimeoutException(HikariPool.java:696)
	at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:181)
	at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:146)
	at com.zaxxer.hikari.HikariDataSource.getConnection(HikariDataSource.java:128)
```

즉, 요청을 완료하지 못하고 대기하고 있다가  30초 이후 타임아웃으로 종료된 것이다.

문제가 발생한 코드를 메소드 호출 순서를 간단하게 보면 다음과 같다.

```java
@Transactional
public long createComment(final MemberId memberId, final Long postId, final CommentRequest commentRequest) {
    //..댓글 생성 코드 생략
    // 여기서 댓글을 생성한 후 댓글 생성 이벤트를 발행한다.
    publisher.publishEvent(new SavedCommentEvent(comment));
    return comment.getId();
}

// 댓글 생성 이벤트 처리
@TransactionalEventListener
public void sendCommentSavedNotification(SavedCommentEvent event) {
    try {
        notificationService.sendCommentNotificationToPostWriter(event.comment());
    } catch (BusinessLogicException e) {
        log.error(e.getMessage(), e);
    }
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void sendCommentNotificationToPostWriter(Comment comment) {
    // .. 자세한 코드 생략
    sendNotification(comment.getPostWriter(), ScreenType.POST, comment.findPostId(), COMMENT_NOTIFICATION_TITLE);
}
```

위의 코드를 보면 처음 댓글 작성에서 트랜잭션을 가지고 이후, 알림 발송하는 과정에서 다른 트랜잭션을 생성하면서 
두 개의 트랜잭션이 독립적으로 수행될 것처럼 보인다.

즉, 댓글 작성에서 댓글 생성 이벤트 발행후 **트랜잭션이 종료되고 커낵션을 반환**할것이라는 생각은 **틀렸다.**

그 이유는 자바에서 기본적으로 특정 스레드에서 예외가 발생하면 해당 스레드에서 콜 스택을 하나씩 타고 올라가서 예외가 전파되기 때문이다.

즉, 댓글 작성 메서드는 알림 발송 메서드가 종료될 때까지 기다려야한다.

------

## 해결 방안

1. 커낵션 풀 사이즈 조정
2. 스레드 분리

1번의 경우 HikariCP wiki에 나와있는 데드락을 피하는 공식을 사용하면된다.

*`pool size = Tn x (Cm - 1) + 1` (Tn: 최대 스레드 수*, *Cm: 단일 스레드에서 사용하는 최대 커넥션 수*)

공식을 설명하면 다음과 같다.

![3-3](/images/2024-03-18-3/3-3.png)

위와 같이 전체 스레드가 8개이고 단일 스레드에서 사용하는 커낵션이 2개라면

*`pool size = 8 x (2 - 1) + 1` = 9*개를 설정하면 데드락이 발생하지 않는다는 것이다.

그 이유는 동시에 8개의 요청이 와도 하나의 커낵션이 각 요청 스레드에 할당되고 작업을 완료하기 위해 하나의 커낵션을 기다리고 있을 텐데, 커낵션 풀에 하나의 커낵션이 존재하기 때문에 하나의 요청은 커낵션을 받아서 작업을 종료할 수 있다. 그 후 종료된 스레드가 커낵션 두개를 반납하면 남은 요청 스레들도 요청을 완료할 수 있기 때문이다.

이는 최대 스레드 수(톰캣의 thread.max 기본은 200)에 따라 커낵션 풀 사이즈를 조정해야한다는 말과 같고 우리가 원하는 바가 아니었다.

우리가 원하는 바는 댓글 작성 메소드는 댓글 생성 이벤트를 발행하고 트랜잭션을 마치고 커낵션을 반환하는 것이다. 이를 위해서는 비동기로 스레드를 분리하여야한다.

------

## 적용 및 확인

비동기를 적용하기 위해서는 다음과 같이 구성 클래스에 `@EnableAsync` 어노테이션을 붙여주면 된다.

```java
@EnableAsync
@SpringBootApplication
public class BackendApplication {

    public static void main(String[] args) {
        SpringApplication.run(BackendApplication.class, args);
    }
}
```

이후 아까 사용했던 코드에

```java
@Transactional
public long createComment(final MemberId memberId, final Long postId, final CommentRequest commentRequest) {
    //..댓글 생성 코드 생략
    // 여기서 댓글을 생성한 후 댓글 생성 이벤트를 발행한다.
    publisher.publishEvent(new SavedCommentEvent(comment));
    return comment.getId();
}

// 댓글 생성 이벤트 처리
@TransactionalEventListener
public void sendCommentSavedNotification(SavedCommentEvent event) {
    try {
        notificationService.sendCommentNotificationToPostWriter(event.comment());
    } catch (BusinessLogicException e) {
        log.error(e.getMessage(), e);
    }
}

@Async
@Transactional
public void sendCommentNotificationToPostWriter(Comment comment) {
    // .. 자세한 코드 생략
    sendNotification(comment.getPostWriter(), ScreenType.POST, comment.findPostId(), COMMENT_NOTIFICATION_TITLE);
}
```

알림 발송 메소드에서 `@Async` 어노테이션을 적용해주면 된다. 
(비동기로 처리하면서 기존 트랜잭션이 없기 때문에 `REQUIRES_NEW` 옵션은 빼주었다.)

이제 `@Async`로 비동기 적용 후 다시 테스트를 해보면 아래와 같이 예외 없이 정상 작동하는 걸 알 수 있다

![3-4](/images/2024-03-18-3/3-4.png)

**추가로 커낵션 풀 사이즈를 5로 정한 이유**

스레드를 분리하여 Hikari CP에서 제공하는 데드락을 피하는 공식을 적용하면 다음과 같다.

 **`pool size = 200(기본 값) x (1 - 1) + 1` =** 1, 

즉 이제 하나의 커낵션만 있어도 해당 메서드에는 데드락이 발생하지 않는다. 
하지만, 데드락을 피하는 공식을 사용해서 나온 풀 사이즈는 효율적이지는 않다.

Hikari CP wiki를 보면 다음과 같은 효율적인 성능 개선을 위한 공식을 제공한다.

```
connections = ((core_count * 2) + effective_spindle_count
```

위의 공식을 기준으로 서버의 사양(EC2 t4g.small)을 대입해보면 다음과 같이 나온다.
 *`connections = 2 * 2 + 0(서버에서 사용하는 스토리지는 EBS의 gp3로 SSD 기반이라 스핀들이 없다)`*

4개를 기준으로 하나씩 더하고 빼보면서 평균적으로 편차가 적은 5개를 사용하였다.

---

## 결론

자바 코드로 테스트를 작성하는 것도 중요하지만, API 설계 후 부하테스트도 진행해야한다.

------

### 참고 자료

[HikariCP wiki](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing#connections--core_count--2--effective_spindle_count)

[HikariCP Dead lock에서 벗어나기 (실전편)](https://techblog.woowahan.com/2663/)
