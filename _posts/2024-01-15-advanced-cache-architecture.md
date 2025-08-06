---
title: 멀티레벨 캐시 아키텍처 구현하기 - Hazelcast + Redis 조합
date: 2024-01-15 14:00:00 +0900
categories: [Backend, Architecture]
tags: [redis, hazelcast, cache, multi-level, architecture]
---

## 들어가며

이전 글에서 Jackson2JsonRedisSerializer 선택 과정을 다뤘는데, 이번에는 프로젝트에서 구현한 멀티레벨 캐시 아키텍처를 공유하려고 합니다. L1(Hazelcast) + L2(Redis) 구조로 설계해서 개발은 완료했지만, 실제 운영에는 단일 캐시만 사용하고 있습니다. 그래도 구현 과정에서 배운 것들을 정리해보겠습니다.

## 왜 멀티레벨 캐시인가?

### 단순 Redis 캐시의 한계

처음에는 단순하게 Redis 캐시만 사용했는데, 이론적으로 몇 가지 문제점이 있을 수 있다고 생각했습니다:

- **네트워크 레이턴시**: 매번 Redis 서버로 요청을 보내야 함
- **Redis 부하**: 모든 캐시 요청이 Redis로 집중됨
- **단일 장애점**: Redis 서버에 문제가 생기면 캐시 전체가 멈춤

그래서 L1 캐시(로컬)와 L2 캐시(Redis)를 조합한 멀티레벨 구조를 설계해봤습니다.

```java
/**
 * 멀티레벨 캐시 아키텍처 구조
 */
@Component
public class CacheArchitecture {
    
    // L1 캐시: Hazelcast (로컬 메모리, 빠름)
    @Autowired private HazelcastInstance hazelcastInstance;
    
    // L2 캐시: Redis (네트워크 캐시, 공유됨)
    @Autowired private RedisTemplate<String, Object> redisTemplate;
    
    // DB 폴백: 자동 DB 조회
    @Autowired private DataFetcher<String, ProductDTO> productDataFetcher;
}
```

### 캐시 계층 구조

```
요청 → L1 캐시 (Hazelcast) → L2 캐시 (Redis) → DB → 응답
       ↓ 1ms 내외              ↓ 3-5ms        ↓ 50-100ms
       초고속                  네트워크 캐시    최종 데이터
```

## 멀티레벨 캐시 구현

### 1. MultiRepository 패턴

핵심 아이디어는 L1/L2 캐시를 하나의 인터페이스로 감싸서 투명하게 처리하는 것입니다:

```java
/**
 * 멀티레벨 캐시 핵심 구현
 * L1(Hazelcast) + L2(Redis) 조합
 */
@Slf4j
@RequiredArgsConstructor
public class MultiRepository<K, V> implements CacheRepository<K, V> {
    
    private final CacheRepository<K, V> l1Cache;  // Hazelcast
    private final CacheRepository<K, V> l2Cache;  // Redis
    
    @Override
    public V get(K key) {
        // 1단계: L1 캐시에서 조회 (가장 빠름)
        V value = l1Cache.get(key);
        if (value != null) {
            log.debug("L1 캐시 히트: {}", key);
            return value;
        }
        
        // 2단계: L2 캐시에서 조회
        value = l2Cache.get(key);
        if (value != null) {
            log.debug("L2 캐시 히트: {}", key);
            
            // L2에서 가져온 데이터를 L1에 저장 (자동 워밍)
            l1Cache.set(key, value);
            return value;
        }
        
        log.debug("캐시 미스: {}", key);
        return null;
    }
    
    @Override
    public boolean set(K key, V value) {
        // 쓰기 시 두 캐시 모두 업데이트
        boolean l1Success = false;
        boolean l2Success = false;
        
        try {
            l1Success = l1Cache.set(key, value);
        } catch (Exception e) {
            log.warn("L1 캐시 저장 실패: {}", key, e);
        }
        
        try {
            l2Success = l2Cache.set(key, value);
        } catch (Exception e) {
            log.error("L2 캐시 저장 실패: {}", key, e);
        }
        
        // 하나라도 성공하면 OK
        return l1Success || l2Success;
    }
}
```

### 2. Reactive 버전 (WebFlux 환경)

WebFlux 환경에서는 비동기 처리가 중요해서 Reactive 버전도 만들었습니다:

