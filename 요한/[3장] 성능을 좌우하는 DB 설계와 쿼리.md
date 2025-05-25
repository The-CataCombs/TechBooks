# 📚 개발 도서 스터디 템플릿

## 🧠 1. 책을 읽기 전에
- **기대하는 점**: 이 책을 선택한 이유, 기대하고 있는 내용
- **알고 싶은 개념 / 주제**: 미리 궁금했던 개념이나 기술

---

## 📂 2. 내용 정리 (자신의 언어로 요약)


### 📌 Chapter [2]: [인덱스는 필요한 만큼만 만들기] (60p)
#### 📚 복합 인덱스의 중복성 이해

##### 🔍 **왜 (a,b) 인덱스가 불필요한가?**

###### **인덱스의 Left-Most Prefix 특성**
복합 인덱스 `(a,b,c)`는 실제로 다음과 같은 검색 패턴을 모두 지원합니다:

```sql
-- ✅ 지원되는 패턴들
WHERE a = ?                    -- (a) 검색 가능
WHERE a = ? AND b = ?          -- (a,b) 검색 가능  
WHERE a = ? AND b = ? AND c = ? -- (a,b,c) 검색 가능

-- ❌ 지원되지 않는 패턴들
WHERE b = ?                    -- 인덱스 활용 불가
WHERE c = ?                    -- 인덱스 활용 불가
WHERE b = ? AND c = ?          -- 인덱스 활용 불가
```

###### **따라서 중복되는 이유**
- `(a,b,c)` 인덱스가 있으면 `(a,b)` 검색도 자동으로 커버됨
- 별도로 `(a,b)` 인덱스를 만들 필요가 없음
- 오히려 **저장공간 낭비 + 쓰기 성능 저하**만 발생

##### 💡 **핵심 원칙**

###### **필요한 인덱스 조합**
```sql
-- 이런 쿼리들이 있다면
SELECT * FROM users WHERE dept = 'IT';                    -- (a)
SELECT * FROM users WHERE dept = 'IT' AND age = 30;       -- (a,b)  
SELECT * FROM users WHERE dept = 'IT' AND age = 30 AND salary > 5000; -- (a,b,c)

-- 인덱스는 하나만!
CREATE INDEX idx_users ON users(dept, age, salary);  -- (a,b,c)
```

###### **불필요한 중복 인덱스**
```sql
-- ❌ 이렇게 하면 안됨
CREATE INDEX idx_users_dept ON users(dept);           -- 불필요!
CREATE INDEX idx_users_dept_age ON users(dept, age);  -- 불필요!
CREATE INDEX idx_users_all ON users(dept, age, salary);
```

**✅ 해야 할 것**:
- 가장 긴 복합 인덱스 하나만 생성

### 📌 Chapter [2]: [미리 집계하기] (63p)
#### 📊 통계 테이블을 이용한 성능 최적화

##### 🔍 **통계 테이블이 필요한 이유**

###### **문제 상황**
```sql
-- 이런 집계 쿼리가 매번 실행된다면?
SELECT 
    category,
    COUNT(*) as order_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount
FROM orders 
WHERE created_at >= '2024-01-01'
GROUP BY category;

-- 😱 문제점: 수백만 건 데이터를 매번 집계 → 느림!
```

###### **해결책: 통계 테이블**
미리 집계된 결과를 별도 테이블에 저장해두고, 빠르게 조회만 하자!

##### 🎯 **통계 테이블 설계**

###### **일별 주문 통계 테이블**
```sql
CREATE TABLE daily_order_stats (
    stat_date DATE,
    category VARCHAR(50),
    order_count INT,
    total_amount DECIMAL(15,2),
    avg_amount DECIMAL(10,2),
    PRIMARY KEY (stat_date, category)
);
```

###### **장점**
- ⚡ **빠른 조회**: 집계 연산 없이 바로 결과 반환
- 📈 **확장성**: 데이터가 늘어나도 조회 성능 일정
- 💾 **리소스 절약**: CPU 사용량 대폭 감소

###### **단점**  
- 🔄 **동기화**: 실시간 데이터와 차이 발생 가능
- 💽 **저장공간**: 추가 테이블 필요
- 🧩 **복잡성**: 집계 로직 별도 관리

##### 💻 **비동기 집계 처리 코드**

