---
title: Spring Redis 직렬화 방식 비교 및 Jackson2JsonRedisSerializer 선택 이유
date: 2023-12-15 11:00:00 +0900
categories: [Backend, Spring]
tags: [redis, serialization, jackson, spring-boot, multi-module]
---

## 들어가며

Redis 캐시를 도입하면서 직렬화 방식 선택이 생각보다 복잡했습니다. Spring Data Redis에서 제공하는 여러 직렬화 방식 중 어떤 걸 써야 할지 고민하다가, 각각의 특징을 정리해보고 우리 상황에 맞는 선택을 했던 과정을 공유합니다. 결론적으로 Jackson2JsonRedisSerializer를 선택했는데, 그 이유와 실제 적용 과정을 정리했습니다.

## 상황 정리

### 직렬화 방식 선택 고민

프로젝트에서 Redis 캐시를 도입하게 되면서 가장 먼저 부딪힌 문제가 직렬화 방식 선택이었습니다. 그냥 대충 넘어가려다가 각 방식의 차이점을 알아보니 생각보다 중요한 선택이더라고요.

```java
// Redis Template 설정 시 직렬화 방식 선택 필요
@Bean
public RedisTemplate<?, ?> redisTemplate() {
    RedisTemplate<?, ?> redisTemplate = new RedisTemplate<>();
    redisTemplate.setConnectionFactory(redisConnectionFactory());
    redisTemplate.setEnableTransactionSupport(true);

    // 어떤 직렬화 방식을 선택할 것인가?
    redisTemplate.setKeySerializer(new JdkSerializationRedisSerializer());        // 기본값
    redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());   // JSON + @class
    redisTemplate.setHashKeySerializer(new Jackson2JsonRedisSerializer());        // 순수 JSON
    redisTemplate.setHashValueSerializer(new StringRedisSerializer());           // 문자열

    return redisTemplate;
}
```

### 우리 프로젝트 상황

프로젝트가 멀티모듈 구조로 되어 있어서 여러 모듈에서 같은 캐시 데이터를 공유해야 하는 상황이었습니다:

```
project-root/
├── product-service/
├── order-service/
├── user-service/
├── common-dto/        ← 공통 DTO 모듈
│   ├── ProductDTO
│   ├── UserDTO
│   └── OrderDTO
```

**필요했던 것들:**
- 여러 모듈에서 동일한 DTO 공유
- 패키지 경로에 의존하지 않는 캐시 데이터
- 타입 안전성
- 모듈 간 결합도 최소화

## Spring Redis 직렬화 방식 비교

Spring Data Redis에서 제공하는 4가지 방식을 하나씩 살펴보겠습니다.

### 1. JdkSerializationRedisSerializer (기본값)

```java
/**
 * JDK 기본 직렬화 방식 분석
 */
@Component
public class JdkSerializationAnalysis {
    
    public void demonstrateJdkSerialization() {
        JdkSerializationRedisSerializer serializer = new JdkSerializationRedisSerializer();
        
        ProductDTO product = ProductDTO.builder()
                .productId("P001")
                .name("Test Product")
                .price(BigDecimal.valueOf(100))
                .build();
                
        byte[] serialized = serializer.serialize(product);
        
        log.info("JDK serialized size: {} bytes", serialized.length);
        // 결과: 일반적으로 JSON보다 2-3배 큰 용량
    }
    
    public void demonstrateVersioningProblem() {
        // serialVersionUID 미설정 시:
        // ProductDTO 클래스에 필드 하나만 추가해도 기존 캐시 데이터 읽기 실패
        
        // serialVersionUID 설정해도:
        // 필드 타입 변경 시 InvalidClassException 발생
        // 예: String name → Name nameObj 같은 변경
    }
}
```

**JDK 직렬화의 문제점:**
- 클래스 구조가 조금만 바뀌어도 기존 캐시 데이터 못 읽음
- 용량이 크고 성능도 별로
- 바이너리 데이터라서 디버깅할 때 내용을 볼 수 없음

### 2. GenericJackson2JsonRedisSerializer

