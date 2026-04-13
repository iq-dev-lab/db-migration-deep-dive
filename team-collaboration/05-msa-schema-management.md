# MSA에서의 스키마 관리

---

## 🎯 핵심 질문

마이크로서비스 아키텍처에서 각 서비스가 독립적인 데이터베이스를 가질 때, 어떻게 스키마 변경을 조율할까요? 공유 DB에서 Database per Service로 마이그레이션할 때의 위험을 어떻게 관리할까요?

## 🔍 왜 이 개념이 실무에서 중요한가

**Database per Service** 원칙은 MSA의 핵심이지만, 스키마 관리를 잘못하면 데이터 불일치, 배포 실패, 또는 데이터 손실이 발생합니다. 특히 모놀리식에서 MSA로 전환하는 과정에서, 공유 테이블을 분리하는 작업은 매우 섬세합니다. 또한 여러 서비스가 관련된 비즈니스 트랜잭션을 처리할 때 스키마 버전 호환성이 중요합니다.

---

## 😱 흔한 실수 (Before)

### 시나리오 1: Database per Service 원칙 무시

**잘못된 구조:**

```
ServiceA (user-service) ─┐
ServiceB (order-service) ├─→ shared_db (모두 같은 DB!)
ServiceC (payment-service) ┘
```

**문제점:**

```
1. 서비스 간 강한 결합도
   ├─ user-service가 공동 테이블 삭제하면
   ├─ order-service, payment-service 장애
   └─ "누가 삭제했어?" 추적 불가

2. 배포 순서 의존성
   ├─ order-service 배포 전에 user-service 배포 필수
   ├─ user-service가 실패하면 order-service도 못 배포
   └─ 배포 자동화 어려움

3. 스케일링 불가능
   ├─ user-service의 부하가 높으면
   ├─ order-service의 쿼리도 느려짐 (같은 DB)
   ├─ 데이터베이스 리소스 공유로 인한 경합
   └─ 독립적인 최적화 불가

4. 스키마 버전 관리 복잡
   ├─ order-service v2.0 배포 중
   ├─ user-service v1.0은 여전히 실행 중
   ├─ 스키마 변경이 양쪽 모두에 영향
   └─ 하위 호환성 유지 어려움

5. 마이그레이션 테스트 어려움
   ├─ 한 서비스의 마이그레이션이 다른 서비스의 쿼리 영향
   ├─ 모든 서비스를 함께 테스트해야 함
   └─ 배포 전 검증 비용 증가
```

### 시나리오 2: Database per Service로 분리하다가 실패

**상황: users 테이블을 user_db에서 order_db로 옮기려고 함**

```
기존 상황:
shared_db:
├─ users
├─ orders
└─ payments

목표:
user_db:      ├─ users
order_db:     ├─ orders
payment_db:   └─ payments
```

**실패 케이스:**

```
1️⃣ 마이그레이션 계획: users → user_db로 복사
   shared_db.users (원본)
   user_db.users (복사본)

2️⃣ 데이터 복사
   INSERT INTO user_db.users 
   SELECT * FROM shared_db.users;
   
3️⃣ order-service 배포 (order_db 사용)
   - 하지만 order 테이블의 FK가 shared_db.users를 참조!
   - 복사 후 새 데이터가 user_db.users에만 저장됨
   - order 테이블의 FK는 shared_db.users를 가리킴
   - → 외래키 제약 위반!

4️⃣ 데이터 불일치
   user_db.users:
   id=1, email=user1@example.com
   
   shared_db.users:
   id=1, email=user1@example.com (이전 버전)
   
   orders:
   user_id=1 (어느 users를 참조하나?)
```

### 시나리오 3: 라이브 스키마 변경 중 불일치

**상황: 여러 버전 앱이 동시 실행 중**

```
배포 상황:
user-service v1.0 → (배포 중)
user-service v2.0 (새 버전, 아직 소수 인스턴스만 실행 중)

버전 차이:
v1.0: users 테이블에 phone 컬럼 있음
v2.0: phone 컬럼을 phone_e164 로 이름 변경
```

**마이그레이션:**

```sql
-- 문제: phone 컬럼 제거, phone_e164 추가
ALTER TABLE users DROP COLUMN phone;
ALTER TABLE users ADD COLUMN phone_e164 VARCHAR(20);
```

**배포 타임라인:**