```java
@Component
public class OrderStatsService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private OrderStatsRepository statsRepository;
    
    // 주문 생성 시 비동기로 통계 업데이트
    @EventListener
    @Async
    public void handleOrderCreated(OrderCreatedEvent event) {
        updateDailyStats(event.getOrder());
    }
    
    private void updateDailyStats(Order order) {
        LocalDate today = order.getCreatedAt().toLocalDate();
        String category = order.getCategory();
        
        // 기존 통계 조회 또는 새로 생성
        DailyOrderStats stats = statsRepository
            .findByDateAndCategory(today, category)
            .orElse(new DailyOrderStats(today, category));
        
        // 통계 업데이트
        stats.incrementOrderCount();
        stats.addAmount(order.getAmount());
        stats.recalculateAverage();
        
        statsRepository.save(stats);
    }
    
    // 빠른 통계 조회
    public List<OrderStatsDto> getDailyStats(LocalDate date) {
        return statsRepository.findByDate(date);
        // 🚀 집계 연산 없이 바로 반환!
    }
}
```

```java
// 이벤트 발행
@Service  
public class OrderService {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public void createOrder(OrderDto orderDto) {
        Order order = orderRepository.save(new Order(orderDto));
        
        // 비동기 통계 업데이트 트리거
        eventPublisher.publishEvent(new OrderCreatedEvent(order));
    }
}
```

###### **집계 주기 선택**
- **실시간**: 매 트랜잭션마다 (부하 큼)
- **배치**: 매시간/매일 (지연 있음, 안정적)
- **하이브리드**: 중요 지표는 실시간 + 나머지는 배치

###### **데이터 정합성 보장**
```java
@Scheduled(cron = "0 0 1 * * ?") // 매일 새벽 1시
public void recalculateStats() {
    // 전날 통계 재계산으로 정합성 보장
    LocalDate yesterday = LocalDate.now().minusDays(1);
    rebuildStatsForDate(yesterday);
}
```

**💡 핵심**: 집계 테이블을 통해 조회 성능을 끌어 올릴 수 있고 전략 선택은 해당 서비스 성격에 맞게 설정 필요

### 📌 Chapter [2]: [배치 쿼리 실행 시간 증가] (79p)
#### 📊 배치 처리 파트

##### 🔍 **크기가 너무 크면 나눠서 처리하자**

```java
@Service
public class AccessLogService {
    
    public List<AccessLog> getAllAccessLogsByMonth(LocalDate startDate, LocalDate endDate) {
        List<AccessLog> allLogs = new ArrayList<>();
        LocalDateTime cursor = startDate.atStartOfDay();
        LocalDateTime end = endDate.atTime(23, 59, 59);
        
        while (true) {
            List<AccessLog> batch = repository.findByDateRange(cursor, end, 100);
            
            if (batch.isEmpty()) break;
            
            allLogs.addAll(batch);
            cursor = batch.get(batch.size() - 1).getAccessDate().plusNanos(1);
        }
        
        return allLogs;
    }
}
```

```sql
-- Repository 쿼리
SELECT * FROM access_log 
WHERE access_date BETWEEN ? AND ? 
ORDER BY access_date ASC 
LIMIT 100
```


**💡 핵심**: 개발에 정답은 없다...


### 📌 Chapter [번호]: [챕터 제목]

- **핵심 요약**:
    - 요점 1: 인덱스 설정에는 정
    - 요점 2:

- **인상 깊은 내용 **:
  > "책에서 인용하고 싶은 내용을 적어보세요."

- **나의 해석 / 생각**:
    - 이 개념을 어디에 적용할 수 있을까?
    - 내가 과거에 겪었던 문제와 연결되는가?



---

## 💬 3. 이야기하고 싶은 질문 / 포인트

> 읽으며 생긴 **의문**, **확인하고 싶은 개념**, **비판하고 싶은 주장** 등을 적습니다.

- ❓ 질문 1: where문에 %단어% 가 들어 있는 경우 풀 스캔을 하게 되고 책에서는 전문 검색 기능을 사용하라는데 전문 검색은 기존 방식과 무슨 차이인건지?(56p)

- ❓ 질문 2: jobqueue에서 w,p,c 이렇게 3개의 status를 가지지만 w인 데이터가 적고 자주 사용하기 때문에 인덱스를 설정하는게 좋다는데 jobqueue 특성상 insert,update가 많아 보이는데 인덱스 설정이 도움이 될지?
다른 방법은 없는지?

- ❓ 확인하고 싶은 개념 1: 주,부 DB에서 복제가 이루이지는 경우 커밋 시점에 일어난다고 하는데 구체적으로 어떠한 과정인지?

- 💭 더 알아보고 싶은 개념: 

---

## 🎯 4. 정리 & 적용 아이디어

- **내가 배운 것 한 줄 요약**:  
  → `이 장을 통해 나는 DB 성능 최적화 기법에 대해 이해했다.`
- **개발에 어떻게 적용해볼까?**
    - 실제 프로젝트나 업무에 적용해볼 수 있는 아이디어

---
