# 테스트 데이터 관리

---

## 🎯 핵심 질문

Flyway로 관리되는 프로덕션 스키마에서 테스트 데이터는 어떻게 격리하고 관리할까? Repeatable 마이그레이션, Testcontainers, @Sql, 트랜잭션 롤백 중 어떤 방식이 언제 적합할까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

테스트 데이터 관리가 제대로 되지 않으면:

1. **테스트 간 오염**: 한 테스트가 남긴 데이터가 다른 테스트에 영향
2. **비결정적 테스트**: 같은 코드인데 테스트 실행 순서에 따라 성공/실패
3. **성능 저하**: 테스트마다 전체 DB를 초기화하느라 수십 초 소요
4. **개발 생산성 저하**: 테스트 실패 원인이 코드가 아닌 데이터 상태 때문
5. **CI/CD 불안정**: 로컬에서는 되는데 CI 파이프라인에서 안 되는 현상

---

## 😱 흔한 실수 (Before)

### 문제 1: 프로덕션 데이터를 테스트에 사용

```java
@SpringBootTest
public class OrderServiceTest {
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    public void testOrderCalculation() {
        // 문제: 프로덕션 DB의 실제 주문 데이터 사용
        // 데이터가 변경되면 테스트 실패
        Order order = orderRepository.findById(1L).orElse(null);
        
        BigDecimal total = orderService.calculateTotal(order);
        // 기댓값이 고정되어 있음
        assertEquals(new BigDecimal("150.50"), total);
        // 하지만 프로덕션 데이터가 변경되었다면? → 테스트 실패
    }
}
```

**문제점**:
- 테스트가 특정 데이터에 의존
- 같은 ID의 주문이 삭제되면 테스트 실패
- 주문 금액이 변경되면 기댓값 업데이트 필요
- CI 파이프라인에서는 다른 DB 사용하면 데이터 없음

### 문제 2: 테스트 데이터를 @BeforeEach에 하드코딩

```java
@SpringBootTest
public class UserServiceTest {
    @Autowired
    private UserRepository userRepository;
    
    @BeforeEach
    public void setUp() {
        // 문제 1: 매번 중복된 INSERT
        User user = new User();
        user.setUsername("testuser");
        user.setEmail("test@example.com");
        userRepository.save(user);
        
        // 문제 2: 데이터 정리 누락
        // 테스트 실패 시 데이터가 남아있어 다음 테스트에 영향
    }
    
    @Test
    public void testCreateUser() {
        // ...
    }
    
    @Test
    public void testFindByEmail() {
        // setUp()에서 생성한 데이터를 사용
        // 하지만 위 testCreateUser()가 실패하면?
        // 데이터가 없거나 중복될 수 있음
    }
}
```

**문제점**:
- @BeforeEach가 매번 실행되어 성능 저하
- 테스트 간 데이터 격리 불명확
- 정리 로직(@AfterEach) 누락

### 문제 3: H2 인메모리 DB로 테스트하면서 SQL 호환성 무시

```java
@SpringBootTest
@AutoConfigureTestDatabase(
    replace = AutoConfigureTestDatabase.Replace.ANY
)
public class UserRepositoryTest {
    // 문제: H2 DB로 자동 변경
    // - MySQL의 AUTO_INCREMENT는 H2의 IDENTITY로 다름
    // - MySQL의 JSON 함수는 H2에 없음
    // - COLLATE utf8mb4_unicode_ci는 H2에 다름
    
    @Test
    public void testJsonColumn() {
        // MySQL: JSON_EXTRACT(metadata, '$.key') 사용 가능
        // H2: JSON_EXTRACT 없음 → 테스트 실패
    }
}
```

**문제점**:
- H2로 테스트해도 프로덕션은 MySQL
- 테스트에서 성공해도 프로덕션에서 실패 (또는 그 반대)
- DB 호환성 검증 불가능

### 문제 4: @Sql 실행 순서 혼란

```java
@SpringBootTest
public class OrderServiceTest {
    
    @Test
    @Sql("classpath:test-data.sql")  // 순서 불명확
    public void testOrderCreation() {
        // test-data.sql이 언제 실행되나?
        // Flyway 마이그레이션 후에? 전에?
        // 트랜잭션은?
    }
}
```

