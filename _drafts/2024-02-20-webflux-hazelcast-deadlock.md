---
title: WebFlux에서 Hazelcast 사용하다 만난 데드락 해결기 - .block() 제거와 비동기 처리
date: 2024-02-20 10:00:00 +0900
categories: [Backend, WebFlux]
tags: [webflux, hazelcast, deadlock, reactive, spring-boot]
---

## 들어가며

Gateway 서비스를 WebFlux로 구현하면서 Hazelcast 캐시와 연동하는 과정에서 간헐적으로 서비스가 멈추는 현상이 발생했습니다. 로그를 보니 특정 상황에서 데드락이 발생하고 있었는데, 원인을 찾아보니 WebFlux의 이벤트 루프에서 블로킹 연산을 실행하는 코드가 있었습니다. 이 문제를 해결하는 과정을 정리해보겠습니다.

## 문제 상황

### 실제 증상
- 특정 시점에서 Gateway 서비스가 완전히 멈춤
- **가장 당황스러웠던 것**: 콘솔에 아무 로그도 찍히지 않음
- CPU 사용률은 낮은데 새로운 요청이 전혀 처리되지 않음
- 서비스 재시작 전까지 계속 멈춰있음

### 발생 환경
```
Gateway Service (WebFlux) → Hazelcast (권한 캐시) → JPA (권한 조회)
```

## WebFlux와 블로킹 연산의 문제점

### 이벤트 루프 모델 이해

WebFlux는 이벤트 루프 기반으로 동작합니다:

```java
/**
 * WebFlux 이벤트 루프 동작 방식
 */
// 이벤트 루프 스레드 (보통 CPU 코어 수만큼)
EventLoop Thread 1: [Request A] → [Request B] → [Request C] → ...
EventLoop Thread 2: [Request D] → [Request E] → [Request F] → ...
```

**핵심 원칙**: 이벤트 루프 스레드에서는 절대 블로킹 연산을 하면 안 됩니다.

### 블로킹 vs 논블로킹 연산

```java
// ❌ 블로킹 연산 (데드락 위험)
@GetMapping("/user/{id}")
public Mono<User> getUser(@PathVariable String id) {
    return Mono.fromCallable(() -> userRepository.findById(id))  // JPA는 블로킹!
        // .subscribeOn(Schedulers.boundedElastic()) // 이게 없으면 문제!
        .map(user -> user.orElseThrow());
}

// ✅ 논블로킹 연산 (안전)
@GetMapping("/user/{id}")
public Mono<User> getUser(@PathVariable String id) {
    return Mono.fromCallable(() -> userRepository.findById(id))
        .subscribeOn(Schedulers.boundedElastic())  // 별도 스레드에서 실행
        .map(user -> user.orElseThrow());
}
```

## 실제 데드락 발생 코드 분석

### 문제가 된 코드

Gateway 서비스에서 권한 확인 로직:

```java
/**
 * 권한 확인 필터 - 데드락 위험 코드
 */
@Component
public class PermissionGatewayDataFetcher implements DataFetcher<String, CacheableApiPermission> {
    
    @Override
    public CacheableApiPermission fetchData(String productId, String apiFullUrl) {
        // ❌ 문제: JPA 블로킹 연산을 이벤트 루프에서 실행
        return Mono.fromCallable(() -> 
            permissionRepository.getPermissionCache(Long.parseLong(productId), apiFullUrl))
            // .subscribeOn(Schedulers.boundedElastic()) // 주석 처리됨!
            .block(); // 치명적 실수!
    }
}
```

### 왜 시스템이 완전히 멈췄나?

1. **WebFlux 이벤트 루프 스레드**에서 권한 확인 요청
2. **Hazelcast 캐시 미스** 발생
3. **JPA 블로킹 연산**을 이벤트 루프에서 실행
4. **이벤트 루프 스레드들이 모두 블로킹**
5. **Hazelcast 클러스터 통신도 같은 스레드 풀 사용** → 클러스터 하트비트 중단
6. **새로운 요청 처리 불가능, 로그 출력도 불가능**

```
Event Loop Thread Pool (4개 스레드)
Thread 1: [BLOCKED] - JPA 호출 대기
Thread 2: [BLOCKED] - JPA 호출 대기  
Thread 3: [BLOCKED] - JPA 호출 대기
Thread 4: [BLOCKED] - JPA 호출 대기
→ Hazelcast 클러스터 통신도 중단!
→ 로그 출력용 스레드도 없음!
```

### 왜 로그가 안 찍혔나?