```java
/**
 * GenericJackson2JsonRedisSerializer 분석
 */
@Component
public class GenericJacksonAnalysis {
    
    public void demonstrateGenericJackson() {
        GenericJackson2JsonRedisSerializer serializer = new GenericJackson2JsonRedisSerializer();
        
        ProductDTO product = ProductDTO.builder()
                .productId("P001")
                .name("Test Product")
                .price(BigDecimal.valueOf(100))
                .build();
                
        byte[] serialized = serializer.serialize(product);
        String jsonString = new String(serialized, StandardCharsets.UTF_8);
        
        log.info("Generic Jackson JSON: {}", jsonString);
        
        // 실제 저장되는 JSON:
        // {
        //   "@class": "com.company.common.dto.ProductDTO",
        //   "productId": "P001",
        //   "name": "Test Product", 
        //   "price": 100
        // }
    }
    
    public void demonstratePackageDependencyProblem() {
        // 문제: @class 필드에 전체 패키지 경로 포함
        // 서로 다른 MSA 서비스에서 동일한 패키지 구조 강제됨
        
        // Service A: com.company.product.dto.ProductDTO
        // Service B: com.company.order.dto.ProductDTO  
        // ← 같은 데이터라도 패키지 다르면 역직렬화 실패
    }
}
```

**GenericJackson2JsonRedisSerializer 문제점:**
- @class 필드 때문에 패키지 경로에 의존하게 됨
- 서로 다른 서비스에서 패키지 구조가 달라지면 문제 발생
- @class 메타데이터로 용량 증가

### 3. Jackson2JsonRedisSerializer

```java
/**
 * Jackson2JsonRedisSerializer 분석
 */
@Configuration
public class Jackson2JsonAnalysis {
    
    @Bean("productRedisTemplate")
    public RedisTemplate<String, ProductDTO> productRedisTemplate() {
        RedisTemplate<String, ProductDTO> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        
        // 타입별 전용 Serializer 설정 필요
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new Jackson2JsonRedisSerializer<>(ProductDTO.class));
        
        return template;
    }
    
    @Bean("userRedisTemplate")  
    public RedisTemplate<String, UserDTO> userRedisTemplate() {
        RedisTemplate<String, UserDTO> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        
        // 타입별로 별도 RedisTemplate 필요
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new Jackson2JsonRedisSerializer<>(UserDTO.class));
        
        return template;
    }
    
    public void demonstrateCleanJson() {
        Jackson2JsonRedisSerializer<ProductDTO> serializer = 
            new Jackson2JsonRedisSerializer<>(ProductDTO.class);
        
        ProductDTO product = ProductDTO.builder()
                .productId("P001")
                .name("Test Product")
                .price(BigDecimal.valueOf(100))
                .build();
                
        byte[] serialized = serializer.serialize(product);
        String jsonString = new String(serialized, StandardCharsets.UTF_8);
        
        log.info("Jackson2 JSON: {}", jsonString);
        
        // 깔끔한 JSON 결과 (@class 없음):
        // {
        //   "productId": "P001", 
        //   "name": "Test Product",
        //   "price": 100
        // }
    }
}
```

**Jackson2JsonRedisSerializer 특징:**
- **장점**: @class 없는 깔끔한 JSON, 타입 안전성
- **단점**: 타입별로 별도 RedisTemplate 빈 설정 필요
- **결론**: 설정 코드는 늘어나지만 안전하고 성능도 좋음

### 4. StringRedisSerializer

```java
/**
 * StringRedisSerializer 분석
 */
@Component
public class StringSerializerAnalysis {
    
    private final ObjectMapper objectMapper = new ObjectMapper();
    
    public void demonstrateStringSerialization() {
        StringRedisSerializer serializer = new StringRedisSerializer();
        
        ProductDTO product = ProductDTO.builder()
                .productId("P001")
                .name("Test Product")
                .price(BigDecimal.valueOf(100))
                .build();
        
        try {
            // 수동 JSON 변환 필요
            String jsonString = objectMapper.writeValueAsString(product);
            byte[] serialized = serializer.serialize(jsonString);
            
            log.info("String serialized size: {} bytes", serialized.length);
            log.info("Pure JSON: {}", jsonString);
            
            // 역직렬화도 수동으로
            String deserializedString = serializer.deserialize(serialized);
            ProductDTO deserializedProduct = objectMapper.readValue(deserializedString, ProductDTO.class);
            
        } catch (Exception e) {
            log.error("String serialization failed", e);
        }
    }
}
```

