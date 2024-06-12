---
layout: single
title: "외부 API 장애 대응"
categories: project
tag: [주차의상상은현실이된다, 외부 API 장애, 서킷브레이커]
typora-root-url: ../
toc: true
author_profile: false
---

# 상황

“주차의 상상은 현실이된다” 프로젝트에서 외부 API를 통해 주기적으로 주차장 데이터를 삽입/갱신 한다.  
이 과정에서 외부 API에서 장애가 발생했을 때, 서버에 영향이 덜 받도록 프로젝트에 적용한 방법을 기록하려고 한다.

## 1. 타임 아웃 처리

외부 API에서 문제가 발생해서 오래 동안 응답이 없는 경우, 스레드는 계속해서 대기하게 된다.  
(프로젝트에서 사용한 RestTemplate의 경우 설정하지 않을 경우 타임아웃에 제한이 없다.)
그렇기 때문에 일정 시간 응답이 없다면 빠르게 타임 아웃으로 예외를 던지는 게 낫다고 판단했다.

타임 아웃은 연결 타임 아웃과 읽기 타임 아웃이 있다.   

- 연결 타임 아웃은 클라이언트가 서버에 연결을 요청하고 연결이 성공할 때까지의 시간을 의미한다.
- 읽기 타임 아웃은 서버와 연결이 성공한 후, 클라이언트가 응답을 기다리는 시간을 의미한다.

```RestTemplate```의 경우 다음과 같이 적용시킬 수 있다. 

```java
@Bean
@Qualifier("parkingApiRestTemplate")
public RestTemplate parkingApiRestTemplate(RestTemplateBuilder restTemplateBuilder) {
    return restTemplateBuilder
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(5))
            .errorHandler(new ParkingApiErrorHandler())
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_UTF8_VALUE)
            .build();
}
```

위와 같이 ```setConnectTimeout()```으로 연결 타임 아웃을 ```setReadTimeout()```으로 읽기 타임 아웃을 설정할 수 있다.



## 2. 헬스 체크

프로젝트에서 요청 마다 많은 주차장 데이터를 가져오는데, 장애가 난 서버에 계속해서 요청을 보내는 것은 해당 서버에 부하만 주기 때문에 본 요청을 보내기 전 서버가 정상 작동하는 지 확인하기 위해 작은 데이터 수를 보내서 상태를 확인하는 게 좋다.

아래는 프로젝트에 적용한 코드이다.

```java
public interface HealthChecker {

    HealthCheckResponse check();
}
```

- 여러 외부 API를 사용하기 때문에 헬스 체크하는 부분은 따로 인터페이스로 생성해주었다.

```java
@Getter
public class HealthCheckResponse {

    boolean isHealthy;
    int totalSize;

    public HealthCheckResponse(boolean isHealthy, int totalSize) {
        this.isHealthy = isHealthy;
        this.totalSize = totalSize;
    }
}
```

- 응답 값의 경우 서버의 상태가 정상인지, API에서 제공하는 전체 데이터 수를 반환하게 했다.

```java
@Override
public HealthCheckResponse check() {
    ResponseEntity<KoreaParkingResponse> response = call(1, 1);
    return new HealthCheckResponse(isHealthy(response), response.getBody().getResponse().getBody().getTotalCount());
}
```

- `call(1, 1)` 의 경우 1 페이지에서 1개 사이즈 만큼 요청하는 메서드이다.

```java
private boolean isHealthy(ResponseEntity<KoreaParkingResponse> response) {
    return response.getStatusCode().is2xxSuccessful() && response.getBody().getResponse().getHeader()
            .getResultCode().equals(NORMAL_RESULT_CODE);
}
```

- 서버가 정상 상태인지 확인 할 때 응답이 200번대 이고, 외부 API의 정상 응답 코드를 기준으로 판단했다.

이후 실제 코드에서 헬스 체크를 성공한 API 들만 실제 호출을 하도록 설정해주었다.



## 3. 재시도 처리

API 요청은 네트워크의 간헐적 문제등으로 실패할 수 있기 때문에, 실패 시에 특정 횟수만큼 재요청 처리를 좋다고 판단했다.  
프로젝트에서는 RestTemplate을 사용하는데 spring-retry를 이용하면 쉽게 재시 도로직을 추가해줄 수 있었다.

의존성은 다음과 같이 추가해주면 된다.

```groovy
implementation 'org.springframework.retry:spring-retry:2.0.6'
```

