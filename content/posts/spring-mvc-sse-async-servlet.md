---
title: "Spring MVC에서 SSE는 어떻게 동작할까?"
date: 2026-01-21
draft: false
tags: ["Spring", "SSE", "Async Servlet", "SseEmitter", "비동기"]
categories: ["Spring"]
summary: "Tomcat 스레드 1개로 여러 SSE 연결을 처리할 수 있는 이유를 알아본다."
---

Tomcat 스레드를 1개로 제한하면 동시 요청 처리가 불가능할까? 실험 결과는 예상과 달랐다.

---

## 실험: 스레드 1개로 두 요청 처리하기

### 환경 설정

```yaml
server:
  tomcat:
    threads:
      max: 1
      min-spare: 1
```

Tomcat 스레드를 1개로 제한했다.

### 테스트 시나리오

1. **SSE 스트리밍 요청**: AI 챗봇 응답을 스트리밍으로 받는다 (약 10초 소요)
2. **Sleep 요청**: 5초간 `Thread.sleep()` 후 응답한다

```kotlin
@GetMapping("/sleep")
fun sleepTest(): Map<String, Any> {
    val startTime = LocalDateTime.now()
    Thread.sleep(5000)
    val endTime = LocalDateTime.now()

    return mapOf(
        "startTime" to startTime,
        "endTime" to endTime,
        "thread" to Thread.currentThread().name,
    )
}
```

두 요청을 거의 동시에 보냈다.

### 예상 결과

스레드가 1개이므로 두 번째 요청은 첫 번째가 끝날 때까지 대기해야 한다.

```
[SSE 요청]   |████████████████████| 10초
[Sleep 요청]                       |█████| 5초 (SSE 완료 후 시작)
총 소요 시간: 약 15초
```

### 실제 결과

```
[SSE 요청]   |████████████████████| 10초
[Sleep 요청] |█████|                5초 (동시 처리)
총 소요 시간: 약 10초
```

**스레드가 1개인데 두 요청이 동시에 처리되었다.**

---

## 원인: Async Servlet

### Thread-per-request 모델의 한계

전통적인 Servlet은 요청당 스레드 하나를 점유한다.

```
[요청] → [Tomcat 스레드 점유] → [처리] → [응답] → [스레드 반환]
```

채팅, 알림, 스트리밍 같은 장시간 연결이 필요한 기능들이 등장하면서, 스레드 고갈 문제가 발생했다. 1,000명이 실시간 알림을 구독하면 1,000개의 스레드가 필요했다.

**연결은 유지하되 스레드는 반환해야 한다.** 이 요구가 Servlet 3.0에서 Async Servlet이 도입된 이유다.

### Async Servlet의 동작 방식

```
[요청] → [Tomcat 스레드] → [AsyncContext 시작] → [스레드 즉시 반환]
                                    ↓
                          [별도 스레드에서 처리]
                                    ↓
                          [AsyncContext로 응답]
```

핵심은 Tomcat 스레드를 즉시 반환한다는 것이다.

---

## SseEmitter의 내부 동작

Spring에서 SSE를 구현할 때 `SseEmitter`를 사용한다. `SseEmitter`는 내부적으로 Async Servlet을 사용한다.

```kotlin
@GetMapping("/stream", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
fun stream(): SseEmitter {
    val emitter = SseEmitter(60_000L)

    executor.execute {
        repeat(10) { i ->
            emitter.send("chunk $i")
            Thread.sleep(1000)
        }
        emitter.complete()
    }

    return emitter  // 즉시 반환, Tomcat 스레드 해제
}
```

`SseEmitter`를 반환하는 순간:

1. Tomcat 스레드는 즉시 해제된다
2. 실제 데이터 전송은 별도 스레드(Executor)에서 수행된다
3. 클라이언트와의 연결은 유지된다

실험에서 SSE 요청이 10초 동안 진행되는 동안에도 Sleep 요청을 처리할 수 있었던 이유다.

---

## 스레드 모델 비교

| 구분 | 동기 방식 | Async Servlet |
|------|----------|---------------|
| 스레드 점유 | 요청 완료까지 | 즉시 반환 |
| 동시 연결 수 | 스레드 수에 제한 | 메모리에 제한 |
| 적합한 용도 | 짧은 요청/응답 | 장시간 연결, 스트리밍 |

---

## 주의사항

### 모든 요청이 Async는 아니다

일반적인 REST API는 여전히 동기 방식이다. 다음 경우에만 Async Servlet이 사용된다.

- `SseEmitter` 반환
- `DeferredResult<T>` 반환
- `Callable<T>` 반환
- `ResponseBodyEmitter` 반환

### 별도 스레드 풀이 필요하다

Tomcat 스레드는 해제되지만, 실제 작업을 수행할 스레드 풀은 별도로 관리해야 한다.

```kotlin
@Bean
fun sseExecutor(): ThreadPoolTaskExecutor =
    ThreadPoolTaskExecutor().apply {
        corePoolSize = 10
        maxPoolSize = 50
        setThreadNamePrefix("sse-")
    }
```

---

## 정리

**"스레드 1개로 제한했는데 왜 블로킹이 안 되지?"**

이 질문의 답은 Async Servlet이다.

`SseEmitter`를 반환하는 순간 Tomcat 스레드는 반환된다. 스레드가 1개여도 SSE 연결을 유지하면서 다른 요청을 처리할 수 있다.

- Tomcat 스레드 수 ≠ 동시 처리 가능한 요청 수 (Async의 경우)
- 스트리밍/장시간 연결에는 Async Servlet이 필수다
- 성능 튜닝 시 동기/비동기 처리 방식을 구분해야 한다
