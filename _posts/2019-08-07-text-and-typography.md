---
title: OpenTelemetry + ELK 기반 Spring Boot 멀티모듈 관측성 시스템 구축기
description: 기존 Prometheus 모니터링을 확장한 트레이스 추적 시스템 도입과 스택 변경 여정
author: backend-engineer
date: 2024-07-12 14:30:00 +0900
categories: [Backend, Monitoring]
tags: [opentelemetry, elk, spring-boot, zipkin, prometheus, observability, microservices]
pin: true
math: false
mermaid: true
image:
  path: /assets/img/posts/opentelemetry-architecture.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: OpenTelemetry 기반 관측성 시스템 아키텍처
---

## 들어가며

Spring Boot 기반 멀티모듈 시스템을 운영하면서 기존 Prometheus + Grafana 모니터링으로는 서비스 간 트랜잭션 흐름 추적에 한계를 느끼게 되었습니다. 특히 OpenAPI 시스템 확장 과정에서 모듈 간 호출 관계와 장애 지점 파악의 어려움이 대두되었고, 이를 해결하기 위해 OpenTelemetry 기반 관측성 시스템을 도입했습니다.

본 글에서는 기존 모니터링 스택을 유지하면서 새로운 트레이싱 기능을 추가하는 과정과, ELK 스택에서 Zipkin + Prometheus 조합으로 변경하게 된 기술적 의사결정 과정을 공유합니다.

---

## 기존 시스템 현황과 한계점 분석

### 기존 모니터링 아키텍처

우리 시스템은 인프라 레벨에서 다양한 Exporter를 활용한 Prometheus + Grafana 조합으로 구성되어 있었습니다:

```mermaid
graph TB
    subgraph "Infrastructure Monitoring"
        A[Node Exporter<br/>시스템 메트릭]
        B[MySQLd Exporter<br/>DB 메트릭]
        C[Process Exporter<br/>프로세스 메트릭]
        D[Redis Exporter<br/>캐시 메트릭]
        E[Push Gateway<br/>배치 작업 메트릭]
    end
    
    subgraph "Application Layer"
        F[Spring Boot Apps<br/>별도 계측 없음]
    end
    
    A --> G[Prometheus]
    B --> G
    C --> G
    D --> G
    E --> G
    
    G --> H[Grafana]
```

> 기존 시스템은 인프라 레벨에서의 메트릭 수집과 시각화에는 효과적이었지만, 애플리케이션 레벨의 비즈니스 로직 추적에는 한계가 있었습니다.
{: .prompt-info }

**기존 시스템의 강점**:
- **인프라 중심 모니터링**: Node Exporter, MySQLd Exporter, Redis Exporter 등을 통한 체계적인 시스템 리소스 모니터링
- **애플리케이션 무침투성**: Spring Boot 애플리케이션에 별도 계측 코드 추가 없이 외부에서 메트릭 수집
- **Grafana 대시보드**: 다양한 익스포터 데이터를 통합한 직관적인 모니터링 환경
- **안정적인 운영**: Process Exporter를 통한 애플리케이션 프로세스 상태 모니터링

**한계점**:
- **애플리케이션 레벨 가시성 부족**: 인프라 메트릭만으로는 비즈니스 로직 수준의 성능 병목 파악 어려움
- **서비스 간 트랜잭션 추적 불가**: API Gateway → Auth Service → Business Service → External API 호출 체인에서 어느 구간에서 지연이 발생하는지 파악 불가
- **요청별 상관관계 추적 한계**: 동일한 사용자 요청이 여러 서비스를 거치면서 발생하는 로그와 메트릭의 연관성 파악 어려움
- **비즈니스 메트릭 부재**: HTTP 상태 코드, 응답 시간 등 기술적 메트릭은 있지만 실제 비즈니스 성공/실패율 등의 지표 부족

### OpenAPI 시스템 확장 요구사항