**문제점**:
- @Sql 실행 순서 불명확 (BEFORE or AFTER?)
- Flyway와의 상호작용 복잡
- 트랜잭션 격리 레벨 설정 필요

---

## ✨ 올바른 접근 (After)

### 올바른 Approach 1: Repeatable 마이그레이션으로 시드 데이터 관리

```sql
-- src/main/resources/db/migration/R__seed_initial_data.sql
-- Repeatable 마이그레이션: V 마이그레이션 완료 후 매번 실행
-- 스키마 변경 시 자동으로 재실행되어 멱등성 보장

-- 1단계: 기존 데이터 DELETE (멱등성)
DELETE FROM users WHERE username IN ('admin', 'system', 'test');
DELETE FROM roles WHERE name IN ('ADMIN', 'USER', 'GUEST');

-- 2단계: 새로운 데이터 INSERT
INSERT INTO roles (id, name, description) VALUES
(1, 'ADMIN', '관리자'),
(2, 'USER', '일반 사용자'),
(3, 'GUEST', '게스트');

INSERT INTO users (id, username, email, role_id, created_at) VALUES
(1, 'admin', 'admin@example.com', 1, NOW()),
(2, 'system', 'system@example.com', 1, NOW()),
(3, 'test', 'test@example.com', 2, NOW());

-- 3단계: AUTO_INCREMENT 동기화 (선택)
ALTER TABLE users AUTO_INCREMENT = 4;
ALTER TABLE roles AUTO_INCREMENT = 4;
```

**특징**:
- V(Version) 마이그레이션 완료 후 R(Repeatable) 마이그레이션 실행
- flyway_schema_history에 `checksum` 없음 (내용 변경해도 상관없음)
- 매번 DELETE + INSERT → 멱등성 보장 (여러 번 실행해도 같은 결과)

**실행 순서**:
```
V1__create_users_table.sql (version: 1)
  ↓
V2__create_roles_table.sql (version: 2)
  ↓
R__seed_initial_data.sql (checksum 무시, 매번 재실행)
  ↓
다음 V 마이그레이션까지 계속 유효
```

### 올바른 Approach 2: Testcontainers + Flyway 조합 (권장)

```java
// pom.xml 추가
// <dependency>
//     <groupId>org.testcontainers</groupId>
//     <artifactId>testcontainers</artifactId>
//     <version>1.19.3</version>
//     <scope>test</scope>
// </dependency>
// <dependency>
//     <groupId>org.testcontainers</groupId>
//     <artifactId>mysql</artifactId>
//     <version>1.19.3</version>
//     <scope>test</scope>
// </dependency>
// <dependency>
//     <groupId>org.testcontainers</groupId>
//     <artifactId>junit-jupiter</artifactId>
//     <version>1.19.3</version>
//     <scope>test</scope>
// </dependency>

package com.example;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;

@SpringBootTest
@Testcontainers
public class OrderServiceIntegrationTest {
    
    @Container
    @ServiceConnection
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test")
        .withInitScript("init.sql");  // 선택사항
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private OrderService orderService;
    
    @Test
    public void testOrderCalculation() {
        // 테스트 시작: 완벽하게 격리된 MySQL 8.0 컨테이너
        // Flyway가 자동으로 src/main/resources/db/migration 실행
        // R__seed 마이그레이션도 자동 실행 → 테스트 데이터 준비됨
        
        Order order = orderRepository.findById(1L)  // R__seed에서 삽입된 데이터
            .orElse(null);
        assertNotNull(order);
        
        BigDecimal total = orderService.calculateTotal(order);
        assertEquals(new BigDecimal("150.50"), total);
    }
    
    @Test
    public void testOrderCreation() {
        // 이 테스트는 이전 테스트와 완전히 격리됨
        // 각 테스트마다 새로운 컨테이너 인스턴스는 아니지만
        // 데이터는 격리됨 (트랜잭션 or 각 테스트마다 정리)
        
        Order order = new Order();
        order.setUserId(1L);
        order.setTotal(new BigDecimal("100.00"));
        
        Order saved = orderRepository.save(order);
        assertNotNull(saved.getId());
    }
}
```