```
T1: v1.0 인스턴스들 (phone 사용)
    v2.0 마이그레이션 시작
    
T2: phone 컬럼 제거!
    → v1.0 인스턴스가 phone 읽으려고 함
    → Unknown column 'phone' 에러!
    
T3: phone_e164 컬럼 생성
    v2.0 인스턴스들이 데이터 쓰기 시작
    
T4: v1.0 인스턴스들이 phone_e164 읽으려고 함
    → 컬럼 존재하지 않음
    → 또 다른 에러!
```

### 시나리오 4: JOIN 쿼리가 끊어짐 (Database per Service 분리 후)

**상황: 여러 DB에 걸친 JOIN**

```
이전 (shared_db):
SELECT u.name, o.order_id, o.total
FROM users u
JOIN orders o ON u.id = o.user_id;
```

**분리 후:**

```
user_db.users
order_db.orders

-- ❌ 불가능한 쿼리
SELECT u.name, o.order_id, o.total
FROM user_db.users u
JOIN order_db.orders o ON u.id = o.user_id;
-- 같은 데이터베이스에서 JOIN할 수 없음!
```

**문제:**

```
1. JOIN 성능 저하
   - 메모리에서 조인 (애플리케이션 레이어)
   - N+1 쿼리 문제
   
2. 데이터 일관성
   - user_db.users 쿼리 후
   - order_db.orders 쿼리 시점에 데이터 변경 가능
   - 일관성 보장 불가

3. 분산 트랜잭션
   - 두 DB에 동시 커밋 불가
   - 트랜잭션 롤백 어려움
```

---

## ✨ 올바른 접근 (After)

### 원칙 1: Database per Service (명확한 경계)

**최종 목표 아키텍처:**

```
user-service ──→ user_db
               └─ Tables: users, user_roles, user_sessions
               
order-service ──→ order_db
               └─ Tables: orders, order_items, order_status
               
payment-service ──→ payment_db
                 └─ Tables: payments, payment_methods, transactions
```

**각 서비스의 책임:**

```yaml
user-service:
  databases:
    - user_db (소유)
  tables:
    - users (생성, 수정)
    - user_roles (생성, 수정)
  maigration-path: db/migration/user/
  constraints:
    - 다른 서비스의 테이블 접근 금지

order-service:
  databases:
    - order_db (소유)
  tables:
    - orders (생성, 수정)
    - order_items (생성, 수정)
  migration-path: db/migration/order/
  constraints:
    - user_db 접근 금지
    - FK: user_id는 데이터 복제로만 관리 (JOIN 아님)
```

### 전략 1: 공유 DB에서 Database per Service로 마이그레이션

**5단계 마이그레이션 플랜:**

#### Phase 1: 준비 단계 (마이그레이션 0~2주)

```sql
-- 1.1 대상 데이터베이스 생성
CREATE DATABASE user_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 1.2 스키마 복제
CREATE TABLE user_db.users LIKE shared_db.users;
CREATE TABLE user_db.user_roles LIKE shared_db.user_roles;

-- 1.3 데이터 복제 (전체)
INSERT INTO user_db.users 
SELECT * FROM shared_db.users;

INSERT INTO user_db.user_roles 
SELECT * FROM shared_db.user_roles;

-- 1.4 데이터 검증
SELECT COUNT(*) FROM shared_db.users;       -- 100만
SELECT COUNT(*) FROM user_db.users;         -- 100만 (같아야 함)

-- 1.5 인덱스 추가
CREATE INDEX idx_users_email ON user_db.users(email);
```

**마이그레이션 파일:**

```sql
-- user-service/V1__20240415_100000__Prepare_user_db.sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255),
    status VARCHAR(50) DEFAULT 'ACTIVE',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE user_roles (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    role_name VARCHAR(50) NOT NULL
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
```

#### Phase 2: 이중 쓰기 (마이그레이션 2~3주)

```sql
-- 2.1 shared_db 쪽에 Trigger 추가 (새 데이터 → user_db로도 동시 기록)
CREATE TRIGGER user_insert_sync
AFTER INSERT ON shared_db.users
FOR EACH ROW
BEGIN
    INSERT INTO user_db.users 
    VALUES (NEW.id, NEW.email, NEW.name, NEW.status, NEW.created_at, NEW.updated_at)
    ON DUPLICATE KEY UPDATE
        email = NEW.email,
        name = NEW.name,
        status = NEW.status,
        updated_at = NEW.updated_at;
END;

-- 2.2 UPDATE도 동기화
CREATE TRIGGER user_update_sync
AFTER UPDATE ON shared_db.users
FOR EACH ROW
BEGIN
    UPDATE user_db.users SET
        email = NEW.email,
        name = NEW.name,
        status = NEW.status,
        updated_at = NEW.updated_at
    WHERE id = NEW.id;
END;

-- 2.3 DELETE도 동기화
CREATE TRIGGER user_delete_sync
AFTER DELETE ON shared_db.users
FOR EACH ROW
BEGIN
    DELETE FROM user_db.users WHERE id = OLD.id;
END;
```