```java
/**
 * WebFlux용 Reactive 멀티레벨 캐시
 */
@RequiredArgsConstructor
public class ReactiveMultiRepository<K, V> implements ReactiveCacheRepository<K, V> {
    
    private final ReactiveCacheRepository<K, V> l1Cache;
    private final ReactiveCacheRepository<K, V> l2Cache;
    
    @Override
    public Mono<V> get(K key) {
        return l1Cache.get(key)
            .flatMap(value -> {
                if (value != null) {
                    return Mono.just(value); // L1 히트
                }
                
                // L1 미스 → L2 조회
                return l2Cache.get(key)
                    .flatMap(l2Value -> {
                        if (l2Value != null) {
                            // L2에서 가져온 데이터를 L1에 저장
                            return l1Cache.set(key, l2Value)
                                .thenReturn(l2Value);
                        }
                        return Mono.empty();
                    });
            });
    }
    
    @Override
    public Mono<Boolean> set(K key, V value) {
        return Mono.zip(
            l1Cache.set(key, value).onErrorResume(e -> {
                log.warn("L1 캐시 저장 실패", e);
                return Mono.just(false);
            }),
            l2Cache.set(key, value).onErrorResume(e -> {
                log.warn("L2 캐시 저장 실패", e);
                return Mono.just(false);
            })
        ).thenReturn(true);
    }
}
```

### 3. Cache Factory 패턴

설정에 따라 캐시 전략을 바꿀 수 있도록 팩토리 패턴을 사용했습니다:

```java
/**
 * 캐시 전략을 선택하는 팩토리
 * 설정에 따라 Multi/Redis/Hazelcast 중 선택
 */
@Component
@Slf4j
public class CacheConfigFactory {
    
    @Value("${cache.flag}") // M: Multi, R: Redis, H: Hazelcast
    private String cacheFlag;
    
    public <K, V> CacheRepository<K, V> createCacheRepository(String mapName, Class<V> vClass) {
        CacheMode mode = CacheMode.fromFlag(cacheFlag.charAt(0));
        
        log.info("캐시 전략 선택: {} => {}", mapName, mode.getDescription());
        
        return switch (mode) {
            case MULTI -> createMultiRepository(mapName, vClass);
            case REDIS -> createRedisRepository(vClass);
            case HAZELCAST -> createHazelcastRepository(mapName, vClass);
        };
    }
    
    private <K, V> CacheRepository<K, V> createMultiRepository(String mapName, Class<V> vClass) {
        // L1: Hazelcast (빠른 액세스)
        CacheRepository<K, V> l1Cache = createHazelcastRepository(mapName, vClass);
        
        // L2: Redis (영속성)
        CacheRepository<K, V> l2Cache = createRedisRepository(vClass);
        
        return new MultiRepository<>(l1Cache, l2Cache);
    }
    
    private <K, V> CacheRepository<K, V> createRedisRepository(Class<V> vClass) {
        RedisTemplate<K, V> redisTemplate = redisConfigFactory.createRedisTemplate(vClass);
        
        // Jackson2JsonRedisSerializer 적용 (이전 글에서 선택한 방식)
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new Jackson2JsonRedisSerializer<>(vClass));
        
        return new RedisRepository<>(redisTemplate);
    }
}
```

## DataFetcher 패턴 - 자동 DB 폴백

### 캐시 미스 시 자동 DB 조회

캐시에 데이터가 없을 때 자동으로 DB에서 가져오는 패턴을 구현했습니다:

```java
/**
 * 캐시 미스 시 자동으로 DB에서 데이터를 가져오는 DataFetcher
 * 서비스 레이어에서 캐시 로직을 신경 쓰지 않아도 됨
 */
@Component
public class ProductDataFetcher implements DataFetcher<String, ProductDTO> {
    
    private final ProductRepository productRepository;
    private final ProductMapper productMapper;
    
    @Override
    public ProductDTO fetchData(String productId) {
        log.info("DB에서 상품 조회: {}", productId);
        
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new EntityNotFoundException("상품 없음: " + productId));
        
        return productMapper.toDTO(product);
    }
    
    @Override
    public List<ProductDTO> fetchDataList(Collection<String> productIds) {
        log.info("DB에서 상품 목록 조회: {} 건", productIds.size());
        
        return productRepository.findByIdIn(productIds)
            .stream()
            .map(productMapper::toDTO)
            .collect(Collectors.toList());
    }
}
```