**장점**:
1. **실제 MySQL 사용**: 프로덕션과 동일한 DB 호환성
2. **완벽한 격리**: 각 테스트마다 독립적인 컨테이너
3. **Flyway 자동 실행**: 마이그레이션과 시드 데이터 자동 준비
4. **성능**: Docker 캐싱으로 첫 실행 후 빠름

**주의사항**:
- Docker 필수 (로컬 개발 환경에 설치 필요)
- CI 파이프라인에서 Docker-in-Docker 필요할 수도 있음

### 올바른 Approach 3: @Sql과 명확한 실행 순서

```java
package com.example.integration;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.jdbc.Sql;
import org.springframework.test.context.jdbc.Sql.ExecutionPhase;
import org.springframework.transaction.annotation.Transactional;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;

@SpringBootTest
@Sql(
    scripts = "classpath:test-data-setup.sql",
    executionPhase = ExecutionPhase.BEFORE_TEST_METHOD
)
@Sql(
    scripts = "classpath:test-data-cleanup.sql",
    executionPhase = ExecutionPhase.AFTER_TEST_METHOD
)
public class UserServiceSqlTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    public void testFindByEmail() {
        // 실행 순서:
        // 1. Flyway 마이그레이션 (Spring Boot 시작 시)
        // 2. test-data-setup.sql 실행 (BEFORE_TEST_METHOD)
        // 3. 테스트 메서드 실행
        // 4. test-data-cleanup.sql 실행 (AFTER_TEST_METHOD)
        
        User user = userRepository.findByEmail("setup@example.com")
            .orElse(null);
        assertNotNull(user);
        assertEquals("setup_user", user.getUsername());
    }
}
```

**test-data-setup.sql**:
```sql
-- 테스트 시작 전 데이터 준비
INSERT INTO users (username, email, created_at) VALUES
('setup_user', 'setup@example.com', NOW());
```

**test-data-cleanup.sql**:
```sql
-- 테스트 종료 후 정리
DELETE FROM users WHERE email = 'setup@example.com';
```

### 올바른 Approach 4: 트랜잭션 롤백을 통한 격리

```java
package com.example.integration;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@Transactional  // 각 테스트를 트랜잭션으로 감싸기
public class UserRepositoryTransactionalTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    public void testSaveUser() {
        // 트랜잭션 시작
        User user = new User();
        user.setUsername("txn_user");
        user.setEmail("txn@example.com");
        
        User saved = userRepository.save(user);
        assertNotNull(saved.getId());
        
        User found = userRepository.findById(saved.getId()).orElse(null);
        assertNotNull(found);
        
        // 테스트 종료: 트랜잭션 자동 ROLLBACK
        // → INSERT된 사용자가 DB에서 제거됨
    }
    
    @Test
    public void testFindByEmail() {
        // 이 테스트는 이전 테스트의 데이터 영향 없음
        // (이전 테스트의 INSERT가 ROLLBACK됨)
        
        User found = userRepository.findByEmail("txn@example.com")
            .orElse(null);
        assertNull(found);  // 없음 (이전 테스트에서 롤백됨)
    }
}
```

**주의사항**:
```java
@Transactional
public class Test {
    @Test
    public void testWithLazyLoading() {
        User user = userRepository.findById(1L).orElse(null);
        
        // 문제: Lazy Loading된 orders가 로드되지 않음
        // (트랜잭션 외부에서는 LazyInitializationException 발생 가능)
        List<Order> orders = user.getOrders();  // ← 위험!
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Repeatable 마이그레이션의 멱등성 보장

```
Flyway 시작
  ↓
Version 마이그레이션 정렬 (V1, V2, V3, ...)
  ↓
각 V 마이그레이션 순서대로 실행
  ├─ V 마이그레이션만 flyway_schema_history에 저장 (checksum 있음)
  └─ 이미 실행된 V 마이그레이션 재실행 불가능 (checksum 검증)
  ↓
모든 V 마이그레이션 완료 후 R 마이그레이션 정렬
  ↓
각 R 마이그레이션 순서대로 실행
  ├─ 매번 실행 (checksum 무시)
  └─ flyway_schema_history에 최신 내용 기반으로 저장
  ↓
다음 실행 시 R 마이그레이션 재비교
  ├─ 내용이 같으면: 스킵
  └─ 내용이 다르면: 재실행