**앱 레벨 처리 (이중 쓰기):**

```java
// UserRepository.java (shared_db 사용 중)
@Transactional
public User save(User user) {
    // 1. shared_db에 저장
    User savedUser = userRepository.save(user);
    
    // 2. user_db에도 동시에 저장 (이중 쓰기)
    userDbRepository.save(user);
    
    return savedUser;
}

// 또는 Event-driven
@Component
public class UserEventPublisher {
    public void publishUserCreated(User user) {
        // user_db에 비동기로 저장
        userDbService.saveAsync(user);
    }
}
```

**마이그레이션 파일:**

```sql
-- shared-db/V2__20240415_110000__Create_sync_triggers.sql
CREATE TRIGGER user_insert_sync AFTER INSERT ON users ...;
CREATE TRIGGER user_update_sync AFTER UPDATE ON users ...;
CREATE TRIGGER user_delete_sync AFTER DELETE ON users ...;
```

#### Phase 3: 읽기 전환 (마이그레이션 3~4주)

```yaml
# user-service/application.yml (v2.0)
spring:
  datasource:
    # 읽기는 user_db에서 시작
    url: jdbc:mysql://user-db:3306/user_db
    username: user_svc
    password: password
    driver-class-name: com.mysql.cj.jdbc.Driver
  
  # shared_db는 이제 읽기 전용
  secondary-datasource:
    url: jdbc:mysql://shared-db:3306/shared_db
    username: read_only
    password: password
    read-only: true
```

**하위 호환성 유지:**

```java
// UserRepository.java (v2.0)
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // user_db에서 읽기
    Optional<User> findByEmail(String email);
}

// user-service v1.0은 여전히 작동
// shared_db에서 읽기하면서 이중 쓰기로 user_db 동기화
```

**마이그레이션 파일:**

```sql
-- user-service/V2__20240415_120000__Sync_existing_data.sql
-- 배포 후 새 데이터는 user_db에만 기록
-- 기존 데이터는 이미 동기화됨 (Phase 2)
```

#### Phase 4: 쓰기 전환 (마이그레이션 4~5주)

```yaml
# user-service/application.yml (v2.5)
spring:
  datasource:
    # 읽기/쓰기 모두 user_db
    url: jdbc:mysql://user-db:3306/user_db
    username: user_svc
    password: password
  
  # shared_db 읽기 전용 (더 이상 쓰지 않음)
  secondary-datasource:
    url: jdbc:mysql://shared-db:3306/shared_db
    username: read_only
    password: password
    read-only: true
```

```java
// UserRepository.java (v2.5)
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // user_db에서만 읽기/쓰기
    Optional<User> findByEmail(String email);
    
    @Modifying
    void deleteByEmail(String email);
}

// 이중 쓰기 제거
@Transactional
public User save(User user) {
    // user_db에만 저장
    return userRepository.save(user);
}
```

**마이그레이션 파일:**

```sql
-- user-service/V3__20240415_130000__Drop_write_from_shared_db.sql
-- Trigger 제거 (더 이상 필요 없음)
DROP TRIGGER user_insert_sync;
DROP TRIGGER user_update_sync;
DROP TRIGGER user_delete_sync;

-- 마지막 데이터 동기화 확인
SELECT COUNT(*) FROM user_db.users;
SELECT COUNT(*) FROM shared_db.users;
-- 같아야 함!
```

#### Phase 5: 정리 (마이그레이션 5~6주)

```sql
-- 5.1 shared_db의 users 테이블 제거
ALTER TABLE shared_db.orders DROP FOREIGN KEY fk_orders_users;
DROP TABLE shared_db.users;
DROP TABLE shared_db.user_roles;

-- 5.2 shared_db 정리
SELECT * FROM shared_db.users; -- 비어있어야 함
```

**마이그레이션 파일:**