이후, 아래 코드와 같이 재시도 횟수를 설정해주고 RestTemplateBuilder에 추가해주면 된다.

```java
@Bean
@Qualifier("parkingApiRestTemplate")
public RestTemplate parkingApiRestTemplate(RestTemplateBuilder restTemplateBuilder) {
    return restTemplateBuilder
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(5))
            .errorHandler(new ParkingApiErrorHandler())
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_UTF8_VALUE)
      			.additionalInterceptors(clientHttpRequestInterceptor())
            .build();
}

private ClientHttpRequestInterceptor clientHttpRequestInterceptor() {
    return (request, body, execution) -> {
        RetryTemplate retryTemplate = new RetryTemplate();
        retryTemplate.setRetryPolicy(new SimpleRetryPolicy(3));
        try {
            return retryTemplate.execute(context -> execution.execute(request, body));
        } catch (Throwable throwable) {
            throw new RuntimeException(throwable);
        }
    };
}
```



## 4. 서킷 브레이커 적용

외부 시스템 장애 지속시 계속해서 타임 아웃/500번대 응답을 보낼 것이다. 그럼에도 우리 서비스에서 계속해서 요청을 보내는 것은 우리 서버의 자원 낭비뿐만 아니라, 사용하는 외부 서버에도 부하를 준다고 생각했다.

그래서 오류 지속시 일정 시간동안 기능 실행을 차단하고 지정한 시간이 지나고 다시 정상 작동하도록 서킷 브레이커를 적용했다.

처음 구현해 준 것은 요청 카운터이다. 이를 이용해서 API 요청할 때마다 카운터를 증가시켜주고 만약 예외가 발생하면 예외도 따로 카운터 해준다. 아래는 실제 카운터를 구현한 코드이다.

```java
public class ApiCounter {

    private final int MIN_TOTAL_COUNT;

    private AtomicInteger totalCount;
    private AtomicInteger errorCount;
    private boolean isOpened;

    public ApiCounter() {
        this.MIN_TOTAL_COUNT = 10;
        this.totalCount = new AtomicInteger(0);
        this.errorCount = new AtomicInteger(0);
        this.isOpened = false;
    }

    public ApiCounter(int minTotalCount) {
        this.MIN_TOTAL_COUNT = minTotalCount;
        this.totalCount = new AtomicInteger(0);
        this.errorCount = new AtomicInteger(0);
        this.isOpened = false;
    }

    public void countUp() {
        while (true) {
            int expected = getTotalCount();
            int newValue = expected + 1;
            if (totalCount.compareAndSet(expected, newValue)) {
                return;
            }
        }
    }

    public void errorCountUp() {
        countUp();
        while (true) {
            int expected = getErrorCount();
            int newValue = expected + 1;
            if (errorCount.compareAndSet(expected, newValue)) {
                return;
            }
        }
    }

    public void reset() {
        totalCount = new AtomicInteger(0);
        errorCount = new AtomicInteger(0);
        isOpened = false;
    }

    public boolean isOpened() {
        return isOpened;
    }

    public void open() {
        isOpened = true;
    }

    public boolean isErrorRateOverThan(double errorRate) {
        int currentTotalCount = getTotalCount();
        int currentErrorCount = getErrorCount();
        if (currentTotalCount < MIN_TOTAL_COUNT) {
            return false;
        }
        double currentErrorRate = (double) currentErrorCount / currentTotalCount;
        return currentErrorRate >= errorRate;
    }

    public int getTotalCount() {
        return totalCount.get();
    }
    public int getErrorCount() { return errorCount.get(); }
}
```

카운터는 여러 스레드에서 사용이 가능하므로 `AtomicInteger`를 사용했다.  
처음 구현했을 때는 `synchronized` 키워드를 사용해서 구현했었는데, 이를 사용했을 때는 메소드 레벨로 키워드만 적어주면 되고 하나의 서버에서 동작되기 떄문에 적합하다고 판단했었다.

하지만, 코드 리뷰를 통해 `AtomicInteger`를 고려해보게 되었다.

![image-20240613035203213](/images/2024-05-23-resilience/image-20240613035203213.png)

`AtomicInteger`는 락을 사용하지 않고 원자적 연산을 수행하여 락을 사용하는 `synchronized` 키워드에 비해 빠른 성능을 기대할 수 있다. 또한 `synchronized` 키워드를 사용하면 메서드를 확인하고 스레드 간 경쟁 상태가 일어날거 같으면 직접 키워드를 붙여줘야하기 때문에 실수를 유발할 수 있다.