```

**멱등성 패턴**:
```sql
-- ❌ 비멱등성: 매번 다른 결과
INSERT INTO users (username, email) VALUES ('john', 'john@example.com');
-- 매번 실행하면 중복된 행 증가

-- ✅ 멱등성: 매번 같은 결과
DELETE FROM users WHERE username = 'john';
INSERT INTO users (username, email) VALUES ('john', 'john@example.com');
-- 매번 실행해도 결과 동일 (1행)

-- ✅ 멱등성: ON DUPLICATE KEY UPDATE
INSERT INTO users (username, email) VALUES ('john', 'john@example.com')
ON DUPLICATE KEY UPDATE email = 'john@example.com';
-- 중복 키 있으면 UPDATE, 없으면 INSERT
```

### 2. Testcontainers의 격리 메커니즘

```
@Testcontainers 테스트 클래스 로드
  ↓
@Container 필드 발견
  ↓
JUnit Extension 활성화 (TestcontainersExtension)
  ↓
테스트 클래스 인스턴스화 전
  ├─ 컨테이너 시작 (docker run)
  ├─ 포트 바인딩
  └─ @ServiceConnection으로 DataSource 자동 구성
  ↓
Spring Boot 애플리케이션 시작
  ├─ DataSource: 컨테이너로부터 (localhost:random_port)
  ├─ Flyway 마이그레이션 자동 실행
  └─ R__seed 마이그레이션도 자동 실행
  ↓
테스트 메서드 1 실행
  ├─ 컨테이너 DB 접근
  └─ 데이터 삽입/변경
  ↓
테스트 메서드 2 실행
  ├─ 같은 컨테이너 DB 접근 (재시작 아님)
  └─ 이전 테스트 데이터는 그대로 남음 (격리 필요시 @Transactional)
  ↓
모든 테스트 완료 후
  ├─ 컨테이너 종료 (docker stop)
  └─ 볼륨 삭제
```

### 3. @Transactional 롤백 메커니즘

```
@Transactional 메서드 진입
  ↓
ApplicationContext에서 PlatformTransactionManager 획득
  ↓
TransactionManager.getTransaction(definition) 호출
  ├─ 트랜잭션 시작 (BEGIN)
  └─ Savepoint 생성 (중첩 가능)
  ↓
메서드 바디 실행
  ├─ 모든 DB 작업이 현재 트랜잭션 내에서 실행됨
  └─ 다른 스레드/트랜잭션은 영향 없음 (격리 수준 따라 다름)
  ↓
메서드 정상 종료
  ↓
TransactionInterceptor가 예외 확인
  └─ 예외 없음 → ROLLBACK
  └─ @Transactional(propagation = SUPPORTS) 등이면 COMMIT
  ↓
메서드 호출자는 INSERT/UPDATE/DELETE가 "없던 것처럼" 본다
```

---

## 💻 실전 실험

### 실험 1: Repeatable 마이그레이션으로 멱등성 확보

**파일 구조**:
```
src/main/resources/db/migration/
├── V1__create_tables.sql
└── R__seed_initial_data.sql
```

**V1__create_tables.sql**:
```sql
CREATE TABLE IF NOT EXISTS categories (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    category_id BIGINT NOT NULL,
    price DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(id)
);
```

**R__seed_initial_data.sql**:
```sql
-- Step 1: 멱등성을 위한 DELETE
DELETE FROM products WHERE name IN ('Laptop', 'Mouse', 'Keyboard');
DELETE FROM categories WHERE name IN ('Electronics', 'Accessories');

-- Step 2: INSERT (새로운 데이터)
INSERT INTO categories (id, name) VALUES
(1, 'Electronics'),
(2, 'Accessories');

INSERT INTO products (id, category_id, name, price) VALUES
(1, 1, 'Laptop', 999.99),
(2, 1, 'Monitor', 299.99),
(3, 2, 'Mouse', 29.99),
(4, 2, 'Keyboard', 79.99);

-- Step 3: AUTO_INCREMENT 동기화
ALTER TABLE categories AUTO_INCREMENT = 3;
ALTER TABLE products AUTO_INCREMENT = 5;
```

**테스트 코드**:
```java
package com.example.integration;

import com.example.domain.Category;
import com.example.domain.Product;
import com.example.repository.CategoryRepository;
import com.example.repository.ProductRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.math.BigDecimal;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@DataJpaTest
@Testcontainers
public class RepeatableMigrationTest {
    