```sql
-- shared-db/V3__20240415_140000__Remove_users_tables.sql
-- 안전장치: 읽기 전용 확인
SELECT COUNT(*) FROM shared_db.users;
-- 0이어야 함! (아니면 실패)

ALTER TABLE shared_db.orders DROP FOREIGN KEY fk_orders_users;
DROP TABLE shared_db.user_roles;
DROP TABLE shared_db.users;
```

### 원칙 2: 서비스 간 통신 (JOIN 대신)

**패턴 1: API 호출 (강한 일관성 필요 시)**

```java
// OrderService.java
public OrderDTO getOrderWithUser(Long orderId) {
    Order order = orderRepository.findById(orderId);
    
    // user-service API 호출
    UserDTO user = userClient.getUser(order.getUserId());
    
    return OrderDTO.of(order, user);
}

// UserClient.java
@FeignClient(name = "user-service")
public interface UserClient {
    @GetMapping("/users/{id}")
    UserDTO getUser(@PathVariable Long id);
}
```

**문제:** 네트워크 호출 비용

```
1. Latency 증가 (100ms × 1000개 주문 = 100초)
2. 부분 실패 가능 (user-service 다운)
3. 캐싱 필요

해결: 캐싱, API Gateway, 배치 처리
```

**패턴 2: 데이터 복제 (최종 일관성)**

```java
// OrderService.java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private Long id;
    
    private Long userId;           // FK
    private String userName;       // 복제된 데이터
    private String userEmail;      // 복제된 데이터
    
    // JOIN 불필요, 조인 데이터는 orders 테이블에 이미 있음
    public void display() {
        System.out.println("Order: " + id + ", User: " + userName);
    }
}

// UserEvent (구독)
@Component
public class UserEventListener {
    @KafkaListener(topics = "user-events")
    public void onUserUpdated(UserUpdatedEvent event) {
        // user-service에서 발행한 이벤트
        Order order = orderRepository.findByUserId(event.getUserId());
        order.setUserName(event.getName());
        order.setUserEmail(event.getEmail());
        orderRepository.save(order);
        // → orders 테이블의 복제 데이터 동기화
    }
}
```

**이벤트 흐름:**

```
1. user-service에서 user.name 변경
   → UserNameChangedEvent 발행

2. order-service가 이벤트 구독
   → orders 테이블의 user_name 업데이트

3. order-service는 JOIN 없이 orders 테이블만 쿼리
   → 빠른 조회 + 높은 성능
```

### 전략 2: Saga 패턴으로 분산 트랜잭션 관리

**상황: 주문 생성 시 여러 서비스 조율 필요**

```
CreateOrder 요청:
1. order-service: 주문 생성 (PENDING)
2. inventory-service: 재고 감소
3. payment-service: 결제 처리
4. 결제 실패 → 모든 변경 롤백?

```

**Orchestration Saga (권장):**

```java
// OrderSaga.java
@Component
public class OrderSaga {
    
    @Transactional
    public OrderResult createOrder(CreateOrderRequest request) {
        // Step 1: 주문 생성
        Order order = orderService.createOrder(request);
        
        try {
            // Step 2: 재고 차감
            inventoryClient.reserveItems(order.getItems());
            
            // Step 3: 결제 처리
            PaymentResult payment = paymentClient.charge(order.getTotalPrice());
            
            // Step 4: 주문 완료
            order.setStatus("CONFIRMED");
            orderService.update(order);
            
            return OrderResult.success(order);
            
        } catch (Exception e) {
            // Compensation: 보상 트랜잭션
            order.setStatus("CANCELLED");
            orderService.update(order);
            
            // Rollback all changes
            inventoryClient.releaseItems(order.getItems());
            
            throw new OrderCreationFailedException(e);
        }
    }
}

// 보상 로직
private void compensateInventoryReservation(Order order) {
    try {
        inventoryClient.releaseItems(order.getItems());
    } catch (Exception e) {
        // 보상 실패? → Dead Letter Queue로 전달
        deadLetterQueue.send(order);
    }
}
```

**Event Sourcing Saga (선택사항):**

