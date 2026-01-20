---
title: "Spring에서 별도 스레드로 작업 실행하기"
date: 2026-01-19
draft: false
tags: ["Spring", "비동기", "스레드", "Async", "ThreadPoolTaskExecutor"]
categories: ["Spring"]
summary: "Spring 비동기 처리의 종류와 선택 기준을 정리한다."
---

Tomcat 스레드 풀은 기본 200개다. Controller에서 오래 걸리는 작업을 직접 수행하면 스레드가 묶이고, 모든 스레드가 점유되면 새 요청을 받을 수 없다.

별도 스레드에서 작업을 수행하면 요청 스레드를 빠르게 반납할 수 있다. 이 글에서는 Spring에서 별도 스레드를 사용하는 방법과 `ThreadPoolTaskExecutor` 설정을 정리한다.

---

## 1. Spring MVC의 기본 스레드 모델

Spring MVC는 Servlet 기반 동기 처리 모델을 사용한다.

- 요청 1개당 Tomcat 워커 스레드 1개가 할당된다
- Controller 메서드가 종료될 때까지 스레드는 점유된다
- 응답 완료 후 스레드는 풀로 반환된다

Controller에서 오래 걸리는 작업을 수행하면 스레드가 묶인다.

**스레드 점유 시간이 곧 서버 처리량의 한계다.** Tomcat 스레드 풀은 기본 200개로 제한된다. 모든 스레드가 점유되면 새 요청을 받을 수 없다.

---

## 2. 별도 스레드가 필요한 상황

- 이메일 발송, 알림 전송
- 외부 API 호출 (지연 시간이 긴 경우)
- 대용량 파일 처리
- 로그 적재, 통계 집계
- SSE나 스트리밍 응답

공통점은 **HTTP 응답을 지연시킬 필요가 없는 작업**이라는 점이다.

---

## 3. @Async로 비동기 처리하기

Spring이 제공하는 가장 간단한 방식이다.

```kotlin
@Service
class NotificationService {

    @Async
    fun sendEmail() {
        // 별도 스레드에서 실행
    }
}
```

### 특징

- `@EnableAsync` 설정이 필요하다
- 내부적으로 `TaskExecutor`를 사용한다
- 요청 스레드는 즉시 반환된다
- 반환 타입은 `void` 또는 `CompletableFuture`다

### 주의점

- 같은 클래스 내부 호출에서는 동작하지 않는다 (프록시 기반)
- 트랜잭션 컨텍스트는 전파되지 않는다
- 예외는 호출자에게 전달되지 않는다

**왜 내부 호출에서 동작하지 않는가?** `@Async`는 Spring AOP 프록시로 동작한다. `this.method()` 호출은 프록시를 거치지 않고 실제 객체를 직접 호출한다.

**왜 트랜잭션이 전파되지 않는가?** 트랜잭션은 ThreadLocal 기반이다. 별도 스레드는 ThreadLocal을 공유하지 않는다.

단순한 백그라운드 작업에 적합하다.

---

## 4. Async Servlet으로 요청 스레드 반납하기

Spring MVC는 Servlet Async API 기반 비동기 처리를 지원한다. `Callable`, `DeferredResult`, `SseEmitter`가 대표적이다.

```kotlin
@GetMapping("/async")
fun async(): Callable<String> {
    return Callable {
        "ok"
    }
}
```

### 동작 흐름

1. 요청 수신 후 초기 처리만 Tomcat 스레드에서 수행
2. Async 컨텍스트 시작, 요청 스레드 즉시 반환
3. 별도 스레드에서 실제 작업 수행
4. 응답 시점에 DispatcherServlet이 다시 관여

### 장점

- 요청 스레드를 빠르게 반납한다
- HTTP 요청/응답 흐름과 자연스럽게 연결된다
- 기존 Spring MVC 구조를 유지한다

SSE, 장시간 처리, 외부 API 연동에 적합하다.

---

## 5. Thread 직접 생성을 피해야 하는 이유

```kotlin
Thread {
    doSomething()
}.start()
```

위 방식은 권장되지 않는다.

- 스레드 풀 관리가 되지 않는다
- 애플리케이션 종료 시 정리되지 않는다
- 모니터링, MDC, 트랜잭션과 연동되지 않는다

`ThreadPoolTaskExecutor`처럼 컨테이너가 관리하는 스레드 풀을 사용해야 한다.

---

## 6. ThreadPoolExecutor vs ThreadPoolTaskExecutor

`ThreadPoolExecutor`를 직접 만드는 것보다 `ThreadPoolTaskExecutor`가 실무적으로 유리하다. 후자는 전자를 감싸서 Spring 생태계와 통합한다.