    @Container
    @ServiceConnection
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private CategoryRepository categoryRepository;
    
    @Test
    public void testSeedDataLoaded() {
        // R__seed_initial_data.sql이 자동 실행됨
        
        List<Category> categories = categoryRepository.findAll();
        assertEquals(2, categories.size());
        
        List<Product> products = productRepository.findAll();
        assertEquals(4, products.size());
        
        Product laptop = productRepository.findById(1L).orElse(null);
        assertNotNull(laptop);
        assertEquals("Laptop", laptop.getName());
        assertEquals(new BigDecimal("999.99"), laptop.getPrice());
    }
    
    @Test
    public void testAddNewProduct() {
        // 기존 시드 데이터는 그대로 있음
        Category electronics = categoryRepository.findById(1L).orElse(null);
        assertNotNull(electronics);
        
        // 새 상품 추가
        Product newProduct = new Product();
        newProduct.setName("Tablet");
        newProduct.setCategory(electronics);
        newProduct.setPrice(new BigDecimal("499.99"));
        
        Product saved = productRepository.save(newProduct);
        assertNotNull(saved.getId());
        
        // 전체 상품: 시드 4개 + 새로 추가한 1개 = 5개
        long count = productRepository.count();
        assertEquals(5, count);
    }
}
```

**실행 결과**:
```
[INFO] Starting 'docker pull mysql:8.0'
[INFO] Docker image pulled successfully
[INFO] Starting container: mysql:8.0
[INFO] Container started in 2.5 seconds
[INFO] Running Flyway migration...
[INFO] Successfully applied 1 migration
[INFO] Executing R__seed_initial_data.sql
[INFO] Test testSeedDataLoaded: PASSED (215ms)
[INFO] Test testAddNewProduct: PASSED (189ms)
[INFO] Shutting down container...
```

---

### 실험 2: Testcontainers + Flyway 완전 통합

**application-test.properties**:
```properties
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.jpa.hibernate.ddl-auto=validate
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baselineOnMigrate=true
```

**OrderServiceIntegrationTest.java**:
```java
package com.example.integration;

import com.example.domain.Order;
import com.example.domain.User;
import com.example.repository.OrderRepository;
import com.example.repository.UserRepository;
import com.example.service.OrderService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.test.context.TestPropertySource;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.math.BigDecimal;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@Testcontainers
@TestPropertySource(locations = "classpath:application-test.properties")
public class OrderServiceIntegrationTest {
    
    @Container
    @ServiceConnection
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private OrderService orderService;
    
    private User testUser;
    
    @BeforeEach
    public void setUp() {
        // Flyway 마이그레이션과 R__seed는 이미 완료
        // R__seed에서 'test_user' 생성됨
        testUser = userRepository.findByUsername("test_user")
            .orElseGet(() -> {
                User user = new User();
                user.setUsername("test_user");
                user.setEmail("test@example.com");
                return userRepository.save(user);
            });
    }
    
    @Test
    public void testCreateOrder() {
        Order order = new Order();
        order.setUser(testUser);
        order.setTotal(new BigDecimal("150.50"));
        
        Order saved = orderRepository.save(order);
        assertNotNull(saved.getId());
        
        // DB에서 다시 조회
        Order fetched = orderRepository.findById(saved.getId()).orElse(null);
        assertNotNull(fetched);
        assertEquals(new BigDecimal("150.50"), fetched.getTotal());
    }
    
    @Test
    public void testCalculateOrderTotal() {
        // R__seed에서 미리 생성된 주문 데이터 사용
        List<Order> orders = orderRepository.findByUser(testUser);
        
        if (!orders.isEmpty()) {
            Order order = orders.get(0);
            BigDecimal total = orderService.calculateTotal(order);
            assertNotNull(total);
            assertTrue(total.compareTo(BigDecimal.ZERO) > 0);
        }
    }
    
    @Test
    public void testOrderUpdate() {
        Order order = new Order();
        order.setUser(testUser);
        order.setTotal(new BigDecimal("100.00"));
        
        Order saved = orderRepository.save(order);
        Long orderId = saved.getId();
        
        // 업데이트
        saved.setTotal(new BigDecimal("120.00"));
        orderRepository.save(saved);
        
        // 다시 조회
        Order updated = orderRepository.findById(orderId).orElse(null);
        assertNotNull(updated);
        assertEquals(new BigDecimal("120.00"), updated.getTotal());
    }
}
```

**Domain 클래스**:
```java
// User.java
@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String username;
    
    private String email;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders = new ArrayList<>();
}