AtomicInteger가 원자적 연산으로 동시성을 제어하는 방법은 위 구현한 코드를 보면 알 수 있듯이, **Compare-and-swap(CAS)**를 이용해서 동시성을 제어하기 때문이다.  
(값을 변경 할 때, 예상하는 현재 값과 변경할 값을 파라미터로 받아서 예상 값과 현재 값이 동일하면 값을 변경한다)

해당 내용을 테스트할 때는 `CountDownLatch`를 이용해서 값이 잘 카운팅되는지 확인해주었다.

```java
@Test
void 여러_스레드에서_카운트를_증가시킬수있다() throws InterruptedException {
    //given
    ExecutorService executorService = Executors.newFixedThreadPool(30);
    ApiCounter apiCounter = new ApiCounter();
    int threadCount = 1000;
    CountDownLatch latch = new CountDownLatch(threadCount);

    //when
    for (int i = 0; i < threadCount; i++) {
        executorService.submit(() -> {
            try {
                apiCounter.countUp();
            } finally {
                latch.countDown();
            }
        });
    }

    latch.await();

    //then
    assertThat(apiCounter.getTotalCount()).isEqualTo(threadCount);
}
```

이제 이를 이용해서 API에 적용해주어야 하는데, 외부 API를 호출하는 모든 코드에 직접 카운터를 생성하고 카운터를 증가시켜 주는 것은 코드 중복문제가 발생하고 관리가 안될거 같았다.

그래서 스프링 AOP와 어노테이션을 이용해서 중복 문제를 해결했다.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface CircuitBreaker {

    int minTotalCount() default 10;
    double errorRate() default 0.2;
    long resetTime() default 30;
    TimeUnit timeUnit() default TimeUnit.MINUTES;
}
```

- API 마다 최소 카운트, 오류율, 초기화 할 시간을 받을 수 있도록 했다.

그리고 위의 어노테이션을 적용하면 사용할 수 있도록 Aspect를 정의해주었다.

```java
@Slf4j
@Aspect
@Component
public class CircuitBreakerAspect {

    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(10);
    private final Map<Object, ApiCounter> map = new ConcurrentHashMap<>();

    @Around("@annotation(annotation)")
    public Object around(ProceedingJoinPoint proceedingJoinPoint, CircuitBreaker annotation) {
        ApiCounter apiCounter = getApiCounter(proceedingJoinPoint, annotation.minTotalCount());
        if (apiCounter.isOpened()) {
            log.warn("현재 해당 {} API는 오류로 인해 중지되었습니다.", proceedingJoinPoint.getTarget());
            return null;
        }
        try {
            Object result = proceedingJoinPoint.proceed();
            apiCounter.countUp();
            return result;
        } catch (Throwable e) {
            handleError(annotation, apiCounter);
            return null;
        }
    }

    private ApiCounter getApiCounter(ProceedingJoinPoint proceedingJoinPoint, int minTotalCount) {
        Object target = proceedingJoinPoint.getTarget();
        if (!map.containsKey(target)) {
            map.put(target, new ApiCounter(minTotalCount));
        }
        return map.get(target);
    }

    private void handleError(CircuitBreaker annotation, ApiCounter apiCounter) {
        apiCounter.errorCountUp();
        if (apiCounter.isErrorRateOverThan(annotation.errorRate())) {
            apiCounter.open();
            scheduler.schedule(apiCounter::reset, annotation.resetTime(), annotation.timeUnit());
        }
    }
}

```

현재 서비스에는 특정 외부 API마다 호출하는 객체를 싱글톤 빈으로 갖기 때문에 해당 객체를 키로해서 카운터를 생성해주었다.  
실제 메소드를 실행하기 전에, API가 중지되었는지 확인하고 중지되었으면 로그만 띄워주고 null을 반환하게된다.

그러고 실제 메소드를 실행하고 카운터를 증가시켜준다. 만약 예외가 발생하면 예외 카운트를 증가시키고 어노테이션에 정의된 오류율 이상이되면 해당 API를 Open 상태로 변경하고, 주어진 시간이후에 초기화되도록 스케줄링 해주었다.



## 참고

[최범균-외부 API 장애에 영향 덜 받는 3가지 방법](https://www.youtube.com/watch?v=nuRO0ZBFdKk)