### CacheDBProxy - 완전 자동화

DataFetcher와 캐시를 결합해서 완전 자동화된 캐시를 만들었습니다:

```java
/**
 * 캐시 + DB 자동 조회 프록시
 * 캐시 히트/미스를 자동으로 처리해줌
 */
@RequiredArgsConstructor
public class CacheDBProxy<K, V> {
    
    private final CacheRepository<K, V> cacheRepository;
    private final DataFetcher<K, V> dataFetcher;
    
    public V get(K key) {
        // 1. 캐시에서 조회
        V cached = cacheRepository.get(key);
        if (cached != null) {
            log.debug("캐시 히트: {}", key);
            return cached;
        }
        
        // 2. 캐시 미스 → DB 조회
        log.debug("캐시 미스 → DB 조회: {}", key);
        V data = dataFetcher.fetchData(key);
        
        // 3. 조회된 데이터를 캐시에 저장
        if (data != null) {
            cacheRepository.set(key, data);
            log.debug("DB 조회 완료 → 캐시 저장: {}", key);
        }
        
        return data;
    }
    
    public List<V> getList(Collection<K> keys) {
        Map<K, V> cacheResults = new HashMap<>();
        List<K> missedKeys = new ArrayList<>();
        
        // 1. 캐시에서 배치 조회
        for (K key : keys) {
            V cached = cacheRepository.get(key);
            if (cached != null) {
                cacheResults.put(key, cached);
            } else {
                missedKeys.add(key);
            }
        }
        
        // 2. 캐시 미스 키들을 DB에서 배치 조회
        if (!missedKeys.isEmpty()) {
            List<V> dbResults = dataFetcher.fetchDataList(missedKeys);
            
            // 3. DB 결과를 캐시에 저장
            for (int i = 0; i < missedKeys.size(); i++) {
                K key = missedKeys.get(i);
                V value = dbResults.get(i);
                if (value != null) {
                    cacheRepository.set(key, value);
                    cacheResults.put(key, value);
                }
            }
        }
        
        return keys.stream()
            .map(cacheResults::get)
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
}
```

## 분산 캐시 무효화 전략

### 1. Pub/Sub 기반 캐시 무효화

```java
/**
 * 분산 환경에서 캐시 무효화를 위한 메시징 시스템
 * 한 서비스에서 캐시를 업데이트하면 다른 서비스의 캐시도 자동 무효화
 */
@Service
@Slf4j
public class CacheInvalidationService {
    
    private final CacheMessageService messageService;
    private final Map<String, CacheRepository<?, ?>> cacheRepositories;
    
    public void invalidateCache(String cacheKey) {
        // 1. 로컬 캐시 무효화
        invalidateLocalCache(cacheKey);
        
        // 2. 다른 서비스에 무효화 메시지 전송
        CacheInvalidationMessage message = CacheInvalidationMessage.builder()
            .cacheKey(cacheKey)
            .timestamp(Instant.now())
            .serviceId(getServiceId())
            .build();
        
        messageService.publishInvalidation(message);
        log.info("캐시 무효화 메시지 전송: {}", cacheKey);
    }
    
    @EventListener
    public void handleCacheInvalidation(CacheInvalidationMessage message) {
        // 다른 서비스에서 온 무효화 메시지 처리
        if (!message.getServiceId().equals(getServiceId())) {
            invalidateLocalCache(message.getCacheKey());
            log.info("원격 캐시 무효화 처리: {}", message.getCacheKey());
        }
    }
    
    private void invalidateLocalCache(String cacheKey) {
        cacheRepositories.values().forEach(cache -> {
            try {
                cache.clear(cacheKey);
            } catch (Exception e) {
                log.warn("캐시 무효화 실패: {}", cacheKey, e);
            }
        });
    }
}
```

### 2. 분산 락을 활용한 동시성 제어