// Order.java
@Entity
@Table(name = "orders")
@Data
@NoArgsConstructor
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
    
    @Column(precision = 10, scale = 2)
    private BigDecimal total;
}
```

---

### 실험 3: @Sql + @Transactional 복합 사용

**application.yml**:
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/testdb
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
```

**test-data-common.sql**:
```sql
-- 모든 테스트에서 필요한 기본 데이터
INSERT INTO roles (id, name) VALUES
(1, 'ADMIN'),
(2, 'USER'),
(3, 'GUEST');

INSERT INTO users (id, role_id, username, email) VALUES
(1, 1, 'admin', 'admin@example.com'),
(2, 2, 'john', 'john@example.com'),
(3, 2, 'jane', 'jane@example.com');
```

**UserServiceSqlTest.java**:
```java
package com.example.integration;

import com.example.domain.User;
import com.example.repository.UserRepository;
import com.example.service.UserService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.context.annotation.Import;
import org.springframework.test.context.jdbc.Sql;
import org.springframework.test.context.jdbc.Sql.ExecutionPhase;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@DataJpaTest
@Import(UserService.class)
@Transactional
@Sql(
    scripts = "classpath:test-data-common.sql",
    executionPhase = ExecutionPhase.BEFORE_TEST_METHOD
)
public class UserServiceSqlTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private UserService userService;
    
    @Test
    @Sql(
        scripts = "classpath:test-data-promotion.sql",
        executionPhase = ExecutionPhase.BEFORE_TEST_METHOD
    )
    public void testPromoteUserToAdmin() {
        // 실행 순서:
        // 1. Flyway 마이그레이션 (DataJpaTest 시작 시)
        // 2. @Sql(BEFORE) - test-data-common.sql
        // 3. @Sql(BEFORE) - test-data-promotion.sql (메서드 레벨)
        // 4. 트랜잭션 시작
        // 5. 테스트 메서드 실행
        // 6. 트랜잭션 롤백
        // 7. @Sql(AFTER) 실행 (없음)
        
        User user = userRepository.findByUsername("john").orElse(null);
        assertNotNull(user);
        assertEquals(2, user.getRole().getId());  // USER role
        
        userService.promoteToAdmin(user.getId());
        
        User promoted = userRepository.findById(user.getId()).orElse(null);
        assertNotNull(promoted);
        assertEquals(1, promoted.getRole().getId());  // ADMIN role
    }
    
    @Test
    public void testFindAllUsers() {
        // test-data-common.sql만 실행
        List<User> users = userRepository.findAll();
        assertEquals(3, users.size());
    }
    
    @Test
    public void testDeleteUser() {
        User user = userRepository.findByUsername("jane").orElse(null);
        assertNotNull(user);
        Long userId = user.getId();
        
        userRepository.delete(user);
        
        User deleted = userRepository.findById(userId).orElse(null);
        assertNull(deleted);
        
        // 하지만 트랜잭션 롤백되므로
        // 다음 테스트에서는 jane이 그대로 있음
    }
    
    @Test
    public void testJaneStillExists() {
        // 이전 testDeleteUser()의 DELETE가 롤백됨
        User jane = userRepository.findByUsername("jane").orElse(null);
        assertNotNull(jane);  // 존재함!
    }
}
```

**test-data-promotion.sql**:
```sql
-- testPromoteUserToAdmin에서만 사용
-- john 사용자를 프로모션 대상으로 준비
UPDATE users SET role_id = 2 WHERE username = 'john';
```

