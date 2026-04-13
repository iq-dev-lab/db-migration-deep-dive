# 외래 키 제약 관리

---

## 🎯 핵심 질문

- 외래 키 제약을 추가할 때 모든 행을 검증하는 이유는 무엇인가?
- MSA(Microservice Architecture)에서 외래 키를 제거하는 이유는?
- 외래 키 없이 무결성을 보장하려면 어떻게 해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

외래 키는 데이터 무결성을 보장하는 강력한 도구이지만:
1. 추가 시 모든 기존 행 검증 필요 → 시간 소요
2. 운영 중 데이터 정합성 검증 → 쓰기 성능 저하
3. MSA 환경에서 서비스 간 결합도 증가 → 배포 순서 의존성

언제 외래 키가 필요한지, 언제 제거해야 하는지 알면:
- 성능과 무결성의 균형 유지
- MSA 환경에서 서비스 독립성 보장
- 데이터 일관성 전략 수립

---

## 😱 흔한 실수 (Before — 무분별한 외래 키 추가)

```sql
-- Before: 검증 없이 외래 키 추가
ALTER TABLE orders 
ADD CONSTRAINT fk_orders_users 
FOREIGN KEY (user_id) REFERENCES users(id);
-- 응답: 10분 대기 (1000만 행 검증)
-- 결과: orders 테이블 Lock (읽기만 가능)

-- 문제 상황:
-- ├─ 고아 레코드 존재 (user_id가 users에 없음)
-- │  └─ Error 1452: Cannot add or update a child row
-- │     (기존 위반 데이터 때문에 실패)
-- │
-- ├─ 데이터 정합성 확인 필수 (미리)
-- │  └─ 확인 쿼리 실행 = 시간 낭비
-- │
-- └─ MSA 환경에서 배포 순서 문제
--    ├─ users 마이크로서비스
--    └─ orders 마이크로서비스
--    → orders 배포 시 users DB를 참조하는 문제!

-- Before: 외래 키로 인한 성능 저하
-- DML 발생 시마다 참조 무결성 확인 (오버헤드)
INSERT INTO orders (user_id, amount)
VALUES (100, 1000);
-- 내부 동작:
-- 1. INSERT 유효성 확인
-- 2. users 테이블에서 id=100 존재 확인 (추가 쿼리)
-- 3. 없으면 오류 발생
-- → 모든 INSERT가 2배 느려질 수 있음
```

**결과**: 성능 저하, 데이터 정합성 문제, 배포 복잡도 증가

---

## ✨ 올바른 접근 (After — 선택적 외래 키 관리)

```sql
-- After 1: Monolith 환경 (외래 키 유지)
-- 모든 데이터가 한 DB, 한 어플리케이션에서 관리
ALTER TABLE orders 
ADD CONSTRAINT fk_orders_users 
FOREIGN KEY (user_id) REFERENCES users(id);
-- 배포 가능, 데이터 무결성 자동 보장

-- After 2: MSA 환경 (외래 키 제거)
-- users: 별도 마이크로서비스 DB
-- orders: 별도 마이크로서비스 DB
-- → 외래 키 제약 없음

-- 대신 애플리케이션 계층에서 검증
public class OrderService {
    public Order createOrder(Long userId, BigDecimal amount) {
        // 1. users 마이크로서비스에서 사용자 존재 확인
        User user = userService.getUser(userId);  // REST API 호출
        if (user == null) {
            throw new UserNotFoundException("User not found: " + userId);
        }
        
        // 2. 주문 생성
        Order order = new Order();
        order.setUserId(userId);
        order.setAmount(amount);
        orderRepository.save(order);
        
        return order;
    }
}

-- 또는 배치 작업으로 고아 레코드 주기적 정리
-- (Orphaned record cleanup)
-- 매일 자정에:
-- DELETE FROM orders 
-- WHERE user_id NOT IN (SELECT id FROM users_cache)
-- LIMIT 10000;
```

**결과**: 성능 유지, MSA 서비스 독립성 보장

---

## 🔬 내부 동작 원리

### 1. 외래 키 추가 시 검증 과정

