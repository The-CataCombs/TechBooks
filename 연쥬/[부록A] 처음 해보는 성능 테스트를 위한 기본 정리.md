# 🚀 부록A 처음 해보는 성능 테스트를 위한 기본 정리

## 🥅 책 읽기 전

- 성능 테스트를 제대로 해본 적이 없습니다. 이번 장에서 미리 공부해두고 필요할 때 사용해봐야겠어요.

## 📚 책 내용 정리

### 성능 테스트란?

시스템이 특정 조건 하에서 얼마나 빠르게, 안정적으로, 그리고 확장 가능하게 작동하는지를 측정하는 테스트

### 성능 테스트의 목적

- **병목 지점 식별**: 시스템의 성능 제약 요소 발견
- **용량 계획**: 필요한 하드웨어 리소스 산정
- **안정성 검증**: 높은 부하에서도 시스템이 안정적으로 동작하는지 확인
- **사용자 경험 보장**: 실제 사용자가 경험할 응답 시간 검증

### 성능 테스트 종류

#### 1. 부하(Load) 테스트

**목적**: 예상되는 정상적인 부하에서 시스템 성능 측정

```
시나리오 예시:
- 평상시 동시 사용자 수: 1,000명
- 테스트 지속 시간: 30분
- 목표: 평균 응답시간 < 2초, 에러율 < 1%
```

**특징**:

- 일반적인 운영 조건에서의 성능 확인
- 기대되는 사용자 부하량으로 테스트
- SLA(Service Level Agreement) 충족 여부 확인

#### 2. 스트레스(Stress) 테스트

**목적**: 시스템의 한계점 찾기 및 장애 상황에서의 동작 확인

```
시나리오 예시:
- 점진적으로 부하 증가: 100 → 500 → 1,000 → 2,000 → 5,000명
- 시스템이 실패할 때까지 부하 증가
- 실패 후 복구 능력 테스트
```

**특징**:

- 정상 부하를 초과하는 극한 상황 테스트
- 시스템의 최대 처리 용량 확인
- 장애 발생 시점과 복구 능력 측정

#### 3. 지속 부하(Soak) 테스트

**목적**: 장시간 부하 상태에서 시스템 안정성 확인

```
시나리오 예시:
- 일정한 부하 유지: 800명 동시 사용자
- 테스트 기간: 12시간 ~ 24시간
- 메모리 누수, 성능 저하 모니터링
```

**특징**:

- 내구성(Endurance) 테스트라고도 함
- 메모리 누수, 리소스 고갈 등 장기간 운영 시 발생하는 문제 발견
- 점진적 성능 저하(Performance Degradation) 확인

#### 4. 스파이크(Spike) 테스트

**목적**: 급작스러운 부하 증가에 대한 시스템 반응 확인

```
시나리오 예시:
- 기본 부하: 100명
- 갑작스런 증가: 100명 → 1,000명 (10초 내)
- 스파이크 지속: 5분
- 다시 감소: 1,000명 → 100명
```

**특징**:

- 이벤트, 할인, 뉴스 등으로 인한 급작스러운 트래픽 증가 상황 모방
- 오토 스케일링 기능 테스트
- 부하 급증 시 시스템 안정성 확인

### 포화점과 버클존

#### 성능 곡선의 이해

```
처리량(TPS)
    ↑
    |     포화점
    |    /‾‾‾\
    |   /     \
    |  /       \    버클존
    | /         \  /
    |/           \/
    |________________→ 동시 사용자 수
    정상 구간   한계점  과부하 구간
```

#### 포화점 (Saturation Point)

- **정의**: 시스템이 감당할 수 있는 성능의 한계 지점
- **특징**: 이 지점까지는 부하 증가에 따라 처리량도 비례적으로 증가
- **중요성**: 운영 시 이 지점을 넘지 않도록 용량 계획 수립

#### 버클존 (Buckle Zone)

- **정의**: 포화점을 지나 성능이 급격히 저하되기 시작하는 구간
- **현상**:
  - 응답 시간 급격히 증가
  - 처리량 감소
  - 에러율 급증
  - 시스템 불안정

### 주요 측정 지표

#### 1. 응답 시간 (Response Time)