**실행 결과**:
```
[INFO] Running com.example.integration.UserServiceSqlTest
[INFO] 1. testPromoteUserToAdmin
  [SQL] test-data-common.sql 실행
  [SQL] test-data-promotion.sql 실행
  [TEST] john 사용자를 ADMIN으로 프로모션
  [ROLLBACK] 모든 변경사항 취소
[INFO] 2. testFindAllUsers
  [SQL] test-data-common.sql 실행 (새로 실행, 이전 롤백 후)
  [TEST] 3명의 사용자 확인
  [ROLLBACK]
[INFO] 3. testDeleteUser
  [SQL] test-data-common.sql 실행
  [TEST] jane 사용자 삭제
  [ROLLBACK] DELETE 롤백 → jane이 다시 복구됨
[INFO] 4. testJaneStillExists
  [SQL] test-data-common.sql 실행
  [TEST] jane이 여전히 존재함 (이전 테스트의 영향 없음)
  [ROLLBACK]
[INFO] Tests run: 4, Failures: 0
```

---

## 📊 성능/비용 비교

| 접근법 | 초기화 시간 | 테스트당 시간 | 격리 수준 | 프로덕션 유사도 |
|------|----------|-------------|---------|--------------|
| **Repeatable SQL** | 100-200ms | 50-100ms | 낮음 (공유 DB) | 높음 (실제 SQL) |
| **Testcontainers** | 2-5초 (첫 실행), 100ms (캐시) | 200-500ms | 높음 (독립 컨테이너) | 최고 (실제 MySQL) |
| **@Sql** | 100-200ms | 100-150ms | 중간 (트랜잭션 필요) | 높음 (실제 SQL) |
| **@Transactional** | 100-200ms | 50-100ms | 높음 (자동 롤백) | 낮음 (트랜잭션 격리) |
| **H2 인메모리** | 50-100ms | 30-50ms | 높음 | 낮음 (호환성 차이) |

---

## ⚖️ 트레이드오프

### 1. Repeatable 마이그레이션의 관리 복잡성
- **장점**: 프로덕션과 동일한 SQL, 멱등성 보장
- **단점**: DELETE + INSERT 패턴이 복잡할 수 있음
- **해결**: 테스트 데이터 별도 폴더 (`db/migration/test`)

### 2. Testcontainers의 리소스 소비
- **장점**: 완벽한 격리, 실제 MySQL 호환성
- **단점**: Docker 의존성, 초기 시작 느림
- **해결**: Docker 캐싱, Testcontainers 재사용 (static @Container)

### 3. @Sql의 실행 순서 복잡성
- **장점**: 유연한 데이터 준비
- **단점**: BEFORE/AFTER 이해 필요
- **해결**: 명확한 이름: `test-data-setup.sql`, `test-data-cleanup.sql`

### 4. 트랜잭션 격리와 Lazy Loading
- **장점**: 간단한 테스트 정리
- **단점**: LazyInitializationException 위험
- **해결**: `@Transactional(readOnly = false)` 또는 Eager Loading

---

## 📌 핵심 정리

1. **Repeatable 마이그레이션은 DELETE + INSERT로 멱등성 보장**
   - R__ 파일은 V__ 마이그레이션 후 매번 재실행
   - 스키마 변경 후 자동으로 시드 데이터 갱신

2. **Testcontainers는 가장 신뢰할 수 있는 통합 테스트**
   - 실제 MySQL 8.0 사용 (호환성 검증 가능)
   - 각 테스트 완전 격리 (컨테이너 재사용은 가능)
   - 처음엔 느리지만 Docker 캐싱으로 빨라짐

3. **@Sql은 유연하지만 복잡성 주의**
   - BEFORE_TEST_METHOD: 테스트 전 데이터 준비
   - AFTER_TEST_METHOD: 테스트 후 정리
   - Flyway와 실행 순서 명확히 할 것

4. **@Transactional은 가장 간단하지만 주의 필요**
   - 각 테스트가 자동으로 ROLLBACK
   - Lazy Loading할 때는 별도 처리 필요
   - 트랜잭션 격리 수준 고려

5. **프로덕션과 동일한 환경에서 테스트 권장**
   - H2 인메모리보다 Testcontainers + MySQL
   - 테스트에서는 성공하는데 프로덕션에서 실패하는 현상 방지

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: Repeatable 마이그레이션에서 DELETE 없이 INSERT만 하면 안 될까?</strong></summary>

**A**: 비멱등성 때문에 위험합니다.

**DELETE 없는 경우**:
```sql
-- R__seed_data.sql (DELETE 없음)
INSERT INTO users (username, email) VALUES ('admin', 'admin@example.com');
INSERT INTO users (username, email) VALUES ('system', 'system@example.com');
```

