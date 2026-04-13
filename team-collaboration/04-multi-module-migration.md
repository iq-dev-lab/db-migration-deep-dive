# 멀티 모듈 마이그레이션

---

## 🎯 핵심 질문

여러 Spring Boot 모듈이 같은 데이터베이스를 공유할 때 마이그레이션을 어떻게 관리할까요? 모듈별로 독립적으로 관리하면서도 충돌을 피할 수 있을까요?

## 🔍 왜 이 개념이 실무에서 중요한가

대규모 프로젝트에서는 **하나의 데이터베이스를 여러 모듈이 공유**하는 경우가 많습니다. 예를 들어, `user-service`, `order-service`, `payment-service` 같은 여러 모듈이 모두 같은 DB에 접근합니다. 이 상황에서 마이그레이션을 제대로 관리하지 않으면 버전 충돌, 의존성 누락, 또는 배포 순서 문제가 발생합니다. 올바른 전략을 통해 모듈 간 독립성을 유지하면서 스키마 변경을 조율해야 합니다.

---

## 😱 흔한 실수 (Before)

### 시나리오 1: 각 모듈이 독립적으로 마이그레이션 관리

**프로젝트 구조:**

```
my-monolith/
├─ user-service/
│  ├─ src/main/resources/db/migration/
│  │  ├─ V1__Add_users_table.sql
│  │  └─ V2__Add_user_roles.sql
│  └─ pom.xml
│
├─ order-service/
│  ├─ src/main/resources/db/migration/
│  │  ├─ V1__Add_orders_table.sql
│  │  └─ V2__Add_order_items.sql
│  └─ pom.xml
│
└─ payment-service/
   ├─ src/main/resources/db/migration/
   │  ├─ V1__Add_payments_table.sql
   │  └─ V2__Add_transactions.sql
   └─ pom.xml
```

**문제 1: 버전 충돌**

```
각 모듈이 V1, V2, V3을 정의하면:
- user-service: V1 (users), V2 (user_roles)
- order-service: V1 (orders), V2 (order_items)
- payment-service: V1 (payments), V2 (transactions)

Flyway 실행 순서:
┌──────────────────────────────────────────┐
│ 어느 V1이 먼저 실행되나?                  │
│ user-service의 V1인가?                  │
│ order-service의 V1인가?                  │
│ payment-service의 V1인가?                │
└──────────────────────────────────────────┘

flyway_schema_history:
V1 ✓ (user-service)
V1 ✗ CONFLICT! (order-service와 payment-service)
```

**문제 2: 의존성 누락**

```
order-service의 V2:
ALTER TABLE orders ADD COLUMN user_id BIGINT;
ALTER TABLE orders ADD CONSTRAINT 
    FOREIGN KEY (user_id) REFERENCES users(id);

하지만 users 테이블은 user-service에 정의됨!

만약 order-service가 먼저 실행되면:
❌ users 테이블이 없음 → FOREIGN KEY 오류!
```

**문제 3: 배포 순서 강제**

```
배포 순서가 매우 중요함:
1️⃣ user-service 배포 (users 테이블 생성)
2️⃣ order-service 배포 (orders 테이블 + FK)
3️⃣ payment-service 배포 (payments 테이블)

하지만 자동화하기 어려움:
- CI/CD 파이프라인이 모듈 순서를 알아야 함
- 모듈 간 의존성 명시 필요
- 순서 변경 시 자동 테스트 필요
```

### 시나리오 2: 모든 마이그레이션을 한 모듈에 중앙화

**프로젝트 구조:**

```
my-monolith/
├─ migration-service/ (별도 모듈)
│  └─ src/main/resources/db/migration/
│     ├─ V1__Add_users_table.sql
│     ├─ V2__Add_orders_table.sql
│     ├─ V3__Add_payments_table.sql
│     └─ ... (모든 마이그레이션)
│
├─ user-service/ (마이그레이션 없음)
├─ order-service/ (마이그레이션 없음)
└─ payment-service/ (마이그레이션 없음)
```