```java
@Component
public class OrderEventSaga {
    
    public void createOrder(CreateOrderRequest request) {
        // Event 발행 (데이터베이스에 저장)
        eventStore.append(new OrderCreatedEvent(request));
        
        // 이벤트가 발행되면 각 서비스가 구독
    }
}

// inventory-service 구독
@Component
public class InventoryEventListener {
    @KafkaListener(topics = "order-events")
    public void onOrderCreated(OrderCreatedEvent event) {
        try {
            inventoryService.reserve(event.getItems());
            eventBus.publish(new InventoryReservedEvent(event));
        } catch (Exception e) {
            eventBus.publish(new InventoryReservationFailedEvent(event));
        }
    }
}

// payment-service 구독
@Component
public class PaymentEventListener {
    @KafkaListener(topics = "order-events")
    public void onInventoryReserved(InventoryReservedEvent event) {
        try {
            paymentService.charge(event.getOrderId());
            eventBus.publish(new PaymentCompletedEvent(event));
        } catch (Exception e) {
            eventBus.publish(new PaymentFailedEvent(event));
            // Compensation: inventory 해제
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Trigger 기반 이중 쓰기의 한계

```sql
-- shared_db.users에 Trigger 설정
CREATE TRIGGER user_insert_sync
AFTER INSERT ON users
FOR EACH ROW
BEGIN
    INSERT INTO user_db.users VALUES (...);
END;

-- 문제 1: 데이터 불일치
INSERT INTO shared_db.users VALUES (...);
-- shared_db에는 성공, user_db에는 실패?
-- → Trigger 내부에서 에러 발생
-- → 전체 INSERT 롤백됨!

-- 문제 2: 순환 Trigger
-- user_db에도 Trigger를 설정하면?
INSERT INTO user_db.users VALUES (...);
→ Trigger가 shared_db.users에 다시 INSERT
→ shared_db의 Trigger가 user_db에 다시 INSERT
→ 무한 루프!

-- 문제 3: 네트워크 지연
-- user_db가 느리면?
INSERT INTO shared_db.users VALUES (...);
-- Trigger가 user_db 쓰기를 기다림 (SLOW!)
```

**해결책: 애플리케이션 레벨 이중 쓰기**

```java
@Transactional
public User save(User user) {
    // shared_db에 저장
    userRepository.save(user);
    
    // user_db에도 저장 (Trigger 아님, 앱에서)
    userDbRepository.save(user);
    
    // 둘 다 성공하면 커밋, 하나 실패하면 롤백
}

// 실패 시
try {
    userRepository.save(user);
    userDbRepository.save(user);
} catch (Exception e) {
    // 둘 다 롤백됨
    throw e;
}
```

### 2. 최종 일관성(Eventual Consistency) 모델

```
T0: user-service에서 user.name 변경
    UPDATE users SET name = 'New Name' WHERE id = 1;
    → UserNameChangedEvent 발행

T0.1ms: 이벤트 메시지 큐에 저장

T100ms: order-service가 이벤트 수신
        UPDATE orders SET user_name = 'New Name' WHERE user_id = 1;

T200ms: 모든 서비스의 orders 테이블에 반영됨

결과:
- T0~T100ms: 불일치 (일시적)
- T100ms 이후: 일치 (최종 일관성)

장점:
┌──────────────────────────────────┐
│ 높은 성능 (동기화 아님)         │
│ 높은 확장성 (느슨한 결합)      │
└──────────────────────────────────┘

단점:
┌──────────────────────────────────┐
│ 일시적 불일치 가능              │
│ 디버깅 어려움 (비동기)          │
└──────────────────────────────────┘
```

---

## 💻 실전 실험

### 실험 1: 공유 DB에서 Database per Service로 단계적 마이그레이션

**시작 상태:**

```bash
# shared_db 생성 및 데이터 준비
docker run -d --name mysql-shared \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=shared_db \
  mysql:8.0

mysql -h localhost -u root -ppassword shared_db << 'EOF'
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE,
    name VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    total DECIMAL(10, 2),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

INSERT INTO users (email, name) VALUES 
  ('user1@example.com', 'User 1'),
  ('user2@example.com', 'User 2'),
  ('user3@example.com', 'User 3');

INSERT INTO orders (user_id, total) VALUES 
  (1, 100.00),
  (1, 200.00),
  (2, 150.00);
EOF
```

**Phase 1: user_db 생성 및 데이터 복제**

```bash
# user_db 생성
mysql -h localhost -u root -ppassword << 'EOF'
CREATE DATABASE user_db;
USE user_db;

CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE,
    name VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 데이터 복제
INSERT INTO user_db.users 
SELECT * FROM shared_db.users;

COMMIT;
EOF