##### 측정 항목

- **평균 응답시간**: 전체 요청의 평균 처리 시간
- **최대 응답시간**: 가장 오래 걸린 요청의 처리 시간
- **최소 응답시간**: 가장 빠른 요청의 처리 시간
- **중앙값(Median)**: 응답시간을 정렬했을 때 중간값
- **백분위수**: 99%, 95%, 90% 백분위 응답시간

##### 백분위수의 중요성

```
예시: 1000개 요청의 응답시간 분석
- 평균: 500ms
- 95% 백분위: 800ms → 95%의 사용자가 800ms 이내에 응답 받음
- 99% 백분위: 1500ms → 99%의 사용자가 1500ms 이내에 응답 받음

평균만으로는 일부 사용자의 나쁜 경험을 놓칠 수 있음!
```

#### 2. 처리량 (Throughput)

##### 측정 단위

- **TPS (Transaction Per Second)**: 초당 처리 트랜잭션 수
- **RPS (Request Per Second)**: 초당 처리 요청 수
- **동시 사용자 수**: 동시에 시스템을 사용하는 사용자 수

##### 실무 목표 설정 예시

```
전자상거래 사이트 목표:
- 일반 시간: 1,000 TPS
- 피크 시간: 3,000 TPS
- 특별 이벤트: 5,000 TPS
```

#### 3. 에러율 (Error Rate)

##### 계산 방법

```
에러율 = (실패한 요청 수 / 전체 요청 수) × 100

예시:
- 전체 요청: 10,000개
- 실패 요청: 50개
- 에러율: (50 / 10,000) × 100 = 0.5%
```

##### 허용 가능한 에러율

- **일반적인 웹 서비스**: < 1%
- **금융 서비스**: < 0.1%
- **의료 시스템**: < 0.01%

#### 4. 시스템 리소스 사용률

##### CPU 사용률

```bash
# Linux에서 CPU 사용률 확인
top
htop
# 또는
sar -u 1 10  # 1초 간격으로 10회 측정
```

##### 메모리 사용률

```bash
# 메모리 사용률 확인
free -m
# 또는
ps aux --sort=-%mem | head
```

##### 적정 사용률 가이드

- **CPU**: 70% 이하 (스파이크 대비 여유분 필요)
- **메모리**: 80% 이하
- **디스크 I/O**: 90% 이하

### 성능 테스트 설계

#### 1. 시스템의 트래픽 패턴 분석

##### 실제 트래픽 패턴 예시

```
전자상거래 사이트 일일 트래픽 패턴:
시간대    | 사용자 비율 | 예상 동시 사용자
06:00-09:00 |    5%     |      500명
09:00-12:00 |   15%     |    1,500명
12:00-14:00 |   25%     |    2,500명  ← 점심시간 피크
14:00-18:00 |   20%     |    2,000명
18:00-22:00 |   30%     |    3,000명  ← 저녁시간 피크
22:00-06:00 |    5%     |      500명
```

#### 2. 동시 요청 사용자 수 / 트래픽 규모 산정

##### 계산 방법

```
공식: 동시 사용자 수 = (시간당 페이지뷰 × 평균 세션 시간) / 3600초

예시:
- 시간당 페이지뷰: 100,000
- 평균 세션 시간: 300초 (5분)
- 동시 사용자 수: (100,000 × 300) / 3600 ≈ 8,333명
```

#### 3. 기능별 요청 비율

##### 웹 애플리케이션 예시

```
기능               | 비율  | 상대적 부하
메인 페이지         | 30%   | 낮음
상품 목록/검색      | 25%   | 중간
상품 상세          | 20%   | 중간
장바구니           | 15%   | 높음 (DB 쓰기)
주문/결제          |  8%   | 매우 높음
로그인/회원가입     |  2%   | 중간
```

#### 4. 데이터 크기 고려

##### 테스트 데이터 설계

```java
// 실제 운영 데이터와 유사한 크기로 설계
@Entity
public class Product {
    private String name;        // 평균 20자
    private String description; // 평균 500자
    private List<String> images; // 5개 이미지 URL
    private List<Review> reviews; // 평균 50개 리뷰
}

테스트 DB 크기:
- 상품: 100만개
- 사용자: 50만명
- 주문: 1000만건
- 리뷰: 500만개
```