**장점:**
- 버전 충돌 없음 (단일 파이프라인)
- 실행 순서 명확 (V1 → V2 → V3)

**단점:**

```
1. 모듈 간 강한 결합도
   ├─ order-service 코드 변경 시 migration-service도 수정
   ├─ 모듈 배포 순서 의존
   └─ migration-service가 항상 먼저 배포되어야 함

2. 스케일링 어려움
   ├─ 모듈이 100개가 되면?
   ├─ 마이그레이션 파일도 100개+
   └─ 의존성 추적 불가능

3. 모듈 독립성 상실
   ├─ 신규 모듈 추가 시 migration-service에서 작업
   ├─ 여러 팀이 같은 파일 수정
   └─ 충돌 증가

4. 마이그레이션 버전이 비즈니스 버전과 무관
   ├─ user-service v2.0, order-service v1.5 배포하는데
   ├─ migration-service v3.2 배포?
   └─ 버전 관계 추적 어려움
```

---

## ✨ 올바른 접근 (After)

### 전략 1: 각 모듈의 독립적 마이그레이션 + 타임스탐프 버전

**핵심 아이디어:** 모듈별 마이그레이션은 **타임스탐프 기반 버전**으로 통합 버전 관리

**프로젝트 구조:**

```
my-monolith/
├─ user-service/
│  ├─ src/main/resources/db/migration/
│  │  ├─ V1__20240101_100000__Initial_users.sql
│  │  └─ V2__20240415_143000__Add_user_roles.sql
│  └─ pom.xml
│
├─ order-service/
│  ├─ src/main/resources/db/migration/
│  │  ├─ V1__20240102_100000__Initial_orders.sql
│  │  └─ V2__20240415_144000__Add_order_status.sql
│  └─ pom.xml
│
└─ payment-service/
   ├─ src/main/resources/db/migration/
   │  ├─ V1__20240103_100000__Initial_payments.sql
   │  └─ V2__20240415_145000__Add_transaction_type.sql
   └─ pom.xml
```

**Flyway 정렬 결과:**

```
V1__20240101_100000__Initial_users.sql        (user-service)
V1__20240102_100000__Initial_orders.sql       (order-service)
V1__20240103_100000__Initial_payments.sql     (payment-service)
V2__20240415_143000__Add_user_roles.sql       (user-service)
V2__20240415_144000__Add_order_status.sql     (order-service)
V2__20240415_145000__Add_transaction_type.sql (payment-service)

정렬 기준: V숫자 < V숫자, 같으면 타임스탐프
결과: 자동으로 올바른 순서!
```

**Spring Boot 설정 (각 모듈):**

```yaml
# user-service/application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    validate-on-migrate: true

# order-service/application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    validate-on-migrate: true

# payment-service/application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    validate-on-migrate: true
```

**배포 순서 (자유도 높음):**

```
각 모듈이 독립적으로 배포:

시나리오 1:
user-service → order-service → payment-service
결과: 모두 성공 (타임스탐프로 정렬됨)

시나리오 2:
order-service → payment-service → user-service
결과: 모두 성공 (타임스탐프로 정렬됨)

시나리오 3 (동시 배포):
모든 모듈 동시 배포 ✓
Flyway가 알아서 정렬 + 실행
```

### 전략 2: 모듈별 flyway_schema_history 테이블 분리

**상황:** 마이그레이션 이력을 모듈별로 따로 추적하고 싶은 경우

**프로젝트 구조:**

```
├─ user-service/
│  └─ db/migration/
│     └─ V1__Add_users.sql
│
├─ order-service/
│  └─ db/migration/
│     └─ V1__Add_orders.sql
│
└─ payment-service/
   └─ db/migration/
      └─ V1__Add_payments.sql
```

**Spring Boot 설정 (각 모듈별 history 테이블):**

