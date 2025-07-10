## 🧪 3장 성능을 좌우하는 DB 설계와 쿼리

### 🧠 책을 읽기 전에

- 사실. 쿼리 성능.. 고려할만한 작업을 해보지 않은 백린이입니담..
  - 예전에 맡은 서비스에서 롱쿼리 이슈가 있어서 강제로 중지시켰던 적이 있었는데, 이번 장을 읽고나서 다음에 그런 일이 또 발생한다면 저도 쿼리튜닝을 할 수 있는 인재가 되기 위해 열심히 읽어보겠습니다.

### **📌 인상 깊은 내용**

#### ✅ 선택도

- 인덱스에서 특정 칼럼의 고유한 값 비율 = 선택도
  - 선택도가 높다 = 해당 칼럼에 고유한 값이 많다 ⇒ 인덱스를 이용한 조회 효율이 높아짐
  - 그러나 항상 그런건 아니라는 점
    - ex) status 칼럼: W(대기), P(처리 중), C(완료)
      - 대부분 C이고 적은 수만 W,P임.
      - 선택도는 낮지만 ‘W’, ‘P’인 거를 조회한다면 인덱스로 사용하기 좋은 칼럼임

#### ✅ COUNT(\*)쿼리

- '전체 개수 세지 않기'와 관련된 내용인데요,
- 가끔 프론트 UI 보면, 페이지네이션에서 전체 개수를 보여주는 로직이 있는 경우가 있음
  - 저자는 성능 문제가 심해질 경우, 전체 개수를 보여주는 부분을 빼는 방안도 건의해보는 게 좋다고 했음
  - 나도 능동적인 사람이 되어야겠다는 생각을 잠깐 해봤어요.

#### ✅ 서비스를 멈추게 한 테이블 변경

- 수천만 건의 데이터가 있는 테이블에서 열거 타입 칼럼을 변경하는 과정에서 문제가 발생
- 열거 타입 컬럼을 변경할 때 온라인 DDL을 사용하지 않았고, 테이블 변경을 위한 복사 과정이 시작됨
- 데이터가 많았기 때문에 십 분이 지나도 테이블 변경은 끝나지 않음
  <br/>
- **문제 상황 분석**

  - 수천만 건 데이터에서 ENUM 변경
    ```sql
    ALTER TABLE users
    MODIFY COLUMN status ENUM('active', 'inactive', 'pending', 'suspended');
    ```
  - 온라인 DDL 미사용으로 인한 문제:
    - MySQL에서 ENUM 변경은 기본적으로 테이블 재구성(rebuild) 필요
    - 전체 테이블을 복사하는 과정이 시작됨

- **실제 발생하는 문제들**

  1. 테이블 락 발생

  ```sql
  -- 변경 중 테이블이 잠김
  SELECT \* FROM users WHERE id = 1; -- 대기 상태
  INSERT INTO users VALUES (...); -- 대기 상태
  UPDATE users SET name = 'test'; -- 대기 상태
  ```

  2. 시스템 리소스 과부하

  ```
  임시 테이블 생성 → 데이터 복사 → 인덱스 재생성
  ↓

  - 디스크 I/O 폭증
  - 메모리 사용량 급증
  - CPU 사용률 100%
  ```

  3. 연쇄 장애

  - DB 응답 없음 → 애플리케이션 타임아웃 → 사용자 서비스 중단

- **온라인 DDL을 사용했다면?**

  - MySQL 8.0+ 온라인 DDL:
    ```sql-- 서비스 중단 없이 변경
    ALTER TABLE users
    MODIFY COLUMN status ENUM('active', 'inactive', 'pending', 'suspended'),
    ALGORITHM=INPLACE, LOCK=NONE;
    ```
  - 장점:
    - 읽기/쓰기 작업 계속 가능
    - 락 최소화
    - 점진적 변경

- **올바른 대응 방법**

  1. 사전 계획

  ```sql
   -- 변경 영향도 확인
   EXPLAIN FORMAT=JSON
   ALTER TABLE users MODIFY COLUMN status ENUM(...);
  ```

  2. 단계별 변경

  ```sql
  -- 1단계: 새 컬럼 추가
  ALTER TABLE users ADD COLUMN new_status ENUM(...);

  -- 2단계: 데이터 마이그레이션 (배치 처리)
  UPDATE users SET new_status = status WHERE id BETWEEN 1 AND 10000;

  -- 3단계: 애플리케이션 코드 변경

  -- 4단계: 기존 컬럼 제거
  ALTER TABLE users DROP COLUMN status;
  ALTER TABLE users CHANGE new_status status ENUM(...);
  ```

  3. 유지보수 창 활용
     - 서비스 이용이 적은 시간대 선택
     - 사전 공지 및 모니터링 준비

#### ✅ postgresql에서는 비슷한 이슈가 없는가?

- PostgreSQL에서도 비슷한 이슈가 있지만, 상황이 조금 다르다고 함