```
ALTER TABLE orders 
ADD CONSTRAINT fk_orders_users 
FOREIGN KEY (user_id) REFERENCES users(id);

Timeline:

T0: SHARED LOCK 획득
    orders 테이블 Lock

T1-T10: 데이터 검증
    ├─ 1단계: orders의 모든 행 스캔 (1000만 건)
    │  └─ user_id 값 추출
    │
    ├─ 2단계: 각 user_id가 users에 존재하는지 확인
    │  └─ SELECT COUNT(*) FROM users WHERE id = ?
    │     (각 고유 user_id마다, 또는 배치 검증)
    │
    ├─ 3단계: 불일치 행 발견
    │  └─ Example: user_id = 999999 (users에 없음)
    │  └─ Error 1452: Cannot add or update a child row
    │
    └─ 4단계: 검증 실패, ROLLBACK
       (제약 추가 취소)

T11: Lock 해제

최악 시나리오:
- 1000만 행 모두 검증
- 각 행의 user_id 조회 (user_id가 인덱스되지 않으면 Full Scan)
- 최악 시간: 수십 분

최선 시나리오:
- 1000만 행 모두 검증 (순차 읽기)
- user_id는 이미 인덱스됨 (빠른 lookup)
- 시간: 5~10분
```

### 2. 외래 키가 있을 때의 DML 오버헤드

```
외래 키 없는 INSERT:

INSERT INTO orders (user_id, amount) VALUES (100, 1000);

실행:
1. 버퍼 풀 확인
2. 페이지 할당
3. 행 삽입
4. 인덱스 업데이트
5. Redo Log 기록
소요 시간: ~1ms

─────────────────────────────────

외래 키 있는 INSERT:

INSERT INTO orders (user_id, amount) VALUES (100, 1000);

실행:
1. 버퍼 풀 확인
2. 페이지 할당
3. 행 삽입
4. 인덱스 업데이트
5. ★ 참조 무결성 확인:
   SELECT 1 FROM users WHERE id = 100 LIMIT 1;
   └─ 별도 쿼리 실행 (users 테이블 Lock)
6. Redo Log 기록
소요 시간: ~5ms (5배)

최악: users 테이블에 인덱스가 없으면
- Full Table Scan 시간 추가
- 시간: ~50ms (50배)
```

### 3. MSA에서 외래 키의 문제

```
Monolith Architecture (외래 키 OK):

┌─────────────────────────────┐
│ 단일 Database              │
├─────────────────────────────┤
│ users 테이블               │
│ orders 테이블              │
│   ├─ user_id (FK)          │
│   └─ → users.id 참조       │
└─────────────────────────────┘

배포:
1. 데이터 마이그레이션 (FK 추가)
2. 애플리케이션 배포
3. 완료 (모두 같은 DB)

─────────────────────────────

Microservice Architecture (외래 키 문제):

┌──────────────────┐      ┌──────────────────┐
│ users DB         │      │ orders DB        │
├──────────────────┤      ├──────────────────┤
│ users 테이블     │      │ orders 테이블    │
│ ├─ id (PK)       │      │ ├─ id (PK)       │
│ └─ name          │      │ ├─ user_id (FK?) │
└──────────────────┘      │ └─ amount        │
     ↑                    └──────────────────┘
     │
     └─ 다른 데이터베이스!

문제 1: 데이터 일관성
┌────────────────┐
│ users DB       │
│ (마스터)       │
│                │
│ id=100 DELETE  │
└────────────────┘
         │
         └─ 복제 지연 (수초)
         
┌────────────────┐
│ orders DB      │
│ (복제본)       │
│                │
│ id=100 아직    │
│ 존재함         │
└────────────────┘

orders에 FK 제약이 있으면:
T1: users에서 id=100 DELETE
T2: 복제 지연 (3초)
T3: orders에 FK INSERT 시도
    → users에서 아직 id=100 존재 (OK)
T4: 복제 완료 (id=100 삭제됨)
T5: 이제 데이터 불일치 (고아 레코드)

문제 2: 배포 순서 의존성
┌──────────────────────────────┐
│ services 배포 순서            │
├──────────────────────────────┤
│ 1. users 마이크로서비스 배포  │
│    (DB 마이그레이션: 컬럼 추가) │
│    ↓                          │
│ 2. orders 마이크로서비스 배포 │
│    (FK 추가)                  │
│    └─ 만약 1번이 실패?       │
│       2번도 실패!             │
└──────────────────────────────┘

결론:
FK는 같은 DB 내 테이블에만 효과적
다른 DB의 테이블을 참조하는 FK는 보장 불가
→ 외래 키 제거, 애플리케이션 계층에서 관리
```