**StringRedisSerializer 특징:**
- **장점**: 용량 최소, 완전한 타입 독립성, 속도 빠름
- **단점**: 수동 JSON 변환 필요, 타입 안전성 부족
- **적용**: 단순한 캐시나 성능이 최우선인 경우

## 최종 선택: Jackson2JsonRedisSerializer

### 선택 이유

여러 방식을 비교해본 결과 **Jackson2JsonRedisSerializer**를 선택했습니다:

```java
/**
 * Jackson2JsonRedisSerializer 실제 적용
 * 타입별로 RedisTemplate을 만들어서 사용
 */
@Configuration
public class RedisSerializationConfig {
    
    // 공통 DTO별 전용 RedisTemplate 생성
    @Bean("productRedisTemplate")
    public RedisTemplate<String, ProductDTO> productRedisTemplate() {
        RedisTemplate<String, ProductDTO> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        
        // ProductDTO는 공통 모듈에서 공유
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new Jackson2JsonRedisSerializer<>(ProductDTO.class));
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new Jackson2JsonRedisSerializer<>(ProductDTO.class));
        
        template.afterPropertiesSet();
        return template;
    }
    
    @Bean("userRedisTemplate")
    public RedisTemplate<String, UserDTO> userRedisTemplate() {
        RedisTemplate<String, UserDTO> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new Jackson2JsonRedisSerializer<>(UserDTO.class));
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new Jackson2JsonRedisSerializer<>(UserDTO.class));
        
        template.afterPropertiesSet();
        return template;
    }
    
    @Bean("orderRedisTemplate")
    public RedisTemplate<String, OrderDTO> orderRedisTemplate() {
        RedisTemplate<String, OrderDTO> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new Jackson2JsonRedisSerializer<>(OrderDTO.class));
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new Jackson2JsonRedisSerializer<>(OrderDTO.class));
        
        template.afterPropertiesSet();
        return template;
    }
}
```

### 실제 사용 예시

```java
/**
 * ProductService에서 실제 사용
 */
@Service
@Slf4j
public class ProductService {
    
    private final RedisTemplate<String, ProductDTO> productRedisTemplate;
    private final ProductRepository productRepository;
    
    public ProductService(@Qualifier("productRedisTemplate") RedisTemplate<String, ProductDTO> productRedisTemplate,
                         ProductRepository productRepository) {
        this.productRedisTemplate = productRedisTemplate;
        this.productRepository = productRepository;
    }
    
    public ProductDTO getProduct(String productId) {
        String cacheKey = "product:" + productId;
        
        // 캐시에서 조회
        ProductDTO cached = productRedisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            log.debug("캐시에서 조회: {}", productId);
            return cached;
        }
        
        // DB 조회
        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new EntityNotFoundException("상품 없음: " + productId));
        
        // Entity → DTO 변환
        ProductDTO productDTO = convertToDTO(product);
        
        // 캐시에 저장
        productRedisTemplate.opsForValue().set(cacheKey, productDTO, Duration.ofHours(1));
        
        log.info("DB 조회 후 캐시 저장: {}", productId);
        return productDTO;
    }
    
    public void updateProduct(ProductDTO productDTO) {
        // DB 업데이트
        Product product = convertToEntity(productDTO);
        productRepository.save(product);
        
        // 캐시 무효화
        String cacheKey = "product:" + productDTO.getProductId();
        productRedisTemplate.delete(cacheKey);
        
        log.info("상품 업데이트 및 캐시 무효화: {}", productDTO.getProductId());
    }
    
    private ProductDTO convertToDTO(Product product) {
        return ProductDTO.builder()
                .productId(product.getProductId())
                .name(product.getName())
                .description(product.getDescription())
                .price(product.getPrice())
                .category(product.getCategory())
                .stockCount(product.getStockCount())
                .status(product.getStatus())
                .createdAt(product.getCreatedAt())
                .updatedAt(product.getUpdatedAt())
                .build();
    }
}
```