# 검증
mysql -h localhost -u root -ppassword user_db << 'EOF'
SELECT COUNT(*) as user_count FROM users;
-- Result: 3
EOF
```

**Phase 2: 이중 쓰기 Trigger 생성**

```bash
mysql -h localhost -u root -ppassword shared_db << 'EOF'
-- INSERT Trigger
DELIMITER //
CREATE TRIGGER user_insert_sync
AFTER INSERT ON users
FOR EACH ROW
BEGIN
    INSERT INTO user_db.users (id, email, name, created_at)
    VALUES (NEW.id, NEW.email, NEW.name, NEW.created_at)
    ON DUPLICATE KEY UPDATE
        email = NEW.email,
        name = NEW.name;
END //
DELIMITER ;

-- UPDATE Trigger
DELIMITER //
CREATE TRIGGER user_update_sync
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
    UPDATE user_db.users SET
        email = NEW.email,
        name = NEW.name
    WHERE id = NEW.id;
END //
DELIMITER ;
EOF

# Trigger 검증
mysql -h localhost -u root -ppassword shared_db << 'EOF'
SHOW TRIGGERS;
EOF
```

**Phase 2 테스트: 이중 쓰기 작동 확인**

```bash
# shared_db에 새 사용자 추가
mysql -h localhost -u root -ppassword shared_db << 'EOF'
INSERT INTO users (email, name) VALUES ('user4@example.com', 'User 4');
COMMIT;
EOF

# user_db에도 자동으로 추가되었는지 확인
mysql -h localhost -u root -ppassword user_db << 'EOF'
SELECT * FROM users WHERE email = 'user4@example.com';
-- Result: user4 존재! (Trigger가 동기화함)
EOF

# shared_db에서 업데이트
mysql -h localhost -u root -ppassword shared_db << 'EOF'
UPDATE users SET name = 'Updated User 1' WHERE id = 1;
COMMIT;
EOF

# user_db도 업데이트되었는지 확인
mysql -h localhost -u root -ppassword user_db << 'EOF'
SELECT name FROM users WHERE id = 1;
-- Result: 'Updated User 1' ✓
EOF
```

**Phase 3: 읽기 전환 (애플리케이션 설정 변경)**

```yaml
# user-service v2.0/application.yml
spring:
  datasource:
    primary:
      url: jdbc:mysql://localhost:3306/user_db
      username: user_svc
      password: password
    
    # 이중 쓰기 검증용 (읽기 전용)
    secondary:
      url: jdbc:mysql://localhost:3306/shared_db
      username: read_only
      password: password
      read-only: true

  flyway:
    url: jdbc:mysql://localhost:3306/user_db
```

**Phase 4: 쓰기 전환**

```bash
# shared_db의 Trigger 제거 (더 이상 이중 쓰기 필요 없음)
mysql -h localhost -u root -ppassword shared_db << 'EOF'
DROP TRIGGER IF EXISTS user_insert_sync;
DROP TRIGGER IF EXISTS user_update_sync;
COMMIT;
EOF

# 애플리케이션 설정
# shared_db 읽기 전용, user_db 읽기/쓰기
```

**Phase 5: 정리**

```bash
# shared_db의 users 테이블 삭제 (orders FK 제거 후)
mysql -h localhost -u root -ppassword shared_db << 'EOF'
-- 먼저 orders 테이블 정리
-- (실제로는 order-service도 마이그레이션 해야 함)

-- 안전장치: 데이터 개수 확인
SELECT COUNT(*) FROM users; -- 0이어야 함 (이미 삭제됨)

-- 삭제
DROP TABLE users;
DROP TABLE user_roles;
COMMIT;
EOF
```

### 실험 2: 이벤트 기반 데이터 동기화

```java
// kafka-topic: user-events
// producer: user-service
// consumer: order-service

// user-service에서 이벤트 발행
@Component
public class UserEventPublisher {
    @Autowired
    private KafkaTemplate<String, UserEvent> kafkaTemplate;
    
    public void publishUserNameChanged(User user) {
        UserEvent event = new UserEvent(
            user.getId(),
            "USER_NAME_CHANGED",
            user.getName()
        );
        kafkaTemplate.send("user-events", event);
    }
}

// order-service에서 이벤트 구독
@Component
public class UserEventListener {
    @Autowired
    private OrderRepository orderRepository;
    