WebFlux에서 로그 출력도 보통 이벤트 루프 스레드에서 처리되는데, 모든 스레드가 블로킹되면 로그 출력조차 불가능합니다. 그래서 시스템이 멈춘 것처럼 보였던 것입니다.

## 해결 과정

### 1. 블로킹 연산 식별

문제가 되는 코드들을 찾아냈습니다:

```java
// 1. JPA 호출 (블로킹)
return Mono.fromCallable(() -> permissionRepository.getPermissionCache(...))
    // .subscribeOn(Schedulers.boundedElastic()) // 주석 처리됨!

// 2. Hazelcast 동기 호출 (블로킹)
IMap<String, String> map = hazelcastInstance.getMap("cache");
String value = map.get(key); // 블로킹 연산

// 3. Redis 동기 호출 (블로킹)
redisTemplate.opsForValue().get(key); // 블로킹 연산
```

### 2. 비동기 버전으로 변경

#### A. JPA 호출 수정

```java
/**
 * JPA 블로킹 연산 → 비동기 처리
 */
@Override
public Mono<CacheableApiPermission> fetchData(String productId, String apiFullUrl) {
    return Mono.fromCallable(() -> 
        permissionRepository.getPermissionCache(Long.parseLong(productId), apiFullUrl))
        .subscribeOn(Schedulers.boundedElastic()) // 별도 스레드에서 실행
        .map(this::convertToCache);
}
```

#### B. Hazelcast 비동기 호출 구현

```java
/**
 * Hazelcast 동기 호출 → 비동기 호출
 */
public class ReactiveHazelcastRepository<K, V> {
    
    @Override
    public Mono<V> get(K key) {
        // ❌ 기존 동기 방식
        // return Mono.fromCallable(() -> hazelcastMap.get(key));
        
        // ✅ 비동기 방식
        return Mono.fromCompletionStage(hazelcastMap.getAsync(key));
    }
    
    @Override
    public Mono<Boolean> set(K key, V value) {
        return Mono.fromCompletionStage(hazelcastMap.setAsync(key, value))
            .map(prevValue -> true);
    }
}
```

### 3. 완전한 비동기 처리 체인 구성

```java
/**
 * 권한 확인 필터 - 완전 비동기 버전
 */
@Component
public class ApiPermissionFilter implements GatewayFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String productId = extractProductId(exchange);
        String apiUrl = extractApiUrl(exchange);
        
        return checkPermission(productId, apiUrl)
            .flatMap(hasPermission -> {
                if (hasPermission) {
                    return chain.filter(exchange);
                } else {
                    return unauthorized(exchange);
                }
            });
    }
    
    private Mono<Boolean> checkPermission(String productId, String apiUrl) {
        return cacheProxy.get(productId, apiUrl)  // 캐시 조회 (비동기)
            .switchIfEmpty(
                dataFetcher.fetchData(productId, apiUrl)  // DB 조회 (비동기)
                    .flatMap(permission -> 
                        cacheProxy.set(productId, apiUrl, permission)  // 캐시 저장 (비동기)
                            .thenReturn(permission)
                    )
            )
            .map(permission -> permission.isAllowed());
    }
}
```

## Hazelcast Pub/Sub 비동기 구현

### 문제가 된 동기 Pub/Sub

```java
/**
 * ❌ 동기 방식의 Pub/Sub (블로킹 위험)
 */
public class SyncHazelcastMessageService {
    
    public void publishMessage(String topic, String message) {
        ITopic<String> hazelcastTopic = hazelcastInstance.getTopic(topic);
        hazelcastTopic.publish(message); // 동기 호출 - 블로킹 가능
    }
}
```

### 비동기 Pub/Sub 구현

```java
/**
 * ✅ 비동기 방식의 Pub/Sub
 */
@Component
public class ReactiveHazelcastMessageService {
    
    private final HazelcastInstance hazelcastInstance;
    
    public Mono<Void> publishMessage(String topicName, String message) {
        return Mono.fromCallable(() -> {
            ITopic<String> topic = hazelcastInstance.getTopic(topicName);
            topic.publish(message);
            return null;
        })
        .subscribeOn(Schedulers.boundedElastic()) // 별도 스레드에서 실행
        .then();
    }
    
    public Flux<String> subscribeToTopic(String topicName) {
        return Flux.create(sink -> {
            ITopic<String> topic = hazelcastInstance.getTopic(topicName);
            
            String listenerId = topic.addMessageListener(message -> {
                sink.next(message.getMessageObject());
            });
            
            // 구독 해제 처리
            sink.onDispose(() -> topic.removeMessageListener(listenerId));
        })
        .subscribeOn(Schedulers.boundedElastic()); // 별도 스레드에서 실행
    }
}
```