### 4. 외래 키 없이 무결성 보장하기

```
전략 1: 애플리케이션 검증 (즉각적)

INSERT INTO orders (user_id, amount)
VALUES (100, 1000);

애플리케이션 (Application Layer):
1. INSERT 전에 user_id=100이 users에 있는지 확인
   User user = userRepository.findById(100);
   if (user == null) throw new UserNotFoundException();

2. 있으면 INSERT 진행
   Order order = new Order();
   order.setUserId(100);
   order.setAmount(1000);
   orderRepository.save(order);

장점:
- DB 제약 없음 (성능 우수)
- MSA 친화적

단점:
- 데이터베이스 직접 조작 시 검증 안 됨
- 동시성 조건 (Race condition) 가능성

─────────────────────────────

전략 2: 배치 정리 (주기적)

매일 자정에:
DELETE FROM orders 
WHERE user_id NOT IN (
    SELECT id FROM users
)
LIMIT 10000;

또는 캐시 사용:
-- users_cache 테이블 (매시간 동기화)
CREATE TABLE users_cache (
  id BIGINT PRIMARY KEY,
  synced_at TIMESTAMP
);

-- 매시간 UPDATE
INSERT INTO users_cache (id, synced_at)
SELECT id, NOW() FROM users
ON DUPLICATE KEY UPDATE synced_at = NOW();

-- 배치 정리
DELETE FROM orders 
WHERE user_id NOT IN (SELECT id FROM users_cache);

장점:
- Eventually Consistent (최종 일관성)
- 별도 배치 작업이므로 DML 영향 없음

단점:
- 일시적으로 고아 레코드 존재
- BI/분석에 부정확한 데이터 가능

─────────────────────────────

전략 3: Event-driven (이벤트 기반)

users 마이크로서비스에서 사용자 삭제 시:
1. User Deleted 이벤트 발행 (Kafka, RabbitMQ)
2. orders 마이크로서비스에서 구독
3. user_id = X인 주문 상태 업데이트 또는 삭제

Example (Kafka):
// users-service
public void deleteUser(Long userId) {
    usersRepository.deleteById(userId);
    // Deleted event 발행
    kafkaTemplate.send("user.deleted", 
        new UserDeletedEvent(userId, NOW()));
}

// orders-service
@KafkaListener(topics = "user.deleted")
public void handleUserDeleted(UserDeletedEvent event) {
    // orders 정리
    ordersRepository.deleteByUserId(event.getUserId());
    // 또는 상태 업데이트
    ordersRepository.updateStatusByUserId(
        event.getUserId(), 
        OrderStatus.ABANDONED
    );
}

장점:
- Loosely Coupled (느슨한 결합)
- MSA의 이상적인 패턴
- 자동 동기화

단점:
- 복잡한 아키텍처
- 이벤트 유실 가능성
- 구현 난이도 높음
```

---

## 💻 실전 실험

### 실험 1: 외래 키 추가 시 검증 과정

```bash
docker run -d \
  --name mysql80_fk \
  -e MYSQL_ROOT_PASSWORD=root \
  -p 3306:3306 \
  mysql:8.0

mysql -h 127.0.0.1 -u root -proot -e "CREATE DATABASE testdb;"
```

```sql
USE testdb;

-- 테이블 생성
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100),
  email VARCHAR(100)
);

CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT,
  amount DECIMAL(10,2)
);

-- 데이터 삽입
INSERT INTO users (name, email) 
VALUES ('John', 'john@example.com');

INSERT INTO orders (user_id, amount) 
VALUES 
  (1, 100.0),    -- 유효 (user_id=1 존재)
  (1, 200.0),
  (999, 300.0);  -- 고아 레코드 (user_id=999 없음)

-- 사전 확인: 고아 레코드 검증
SELECT COUNT(*) as orphaned_records 
FROM orders 
WHERE user_id NOT IN (SELECT id FROM users);
-- 결과: 1

-- 외래 키 추가 시도 (실패!)
ALTER TABLE orders 
ADD CONSTRAINT fk_orders_users 
FOREIGN KEY (user_id) REFERENCES users(id);

-- Error 1452: Cannot add or update a child row: 
-- foreign key constraint fails

-- 해결: 고아 레코드 먼저 정리
DELETE FROM orders WHERE user_id NOT IN (SELECT id FROM users);

-- 이제 외래 키 추가 성공
ALTER TABLE orders 
ADD CONSTRAINT fk_orders_users 
FOREIGN KEY (user_id) REFERENCES users(id);

-- 확인
SHOW CREATE TABLE orders;
-- FOREIGN KEY ... REFERENCES users(id)

-- 외래 키 제약 테스트
-- 1. 유효한 user_id로 INSERT (성공)
INSERT INTO orders (user_id, amount) VALUES (1, 400.0);
-- OK

-- 2. 유효하지 않은 user_id로 INSERT (실패)
INSERT INTO orders (user_id, amount) VALUES (999, 500.0);
-- Error 1452: Cannot add or update a child row
```