#### 5. 워밍업 (Warm-up) 설정

##### 워밍업이 필요한 이유

- JVM 최적화 (Just-In-Time 컴파일)
- DB 연결 풀 초기화
- 캐시 데이터 로딩
- CDN 캐시 워밍업

##### 워밍업 시나리오

```
1단계: 10명 동시 사용자로 5분간 실행
2단계: 50명 동시 사용자로 5분간 실행
3단계: 100명 동시 사용자로 5분간 실행
4단계: 실제 테스트 시작
```

#### 6. 적절한 목표치 설정

##### SMART 원칙 적용

```
Specific (구체적):
- "빠르게" ❌ → "평균 응답시간 2초 이하" ✅

Measurable (측정 가능):
- "많은 사용자" ❌ → "동시 사용자 1000명" ✅

Achievable (달성 가능):
- 현재 하드웨어로 실현 가능한 수준

Relevant (관련성):
- 비즈니스 요구사항과 연결

Time-bound (시간 제한):
- "언젠가" ❌ → "2024년 Q1까지" ✅
```

### 성능 테스트 도구

#### 1. nGrinder

##### 특징

- **언어**: Java/Groovy
- **장점**:
  - 웹 기반 관리 인터페이스
  - 분산 테스트 지원
  - 실시간 모니터링
- **단점**:
  - 설치/설정 복잡
  - 리소스 사용량 높음

##### 사용 예시

```groovy
@Test
public void test() {
    HTTPRequest request = new HTTPRequest()

    // 로그인
    NVPair[] loginData = [
        new NVPair("username", "testuser"),
        new NVPair("password", "password123")
    ]
    HTTPResponse loginResponse = request.POST("http://localhost:8080/login", loginData)

    if (loginResponse.statusCode == 200) {
        // 상품 목록 조회
        HTTPResponse response = request.GET("http://localhost:8080/api/products")

        if (response.statusCode == 200) {
            grinder.statistics.forLastTest.success = 1
        } else {
            grinder.statistics.forLastTest.success = 0
        }
    }
}
```

#### 2. k6

##### 특징

- **언어**: JavaScript
- **장점**:
  - 설치 간단
  - 클라우드 서비스 지원
  - 개발자 친화적
- **단점**:
  - 상대적으로 새로운 도구
  - 한국어 자료 부족

##### 사용 예시

```javascript
import http from "k6/http";
import { check, sleep } from "k6";

export let options = {
  stages: [
    { duration: "2m", target: 100 }, // 2분간 100명까지 증가
    { duration: "5m", target: 100 }, // 5분간 100명 유지
    { duration: "2m", target: 200 }, // 2분간 200명까지 증가
    { duration: "5m", target: 200 }, // 5분간 200명 유지
    { duration: "2m", target: 0 }, // 2분간 0명까지 감소
  ],
};

export default function () {
  // 로그인
  let loginResponse = http.post("http://localhost:8080/login", {
    username: "testuser",
    password: "password123",
  });

  check(loginResponse, {
    "로그인 성공": (r) => r.status === 200,
  });

  if (loginResponse.status === 200) {
    // 상품 목록 조회
    let response = http.get("http://localhost:8080/api/products");

    check(response, {
      "상품 목록 조회 성공": (r) => r.status === 200,
      "응답시간 2초 이하": (r) => r.timings.duration < 2000,
    });
  }

  sleep(1);
}
```

#### 3. Gatling

##### 특징

- **언어**: Scala
- **장점**:
  - 높은 성능
  - 아름다운 HTML 리포트
  - 비동기 아키텍처
- **단점**:
  - Scala 학습 필요
  - 설정 복잡

##### 사용 예시

```scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class OrderSimulation extends Simulation {

  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")

  val scn = scenario("주문 시나리오")
    .exec(
      http("로그인")
        .post("/login")
        .formParam("username", "testuser")
        .formParam("password", "password123")
        .check(status.is(200))
    )
    .pause(1)
    .exec(
      http("상품 목록")
        .get("/api/products")
        .check(status.is(200))
        .check(responseTimeInMillis.lt(2000))
    )
    .pause(2)
    .exec(
      http("주문 생성")
        .post("/api/orders")
        .body(StringBody("""{"productId": 1, "quantity": 2}"""))
        .asJson
        .check(status.is(201))
    )

  setUp(
    scn.inject(
      atOnceUsers(10),
      rampUsers(100) during (5 minutes)
    )
  ).protocols(httpProtocol)
}
```

#### 4. JMeter

##### 특징

- **언어**: GUI 기반 (XML 설정)
- **장점**:
  - GUI 인터페이스
  - 풍부한 플러그인
  - 다양한 프로토콜 지원
- **단점**:
  - GUI로 인한 성능 제약
  - 복잡한 시나리오 작성 어려움

#### 도구 선택 가이드

| 기준            | nGrinder | k6            | Gatling     | JMeter |
| --------------- | -------- | ------------- | ----------- | ------ |
| **학습 난이도** | 중간     | 쉬움          | 어려움      | 쉬움   |
| **성능**        | 보통     | 높음          | 매우 높음   | 낮음   |
| **확장성**      | 좋음     | 매우 좋음     | 좋음        | 보통   |
| **리포트**      | 좋음     | 좋음          | 매우 좋음   | 보통   |
| **추천 상황**   | Java 팀  | 클라우드 환경 | 고성능 필요 | 입문자 |

### 실행 시 주의사항

#### 1. 테스트 환경 분리

❌ 잘못된 예시

```
동일한 서버에서 실행:
┌─────────────────────────────┐
│  테스트 서버 (CPU: 4코어)       │
│  ├── 애플리케이션 (2코어)        │
│  └── 부하 생성기 (2코어)        │  ← 리소스 경합 발생!
└─────────────────────────────┘
```

✅ 올바른 예시

```
분리된 환경:
┌─────────────────┐    ┌─────────────────┐
│ 부하 생성기 서버    │ => │ 테스트 대상 서버    │
│ (nGrinder Agent)│    │ (Spring Boot)   │
└─────────────────┘    └─────────────────┘
```

#### 2. 서버 설정 최적화

##### 운영 환경과 동일한 설정 적용

```bash
# /etc/security/limits.conf
# 파일 디스크립터 한도 증가
* soft nofile 65536
* hard nofile 65536

# /etc/sysctl.conf
# TCP 연결 관련 설정
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_fin_timeout = 30
```

##### Spring Boot 설정 최적화

```yaml
# application.yml
server:
  tomcat:
    threads:
      max: 200 # 최대 스레드 수
      min-spare: 10 # 최소 스레드 수
    max-connections: 8192 # 최대 연결 수
    accept-count: 100 # 대기 큐 크기

spring:
  datasource:
    hikari:
      maximum-pool-size: 50 # DB 커넥션 풀 크기
      minimum-idle: 10
      connection-timeout: 30000
      idle-timeout: 600000
```

#### 3. 네트워크 환경 고려

❌ 운영 네트워크에서 테스트

```
문제점:
- 실제 사용자에게 영향
- 네트워크 대역폭 포화
- 방화벽/보안 장비 영향
```

✅ 격리된 테스트 네트워크

```
테스트 전용 네트워크:
- 운영 트래픽과 분리
- 충분한 대역폭 확보
- 동일한 네트워크 토폴로지
```

#### 4. 외부 서비스 Mock 처리

문제 상황

```java
// ❌ 실제 외부 API 호출
@Service
public class PaymentService {
    public PaymentResult processPayment(PaymentRequest request) {
        // 실제 PG사 API 호출 - 비용 발생, 테스트 데이터 오염
        return pgApiClient.pay(request);
    }
}
```

해결책