    @KafkaListener(topics = "user-events")
    public void handleUserEvent(UserEvent event) {
        if ("USER_NAME_CHANGED".equals(event.getEventType())) {
            // orders 테이블의 user_name 복제 데이터 업데이트
            List<Order> orders = orderRepository.findByUserId(event.getUserId());
            orders.forEach(order -> {
                order.setUserName(event.getUserName());
                orderRepository.save(order);
            });
        }
    }
}
```

**테스트:**

```bash
# user-service 실행 (Kafka producer)
java -jar user-service.jar &

# order-service 실행 (Kafka consumer)
java -jar order-service.jar &

# user name 변경 API 호출
curl -X PUT http://localhost:8001/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name": "New Name"}'

# 이벤트 확인 (user-service 로그)
# UserNameChangedEvent 발행: user_id=1, name='New Name'

# order-service에서 동기화 확인 (1초 이후)
curl http://localhost:8002/orders/1
# Response: {"id": 1, "user_name": "New Name"}
```

---

## 📊 성능/비용 비교

| 전략 | 일관성 | 성능 | 복잡도 | 권장도 |
|------|------|------|-------|------|
| **공유 DB** | 강함 | 빠름 | 낮음 | ❌ |
| **API 호출** | 강함 | 느림 | 중간 | ⚠️ |
| **데이터 복제 (비동기)** | 최종 | 매우 빠름 | 높음 | ✅✅ |
| **Saga 패턴** | 최종 | 중간 | 매우 높음 | ✅ |
| **이벤트 소싱** | 최종 | 중간 | 매우 높음 | ✅ (선택적) |

**권장:** Database per Service + 이벤트 기반 동기화

---

## ⚖️ 트레이드오프

### 1. 강한 일관성 vs 높은 성능

```
공유 DB (강한 일관성)
├─ JOIN 사용 가능
├─ 트랜잭션 보장
└─ 성능: 좋음 (단일 DB)

최종 일관성 (높은 성능)
├─ JOIN 불가능
├─ 일시적 불일치 가능
├─ 이벤트 처리 지연 가능 (1~10초)
└─ 성능: 매우 좋음 (수평 확장 가능)

권장:
- 금융: 강한 일관성 필요 (공유 DB 또는 2단계 커밋)
- 전자상거래: 최종 일관성 충분 (이벤트 기반)
- 로그/분석: 최종 일관성 (배치 처리)
```

### 2. 마이그레이션 비용 vs 장기 유지보수 비용

```
현재 공유 DB 유지
├─ 단기: 비용 0 (변경 없음)
└─ 장기: 높은 비용 (확장 불가, 디버깅 어려움)

Database per Service 마이그레이션
├─ 단기: 높은 비용 (5~10주 작업)
├─ 중기: 중간 비용 (이벤트 기반 코드 작성)
└─ 장기: 낮은 비용 (각 팀이 독립적)

장기 프로젝트라면: Database per Service 가치 있음
```

---

## 📌 핵심 정리

1. **Database per Service 원칙**: 각 서비스가 자신의 DB만 소유
2. **5단계 마이그레이션**: 준비 → 이중쓰기 → 읽기전환 → 쓰기전환 → 정리
3. **이벤트 기반 동기화**: 최종 일관성으로 높은 성능 확보
4. **JOIN 대신 복제**: 테이블 데이터 복제로 빠른 조회
5. **Saga 패턴**: 분산 트랜잭션 관리 (보상 로직)
6. **점진적 마이그레이션**: 한 번에 모든 것을 바꾸지 않음
7. **모니터링 필수**: 데이터 불일치 감지 및 재동기화

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 이벤트 기반 동기화 중 메시지가 손실되면 어떻게 할까요?</strong></summary>

**상황:**

```
T1: user-service가 UserNameChangedEvent 발행
T2: Kafka에 저장됨
T3: order-service가 이벤트 수신 중...
T4: 네트워크 끊김! 메시지 손실?
```

**해결책:**

```
1. Kafka의 복제 (Replication)
   ├─ 각 파티션을 3개 브로커에 복제
   ├─ 브로커 1개 장애 → 2개가 남음
   └─ 데이터 손실 0

2. Consumer Offset 관리
   ├─ order-service가 처리한 메시지 위치 저장
   ├─ 재연결 시 마지막 위치부터 다시 시작
   └─ 중복 처리 가능 (이상적으로는 멱등성 보장)

3. Dead Letter Queue (DLQ)
   ├─ 처리 실패한 메시지 → DLQ로 이동
   ├─ 별도 모니터링/재처리
   └─ 운영자 개입 가능

예시 코드:

@KafkaListener(topics = "user-events")
public void handleUserEvent(UserEvent event) {
    try {
        orderRepository.updateUserName(event.getUserId(), event.getUserName());
    } catch (Exception e) {
        // DLQ로 이동
        deadLetterQueueTemplate.send("user-events-dlq", event);
        
        // 알림
        alertService.notify("UserEvent 처리 실패");
        
        throw e;  // 재시도
    }
}
```

**결론:** Kafka + 멱등성 처리 = 거의 100% 안전

</details>

<details>
<summary><strong>Q2: 두 개 이상의 이벤트가 동시에 발행되고 순서가 바뀌면?</strong></summary>

**상황:**

```
user-service:
T1: user.name = 'Alice' → NameChangedEvent(Alice) 발행
T2: user.name = 'Bob'   → NameChangedEvent(Bob) 발행

Kafka (순서 보장 없음):
T3: order-service 수신: Bob (먼저 도착)
T4: order-service 수신: Alice (나중에 도착)

결과:
orders.user_name = 'Alice' (최종)
하지만 실제로는 'Bob'이어야 함!
```

**해결책:**

```
1. Event Versioning (타임스탬프)
   UserNameChangedEvent {
     userId: 1,
     name: 'Alice',
     timestamp: 1000  ← 버전 정보
   }
   
   구독자:
   if (event.timestamp > lastUpdate.timestamp) {
     update(event);  // 최신 이벤트만 적용
   }

2. Event Ordering (Kafka 파티셔닝)
   Topic: user-events
   Partition 0: user_id=1의 모든 이벤트
   Partition 1: user_id=2의 모든 이벤트
   
   같은 user_id → 같은 파티션 → 순서 보장!
   
   코드:
   @Bean
   public ProducerFactory<String, UserEvent> producerFactory() {
       return new DefaultKafkaProducerFactory<>(
           new KafkaProducerConfig(),
           new StringSerializer(),
           new JsonSerializer<>()
       );
   }
   
   kafkaTemplate.send(
       new ProducerRecord<>(
           "user-events",
           event.getUserId().toString(),  // ← 파티션 키
           event
       )
   );

3. 멱등성 처리
   UPDATE users SET name = 'Alice'
   WHERE id = 1 AND updated_at < event.timestamp;
   
   → 같은 이벤트 재처리 안전
```

**권장:** Event Versioning + Kafka 파티셔닝

</details>

<details>
<summary><strong>Q3: 마이그레이션 중 롤백하려면?</strong></summary>

**상황:**

```
Phase 3 (읽기 전환)에서 문제 발생:
- user_db.users 데이터 손상
- order-service가 잘못된 데이터 읽음
- 즉시 롤백 필요!
```

**롤백 전략:**

```
Phase 1, 2 (준비 + 이중쓰기): 롤백 쉬움
└─ user_db는 사용 중 아님
└─ shared_db만 원래대로
└─ 비용: 낮음

Phase 3 (읽기 전환): 롤백 가능하지만 신중
├─ app 설정을 shared_db로 역전
├─ user_db의 변경사항 재검증 필요
└─ 비용: 중간

Phase 4 (쓰기 전환): 롤백 어려움
├─ 이중 쓰기 코드 복구 필요
├─ shared_db와 user_db 재동기화
└─ 비용: 높음

Phase 5 (정리) 이후: 롤백 불가능
└─ shared_db 테이블 삭제됨
└─ 최후의 보루: 백업에서 복구
```

**롤백 계획:**

```yaml
Rollback Phase 3:
  1. 앱 설정 변경: shared_db로 다시 읽기
  2. user_db의 변경사항 분석
  3. shared_db와 재동기화
  4. 데이터 검증 (integrity check)
  
Rollback Phase 4:
  1. 이중 쓰기 코드 복구
  2. user_db에서 shared_db로 데이터 역동기화
  3. 충돌 해결
  4. 서서히 트래픽 전환
  
Rollback Phase 5 이후:
  1. DB 백업에서 복구
  2. 데이터 손실 가능
  3. 최대한 빨리 복구 (분 단위)
```

**교훈:** 각 Phase를 신중히 검증한 후 다음 단계로 진행!

</details>

---

<div align="center">

**[⬅️ 이전: 멀티 모듈 마이그레이션](./04-multi-module-migration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — 배포 파이프라인에서의 마이그레이션 시점 ➡️](../cicd-integration/01-migration-timing-in-pipeline.md)**

</div>