### 실험 2: 외래 키의 성능 오버헤드

```sql
-- 테이블 생성 (1000만 행)
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100),
  INDEX idx_id (id)
);

CREATE TABLE orders_with_fk (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT,
  amount DECIMAL(10,2),
  FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE TABLE orders_without_fk (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT,
  amount DECIMAL(10,2)
);

-- 테스트 데이터
INSERT INTO users (name) VALUES ('User 1'), ('User 2'), ... (1000만 건);

-- 성능 테스트 1: 외래 키 없음
SET @start = NOW(6);
INSERT INTO orders_without_fk (user_id, amount) 
SELECT FLOOR(RAND()*1000000) + 1, ROUND(RAND()*1000, 2)
FROM (SELECT 1 UNION SELECT 2) t1
LIMIT 1000000;
SELECT TIMEDIFF(NOW(6), @start) as elapsed_no_fk;
-- 결과: 약 10초

-- 성능 테스트 2: 외래 키 있음
SET @start = NOW(6);
INSERT INTO orders_with_fk (user_id, amount) 
SELECT FLOOR(RAND()*1000000) + 1, ROUND(RAND()*1000, 2)
FROM (SELECT 1 UNION SELECT 2) t1
LIMIT 1000000;
SELECT TIMEDIFF(NOW(6), @start) as elapsed_with_fk;
-- 결과: 약 50초 (5배 느림)

-- 성능 비교
SELECT 
  'without_fk' as scenario, 10 as seconds
UNION ALL
SELECT 
  'with_fk' as scenario, 50 as seconds;
```

### 실험 3: MSA 환경 시뮬레이션 (외래 키 없이 검증)

```java
// orders-service/OrderService.java
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private UserClient userClient;  // users 마이크로서비스 호출
    
    public OrderResponse createOrder(CreateOrderRequest request) {
        Long userId = request.getUserId();
        BigDecimal amount = request.getAmount();
        
        // 1. users 마이크로서비스에서 사용자 확인
        // (외래 키 대신 애플리케이션 검증)
        try {
            UserResponse user = userClient.getUser(userId);
            if (user == null) {
                throw new UserNotFoundException("User " + userId + " not found");
            }
        } catch (Exception e) {
            throw new ServiceUnavailableException("Users service unavailable", e);
        }
        
        // 2. 주문 생성 (FK 제약 없음, 빠름)
        Order order = new Order();
        order.setUserId(userId);
        order.setAmount(amount);
        Order saved = orderRepository.save(order);
        
        return new OrderResponse(saved.getId(), saved.getUserId(), saved.getAmount());
    }
}

// OrderRepository: 외래 키 없음
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    // 외래 키 제약 없이 user_id만 저장
}

// users-service 삭제 이벤트 처리
@Service
public class UserDeleteEventHandler {
    @Autowired
    private OrderRepository orderRepository;
    
    @KafkaListener(topics = "user.deleted")
    public void handleUserDeleted(String userId) {
        // 사용자 삭제 시 해당 주문 상태 업데이트
        orderRepository.updateStatusByUserId(
            Long.parseLong(userId), 
            OrderStatus.USER_DELETED
        );
    }
}

// 배치 정리 (고아 레코드)
@Component
public class OrphanedRecordCleanup {
    @Autowired
    private OrderRepository orderRepository;
    
    @Scheduled(cron = "0 0 * * * *")  // 매시간
    public void cleanupOrphanedRecords() {
        // 실제로는 users 캐시와 비교
        // 또는 users-service API 호출로 모든 user_id 조회
        
        // 단순화: 1시간 이상 user_deleted 상태 제거
        orderRepository.deleteOldDeletedOrders(
            LocalDateTime.now().minusHours(24)
        );
    }
}
```

