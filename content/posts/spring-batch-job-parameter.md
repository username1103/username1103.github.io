---
title: "Spring Batch JobParameter가 로컬에서 사용했던 값으로 실행된 이유"
date: 2025-11-23
tags: ["Spring Batch", "JobParameter", "RunIdIncrementer", "Jenkins", "Docker"]
categories: ["Spring"]
summary: "RunIdIncrementer 사용 시 의도하지 않은 JobParameter가 추가된 원인과 해결 과정을 정리한다."
---

신규 배치 잡을 개발했다. 로컬에서 정상 동작을 확인한 뒤 개발 서버에 배포했다.

그런데 개발 환경 배치 실행 시 항상 로컬에서 사용했던 JobParameter로만 실행됐다.

Jenkins에서 jobParameter를 넘겨 실행해도, 스프링 배치는 로컬에서 실행한 JobParameter를 고정해서 사용했다.

새로 만든 잡에서는 동일 JobParameter로도 반복 실행할 수 있도록 `RunIdIncrementer`를 사용한다. 이 incrementer는 `run.id`를 자동 증가시켜 새로운 JobInstance를 생성한다.

`run.id`는 정상적으로 1, 2, 3... 증가했다. **그러나 다른 파라미터들은 설정한 파라미터가 아닌, 로컬에서 실행했던 파라미터만 적용되고 있었다.**

원인을 파악해 보기로 했다.

## JobParameter의 구성 과정

스프링 배치는 다음 순서로 JobParameter를 만든다.

```java
public class JobParametersBuilder {
    ...
    
    public JobParametersBuilder getNextJobParameters(Job job) {
        JobInstance lastInstance = jobExplorer.getLastJobInstance(name);
        JobExecution previousExecution = jobExplorer.getLastJobExecution(lastInstance);
        
        nextParameters = incrementer.getNext(previousExecution.getJobParameters());
        ...
        nextParametersMap.putAll(this.parameterMap); // 현재 실행 파라미터 우선
    }
}
```

순서를 정리하면 다음과 같다.

1. 이전 실행의 JobParameter를 조회한다.
2. incrementer를 통해 nextParameters를 결정한다.
3. 이번 실행에서 전달한 jobParameter를 마지막에 merge한다.

---

## RunIdIncrementer

위 코드에서 사용된 incrementer는 RunIdIncrementer이다.

```java
public JobParameters getNext(@Nullable JobParameters parameters) {
    JobParameters params = (parameters == null) ? new JobParameters() : parameters;
    JobParameter<?> runIdParameter = params.getParameters().get(this.key);
    long id = 1;
    if (runIdParameter != null) {
        id = Long.parseLong(runIdParameter.getValue().toString()) + 1;
    }
    return new JobParametersBuilder(params).addLong(this.key, id).toJobParameters();
}
```

동작은 다음과 같다.

- 파라미터로 받은 JobParameter들에서 `run.id`가 있으면 +1 한다.
- 없으면 1로 시작한다.
- **나머지 JobParameter는 건드리지 않는다.**

RunIdIncrementer는 오직 `run.id`만 증가시킨다.

---

### JobParameter 우선순위

| 순위 | 출처 |
|------|------|
| 1 (낮음) | 이전 실행의 JobParameter |
| 2 | Incrementer(getNext) 결과 |
| 3 (높음) | **이번 실행 시 전달된 JobParameter** |

이번 실행에서 넘긴 파라미터가 최우선이다. 그런데 왜 적용되지 않았을까?

---

## Jenkins에서 Docker로 파라미터가 전달되지 않음

스프링 배치 로직상 이번 실행 파라미터가 최우선이 맞다. 그런데 Jenkins에서 넘긴 JobParameter가 반영되지 않았다.

원인은 단순했다.

> Dockerfile 내부에서 배치를 실행할 때, Jenkins가 전달한 jobParameter를 넘기지 않고 있었다.

파라미터 전달 경로는 다음과 같다.

```
Jenkins → Docker run → Spring Boot
```

이 체인에서 Docker가 파라미터를 누락했다. 그 결과 다음과 같은 일이 발생했다.

1. 스프링은 "이번 실행 파라미터 없음"으로 인식한다.
2. 이전 JobParameter를 그대로 사용한다.
3. Incrementer는 `run.id`만 증가시킨다.
4. `run.id`만 달라지고, 나머지는 로컬 실행의 파라미터가 유지된다.

Dockerfile을 수정해 Jenkins에서 받은 파라미터를 그대로 전달하도록 했다. 문제는 바로 해결되었다.

---

## 정리

**RunIdIncrementer는 `run.id`만 증가시킨다.** 다른 파라미터는 건드리지 않는다.

**Spring Batch는 이전 실행의 JobParameter를 기본값으로 사용한다.** Incrementer 적용 후, 이번 실행 파라미터를 merge한다.

이번 실행의 JobParameter가 적용되지 않은 이유는 Dockerfile에서 파라미터를 누락했기 때문이고,
로컬 실행시 적재된 실행 기록의 JobParameter가 사용되어 혼돈을 준 것이다.