## 적용 결과

### 실제 테스트 해보기

```java
@SpringBootTest
class SerializationTest {
    
    @Test
    void 직렬화_방식_비교() {
        ProductDTO testProduct = createTestProduct();
        
        // 1. GenericJackson2JsonRedisSerializer
        SerializationResult genericResult = testGenericJackson(testProduct);
        
        // 2. Jackson2JsonRedisSerializer
        SerializationResult jackson2Result = testJackson2(testProduct);
        
        log.info("Generic Jackson - 크기: {}bytes, @class 포함: {}", 
                genericResult.getSize(), genericResult.hasClassField());
        log.info("Jackson2 - 크기: {}bytes, 타입 안전: {}", 
                jackson2Result.getSize(), jackson2Result.isTypeSafe());
    }
    
    @Test
    void 패키지_독립성_확인() {
        // @class 필드 없이 잘 동작하는지 테스트
        ProductDTO originalProduct = createTestProduct();
        
        Jackson2JsonRedisSerializer<ProductDTO> serializer = 
            new Jackson2JsonRedisSerializer<>(ProductDTO.class);
        
        byte[] serialized = serializer.serialize(originalProduct);
        String jsonString = new String(serialized, StandardCharsets.UTF_8);
        
        // @class 필드가 없는지 확인
        assertThat(jsonString).doesNotContain("@class");
        
        // 깔끔한 JSON 확인
        log.info("저장된 JSON: {}", jsonString);
        
        // 역직렬화 성공 확인
        ProductDTO deserialized = serializer.deserialize(serialized);
        assertThat(deserialized).isEqualTo(originalProduct);
    }
    
    @Test 
    void 타입_안전성_확인() {
        // 컴파일 타임에 타입 체크가 되는지 확인
        RedisTemplate<String, ProductDTO> productTemplate = new RedisTemplate<>();
        productTemplate.setValueSerializer(new Jackson2JsonRedisSerializer<>(ProductDTO.class));
        
        RedisTemplate<String, UserDTO> userTemplate = new RedisTemplate<>();
        userTemplate.setValueSerializer(new Jackson2JsonRedisSerializer<>(UserDTO.class));
        
        // 잘못된 타입 할당 시 컴파일 에러 발생
        // productTemplate.opsForValue().set("key", new UserDTO()); // 컴파일 에러!
        
        log.info("타입 안전성 OK");
    }
    
    private ProductDTO createTestProduct() {
        return ProductDTO.builder()
                .productId("P001")
                .name("테스트 상품")
                .description("테스트용 상품입니다")
                .price(BigDecimal.valueOf(10000))
                .category("테스트")
                .stockCount(100)
                .build();
    }
}
```

## 실제 적용 후 느낀 점

### 장점들
- **타입 안전성**: 컴파일 타임에 타입 체크가 되니까 실수가 줄어듦
- **디버깅 편의성**: JSON 형태로 저장되어서 Redis CLI로 직접 확인 가능
- **패키지 독립성**: @class 필드가 없어서 서비스 간 패키지 구조에 의존하지 않음
- **성능**: @class 메타데이터가 없어서 용량이 줄어듦

### 단점들
- **설정 번거로움**: DTO 타입마다 RedisTemplate을 만들어야 함
- **초기 비용**: 처음 설정할 때 코드가 좀 많아짐

### 결론

설정 코드가 조금 늘어나더라도 안정성과 유지보수성을 생각하면 충분히 가치 있는 선택이었다고 생각합니다. 특히 멀티모듈 환경에서는 더욱 그렇네요.

다음 글에서는 실제로 구현한 멀티레벨 캐시 아키텍처에 대해 다뤄보겠습니다.

---

*실제 프로젝트에서 적용한 경험을 바탕으로 작성했습니다.*