- **PostgreSQL의 ENUM 변경**

  - **기본적으로 더 안전함:**

  ```sql
  -- PostgreSQL에서 ENUM 값 추가 (안전)
  ALTER TYPE status_enum ADD VALUE 'suspended';

  -- 하지만 ENUM 값 제거나 순서 변경은 여전히 위험
  ALTER TYPE status_enum DROP VALUE 'old_status';  -- 불가능!
  ```

- **PostgreSQL에서 문제가 되는 경우들**

  - **1. 컬럼 타입 변경**

  ```sql
  -- 이런 변경은 여전히 테이블 리라이트 발생
  ALTER TABLE users ALTER COLUMN status TYPE text;
  -- 수천만 건에서는 MySQL과 동일한 문제 발생
  ```

  - **2. NOT NULL 제약 조건 추가**

  ```sql
  -- PostgreSQL 11 이전: 전체 테이블 스캔 + 락
  ALTER TABLE users ALTER COLUMN email SET NOT NULL;

  -- PostgreSQL 12+: 더 안전하지만 여전히 주의 필요
  ```

  - **3. 새로운 컬럼에 DEFAULT 값**

  ```sql
  -- PostgreSQL 11 이전: 모든 행 업데이트 (위험)
  ALTER TABLE users ADD COLUMN created_at timestamp DEFAULT now();

  -- PostgreSQL 11+: 메타데이터만 변경 (안전)
  ```

- **PostgreSQL의 장점**

  - **더 스마트한 DDL:**

    ```sql
    -- ENUM 확장은 락 없이 가능
    ALTER TYPE user_status ADD VALUE 'premium';

    -- 컬럼 추가도 더 효율적
    ALTER TABLE users ADD COLUMN phone varchar(20);  -- 즉시 완료
    ```

- **여전히 주의해야 할 PostgreSQL 작업들**

  - **1. 인덱스 생성**

    ```sql
    -- 기본: 테이블 락 발생
    CREATE INDEX idx_user_email ON users(email);

    -- 안전: 온라인 인덱스 생성
    CREATE INDEX CONCURRENTLY idx_user_email ON users(email);
    ```

  - **2. 외래 키 추가**

    ```sql
    -- 위험: 전체 테이블 검증
    ALTER TABLE orders ADD CONSTRAINT fk_user
    FOREIGN KEY (user_id) REFERENCES users(id);

    -- 안전: 단계별 추가
    ALTER TABLE orders ADD CONSTRAINT fk_user
    FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID;
    -- 나중에 VALIDATE CONSTRAINT로 검증
    ```

  - **3. 컬럼 타입 변경**

    ```sql
    -- 여전히 위험
    ALTER TABLE users ALTER COLUMN id TYPE bigint;  -- 전체 테이블 리라이트
    ```

=> 테이블 변경 작업은 신중히 해야겠다..

<br />

### 🔍 추가로 읽어본 글