```java
/**
 * 분산 환경에서 캐시 동시성 제어
 * 여러 서비스가 동시에 같은 데이터를 조회할 때 DB 중복 접근 방지
 */
@Component
public class DistributedCacheService {
    
    private final CacheDBProxy<String, ProductDTO> productCache;
    private final RedisTemplate<String, String> lockTemplate;
    
    public ProductDTO getProductWithLock(String productId) {
        String lockKey = "lock:product:" + productId;
        String cacheKey = "product:" + productId;
        
        // 1. 분산 락 획득 시도
        Boolean acquired = lockTemplate.opsForValue()
            .setIfAbsent(lockKey, "locked", Duration.ofSeconds(10));
        
        if (Boolean.TRUE.equals(acquired)) {
            try {
                // 2. 락을 획득한 경우 캐시 조회 및 DB 접근
                return productCache.get(productId);
            } finally {
                // 3. 락 해제
                lockTemplate.delete(lockKey);
            }
        } else {
            // 4. 락 획득 실패 시 캐시만 조회 (DB 접근 방지)
            ProductDTO cached = productCache.cacheRepository.get(productId);
            if (cached != null) {
                return cached;
            }
            
            // 5. 잠시 대기 후 재시도
            try {
                Thread.sleep(100);
                return getProductWithLock(productId);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("캐시 조회 중단", e);
            }
        }
    }
}
```

## 고급 Hazelcast 패턴

### 1. EntryProcessor를 활용한 분산 연산

```java
/**
 * Hazelcast EntryProcessor를 사용한 분산 연산
 * 네트워크 오버헤드 없이 데이터가 있는 노드에서 직접 연산 수행
 */
public class HashFieldIncrementProcessor<HK, HV> implements EntryProcessor<Object, Map<HK, HV>, Long> {
    
    private HK field;
    private long delta;
    
    @Override
    public Long process(Map.Entry<Object, Map<HK, HV>> entry) {
        Map<HK, HV> value = entry.getValue();
        if (value == null) {
            value = new HashMap<>();
            entry.setValue(value);
        }
        
        // 현재 값 추출
        HV currentValue = value.get(field);
        long numericValue = extractNumericValue(currentValue);
        
        // 증가 연산
        long newValue = numericValue + delta;
        value.put(field, (HV) Long.valueOf(newValue));
        entry.setValue(value);
        
        return newValue;
    }
    
    private long extractNumericValue(HV currentValue) {
        if (currentValue instanceof String) {
            try {
                return Long.parseLong((String) currentValue);
            } catch (NumberFormatException e) {
                return 0;
            }
        } else if (currentValue instanceof Number) {
            return ((Number) currentValue).longValue();
        }
        return 0;
    }
}
```

### 2. 실제 사용 예제

```java
/**
 * 실무에서 사용하는 카운터 캐시 서비스
 * 실시간 조회수, 좋아요 수 등의 카운터 관리
 */
@Service
public class CounterCacheService {
    
    private final HazelcastInstance hazelcastInstance;
    
    public long incrementViewCount(String productId) {
        IMap<String, Map<String, Long>> counterMap = 
            hazelcastInstance.getMap("product-counters");
        
        // EntryProcessor로 분산 연산 수행
        return counterMap.executeOnKey(productId, 
            new HashFieldIncrementProcessor<>("viewCount", 1L));
    }
    
    public Map<String, Long> getProductStats(String productId) {
        IMap<String, Map<String, Long>> counterMap = 
            hazelcastInstance.getMap("product-counters");
        
        return counterMap.get(productId);
    }
}
```

## 실제 성능 측정

### 캐시 히트율 모니터링

성능 최적화를 위해 캐시 히트율을 모니터링했습니다:

```java
/**
 * 캐시 성능 모니터링
 * Micrometer로 메트릭 수집
 */
@Component
public class CacheMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    private final Counter l1HitCounter;
    private final Counter l2HitCounter;
    private final Counter cacheMissCounter;
    
    public CacheMetricsCollector(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.l1HitCounter = Counter.builder("cache.hit")
            .tag("level", "L1")
            .register(meterRegistry);
        this.l2HitCounter = Counter.builder("cache.hit")
            .tag("level", "L2")
            .register(meterRegistry);
        this.cacheMissCounter = Counter.builder("cache.miss")
            .register(meterRegistry);
    }
    
    public void recordL1Hit() {
        l1HitCounter.increment();
    }
    
    public void recordL2Hit() {
        l2HitCounter.increment();
    }
    
    public void recordCacheMiss() {
        cacheMissCounter.increment();
    }
}
```

### 구현 결과