새로 구축하게 된 OpenAPI 시스템은 기존 인프라 중심 모니터링으로는 충분히 관측하기 어려운 특성을 가졌습니다:

- **복잡한 서비스 의존성**: API Gateway → Auth Service → Business Service → External API 호출의 다단계 처리
- **높은 가용성 요구**: OpenAPI 장애가 곧 고객사 서비스 중단으로 이어지는 크리티컬한 시스템
- **다양한 호출 패턴**: 동기/비동기 호출, 배치 처리, 이벤트 기반 처리 등 복합적인 아키텍처
- **애플리케이션 레벨 병목**: 기존 Node Exporter, Process Exporter로는 파악할 수 없는 비즈니스 로직 내부의 성능 이슈

> 기존 인프라 메트릭으로는 "서버는 정상, DB도 정상, 프로세스도 정상"인 상황에서 실제 사용자 요청이 느려지는 원인을 찾기 어려웠습니다.
{: .prompt-warning }

---

## OpenTelemetry 도입 설계와 아키텍처 구성

### 확장된 모니터링 아키텍처 설계

기존 인프라 레벨 Prometheus 모니터링을 유지하면서 애플리케이션 레벨 관측성 기능을 추가하는 방향으로 설계했습니다:

```mermaid
graph TB
    subgraph "기존 Infrastructure Monitoring (유지)"
        I1[Node Exporter]
        I2[MySQLd Exporter]
        I3[Process Exporter]
        I4[Redis Exporter]
        I5[Push Gateway]
    end
    
    subgraph "새로운 Application Observability"
        A[A-Service<br/>OpenTelemetry Java Agent<br/>Spring Boot Starter]
        B[B-Service<br/>OpenTelemetry Java Agent<br/>Spring Boot Starter]
        C[C-Service<br/>OpenTelemetry Java Agent<br/>Spring Boot Starter]
    end
    
    I1 --> P[Prometheus<br/>기존 + 새로운 메트릭]
    I2 --> P
    I3 --> P
    I4 --> P
    I5 --> P
    
    A --> D[OpenTelemetry Collector]
    B --> D
    C --> D
    
    D --> E[Zipkin<br/>Traces]
    D --> P
    D --> G[Logstash<br/>Logs]
    
    G --> H[Elasticsearch]
    
    E --> J[Vue.js Web Dashboard]
    P --> K[Grafana<br/>기존 대시보드 유지]
    P --> J
    H --> J
```

> 기존 인프라 모니터링 시스템을 그대로 유지하면서 애플리케이션 레벨의 관측성만 새롭게 추가하는 점진적 확장 방식을 채택했습니다.
{: .prompt-tip }

### OpenTelemetry 이중 계측 전략

효과적인 트레이싱을 위해 자동 계측과 수동 계측을 병행했습니다:

#### 자동 계측 (OpenTelemetry Java Agent)

1. HTTP 클라이언트/서버 요청 자동 추적
2. 데이터베이스 쿼리 (JPA, MyBatis) 자동 계측
3. Redis, Kafka 등 미들웨어 호출 자동 추적

#### 수동 계측 (OpenTelemetry Spring Starter)

1. 비즈니스 로직 구간별 상세 추적
2. 사용자 정의 속성 추가 (사용자 ID, 요청 타입 등)
3. 에러 상황별 세분화된 추적

### 실제 적용 사례

**Spring Boot 애플리케이션 설정**:

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus, metrics
  tracing:
    sampling:
      probability: 0.1  # 10% 샘플링

otel:
  exporter:
    otlp:
      endpoint: http://otel-collector:4317
  resource:
    attributes:
      service.name: api-gateway-service
      service.version: 1.2.0
      environment: production
```
{: file='src/main/resources/application.yml'}

**비즈니스 로직 트레이싱**:

```java
@RestController
@RequiredArgsConstructor
public class OpenApiController {
    
    private final Tracer tracer = GlobalOpenTelemetry.getTracer("openapi-service");
    private final ExternalApiService externalApiService;
    