```java
// ✅ Mock 서비스 사용
@Service
@Profile("test")
public class MockPaymentService implements PaymentService {

    public PaymentResult processPayment(PaymentRequest request) {
        // 실제 응답과 유사한 지연시간 시뮬레이션
        try {
            Thread.sleep(500); // 평균 500ms 지연
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // 성공률 95% 시뮬레이션
        if (Math.random() < 0.95) {
            return PaymentResult.success(request.getAmount());
        } else {
            return PaymentResult.failure("결제 실패");
        }
    }
}

// WireMock 사용 예시
@Component
public class ExternalApiMockServer {

    @PostConstruct
    public void startMockServer() {
        WireMockServer wireMockServer = new WireMockServer(8089);
        wireMockServer.start();

        // 외부 API Mock 설정
        wireMockServer.stubFor(post(urlEqualTo("/api/payment"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("{\"status\":\"success\",\"transactionId\":\"12345\"}")
                .withFixedDelay(500))); // 500ms 지연 시뮬레이션
    }
}
```

#### 5. 데이터 일관성 유지

테스트 데이터 관리

```java
@Component
public class TestDataManager {

    @EventListener
    public void resetTestData(PerformanceTestStartEvent event) {
        // 테스트 시작 전 데이터 초기화
        cleanupTestData();
        setupInitialData();
    }

    private void setupInitialData() {
        // 100만개 상품 데이터 생성
        for (int i = 1; i <= 1_000_000; i++) {
            Product product = Product.builder()
                .id((long) i)
                .name("테스트 상품 " + i)
                .price(BigDecimal.valueOf(10000 + (i % 90000)))
                .stock(100)
                .build();
            productRepository.save(product);
        }

        // 테스트 사용자 계정 생성
        for (int i = 1; i <= 10000; i++) {
            User user = User.builder()
                .username("testuser" + i)
                .email("test" + i + "@example.com")
                .build();
            userRepository.save(user);
        }
    }
}
```

## 더 찾아본 것

### 1. SLA (Service Level Agreement) 🤝

#### SLA란?

**서비스 수준 협약** - 서비스 제공자와 고객 간의 성능 약속

#### 쉬운 비유

```
🍕 피자 배달의 SLA
"30분 내 배달, 안 지키면 무료!"

📱 카카오톡의 SLA
"메시지 전송 성공률 99.9% 보장"
```

#### 일반적인 SLA 지표

1. 가용성 (Availability)

```
99.9% = 1년에 8.76시간 장애 허용
99.99% = 1년에 52.56분 장애 허용
99.999% = 1년에 5.26분 장애 허용
```

2. 응답시간 (Response Time)

```
웹 페이지: "95%의 요청이 2초 이내 응답"
API 호출: "평균 응답시간 500ms 이하"
검색 기능: "99%의 검색이 1초 이내 완료"
```

3. 처리량 (Throughput)

```
"최소 1,000 TPS 처리 가능"
"동시 사용자 10,000명 지원"
```

#### 실무 SLA 예시

```
🛒 쇼핑몰 SLA
- 서비스 가용성: 99.9% (월 43분 장애 허용)
- 메인 페이지 로딩: 95%가 2초 이내
- 상품 검색: 평균 1초 이내
- 주문 처리: 99%가 5초 이내
- 에러율: 1% 이하

💰 위반 시 페널티
- 가용성 99% 미달 시: 월 서비스 요금 10% 환불
- 응답시간 초과 시: 성능 개선 보고서 제출
```

### 2. 워밍업(Warm-up)이 필요한 이유 🔥

#### 워밍업이란?

성능 테스트 전에 시스템을 미리 "데워주는" 과정

#### 🚗 자동차 비유

```
겨울 아침 자동차:
1. 시동 직후: 엔진이 차가워서 성능 안 좋음
2. 몇 분 후: 엔진이 데워져서 본래 성능 발휘

애플리케이션도 마찬가지!
```

#### 워밍업이 필요한 이유

1. JVM 최적화 ☕

```java
// 처음 실행 시
public int calculatePrice(int price) {
    return price * 1.1; // 인터프리터 모드 (느림)
}

// 여러 번 실행 후 (JIT 컴파일 후)
public int calculatePrice(int price) {
    return price * 1.1; // 컴파일된 기계어 (빠름)
}
```

**결과**: 처음보다 10-100배 빨라질 수 있음!

2. 커넥션 풀 초기화 🔌