**결과**:
- 첫 실행: 2행 삽입 ✅
- 두 번째 실행: 중복 키 오류 ❌ (UNIQUE 제약)
- 또는 두 번째 실행: 4행이 됨 ❌ (PRIMARY KEY 없으면)

**DELETE가 있는 경우**:
```sql
-- R__seed_data.sql (DELETE 포함)
DELETE FROM users WHERE username IN ('admin', 'system');
INSERT INTO users (username, email) VALUES ('admin', 'admin@example.com');
INSERT INTO users (username, email) VALUES ('system', 'system@example.com');
```

**결과**:
- 첫 실행: 2행 ✅
- 두 번째 실행: 여전히 2행 ✅ (멱등성 보장)

**또는 ON DUPLICATE KEY UPDATE 사용**:
```sql
INSERT INTO users (username, email) VALUES ('admin', 'admin@example.com')
ON DUPLICATE KEY UPDATE email = 'admin@example.com';
```

**권장**: DELETE + INSERT (명확) > ON DUPLICATE KEY UPDATE (고급)
</details>

<details>
<summary><strong>Q2: Testcontainers를 사용하면 CI 파이프라인이 Docker 필요해서 복잡해질까?</strong></summary>

**A**: 최신 CI 환경에서는 대부분 Docker 지원합니다.

**GitHub Actions 예시**:
```yaml
name: Test
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    # Docker 자동 지원 (별도 설정 불필요)
    
    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
      - name: Run tests with Testcontainers
        run: mvn test
        # 그냥 실행하면 됨 (Docker 자동)
```

**로컬 개발**:
- Docker Desktop 설치 필요 (한 번만)
- `mvn test` 실행하면 자동으로 컨테이너 실행

**요구사항**:
- CI: docker 데몬 실행 필요 (대부분의 CI 플랫폼 기본 지원)
- 로컬: Docker Desktop 또는 Docker 설치

**만약 Docker 불가능한 경우**:
- Testcontainers + JdbcTemplate 대신
- H2 인메모리 + Flyway (빠르지만 호환성 주의)
</details>

<details>
<summary><strong>Q3: 트랜잭션 롤백으로 테스트할 때 자신이 생성한 ID를 못 쓰는데 괜찮을까?</strong></summary>

**A**: 걱정할 필요 없습니다. 트랜잭션 내에서는 ID를 얻을 수 있습니다.

**코드**:
```java
@SpringBootTest
@Transactional
public class IdGenerationTest {
    @Autowired
    private UserRepository userRepository;
    
    @Test
    public void testIdGeneration() {
        User user = new User();
        user.setUsername("test");
        
        // 트랜잭션 내에서 save하면 ID 할당됨
        User saved = userRepository.save(user);
        Long id = saved.getId();  // ✅ ID 얻을 수 있음
        assertNotNull(id);
        
        // 같은 트랜잭션 내에서 조회
        User found = userRepository.findById(id).orElse(null);
        assertNotNull(found);  // ✅ 찾을 수 있음
        
        // 테스트 종료: 트랜잭션 롤백
        // INSERT가 취소되지만 테스트 내에서는 이미 ID를 사용했음
    }
}
```

**Flush/Clear 주의**:
```java
@Test
public void testFlushAndClear() {
    User user = new User();
    user.setUsername("test");
    userRepository.save(user);
    Long id = user.getId();  // ✅ ID 있음
    
    // 문제: flush + clear
    userRepository.flush();  // DB에 강제 flush
    entityManager.clear();   // 캐시 비우기
    
    // 이 후 조회
    User found = userRepository.findById(id).orElse(null);
    assertNotNull(found);  // 여전히 찾을 수 있음
    
    // 이유: 같은 트랜잭션, flush()가 해도 트랜잭션 내
    // 롤백은 테스트 종료 후
}
```

**결론**: 트랜잭션 롤백은 테스트 **외부**에는 영향 없고, 테스트 **내부**에서는 정상 작동 ✅
</details>

---

<div align="center">

**[⬅️ 이전: Spring Boot + Flyway 자동 설정](./01-spring-boot-flyway-autoconfigure.md)** | **[홈으로 🏠](../README.md)** | **[다음: JPA Entity와 마이그레이션 동기화 ➡️](./03-jpa-entity-sync.md)**

</div>