### 실제 사용 예시

```java
/**
 * 캐시 무효화 메시지 발행
 */
@Service
public class CacheInvalidationService {
    
    private final ReactiveHazelcastMessageService messageService;
    
    public Mono<Void> invalidateCache(String cacheKey) {
        CacheInvalidationMessage message = new CacheInvalidationMessage(cacheKey, Instant.now());
        
        return messageService.publishMessage("cache-invalidation", 
            JsonUtils.toJson(message))
            .doOnSuccess(v -> log.info("캐시 무효화 메시지 발행: {}", cacheKey))
            .doOnError(e -> log.error("캐시 무효화 메시지 발행 실패: {}", cacheKey, e));
    }
}
```

## 문제 재현 방법

실제로 이 문제를 재현해보려면:

### 1. 데드락 재현 코드

```java
/**
 * 데드락을 재현하는 코드
 */
@RestController
public class DeadlockController {
    
    @GetMapping("/deadlock-test")
    public Mono<String> deadlockTest() {
        return Mono.fromCallable(() -> {
            // 이벤트 루프에서 블로킹 연산 실행
            Thread.sleep(5000); // 5초 대기 (DB 조회 시뮬레이션)
            return "response";
        })
        // .subscribeOn(Schedulers.boundedElastic()) // 이 줄을 주석처리하면 데드락!
        .map(result -> "Result: " + result);
    }
}
```

### 2. 테스트 방법

```bash
# 동시에 여러 요청 보내기 (이벤트 루프 스레드 수보다 많이)
for i in {1..10}; do
  curl "http://localhost:8080/deadlock-test" &
done
```

### 3. 예상 결과
- 처음 몇 개 요청이 들어오면 시스템이 완전히 멈춤
- 새로운 요청 응답 없음
- 콘솔에 아무 로그도 안 찍힘

## 해결 후 결과

### 실제 개선 결과
- **가장 중요한 것**: 시스템이 더 이상 멈추지 않음
- 응답 시간 안정화됨
- 로그가 정상적으로 출력됨

## 디버깅 팁

### 데드락 발생 시 확인 방법

1. **스레드 덤프 확인**
```bash
# 프로세스 ID 확인
jps

# 스레드 덤프 생성
jstack <pid> > thread_dump.txt

# reactor-http 스레드들이 모두 BLOCKED 상태인지 확인
grep -A 10 "reactor-http" thread_dump.txt
```

2. **이벤트 루프 스레드 확인**
- 모든 `reactor-http-nio` 스레드가 BLOCKED 상태
- `java.lang.Thread.sleep` 또는 JPA 호출에서 대기 중

### 예방 방법

1. **코드 리뷰 시 체크포인트**
- `Mono.fromCallable()` 사용 시 반드시 `subscribeOn()` 확인
- JPA, JDBC 등 블로킹 라이브러리 사용 시 주의
- `Thread.sleep()` 같은 블로킹 호출 금지

2. **개발 환경 설정**
```yaml
# application.yml
reactor:
  netty:
    # 이벤트 루프 스레드 수 명시적 설정
    ioWorkerCount: 4
```

## 배운 점들

### 1. WebFlux 사용 시 주의사항

- **절대 금지**: 이벤트 루프에서 블로킹 연산
- **필수**: 블로킹 연산 시 `subscribeOn(Schedulers.boundedElastic())` 사용
- **중요**: 시스템이 멈추면 로그도 안 찍히니 미리 예방해야 함

### 2. Hazelcast와 WebFlux 조합 시 주의점

- Hazelcast 클러스터 통신도 이벤트 루프 스레드 사용
- 이벤트 루프가 막히면 클러스터 하트비트도 중단됨
- 비동기 API 사용 필수

### 3. 실제 겪어본 교훈

- **가장 무서운 것**: 아무 로그도 안 찍히는 상황
- **디버깅 팁**: 스레드 덤프가 최고의 친구
- **예방이 최선**: 코드 리뷰에서 블로킹 연산 체크 필수

## 마무리

이번 문제를 겪으면서 WebFlux의 동작 원리를 뼈저리게 느꼈습니다. 

특히 시스템이 완전히 멈춰서 로그도 안 찍히는 상황은 정말 당황스러웠는데, 이제는 그 이유를 명확히 알게 되었습니다. 

앞으로는 블로킹 연산을 사용할 때 항상 `subscribeOn()`을 함께 사용하는 습관을 갖게 될 것 같습니다.

---

*실제 겪은 데드락 문제 해결 경험을 바탕으로 작성했습니다.*