```yaml
# 데이터베이스 커넥션 풀 설정
spring:
  datasource:
    hikari:
      maximum-pool-size: 20 # 최대 20개 연결

처음: 연결 1개 → 요청 시마다 새 연결 생성 (느림)
워밍업 후: 연결 20개 → 즉시 사용 가능 (빠름)
```

3. 캐시 데이터 로딩 💾

```java
@Cacheable("products")
public List<Product> getPopularProducts() {
    return productRepository.findTop10();
}

처음: DB에서 조회 (500ms)
워밍업 후: 캐시에서 조회 (5ms)
```

#### 워밍업 방법

단계별 워밍업 시나리오

```
1단계: 10명 사용자 × 5분
└── 목적: 기본 시스템 초기화

2단계: 50명 사용자 × 5분
└── 목적: 커넥션 풀, 스레드 풀 완성

3단계: 100명 사용자 × 5분
└── 목적: JIT 컴파일, 캐시 워밍업

4단계: 실제 테스트 시작! 🚀
```

워밍업 없을 때 vs 있을 때

```
워밍업 없음:
시간  |  0분   1분   2분   3분   4분   5분
응답시간| 5초   3초   2초   1초   1초   1초
└── 처음 5분간 성능이 점진적으로 개선

워밍업 있음:
시간  |  0분   1분   2분   3분   4분   5분
응답시간| 1초   1초   1초   1초   1초   1초
└── 처음부터 일정한 성능
```

### 3. 동시 사용자 수 계산하고 설정하기 👥

#### 기본 공식

```
동시 사용자 수 = (시간당 방문자 수 × 평균 머무는 시간) ÷ 3600초
```

#### 🍕 피자집 비유

```
피자집에서 동시에 식사하는 손님 수는?

1시간에 120명 방문 (시간당 방문자)
평균 30분 식사 (1800초)

동시 식사 손님 = (120 × 1800) ÷ 3600 = 60명
```

#### 실제 계산 예시

📱 쇼핑몰 앱

```
데이터 수집:
- 시간당 페이지뷰: 100,000번
- 평균 세션 시간: 5분 (300초)
- 사용자당 평균 페이지뷰: 10번

계산:
시간당 사용자 수 = 100,000 ÷ 10 = 10,000명
동시 사용자 수 = (10,000 × 300) ÷ 3600 = 833명
```

#### 시간대별 패턴 고려

📊 일반적인 웹사이트 패턴

```
시간대     | 비율  | 동시 사용자 (기준: 1000명)
06:00-09:00|  5%  |   50명
09:00-12:00| 15%  |  150명
12:00-14:00| 25%  |  250명  ← 점심시간 피크
14:00-18:00| 20%  |  200명
18:00-22:00| 30%  |  300명  ← 저녁시간 피크
22:00-06:00|  5%  |   50명
```

#### 안전 마진 적용

🛡️ 여유분 계산

```
계산된 동시 사용자: 1,000명

안전 마진 적용:
- 일반적: 1,000 × 1.5 = 1,500명
- 보수적: 1,000 × 2.0 = 2,000명
- 이벤트 대비: 1,000 × 3.0 = 3,000명

이유: 예상치 못한 상황 대비
- 갑작스러운 인기 상품 등장
- SNS에서 화제가 되는 경우
- 경쟁사 장애로 트래픽 몰림
```

#### 실무 설정 방법

🎯 목표 설정

```java
// k6 테스트 시나리오
export let options = {
  stages: [
    { duration: '2m', target: 100 },   // 2분간 100명까지 증가
    { duration: '5m', target: 100 },   // 5분간 100명 유지
    { duration: '2m', target: 500 },   // 2분간 500명까지 증가
    { duration: '10m', target: 500 },  // 10분간 500명 유지
    { duration: '3m', target: 1000 },  // 3분간 1000명까지 증가
    { duration: '10m', target: 1000 }, // 10분간 1000명 유지
    { duration: '2m', target: 0 },     // 2분간 0명까지 감소
  ],
};
```

## 👀 질문

- 운영 환경과 동일하게 테스트하고 싶어도, 데이터의 품질이 다르더라구요.. 다들 어떤식으로 테스트 하시는지 궁금합니다.

## 🎀 소감

- 실습을 해보진 않아서 엄청 와닿진 않는 것 같습니다.