- [DB 페이지네이션을 최적화하는 여러 방법들](https://taegyunwoo.github.io/tech/Tech_DBPagination?utm_source=chatgpt.com)
  - LIMIT-OFFSET, NO-OFFSET, 커버링 인덱스 비교

<br />

### 💭 더 알아보고 싶은 개념

#### ✅ 커버링 인덱스

- 사실 첨 들어봄. 공부할 만한 자료 있으면 추천해주세요!

#### ✅ 연관관계가 있는 게 좋은가?

- **연관 관계 성능 분석**

  - **FK 제약조건이 있을 때의 성능 비용**

    - INSERT 시 매번 참조 테이블 존재 확인으로 추가 SELECT 쿼리 발생
      ```sql
      INSERT INTO orders (user_id, product_id) VALUES (123, 456);
      -- ↓ 내부적으로 실행됨
      -- SELECT 1 FROM users WHERE id = 123;      -- 존재 확인
      -- SELECT 1 FROM products WHERE id = 456;   -- 존재 확인
      ```
    - DELETE 시 모든 자식 테이블에서 참조 데이터 존재 여부 확인 필요
      ```sql
      DELETE FROM users WHERE id = 123;
      -- ↓ 내부적으로 실행됨
      -- SELECT COUNT(*) FROM orders WHERE user_id = 123;
      -- SELECT COUNT(*) FROM reviews WHERE user_id = 123;
      ```
    - 대용량 배치 작업에서 각 행마다 FK 검증으로 심각한 성능 저하
      ```sql
      -- 100만 건 INSERT 시 각 행마다 FK 검증 = 200만 번의 추가 SELECT
      INSERT INTO order_items SELECT ... FROM temp_data; -- 매우 느림
      ```

  - **FK 제약조건이 있을 때의 성능 이점**
    - 옵티마이저가 FK 정보를 활용해 JOIN 쿼리 최적화 가능
      ```sql
      SELECT u.name, COUNT(o.id)
      FROM users u
      LEFT JOIN orders o ON u.id = o.user_id  -- FK 존재 시 최적화됨
      GROUP BY u.id;
      ```
    - 애플리케이션에서 별도 데이터 검증 로직 불필요 (DB 레벨에서 보장)

- **FK 제약조건이 없을 때의 성능 이점**

  - INSERT/UPDATE/DELETE 작업이 검증 없이 즉시 실행
    ```sql
    INSERT INTO orders (user_id, product_id) VALUES (999999, 888888);
    -- 존재하지 않는 ID여도 바로 삽입됨
    ```
  - 대용량 데이터 이관 및 배치 처리 작업 속도 향상
    ```sql
    LOAD DATA INFILE 'orders.csv' INTO TABLE orders; -- FK 검증 없음
    ```

- **FK 제약조건이 없을 때의 성능 비용**

  - 애플리케이션 레벨에서 매번 데이터 존재 여부 확인 쿼리 필요
    ```java
    public void createOrder(Long userId, Long productId) {
        if (!userExists(userId)) {           // 추가 쿼리
            throw new UserNotFoundException();
        }
        if (!productExists(productId)) {     // 추가 쿼리
            throw new ProductNotFoundException();
        }
        orderRepository.save(new Order(userId, productId));
    }
    ```
  - 고아 데이터 발생으로 인한 복잡한 JOIN 및 EXISTS 조건 추가
    ```sql
    SELECT o.* FROM orders o
    WHERE EXISTS (SELECT 1 FROM users u WHERE u.id = o.user_id)
      AND EXISTS (SELECT 1 FROM products p WHERE p.id = o.product_id);
    ```

- **실제 성능 비교 결과**

  - INSERT 성능: FK 있음 45초 vs FK 없음 12초 (100만 건 기준)
  - 복잡한 JOIN 쿼리: FK 있음 1.2초 vs FK 없음 2.8초

- **사용 권장 상황**

  - FK 사용 권장: OLTP 시스템, 소중규모 트래픽, 복잡한 관계형 데이터, 데이터 정합성이 중요한 경우
  - FK 없이 사용: 대용량 배치 처리, 고성능 요구 시스템, 마이크로서비스 간 느슨한 결합
  - 하이브리드 방식: 핵심 테이블만 FK 사용, 로그성 테이블은 성능 우선으로 FK 제외

  ```sql
  -- 핵심 테이블만 FK 사용
  ALTER TABLE orders ADD CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id);

  -- 로그성 테이블은 FK 없음 (성능 우선)
  CREATE TABLE user_activity_logs (
      user_id INT,  -- FK 없음
      action VARCHAR(50),
      created_at TIMESTAMP
  );
  ```

<br />

### 🤔 질문

- 보통 다들 인덱스 많이 설정하나요? DBA가 하나요?
- 데이터가 몇 건 이상일 때부터 인덱스가 있는 것과 없는 것의 속도 차이가 유의미한지??
  - Claude: 일반적으로 1,000-10,000건부터 인덱스의 효과가 유의미하게 나타나며, 10만건 이상에서는 인덱스가 필수
- 다들 어떠한 상황에서 비정규화 하는지?
  - 해지 고객을 관리하는 요건을 받았음. 기존 테이블에 해지 고객 관련 컬럼들을 추가하려했는데, 사수님이 성능 문제로 테이블 분리하여 따로 관리하는 것을 추천했음
    - 이유: 해지 고객은 몇 고객 안되기 때문. 굳이 매번 다른 테이블 다 스캔해서 찾아내는 것보다 나을 것이다.
  - 그러나 무결성이 깨지지 않도록 관리하기가 어렵긴 했음.
    - 분산 트랜잭션도 적용이 안되어 있어서 더 힘들었지만.. 뭐든 트레이드오프가 있구나 싶었어요.
- 분산 트랜잭션 처리 로직이 들어갈 경우 성능이 얼마나 더 나빠지는지?
  - Claude가 말하길
    - 실제 성능 비교 (TPS 기준)
      - 단일 DB 트랜잭션: 5,000 TPS
      - 2PC (2개 DB): 500 TPS (90% 감소)
      - 2PC (5개 DB): 150 TPS (97% 감소)
      - Saga Pattern: 2,000 TPS (60% 감소, 하지만 최종 일관성)
      - Eventually Consistent: 4,500 TPS (10% 감소)
    - 메모리 사용량 비교
      - 일반 트랜잭션: 100MB (1만 동시 트랜잭션)
      - 2PC: 500MB (상태 정보 + 락 정보)
      - Saga: 800MB (보상 정보 + 상태 머신)
      - Event Sourcing: 1.2GB (이벤트 스트림 + 스냅샷)

---

## 🎯 소감

- 다양한 방법을 기억해두고 활용해봐야겠어요.
- 테이블 변경은 신중히..
- 쿼리 실행계획 보기