    @PostMapping("/api/v1/process")
    public ResponseEntity<ApiResponse> processRequest(@RequestBody ApiRequest request) {
        Span span = tracer.spanBuilder("openapi.process.request")
                .setAttribute("api.client.id", request.getClientId())
                .setAttribute("api.request.type", request.getType())
                .startSpan();
                
        try (Scope scope = span.makeCurrent()) {
            // 1. 요청 검증
            validateRequest(request);
            
            // 2. 외부 API 호출
            Span externalSpan = tracer.spanBuilder("external.api.call")
                    .setAttribute("external.endpoint", request.getTargetEndpoint())
                    .startSpan();
                    
            try (Scope externalScope = externalSpan.makeCurrent()) {
                ApiResponse response = externalApiService.callExternalApi(request);
                externalSpan.setAttribute("external.response.status", response.getStatus());
                return ResponseEntity.ok(response);
            } finally {
                externalSpan.end();
            }
            
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```
{: file='src/main/java/com/example/controller/OpenApiController.java'}

---

## 기술 스택 변경 여정: ELK에서 Zipkin + Prometheus로

### 초기 계획: OpenTelemetry + ELK 스택

처음에는 다음과 같은 ELK 기반 아키텍처를 계획했습니다:

```mermaid
graph LR
    A[OpenTelemetry] --> B[Elasticsearch]
    B --> C[Kibana]
```

**ELK 스택 선택 이유**:
- 대용량 로그 데이터 처리에 최적화된 Elasticsearch
- Kibana의 강력한 시각화 및 대시보드 기능
- 로그와 트레이싱 데이터의 통합 분석 가능

### ELK 스택 운영 중 발견된 문제점

> 실제 운영 과정에서 다음과 같은 한계점들이 발견되었습니다.
{: .prompt-warning }

#### 1. Kibana 보안 이슈
- 기본 설정으로는 접근 제어가 미흡
- X-Pack Security 도입 시 라이선스 비용 발생
- 사내 보안 정책과의 충돌

#### 2. Vue.js 대시보드 통합 어려움
- Kibana 데이터를 외부 애플리케이션에서 직접 활용하기 복잡
- 커스텀 대시보드 구축 시 Kibana API의 제약사항
- 기존 Vue.js 기반 관리 시스템과의 일관성 부족

#### 3. Prometheus와의 이원화 문제
- 기존 Prometheus 메트릭과 ELK 트레이싱 데이터가 분리
- 메트릭과 트레이스 상관관계 분석의 어려움
- 운영팀의 도구 학습 부담 증가

### 최종 선택: Zipkin + Prometheus + Logstash 조합

여러 POC(Proof of Concept)를 거쳐 다음 조합으로 최종 결정했습니다:

| 기능 | 기존 ELK | 변경 후 |
|:-----|:---------|:--------|
| 트레이싱 | Kibana | **Zipkin** |
| 메트릭 | 별도 Prometheus | **Prometheus** (유지) |
| 로그 수집 | Logstash | **Logstash** (유지) |
| 로그 저장 | Elasticsearch | **Elasticsearch** (유지) |

**변경 결정의 핵심 요인**:

#### Zipkin 선택 이유
- 가벼운 트레이싱 UI 제공으로 빠른 장애 분석 가능
- REST API 지원으로 Vue.js 대시보드와 쉬운 통합
- 단일 컨테이너 운영으로 운영 복잡도 최소화
- 메모리 사용량이 Jaeger 대비 약 40% 적음

#### Prometheus 유지 이유
- **기존 인프라 모니터링 연속성**: Node Exporter, MySQLd Exporter 등 기존 익스포터 기반 모니터링 시스템과의 일관성 유지
- **운영 노하우 보존**: 기존 Process Exporter를 통한 애플리케이션 프로세스 모니터링 경험과 알러트 룰 재활용
- **Grafana 대시보드 자산 활용**: 기존에 구축된 인프라 대시보드와 새로운 애플리케이션 메트릭의 통합 가능
- **점진적 전환**: 기존 Push Gateway를 통한 배치 작업 모니터링을 유지하면서 새로운 트레이싱 기능 추가

#### Logstash 활용 이유
- 기존 Elasticsearch 인프라 활용
- 다양한 로그 포맷 파싱 및 변환 기능
- OpenTelemetry 데이터의 로그 형태 변환 처리

---

## 구현 과정과 기술적 도전

### OpenTelemetry Collector 설정

트레이싱, 메트릭, 로그를 각각 적절한 백엔드로 라우팅하는 Collector 구성:

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  
  resource:
    attributes:
      - key: environment
        value: production
        action: upsert
      - key: cluster
        value: openapi-cluster
        action: upsert

exporters:
  # Zipkin으로 트레이싱 데이터 전송
  zipkin:
    endpoint: http://zipkin:9411/api/v2/spans
    
  # Prometheus로 메트릭 전송
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: "openapi"
    
  # Logstash로 로그 데이터 전송
  logging:
    loglevel: info

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [zipkin]
      
    metrics:
      receivers: [otlp] 
      processors: [resource, batch]
      exporters: [prometheus]
      
    logs:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [logging]
```
{: file='docker/otel-collector-config.yaml'}

### Vue.js 대시보드 통합

기존 Vue.js 관리 시스템에 트레이싱 기능을 통합했습니다:

```javascript
// Zipkin API 연동
export class TracingService {
  constructor() {
    this.zipkinBaseUrl = process.env.VUE_APP_ZIPKIN_URL;
    this.prometheusBaseUrl = process.env.VUE_APP_PROMETHEUS_URL;
  }
  
  // 특정 서비스의 최근 트레이스 조회
  async getRecentTraces(serviceName, limit = 100) {
    const endTime = Date.now() * 1000; // microseconds
    const startTime = endTime - (24 * 60 * 60 * 1000 * 1000); // 24시간 전
    
    const response = await fetch(
      `${this.zipkinBaseUrl}/api/v2/traces?serviceName=${serviceName}&endTs=${endTime}&lookback=${startTime}&limit=${limit}`
    );
    return response.json();
  }
  
  // 특정 TraceId로 상세 트레이스 조회
  async getTraceDetail(traceId) {
    const response = await fetch(`${this.zipkinBaseUrl}/api/v2/trace/${traceId}`);
    return response.json();
  }
  
  // Prometheus에서 에러율 메트릭 조회
  async getErrorRate(serviceName, timeRange = '1h') {
    const query = `rate(http_requests_total{job="${serviceName}",status=~"5.."}[5m])`;
    const response = await fetch(
      `${this.prometheusBaseUrl}/api/v1/query?query=${encodeURIComponent(query)}`
    );
    return response.json();
  }
}
```
{: file='src/services/TracingService.js'}

### 주요 기술적 도전과 해결책

#### 도전 1: 높은 트래픽에서의 성능 영향

> 초기 100% 트레이싱으로 인한 성능 저하 문제가 발생했습니다.
{: .prompt-danger }

**문제**: 응답 시간 5-10% 증가, CPU 사용률 15% 상승

**해결**: 적응형 샘플링 도입

```java
// 적응형 샘플링 설정
@Configuration
public class TracingConfiguration {
    
    @Bean
    public SdkTracerProvider tracerProvider() {
        return SdkTracerProvider.builder()
                .addSpanProcessor(BatchSpanProcessor.builder(
                        OtlpGrpcSpanExporter.builder()
                                .setEndpoint("http://otel-collector:4317")
                                .build())
                        .setMaxExportBatchSize(512)
                        .setExportTimeout(Duration.ofSeconds(2))
                        .build())
                .setSampler(TraceIdRatioBasedSampler.create(0.1)) // 10% 샘플링
                .build();
    }
}
```
{: file='src/main/java/com/example/config/TracingConfiguration.java'}

#### 도전 2: 비동기 처리에서의 TraceId 전파 문제

**문제**: Spring의 `@Async` 메서드에서 트레이스 컨텍스트 손실

**해결**: TraceableExecutorService 래핑

```java
@Configuration
@EnableAsync
public class AsyncConfiguration implements AsyncConfigurer {
    
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        
        // OpenTelemetry 컨텍스트 전파를 위한 래핑
        return Context.taskWrapping(executor);
    }
}
```
{: file='src/main/java/com/example/config/AsyncConfiguration.java'}

#### 도전 3: 대용량 로그 데이터 처리

**문제**: 일일 수십 GB의 로그 데이터로 인한 Elasticsearch 부하

**해결**: Logstash 파이프라인 최적화와 인덱스 생명주기 관리

```ruby
# logstash.conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [fields][service] {
    mutate {
      add_field => { "service_name" => "%{[fields][service]}" }
    }
  }
  
  # TraceId 기반 필터링
  if [traceId] and [traceId] != "" {
    mutate {
      add_tag => [ "traced" ]
    }
  }
  
  # 불필요한 디버그 로그 제거
  if [level] == "DEBUG" {
    drop { }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "openapi-logs-%{+YYYY.MM.dd}"
    template_name => "openapi-logs"
    template_pattern => "openapi-logs-*"
  }
}
```
{: file='config/logstash.conf'}

---

## 운영 결과와 효과 측정

### 트랜잭션 추적 성능 개선

#### 장애 대응 시간 단축

| 구분 | 이전 | 이후 | 개선율 |
|:-----|:-----|:-----|:-------|
| 평균 장애 대응 시간 | 25분 | 5분 | **80% 단축** |
| 복잡한 호출 체인 분석 | 40-60분 | 3-5분 | **90% 단축** |

> 트레이스 기반 장애 분석으로 MTTR(Mean Time To Recovery)이 대폭 개선되었습니다.
{: .prompt-tip }

### 구체적인 장애 해결 사례

#### 사례 1: 외부 API 호출 지연 문제

**기존 접근법 (인프라 모니터링 중심)**:
1. Grafana에서 Node Exporter 메트릭으로 CPU/메모리 이상 없음 확인
2. MySQLd Exporter에서 DB 성능 지표 정상 확인
3. Process Exporter에서 애플리케이션 프로세스 상태 정상 확인
4. 각 서버별 애플리케이션 로그 파일 수동 검색
5. 외부 API 호출 로그를 찾기 위해 여러 서비스 로그 분석
6. **총 소요 시간: 35분**

**개선된 접근법 (트레이싱 기반)**:
1. Zipkin에서 느린 트레이스 필터링
2. 트레이스 상세 보기에서 외부 API 호출 구간 8초 지연 발견
3. TraceId로 관련 로그 즉시 필터링하여 API 타임아웃 원인 파악
4. **총 소요 시간: 3분**

> 기존 인프라 메트릭으로는 "시스템은 정상이지만 성능이 느리다"는 상황에서 정확한 병목 지점을 찾기 어려웠습니다.
{: .prompt-warning }

#### 사례 2: 데이터베이스 커넥션 풀 고갈

**기존 접근법의 한계**:
- MySQLd Exporter에서는 DB 자체 성능은 정상으로 표시
- Process Exporter에서 애플리케이션 메모리 사용량만 모니터링 가능
- 커넥션 풀 레벨의 세부 정보는 애플리케이션 로그에서만 확인 가능

**트레이싱 기반 해결**:
- Zipkin 트레이스에서 DB 호출 구간의 이상 패턴 발견: 특정 시간대 DB Span의 대기 시간이 급증
- TraceId로 동일 시간대 다른 요청들의 DB 접근 패턴 분석
- 애플리케이션 레벨에서 커넥션 풀 고갈 현상 즉시 파악하여 설정 조정

> 인프라 메트릭으로는 "DB는 정상, 애플리케이션도 정상"이지만 실제로는 애플리케이션-DB 간 연결 레벨에서 문제가 발생한 사례였습니다.
{: .prompt-info }

### 비즈니스 임팩트

#### API 가용성 향상
- SLA 99.9% → 99.95% 달성
- 평균 장애 시간 60분 → 15분으로 단축

#### 운영 효율성
- 온콜 엔지니어의 야간 대응 건수 50% 감소
- 장애 대응을 위한 팀 간 커뮤니케이션 시간 70% 단축

#### 고객 만족도
- OpenAPI 사용 고객사의 장애 관련 문의 60% 감소
- API 응답 시간 안정성 개선으로 고객사 만족도 향상

---

## 스택 변경에 대한 회고와 교훈

### 기술 스택 선택의 교훈

#### 완벽한 솔루션보다 적합한 솔루션

> ELK 스택이 기능적으로는 더 강력했지만, 우리 환경과 요구사항에는 Zipkin + Prometheus 조합이 더 적합했습니다.
{: .prompt-tip }

기존 인프라와의 호환성, 팀의 기술 역량, 운영 복잡도를 종합적으로 고려한 선택이 중요했습니다.

#### 점진적 도입의 중요성

- 기존 Prometheus 시스템을 유지하면서 새로운 트레이싱 기능을 추가하는 방식으로 리스크를 최소화
- 전면적인 스택 교체보다는 단계적 확장이 더 안전하고 효과적

### 향후 개선 방향

#### AI 기반 이상 탐지 도입
- [ ] 현재 임계값 기반 알러트의 한계를 극복하기 위해 머신러닝 기반 이상 패턴 감지 시스템 도입 검토
- [ ] Zipkin 트레이스 데이터를 활용한 성능 저하 예측 모델 구축

#### 비즈니스 메트릭과의 연계 강화
- [ ] 기술적 메트릭을 비즈니스 지표(매출, 사용자 경험)와 연결하여 장애의 비즈니스 임팩트 정량화
- [ ] SLO/SLI 기반 서비스 품질 관리 체계 구축

#### 멀티 클라우드 관측성
- [ ] 향후 클라우드 확장 시에도 일관된 관측성을 유지할 수 있도록 OpenTelemetry 표준 활용 확대

---

## 마무리

OpenTelemetry 기반 관측성 시스템 구축을 통해 기존 인프라 중심 모니터링의 한계를 극복하고 애플리케이션 레벨의 진정한 관측성을 확보할 수 있었습니다. 특히 기존 Prometheus + 다양한 Exporter 기반 인프라 모니터링을 그대로 유지하면서 새로운 트레이싱 기능을 점진적으로 추가한 접근 방식이 성공의 핵심이었습니다.

기술적으로는 ELK에서 Zipkin + Prometheus 조합으로의 스택 변경을 통해 우리 환경에 최적화된 경량화된 솔루션을 구축했으며, 실무적으로는 장애 대응 시간을 **80% 단축**하여 서비스 안정성을 크게 향상시켰습니다.

무엇보다 중요한 것은 **인프라 레벨에서 애플리케이션 레벨로의 관측성 확장**이 단순한 기술 추가가 아니라 운영 패러다임의 전환을 의미한다는 점입니다. 기존 "서버는 정상인데 왜 느린가?"에서 "어떤 비즈니스 로직에서 병목이 발생하는가?"로 질문의 차원이 바뀌었고, 이를 통해 더욱 정확하고 빠른 장애 대응이 가능해졌습니다.

앞으로도 Node Exporter, MySQLd Exporter 등 검증된 인프라 모니터링을 기반으로 하면서, OpenTelemetry 기반 애플리케이션 관측성을 지속적으로 발전시켜 더욱 안정적이고 신뢰할 수 있는 시스템을 구축해 나가겠습니다.