### ThreadPoolExecutor (java.util.concurrent)

- Java 표준 스레드 풀 구현체
- 직접 생성/종료를 관리해야 한다
- MDC 전파, 종료 훅 등을 수동 구성해야 한다

### ThreadPoolTaskExecutor (Spring)

- `ThreadPoolExecutor`를 Spring이 관리 가능한 형태로 래핑한다
- Bean 라이프사이클에 자연스럽게 붙는다
- `TaskDecorator`, graceful shutdown을 공식 지원한다

**Spring 애플리케이션에서는 ThreadPoolTaskExecutor를 Bean으로 등록하는 방식이 표준이다.**

---

## 7. ThreadPoolTaskExecutor 설정 예시

```kotlin
@Bean
fun chatStreamExecutor(): ThreadPoolTaskExecutor =
    ThreadPoolTaskExecutor().apply {
        corePoolSize = 20
        maxPoolSize = 100
        queueCapacity = 200
        keepAliveSeconds = 60
        setThreadNamePrefix("chat-stream-")
        setWaitForTasksToCompleteOnShutdown(true)
        setAwaitTerminationSeconds(60)
        setTaskDecorator(MdcCopyTaskDecorator())
        setRejectedExecutionHandler(ThreadPoolExecutor.AbortPolicy())
    }
```

### 처리 순서

1. 스레드 수가 `corePoolSize(20)` 미만이면 새 스레드로 즉시 처리
2. core를 다 썼다면 `queueCapacity(200)`까지 큐에 적재
3. 큐도 가득 차면 `maxPoolSize(100)`까지 스레드 추가 생성
4. 스레드도 max, 큐도 가득 차면 `RejectedExecutionHandler` 실행

**왜 core → queue → max 순서인가?** 스레드 생성 비용이 큐 대기보다 비싸기 때문이다.

| 항목 | 스레드 생성 | 큐 대기 |
|------|-----------|---------|
| 메모리 | 스택 메모리 할당 (기본 512KB~1MB) | 작업 객체만 저장 (수 KB) |
| CPU | OS 커널 호출, 스케줄러 등록 | 단순 큐 삽입 |

기존 스레드를 최대한 재사용하는 것이 효율적이다. 큐도 가득 차면 그제서야 추가 스레드를 생성한다.

### 최대 수용량

| 항목 | 값 |
|------|-----|
| 동시 실행 | 최대 100개 |
| 대기 큐 | 최대 200개 |
| 총 수용 | 최대 300개 |
| 초과 시 | 예외 발생 |

SSE/스트리밍처럼 작업이 오래 붙잡히는 워크로드에서 중요한 설정이다.

### TaskDecorator: 컨텍스트 전파

요청 스레드와 워커 스레드는 다르다. 로그 상관관계(traceId 등)를 유지하려면 컨텍스트를 복사해야 한다.

`TaskDecorator`는 작업 제출 시점의 컨텍스트를 캡처해서 실행 스레드에 주입한다.

- 장점: 비동기에서도 로그 추적이 끊기지 않는다
- 주의: 누수 방지를 위해 실행 후 `clear()`가 필요하다

### Graceful Shutdown

`setWaitForTasksToCompleteOnShutdown(true)`와 `setAwaitTerminationSeconds(60)` 설정이다.

- 종료 시 신규 작업 수락 중단
- 실행 중인 작업 완료를 최대 60초 대기
- 초과 시 강제 종료

스트리밍 작업은 연결이 길어질 수 있다. 종료 대기 시간은 서비스 특성에 맞춰 조정한다.

### RejectedExecutionHandler: 포화 시 동작

| 정책 | 동작 |
|------|------|
| AbortPolicy | 즉시 예외 (명확한 실패) |
| CallerRunsPolicy | 호출 스레드에서 실행 (백프레셔 효과) |
| DiscardPolicy | 조용히 버림 (가시성 낮음) |
| DiscardOldestPolicy | 오래된 작업 버리고 새 작업 수용 |

스트리밍/채팅처럼 사용자 경험이 중요한 워크로드에서는 거부 정책을 명시적으로 설계해야 한다.

---

## 8. 정리

Spring에서 별도 스레드를 사용하는 방법은 `@Async`, `ThreadPoolTaskExecutor` 등이 있다. 목적에 맞게 선택하면 된다.

Thread를 직접 생성하면 안 된다. 스레드 풀 관리가 되지 않고, 애플리케이션 종료 시 정리되지 않으며, MDC/트랜잭션과 연동되지 않는다. `ThreadPoolTaskExecutor`를 Bean으로 등록해서 사용해야 한다.