### 실험 4: 외래 키 제거 및 검증

```sql
-- 상황: 외래 키가 있던 orders 테이블
-- MSA로 변경하려면 외래 키 제거 필요

-- 1단계: 제약 조건 확인
SELECT CONSTRAINT_NAME FROM INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS
WHERE TABLE_NAME = 'orders' AND CONSTRAINT_SCHEMA = 'testdb';
-- 결과: fk_orders_users

-- 2단계: 고아 레코드 확인
SELECT COUNT(*) as orphaned_count 
FROM orders 
WHERE user_id NOT IN (SELECT id FROM users);

-- 3단계: 필요에 따라 정리
DELETE FROM orders WHERE user_id NOT IN (SELECT id FROM users);

-- 4단계: 외래 키 제거
ALTER TABLE orders 
DROP FOREIGN KEY fk_orders_users;

-- 5단계: 애플리케이션 검증 코드 추가
// Java 코드: userClient.getUser(userId) 호출 추가

-- 6단계: 배치 정리 스케줄 추가
-- 매시간 실행하는 배치 작업 설정
```

---

## 📊 성능/비용 비교

| 전략 | 성능 | 무결성 | MSA 친화 | 복잡도 |
|------|------|--------|---------|--------|
| **외래 키** | -20~80% | 100% | 낮음 | 낮음 |
| **앱 검증** | 0% | 99% | 높음 | 중간 |
| **배치 정리** | 0% | ~95% | 높음 | 낮음 |
| **이벤트 기반** | 0% | 99%+ | 매우 높음 | 높음 |

---

## ⚖️ 트레이드오프

| 선택지 | 장점 | 단점 |
|--------|------|------|
| **외래 키** | 즉각적 무결성, DB 보장 | 성능 저하, MSA 부적합 |
| **앱 검증** | 성능 우수, MSA 친화 | 검증 로직 중복, 일관성 없음 |
| **배치 정리** | 성능 무영향, 자동화 | 일시적 고아, 지연 |
| **이벤트 기반** | 최상의 아키텍처 | 복잡, 이벤트 유실 가능 |

---

## 📌 핵심 정리

1. **외래 키 추가**: 기존 모든 행 검증 필수 (SHARED Lock, 시간 소요)
2. **DML 오버헤드**: 외래 키 있으면 INSERT/UPDATE가 5배 이상 느림
3. **MSA에서 외래 키 문제**:
   - 다른 DB의 테이블 참조 불가
   - 배포 순서 의존성 증가
   - 서비스 간 결합도 증가
4. **대안**:
   - 애플리케이션 계층 검증
   - 배치 정리
   - 이벤트 기반 동기화
5. **선택 기준**:
   - Monolith: 외래 키 유지
   - MSA: 외래 키 제거, 앱 검증 + 배치 정리

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 외래 키 제약이 있을 때, UPDATE로 user_id를 변경하면 어떻게 되는가?</strong></summary>

**답변**:

UPDATE도 외래 키 검증을 거칩니다.

```sql
상황:
CREATE TABLE orders (
  id INT,
  user_id INT,
  FOREIGN KEY (user_id) REFERENCES users(id)
);

INSERT INTO orders VALUES (1, 100);  -- user_id=100 (유효)

-- UPDATE 시도 1: 유효한 user_id로 변경 (성공)
UPDATE orders SET user_id = 200 
WHERE id = 1;
-- 1. user_id = 200이 users에 존재하는지 확인
-- 2. 존재하면 UPDATE 진행
-- 3. orders.user_id: 100 → 200

-- UPDATE 시도 2: 유효하지 않은 user_id로 변경 (실패)
UPDATE orders SET user_id = 999 
WHERE id = 1;
-- 1. user_id = 999이 users에 존재하는지 확인
-- 2. 없으면 오류!
-- Error 1452: Cannot add or update a child row

-- 성능 영향:
-- UPDATE도 INSERT처럼 각 행에 대해 참조 확인
-- UPDATE 성능 = INSERT 성능과 유사 (5배 느림)

-- 대량 UPDATE는 특히 위험:
UPDATE orders SET user_id = user_id + 1;
-- 100만 행 모두 검증 → 매우 느림
-- 외래 키가 없으면: 1초
-- 외래 키가 있으면: 50초+
```