```yaml
# user-service/application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    table: flyway_schema_history_user
    baseline-on-migrate: false

# order-service/application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    table: flyway_schema_history_order
    baseline-on-migrate: false

# payment-service/application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    table: flyway_schema_history_payment
    baseline-on-migrate: false
```

**결과:**

```sql
-- 데이터베이스에 3개의 history 테이블 생성
SHOW TABLES LIKE 'flyway_schema_history%';

+------------------------------+
| Tables_in_testdb             |
+------------------------------+
| flyway_schema_history_user   | ← user-service만 관리
| flyway_schema_history_order  | ← order-service만 관리
| flyway_schema_history_payment| ← payment-service만 관리
+------------------------------+

-- 각 모듈은 자신의 history만 추적
SELECT * FROM flyway_schema_history_user;
V1: Add_users ✓

SELECT * FROM flyway_schema_history_order;
V1: Add_orders ✓

SELECT * FROM flyway_schema_history_payment;
V1: Add_payments ✓

-- 하지만 실제 테이블은 모두 같은 DB!
SHOW TABLES;
users, orders, payments (공유됨)
```

**장점:**
- 모듈별 마이그레이션 이력 추적
- 배포 순서 자유도 높음
- 모듈 간 간섭 없음

**주의:** 모듈이 다른 모듈의 테이블을 수정하려면?

```sql
-- order-service의 마이그레이션에서
-- users 테이블(user-service 소유) 수정
ALTER TABLE users ADD COLUMN order_count INT;

-- ⚠️ 2개 모듈의 이력을 봐야 함:
SELECT * FROM flyway_schema_history_order;   -- 이 쿼리가 저장됨
SELECT * FROM flyway_schema_history_user;    -- 하지만 users 테이블도 변경됨
-- 추적 복잡!

-- 권장: 테이블은 소유 모듈에서만 수정
-- order-service는 자신의 orders 테이블만 수정
```

### 전략 3: 네임스페이스 기반 분리 (권장)

**핵심:** 마이그레이션 경로로 모듈을 명시적으로 분리

**프로젝트 구조:**

```
my-monolith/
├─ database/
│  └─ src/main/resources/db/migration/
│     ├─ user/ (user-service 마이그레이션)
│     │  ├─ V1__20240101_100000__Create_users.sql
│     │  └─ V2__20240415_143000__Add_roles.sql
│     │
│     ├─ order/ (order-service 마이그레이션)
│     │  ├─ V1__20240102_100000__Create_orders.sql
│     │  └─ V2__20240415_144000__Add_status.sql
│     │
│     └─ payment/ (payment-service 마이그레이션)
│        ├─ V1__20240103_100000__Create_payments.sql
│        └─ V2__20240415_145000__Add_method.sql
│
├─ user-service/
│  └─ pom.xml (database 의존성)
│
├─ order-service/
│  └─ pom.xml (database 의존성)
│
└─ payment-service/
   └─ pom.xml (database 의존성)
```

**Spring Boot 설정 (공유 database 모듈):**

```yaml
# database/application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration/user, 
               classpath:db/migration/order,
               classpath:db/migration/payment
    baseline-on-migrate: false
    validate-on-migrate: true
```

**각 모듈 설정:**

```yaml
# user-service/application.yml
spring:
  profiles:
    include: database  # database 모듈 설정 포함

# order-service/application.yml
spring:
  profiles:
    include: database

# payment-service/application.yml
spring:
  profiles:
    include: database
```