실제로는 구현만 해두고 운영에는 적용하지 않았지만, 테스트해본 결과:

```
📊 테스트 결과

L1 캐시 (Hazelcast)
├── 응답 시간: 1ms 내외 (로컬 테스트)
├── 메모리 사용량: 설정에 따라 조절 가능
└── 개발 환경에서 정상 동작 확인

L2 캐시 (Redis)
├── 응답 시간: 3-5ms (개발 환경)
├── 안정적 동작 확인
└── 기존 Redis 캐시와 동일한 성능

아키텍처 자체
├── 코드 구조는 깔끔하게 구현됨
├── 팩토리 패턴으로 전략 변경 가능
└── 실제 운영에는 단일 캐시 사용 중
```

## 구현하면서 배운 점

### 캐시 일관성 관리

구현 과정에서 가장 고민했던 건 캐시 일관성 관리였습니다:

```java
/**
 * 캐시 일관성 보장 전략
 */
@Transactional
public class ProductService {
    
    public void updateProduct(ProductDTO productDTO) {
        // 1. DB 업데이트
        Product updated = productRepository.save(convertToEntity(productDTO));
        
        // 2. 캐시 무효화 (업데이트 아님)
        String cacheKey = "product:" + productDTO.getProductId();
        cacheInvalidationService.invalidateCache(cacheKey);
        
        // 3. 관련 캐시들도 무효화
        invalidateRelatedCaches(productDTO.getProductId());
        
        log.info("상품 업데이트 및 캐시 무효화 완료: {}", productDTO.getProductId());
    }
    
    private void invalidateRelatedCaches(String productId) {
        // 관련된 캐시들도 함께 무효화
        cacheInvalidationService.invalidateCache("product-list:*");
        cacheInvalidationService.invalidateCache("category:*");
        cacheInvalidationService.invalidateCache("search:*");
    }
}
```

### 장애 대응 전략

캐시 장애 시 자동 복구 메커니즘도 설계해봤습니다:

```java
/**
 * 캐시 장애 시 자동 복구
 */
@Component
public class CacheHealthChecker {
    
    @Scheduled(fixedDelay = 30000) // 30초마다 체크
    public void checkCacheHealth() {
        // L1 캐시 상태 확인
        if (!isL1CacheHealthy()) {
            log.warn("L1 캐시(Hazelcast) 장애 감지 - L2 캐시로 전환");
            switchToL2OnlyMode();
        }
        
        // L2 캐시 상태 확인
        if (!isL2CacheHealthy()) {
            log.warn("L2 캐시(Redis) 장애 감지 - 캐시 우회 모드로 전환");
            switchToCacheBypassMode();
        }
    }
    
    private boolean isL1CacheHealthy() {
        try {
            hazelcastInstance.getMap("health-check").put("test", "ok");
            return true;
        } catch (Exception e) {
            return false;
        }
    }
    
    private boolean isL2CacheHealthy() {
        try {
            redisTemplate.opsForValue().set("health-check", "ok");
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

## 마무리

### 배운 점들

멀티레벨 캐시 아키텍처를 구현하면서 배운 점들을 정리하면:

- **설계의 중요성**: 이론적으로는 L1/L2 캐시 조합으로 성능 개선 가능
- **구현의 복잡성**: 단순 캐시 대비 고려할 요소가 많음
- **확장성**: 팩토리 패턴으로 새로운 서비스 추가 용이
- **실용성**: 실제 운영에서는 단순한 구조가 더 나을 수 있음

### 실제 운영에서는...

현재 프로젝트에서는 여러 이유로 단일 캐시만 사용하고 있습니다:

- **복잡성**: 멀티레벨 캐시는 구현 복잡도가 높음
- **운영 부담**: 모니터링할 요소가 많아짐
- **트래픽 규모**: 현재 트래픽에서는 단일 캐시로도 충분
- **팀 리소스**: 구현보다는 다른 기능 개발에 집중

멀티레벨 캐시는 분명 좋은 아키텍처지만, 실제 운영에서는 트래픽 규모와 팀 상황을 고려해야 한다는 걸 배웠습니다.

그래도 구현 과정에서 캐시 아키텍처에 대해 많이 배울 수 있었고, 나중에 필요할 때 바로 적용할 수 있도록 준비되어 있습니다.

---

*구현 경험을 바탕으로 작성했습니다.*