</details>

<details>
<summary><strong>Q2: MSA에서 CASCADE DELETE를 사용하면 안 될까?</strong></summary>

**답변**:

CASCADE DELETE는 MSA에서 더 위험합니다.

```sql
외래 키 with CASCADE DELETE:

CREATE TABLE orders (
  id INT,
  user_id INT,
  FOREIGN KEY (user_id) REFERENCES users(id) 
    ON DELETE CASCADE
);

시나리오:

T1: users에서 사용자 삭제
    DELETE FROM users WHERE id = 100;

T2: CASCADE 발동
    자동으로 orders에서도 삭제
    DELETE FROM orders WHERE user_id = 100;

문제점:
1. 다른 DB의 users에서 삭제하면?
   (MSA의 users-service DB에서 삭제)
   
   orders-service의 orders 테이블은 안전함
   (외래 키가 없으므로)
   
   하지만 만약 외래 키가 있으면?
   복제 지연으로 인해 예기치 않은 삭제 발생 가능

2. 복제 지연 시나리오:
   T1: users DB에서 user_id=100 DELETE
   T2: orders-service의 orders에서 삭제?
       아니다 (다른 DB이므로)
   
   그런데 외래 키가 있다면?
   T1: orders DB에서 user_id=100인 레코드가 
       마스터에서 삭제되면 CASCADE 발동
   T2: users DB의 복제 지연 (5초)
   T3: 5초 동안 주문 데이터 없음?
       (복제 지연으로 인한 불일치)

권장: CASCADE DELETE는 사용하지 말 것
대신:
1. 애플리케이션에서 명시적 삭제
2. 이벤트 기반 동기화
3. 배치 정리

예:
// users-service에서 사용자 삭제 시
public void deleteUser(Long userId) {
    usersRepository.deleteById(userId);
    
    // orders-service에 알림
    kafkaTemplate.send("user.deleted", userId);
    
    // orders-service에서:
    // DELETE FROM orders WHERE user_id = ?
    // 또는 상태 업데이트
}
```

</details>

<details>
<summary><strong>Q3: 외래 키 없이 데이터 일관성을 100% 보장할 수 있는가?</strong></summary>

**답答**:

분산 시스템에서는 100% 보장이 불가능합니다 (CAP Theorem).

```
외래 키가 있는 경우 (단일 DB):
- 일관성(Consistency): 100% (DB가 보장)
- 가용성(Availability): 중간 (Lock으로 인한 대기)
- 분할 허용성(Partition Tolerance): 낮음 (단일 DB)

외래 키 없는 경우 (MSA):
- 일관성(Consistency): 95~99% (앱+배치)
- 가용성(Availability): 높음 (Lock 없음)
- 분할 허용성(Partition Tolerance): 높음 (독립 DB)

분류별 보장 수준:

1. 즉각적 일관성 (Strong Consistency):
   └─ 외래 키 사용 (같은 DB)
   └─ 무결성 100%, 성능 저하

2. 최종 일관성 (Eventual Consistency):
   ├─ 애플리케이션 검증 + 배치
   └─ 일시적 불일치 가능, 결국 일관성

3. 낙관적 일관성 (Optimistic Consistency):
   ├─ 검증 없이 INSERT
   ├─ 배치로 주기적 정리
   └─ 일관성 95% 수준

권장: MSA에서는 Eventual Consistency 수용

예시:
- T1: 사용자 삭제 (users-service)
- T2~T3: 이벤트 처리 지연 (수초)
- T3: orders-service에서 주문 상태 업데이트
- T4: 완전 일관성 (최종)

이 과정에서 일시적 불일치 허용:
- 주문이 존재하지만 사용자는 없는 상태 (수초)
- 하지만 몇 초 후 상태 업데이트됨
- 사용자는 인지 불가 (백엔드 작업)

이것이 MSA의 현실
```

</details>

---

<div align="center">

**[⬅️ 이전: 인덱스 추가](./06-add-index-safely.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — DDL 롤백이 없는 이유 ➡️](../rollback-recovery/01-why-ddl-no-rollback.md)**

</div>