**또는 동적 위치 설정:**

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    locations: classpath*:db/migration/**  # 모든 하위 경로 스캔
```

**장점:**
- 파일 구조가 모듈 의존성을 명시
- 각 팀이 자신의 경로만 관리
- 마이그레이션 충돌 없음

---

## 🔬 내부 동작 원리

### 1. Flyway의 클래스패스 스캔 메커니즘

```java
// Spring Boot의 FlywayProperties
@ConfigurationProperties(prefix = "spring.flyway")
public class FlywayProperties {
    // locations: 마이그레이션 파일을 찾을 경로
    private String[] locations = {"classpath:db/migration"};
    
    // table: 히스토리 추적 테이블명
    private String table = "flyway_schema_history";
    
    // baselineVersion: 초기 버전
    private String baselineVersion = "1";
}

// Spring이 여러 모듈의 마이그레이션을 병합
public List<ResolvedMigration> scan() {
    List<ResolvedMigration> migrations = new ArrayList<>();
    
    // 클래스패스의 모든 경로 스캔
    for (String location : locations) {
        Resource[] resources = resourceLoader.getResources(location + "/**");
        
        for (Resource resource : resources) {
            String filename = resource.getFilename();
            if (filename.matches("V\\d+.*\\.sql")) {
                migrations.add(parseMigration(resource));
            }
        }
    }
    
    // 버전으로 정렬
    Collections.sort(migrations);
    return migrations;
}
```

**실제 동작:**

```
user-service 모듈:
  /user-service/target/classes/db/migration/
    V1__20240101_100000__Create_users.sql
    V2__20240415_143000__Add_roles.sql

order-service 모듈:
  /order-service/target/classes/db/migration/
    V1__20240102_100000__Create_orders.sql
    V2__20240415_144000__Add_status.sql

Flyway 스캔 결과 (클래스패스에서):
✓ V1__20240101_100000__Create_users.sql
✓ V1__20240102_100000__Create_orders.sql
✓ V2__20240415_143000__Add_roles.sql
✓ V2__20240415_144000__Add_status.sql

정렬:
1. V1__20240101_100000__Create_users.sql
2. V1__20240102_100000__Create_orders.sql
3. V2__20240415_143000__Add_roles.sql
4. V2__20240415_144000__Add_status.sql
```

### 2. 모듈 간 테이블 의존성 추적

```
모듈별 책임 영역:

user-service:
├─ 소유 테이블: users, user_roles, user_sessions
├─ 마이그레이션: V1-V10
└─ 외래키 참조: 없음 (기본 데이터)

order-service:
├─ 소유 테이블: orders, order_items, order_history
├─ 마이그레이션: V11-V20
├─ 외래키 참조: users(user_id)
└─ 의존성: user-service v1.0+

payment-service:
├─ 소유 테이블: payments, transactions, payment_methods
├─ 마이그레이션: V21-V30
├─ 외래키 참조: orders(order_id), users(user_id)
└─ 의존성: user-service v1.0+, order-service v11.0+
```

**의존성 정의 파일 (선택사항):**

```yaml
# database/src/main/resources/migration-dependencies.yml
dependencies:
  user-service:
    version: "1"
    depends-on: []  # 의존성 없음
    
  order-service:
    version: "11"
    depends-on:
      - user-service: "1"  # user v1 이상 필요
    
  payment-service:
    version: "21"
    depends-on:
      - user-service: "1"
      - order-service: "11"
```

### 3. 다중 Flyway 인스턴스 관리

각 모듈이 자신의 history 테이블을 가질 때:

```java
// database 모듈의 설정
@Configuration
public class FlywayConfig {
    
    @Bean
    public Flyway userFlyway(DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/user")
            .table("flyway_schema_history_user")
            .load();
    }
    
    @Bean
    public Flyway orderFlyway(DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/order")
            .table("flyway_schema_history_order")
            .load();
    }
    
    @Bean
    public Flyway paymentFlyway(DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/payment")
            .table("flyway_schema_history_payment")
            .load();
    }
    
    @Bean
    public CommandLineRunner runFlywayMigrations(
            Flyway userFlyway, Flyway orderFlyway, Flyway paymentFlyway) {
        return args -> {
            userFlyway.migrate();
            orderFlyway.migrate();
            paymentFlyway.migrate();
        };
    }
}
```

---

## 💻 실전 실험

### 실험 1: 타임스탐프 기반 멀티 모듈 마이그레이션

**프로젝트 구조 생성:**

```bash
mkdir -p multi-module-migration/{user-service,order-service,payment-service}

# user-service
mkdir -p multi-module-migration/user-service/src/main/resources/db/migration

cat > multi-module-migration/user-service/src/main/resources/db/migration/V1__20240101_100000__Create_users.sql << 'EOF'
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
EOF

cat > multi-module-migration/user-service/src/main/resources/db/migration/V2__20240415_143000__Add_user_roles.sql << 'EOF'
CREATE TABLE user_roles (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    role_name VARCHAR(50) NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
EOF

# order-service
mkdir -p multi-module-migration/order-service/src/main/resources/db/migration

cat > multi-module-migration/order-service/src/main/resources/db/migration/V1__20240102_100000__Create_orders.sql << 'EOF'
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(50) DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
EOF

cat > multi-module-migration/order-service/src/main/resources/db/migration/V2__20240415_144000__Add_order_items.sql << 'EOF'
CREATE TABLE order_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE
);

CREATE INDEX idx_order_items_order_id ON order_items(order_id);
EOF

# payment-service
mkdir -p multi-module-migration/payment-service/src/main/resources/db/migration

cat > multi-module-migration/payment-service/src/main/resources/db/migration/V1__20240103_100000__Create_payments.sql << 'EOF'
CREATE TABLE payments (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(50) DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_payments_order_id ON payments(order_id);
CREATE INDEX idx_payments_user_id ON payments(user_id);
EOF

cat > multi-module-migration/payment-service/src/main/resources/db/migration/V2__20240415_145000__Add_payment_methods.sql << 'EOF'
CREATE TABLE payment_methods (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    method_type VARCHAR(50) NOT NULL,
    is_default BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_payment_methods_user_id ON payment_methods(user_id);
EOF
```

**Spring Boot 통합 설정:**

```yaml
# application.yml (모든 모듈이 같은 DB 사용)
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/multitenant
    username: root
    password: password
    driver-class-name: com.mysql.cj.jdbc.Driver
  
  flyway:
    enabled: true
    locations: classpath:db/migration  # 클래스패스 전체 스캔
    baseline-on-migrate: false
    validate-on-migrate: true
    out-of-order: false  # 순차 실행 강제
```

**테스트:**

```bash
# 각 모듈을 차례대로 또는 동시에 빌드/실행
mvn clean install

# 첫 번째 모듈 실행 (Flyway 실행)
java -jar user-service/target/app.jar

# 두 번째 모듈 실행 (Flyway 실행)
java -jar order-service/target/app.jar

# 세 번째 모듈 실행 (Flyway 실행)
java -jar payment-service/target/app.jar

# 결과 확인
mysql -h localhost -u root -p multitenant << 'EOF'
SELECT * FROM flyway_schema_history ORDER BY installed_rank;
+----+---------+---------------------+--------+----------+
| id | version | script              | status | checksum |
+----+---------+---------------------+--------+----------+
| 1  | 1       | V1__20240101_100000_...| OK    | 123...   |
| 2  | 1       | V1__20240102_100000_...| OK    | 456...   |
| 3  | 1       | V1__20240103_100000_...| OK    | 789...   |
| 4  | 2       | V2__20240415_143000_...| OK    | abc...   |
| 5  | 2       | V2__20240415_144000_...| OK    | def...   |
| 6  | 2       | V2__20240415_145000_...| OK    | ghi...   |
+----+---------+---------------------+--------+----------+

SHOW TABLES;
orders, order_items, payments, payment_methods, users, user_roles, flyway_schema_history
EOF
```

### 실험 2: 모듈별 history 테이블 분리

```java
// database 모듈의 설정 클래스
@Configuration
public class MultiModuleFlywayConfig {
    
    @Bean
    public Flyway userServiceFlyway(DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/user")
            .table("flyway_schema_history_user")
            .baselineOnMigrate(true)
            .baselineVersion("0")
            .load();
    }
    
    @Bean
    public Flyway orderServiceFlyway(DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/order")
            .table("flyway_schema_history_order")
            .baselineOnMigrate(true)
            .baselineVersion("0")
            .load();
    }
    
    @Bean
    public Flyway paymentServiceFlyway(DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/payment")
            .table("flyway_schema_history_payment")
            .baselineOnMigrate(true)
            .baselineVersion("0")
            .load();
    }
    
    @Bean
    public CommandLineRunner runMigrations(
            Flyway userServiceFlyway,
            Flyway orderServiceFlyway,
            Flyway paymentServiceFlyway) {
        
        return args -> {
            // 순서 상관없음 (각자 history 테이블 사용)
            userServiceFlyway.migrate();
            orderServiceFlyway.migrate();
            paymentServiceFlyway.migrate();
            
            // 결과 로깅
            System.out.println("User Service migrations: " + 
                userServiceFlyway.info().all().length);
            System.out.println("Order Service migrations: " + 
                orderServiceFlyway.info().all().length);
            System.out.println("Payment Service migrations: " + 
                paymentServiceFlyway.info().all().length);
        };
    }
}
```

**프로젝트 구조:**

```
database-module/
└─ src/main/resources/db/migration/
   ├─ user/
   │  ├─ V1__20240101_100000__Create_users.sql
   │  └─ V2__20240415_143000__Add_roles.sql
   │
   ├─ order/
   │  ├─ V1__20240102_100000__Create_orders.sql
   │  └─ V2__20240415_144000__Add_items.sql
   │
   └─ payment/
      ├─ V1__20240103_100000__Create_payments.sql
      └─ V2__20240415_145000__Add_methods.sql
```

### 실험 3: 배포 순서 자유도 검증

**시나리오:** 모듈을 다양한 순서로 배포해도 정상 작동하는지 확인

```bash
# 시나리오 1: 추천 순서 (의존성 따름)
# 1. user-service (의존성 없음)
# 2. order-service (user 참조)
# 3. payment-service (user, order 참조)
java -jar user-service/target/app.jar &
sleep 10
java -jar order-service/target/app.jar &
sleep 10
java -jar payment-service/target/app.jar &

# 시나리오 2: 역순
# 1. payment-service (의존성 존재하지만...)
# 2. order-service
# 3. user-service
java -jar payment-service/target/app.jar &
sleep 5
java -jar order-service/target/app.jar &
sleep 5
java -jar user-service/target/app.jar &

# 시나리오 3: 동시 배포
java -jar user-service/target/app.jar &
java -jar order-service/target/app.jar &
java -jar payment-service/target/app.jar &

# 모든 시나리오에서:
mysql -h localhost -u root -p testdb << 'EOF'
SELECT * FROM flyway_schema_history ORDER BY installed_rank;
-- 결과: 6개 마이그레이션 모두 OK
-- (배포 순서 상관없음!)
EOF
```

---

## 📊 성능/비용 비교

| 전략 | 버전 충돌 | 배포 순서 | 모듈 독립성 | 관리 복잡도 | 권장도 |
|------|---------|---------|----------|---------|------|
| **독립 마이그레이션** (타임스탐프) | 없음 | 자유 | 높음 | 낮음 | ✅✅✅ |
| **중앙화 마이그레이션** | 없음 | 강제 | 낮음 | 높음 | ⚠️ |
| **모듈별 history** | 없음 | 자유 | 높음 | 중간 | ✅✅ |
| **네임스페이스 분리** | 없음 | 자유 | 높음 | 중간 | ✅✅ |

**권장:** 타임스탐프 + 독립 마이그레이션

---

## ⚖️ 트레이드오프

### 1. 모듈 독립성 vs 중앙 관리

```
독립 마이그레이션 (타임스탐프)
├─ 장점: 각 모듈이 독립적
├─ 장점: 배포 순서 자유
└─ 단점: 전체 마이그레이션 이력 추적 필요

중앙화 마이그레이션
├─ 장점: 단일 history 테이블
├─ 장점: 전체 이력 명확
└─ 단점: 모듈이 결합됨
```

**권장:** 독립 마이그레이션 + 정기적인 전체 감사

### 2. 학습 곡선 vs 자동화

```
간단한 구조 (모든 마이그레이션을 user-service에)
├─ 학습 쉬움
└─ 유지보수 어려움

복잡한 구조 (모듈별 + 네임스페이스)
├─ 초기 학습 시간 필요
└─ 장기 유지보수 쉬움
```

---

## 📌 핵심 정리

1. **타임스탐프 버전으로 자동 정렬** - 버전 충돌 zero
2. **각 모듈이 독립적으로 마이그레이션 관리** - 배포 순서 자유
3. **모듈별 history 테이블** (선택) - 이력 명확화
4. **네임스페이스 분리** (선택) - 파일 구조 명확화
5. **테이블 소유권 명시** - 어느 모듈이 관리하는가?
6. **의존성 문서화** - payment는 user/order에 의존
7. **CI/CD에서 순서 무관하게** - Flyway가 알아서 정렬

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 모듈 A의 마이그레이션이 모듈 B의 테이블을 참조하면 어떻게 해야 할까요?</strong></summary>

**상황:**
```
user-service (모듈 A):
- 소유: users 테이블
- 마이그레이션: V1__Create_users.sql

order-service (모듈 B):
- 소유: orders 테이블
- 마이그레이션: V1__Create_orders.sql
                V2__Add_foreign_key_to_users.sql
                     ↑ users 테이블 참조
```

**타임스탐프 정렬:**
```
V1__20240101_100000__Create_users.sql (user-service)
V1__20240102_100000__Create_orders.sql (order-service)
V2__20240415_143000__Add_fk_to_users.sql (order-service)
                      ↑ users는 이미 생성됨 ✓
```

**안전한 쓰기:**

```sql
-- V2__20240415_143000__Add_fk_to_users.sql (order-service)
-- 사전 조건: users 테이블이 이미 생성됨
-- (타임스탐프가 users v1보다 이후)

ALTER TABLE orders ADD CONSTRAINT fk_orders_users 
FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE;
```

**위험한 상황 회피:**

```
❌ user-service v1 아직 배포 안 됨
   → order-service v2 배포 시도
   → users 테이블 없음!
   → 마이그레이션 실패

✅ 해결책:
1. 타임스탐프로 자동 정렬
2. 의존성 문서화
3. CI/CD에서 검증
```

</details>

<details>
<summary><strong>Q2: 여러 서비스로 분리할 때 마이그레이션을 어떻게 관리해야 할까요? (MSA)</strong></summary>

**현황: 단일 DB, 여러 서비스**
```
user-service ┐
order-service ├─→ 공유 DB
payment-service┘
```

**목표: MSA로 분리**
```
user-service ──→ user_db
order-service ──→ order_db
payment-service ──→ payment_db
```

**마이그레이션 전략:**

```
Phase 1: 데이터 복제
├─ user_db 생성
├─ users 테이블 복사
└─ payment-service는 여전히 공유 DB 사용

Phase 2: 이중 쓰기
├─ 새 데이터는 user_db + 공유 DB에 동시 기록
└─ 일관성 검증

Phase 3: 읽기 전환
├─ user-service가 user_db에서 읽기 시작
├─ 공유 DB 읽기 정지
└─ 쓰기는 여전히 이중

Phase 4: 쓰기 전환
├─ 공유 DB 쓰기 정지
├─ user_db만 쓰기
└─ 이중 쓰기 종료

Phase 5: 정리
├─ 공유 DB의 users 삭제
├─ Flyway history 정리
└─ 모니터링 확대
```

**각 Phase의 마이그레이션:**

```sql
-- Phase 1: user_db 생성
CREATE DATABASE user_db;
CREATE TABLE user_db.users LIKE shared_db.users;
INSERT INTO user_db.users SELECT * FROM shared_db.users;

-- Phase 2: Trigger로 이중 쓰기 (공유 DB 쪽)
CREATE TRIGGER user_sync_insert
AFTER INSERT ON shared_db.users
FOR EACH ROW
BEGIN
  INSERT INTO user_db.users VALUES (...);
END;

-- Phase 3: app 설정 변경 (user-service)
spring.datasource.url = jdbc:mysql://user-db:3306/user_db

-- Phase 4: Trigger 제거
DROP TRIGGER user_sync_insert;

-- Phase 5: 정리
DELETE FROM shared_db.users;
ALTER TABLE shared_db.users DROP FOREIGN KEY ...;
```

**마이그레이션 구조:**

```
user-service-db/
└─ src/main/resources/db/migration/
   ├─ V1__20240101_100000__Create_users.sql
   ├─ V2__20240415_143000__Split_from_shared_db.sql
   └─ V3__20240415_144000__Sync_historical_data.sql

order-service-db/
└─ src/main/resources/db/migration/
   ├─ V1__20240102_100000__Create_orders.sql
   └─ V2__20240415_150000__Remove_user_fk_ref.sql (공유 DB에서 user 참조 제거)
```

</details>

<details>
<summary><strong>Q3: 마이그레이션 파일이 너무 많아지면 관리하기 어려운데, 어떻게 정리할까요?</strong></summary>

**상황:**
```
6개월 운영 후:

user-service/db/migration/
├─ V1__20240101_100000__Create_users.sql
├─ V2__20240102_100000__Add_roles.sql
├─ V3__20240105_100000__Add_permissions.sql
├─ V4__20240110_100000__Modify_role_name.sql
├─ V5__20240115_100000__Add_status.sql
├─ V6__20240120_100000__Fix_status_null.sql  ← 오류 수정
├─ V7__20240125_100000__Add_index.sql
├─ V8__20240201_100000__Rename_column.sql
├─ V9__20240205_100000__Add_audit_columns.sql
├─ V10__20240210_100000__Fix_audit.sql
├─ ... (V50까지 계속)
└─ V50__20240415_100000__Last_migration.sql

파일 수: 50개
추적 복잡도: 높음
```

**정리 전략:**

```
1️⃣ 현재 스키마 스냅샷 생성
   V0__baseline__Current_production_schema.sql
   (모든 마이그레이션을 단일 파일로 통합)

2️⃣ 기존 파일 아카이브
   mv V1__*.sql db/migration/archive/
   mv V2__*.sql db/migration/archive/
   ...
   mv V50__*.sql db/migration/archive/

3️⃣ Flyway baseline 설정
   spring.flyway.baselineVersion=50
   spring.flyway.baselineDescription="Production schema V50"

4️⃣ 신규 마이그레이션부터 다시 시작
   V51__20240420_100000__New_feature.sql
   V52__20240425_100000__Another_feature.sql
```

**결과:**

```
user-service/db/migration/
├─ baseline/ (아카이브)
│  ├─ V1__Create_users.sql
│  ├─ V2__Add_roles.sql
│  ...
│  └─ V50__Last_old_migration.sql
│
├─ current/
│  ├─ V0__baseline__Current_production_schema.sql
│  ├─ V51__New_feature.sql
│  └─ V52__Another_feature.sql

flyway_schema_history:
V0 (baseline) ✓
V51 ✓
V52 ✓
(V1-V50은 무시됨, 이미 적용됨)
```

**주의:**
```
❌ 실행 중인 마이그레이션은 절대 삭제
   (이미 DB에 적용된 파일)

✅ 정리하는 것은 아카이브일 뿐
   (새로운 환경에서도 재현 가능)
```

</details>

---

<div align="center">

**[⬅️ 이전: 마이그레이션 코드 리뷰 체크리스트](./03-code-review-checklist.md)** | **[홈으로 🏠](../README.md)** | **[다음: MSA에서의 스키마 관리 ➡️](./05-msa-schema-management.md)**

</div>
