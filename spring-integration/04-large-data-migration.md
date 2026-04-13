# 대용량 데이터 마이그레이션

---

## 🎯 핵심 질문

수백만 건의 데이터를 마이그레이션해야 할 때, 단일 UPDATE 쿼리로 처리하면 안 되는 이유는? 그리고 배치 처리, Dark Launch, 진행률 모니터링을 어떻게 구현할까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

대용량 데이터 마이그레이션이 제대로 되지 않으면:

1. **서비스 중단**: 장시간 락으로 인해 다른 쿼리들이 대기하여 응답 지연
2. **메모리 부족**: 트랜잭션 로그가 기하급수적으로 증가
3. **Replication 지연**: 슬레이브 DB가 마스터를 따라가지 못해 read replica 사용 불가
4. **배포 실패**: 마이그레이션 중 오류 발생 시 전체 롤백 → 롤백 시간도 매우 김
5. **데이터 무결성 손상**: 실패한 마이그레이션 상태에서 부분적 업데이트

---

## 😱 흔한 실수 (Before)

### 문제 1: 수천만 건을 단일 UPDATE로 처리

```sql
-- ❌ 절대 금지!
UPDATE orders SET status = 'COMPLETED' 
WHERE status = 'DONE' AND created_at < '2026-01-01';

-- 문제점:
-- 1. 5천만 건의 행을 한 번에 잠금 (EXCLUSIVE LOCK)
-- 2. Undo Log 크기: 5천만 건 × row size = 수 GB
-- 3. 다른 모든 INSERT/UPDATE/DELETE 쿼리 대기
-- 4. Replication 지연: 슬레이브가 5천만 건 처리하느라 10분 이상
-- 5. 실패 시 5천만 건 롤백
```

**실제 영향**:
```
14:00:00 마이그레이션 시작
  ↓ (1분)
14:01:00 5천만 건 스캔 및 잠금 → 다른 쿼리 응답 불가
  ↓ (9분)
14:10:00 커밋 중 
  ↓ (2분)
14:12:00 마이그레이션 완료
→ 총 12분간 서비스 응답 불가
```

### 문제 2: 배치 처리 없이 메모리 폭발

```java
// ❌ 위험: 모든 데이터를 메모리에 로드
@Bean
public Job largeDataMigrationJob() {
    return jobBuilder.get("migration")
        .start(step1())
        .build();
}

public Step step1() {
    return stepBuilder.get("step1")
        .<Order, Order>chunk(100000)  // ← 청크 크기는 100K
        .reader(new JdbcCursorItemReader<>() {{
            setSql("SELECT * FROM orders WHERE status = 'DONE'");
            setDataSource(dataSource);
            setRowMapper((rs, rowNum) -> {
                Order order = new Order();
                order.setId(rs.getLong("id"));
                // ... 100K 행 메모리에 로드
                return order;
            });
        }})
        .processor(order -> {
            order.setStatus("COMPLETED");
            return order;
        })
        .writer(orders -> {
            // 100K 행을 한 번에 INSERT
            // Undo Log 증가
        })
        .build();
}

// 문제: 
// - 100K 청크 = 약 20MB 메모리 점유
// - Undo Log = 100K × row size × 재시도 횟수
// - 네트워크 대역폭 낭비
```

### 문제 3: 진행률 모니터링 없이 무한 대기

```java
// ❌ 진행률 알 수 없음
public void migrateData() {
    List<Order> orders = orderRepository.findAllByStatus("DONE");
    
    for (Order order : orders) {
        order.setStatus("COMPLETED");
        orderRepository.save(order);
    }
    
    // 500만 건이 몇 개 처리되었는지 알 수 없음
    // 진행률 0%
    // 얼마나 더 기다려야 하나?
}
```

---

## ✨ 올바른 접근 (After)

### 올바른 Approach 1: Chunk 단위 배치 처리

**원칙**: 수천만 건을 1천~1만 건씩 나눠서 처리

```sql
-- ❌ 위험: 전체를 한 번에
-- UPDATE orders SET status = 'COMPLETED' WHERE status = 'DONE';

-- ✅ 안전: 1천 건씩 배치
-- Batch iteration:
--   Loop 1: id 1~1000
--   Loop 2: id 1001~2000
--   Loop 3: id 2001~3000
--   ... 
--   Loop N: id (N-1)*1000+1 ~ N*1000

-- 마이그레이션 파일 (V099__migrate_order_status_batch.sql)
DELIMITER //
CREATE PROCEDURE migrate_order_status_batch(
    IN batch_size INT,
    IN total_iterations INT
)
BEGIN
    DECLARE current_iteration INT DEFAULT 0;
    DECLARE affected_rows INT DEFAULT 0;
    
    WHILE current_iteration < total_iterations DO
        UPDATE orders 
        SET status = 'COMPLETED', updated_at = NOW()
        WHERE status = 'DONE' 
          AND id BETWEEN (current_iteration * batch_size + 1) 
                    AND ((current_iteration + 1) * batch_size)
        LIMIT batch_size;
        
        SET affected_rows = ROW_COUNT();
        
        -- 로그 테이블에 진행상황 기록
        INSERT INTO migration_progress (procedure_name, iteration, affected_rows, executed_at)
        VALUES ('migrate_order_status_batch', current_iteration, affected_rows, NOW());
        
        -- 커밋 (각 배치마다)
        COMMIT;
        
        -- Replication lag 모니터링
        SELECT SLEEP(1);  -- 1초 대기 (레플리카 따라잡을 시간 제공)
        
        SET current_iteration = current_iteration + 1;
    END WHILE;
    
    -- 완료 로그
    INSERT INTO migration_log (name, status, completed_at)
    VALUES ('migrate_order_status_batch', 'COMPLETED', NOW());
END //
DELIMITER ;

-- 실행 (1천 건씩, 5000회 반복 = 500만 건)
CALL migrate_order_status_batch(1000, 5000);
```

**이점**:
- 각 배치마다 COMMIT → 다른 쿼리에 기회 제공
- Undo Log 크기 제한 (배치 크기만큼)
- 실패 시 마지막 배치부터 재시작 가능
- Replication lag 제어 가능

### 올바른 Approach 2: Java Migration으로 Spring 통합

```java
// src/main/resources/db/migration/V099__migrate_order_status.sql 대신
// org/example/db/migration/V099__MigrateOrderStatusBatch.java 작성

package org.example.db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

import java.time.Instant;

@Slf4j
public class V099__MigrateOrderStatusBatch extends BaseJavaMigration {
    
    private static final int BATCH_SIZE = 1000;
    private static final int TOTAL_BATCHES = 5000;
    private static final long SLEEP_MS = 1000;  // Replication lag 고려
    
    @Override
    public void migrate(Context context) throws Exception {
        JdbcTemplate jdbc = new JdbcTemplate(context.getConnection());
        
        log.info("Starting order status migration: {} total items", 
            TOTAL_BATCHES * BATCH_SIZE);
        
        long startTime = System.currentTimeMillis();
        int totalAffected = 0;
        
        for (int iteration = 0; iteration < TOTAL_BATCHES; iteration++) {
            long iterationStart = System.currentTimeMillis();
            
            // 배치 처리
            int affected = jdbc.update(
                "UPDATE orders SET status = 'COMPLETED', updated_at = NOW() " +
                "WHERE status = 'DONE' " +
                "  AND id BETWEEN ? AND ? " +
                "LIMIT ?",
                iteration * BATCH_SIZE + 1,
                (iteration + 1) * BATCH_SIZE,
                BATCH_SIZE
            );
            
            totalAffected += affected;
            
            // 진행률 기록
            long iterationTime = System.currentTimeMillis() - iterationStart;
            
            log.info("Batch {}/{}: {} rows updated ({}ms)", 
                iteration + 1, TOTAL_BATCHES, affected, iterationTime);
            
            // Replication lag 제어
            if (affected > 0) {
                Thread.sleep(SLEEP_MS);
            }
        }
        
        long totalTime = System.currentTimeMillis() - startTime;
        log.info("Migration completed: {} rows updated in {}ms", 
            totalAffected, totalTime);
    }
}
```

**Flyway 설정** (Java migration 활성화):

```yaml
# application.yml
spring:
  flyway:
    locations: classpath:db/migration
    # Java migration 클래스도 포함 (기본값)
    # (classpath 경로에서 자동 스캔)
```

### 올바른 Approach 3: Dark Launch 패턴

**개념**: 새 컬럼에 데이터를 점진적으로 채우면서 기존 기능은 유지

**예시 - 컬럼 이름 변경 (users.name → users.full_name)**

**Step 1: 신규 컬럼 추가 (마이그레이션)**

```sql
-- V100__add_full_name_column.sql
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);

-- 기존 데이터 한 번에 복사 (이 단계에서는 최소한의 작업)
UPDATE users SET full_name = name WHERE full_name IS NULL;
```

**Step 2: 애플리케이션에서 양쪽 모두 쓰기**

```java
@Entity
@Table(name = "users")
public class User {
    // 기존 필드
    @Column(name = "name")
    private String nameOld;
    
    // 새 필드
    @Column(name = "full_name")
    private String fullName;
    
    // Setter: 양쪽에 동시에 쓰기
    public void setFullName(String fullName) {
        this.fullName = fullName;
        this.nameOld = fullName;  // 호환성 유지
    }
    
    // Getter: 새 필드 우선, 없으면 구 필드
    public String getFullName() {
        return fullName != null ? fullName : nameOld;
    }
}

@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    public void updateUser(Long id, String fullName) {
        User user = userRepository.findById(id).orElseThrow();
        
        // setter가 양쪽 필드 업데이트
        user.setFullName(fullName);
        userRepository.save(user);
        
        // INSERT/UPDATE:
        // INSERT INTO users (name, full_name) VALUES ('John', 'John')
        // UPDATE users SET name = 'John', full_name = 'John'
    }
}
```

**Step 3: 백그라운드 배치로 기존 데이터 마이그레이션**

```java
@Component
public class NameMigrationBatch {
    
    @Autowired
    private JdbcTemplate jdbc;
    
    @Scheduled(fixedRate = 60000)  // 1분마다 실행
    public void migrateOldDataBatch() {
        int updated = jdbc.update(
            "UPDATE users SET full_name = name " +
            "WHERE full_name IS NULL LIMIT 10000"
        );
        
        if (updated > 0) {
            log.info("Migrated {} users", updated);
        }
    }
}
```

**Step 4: 읽기 전환**

```java
public User getUser(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    
    // Before: return user.getNameOld();
    // After:
    return user.getFullName();  // 새 필드에서 읽음
}
```

**Step 5: 구 컬럼 제거**

```sql
-- V101__remove_old_name_column.sql (며칠 후)
ALTER TABLE users DROP COLUMN name;

-- 또는 이름 변경 후 컬럼 삭제
-- ALTER TABLE users DROP COLUMN name;
```

**Dark Launch의 장점**:
- 배포 중 서비스 중단 없음
- 실패 시 롤백 간단 (새 컬럼 무시하면 됨)
- 점진적 마이그레이션으로 리소스 분산
- 데이터 검증 기간 확보

### 올바른 Approach 4: 진행률 모니터링

```sql
-- 마이그레이션 진행률 테이블
CREATE TABLE migration_progress (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    migration_name VARCHAR(255) NOT NULL,
    iteration INT NOT NULL,
    total_iterations INT NOT NULL,
    affected_rows INT DEFAULT 0,
    status VARCHAR(50) DEFAULT 'RUNNING',  -- RUNNING, COMPLETED, FAILED
    started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    error_message TEXT
);

CREATE INDEX idx_migration_progress_name ON migration_progress(migration_name);
```

```java
@Component
@Slf4j
public class MigrationProgressMonitor {
    
    @Autowired
    private JdbcTemplate jdbc;
    
    public void recordBatch(String migrationName, int iteration, 
                           int totalIterations, int affectedRows) {
        jdbc.update(
            "INSERT INTO migration_progress " +
            "(migration_name, iteration, total_iterations, affected_rows) " +
            "VALUES (?, ?, ?, ?)",
            migrationName, iteration, totalIterations, affectedRows
        );
    }
    
    public void recordError(String migrationName, String errorMessage) {
        jdbc.update(
            "UPDATE migration_progress SET status = 'FAILED', error_message = ? " +
            "WHERE migration_name = ? AND status = 'RUNNING'",
            errorMessage, migrationName
        );
    }
    
    public MigrationStats getProgress(String migrationName) {
        return jdbc.queryForObject(
            "SELECT " +
            "  SUM(affected_rows) as totalProcessed, " +
            "  MAX(iteration) as lastIteration, " +
            "  MAX(total_iterations) as totalIterations, " +
            "  status " +
            "FROM migration_progress " +
            "WHERE migration_name = ? " +
            "GROUP BY migration_name",
            (rs, rowNum) -> MigrationStats.builder()
                .totalProcessed(rs.getLong("totalProcessed"))
                .lastIteration(rs.getInt("lastIteration"))
                .totalIterations(rs.getInt("totalIterations"))
                .status(rs.getString("status"))
                .build(),
            migrationName
        );
    }
}

@Data
@Builder
class MigrationStats {
    private Long totalProcessed;
    private Integer lastIteration;
    private Integer totalIterations;
    private String status;
    
    public double getPercentage() {
        if (totalIterations == 0) return 0;
        return (double) lastIteration / totalIterations * 100;
    }
}
```

**REST API로 진행률 조회**:

```java
@RestController
@RequestMapping("/admin/migrations")
public class MigrationAdminController {
    
    @Autowired
    private MigrationProgressMonitor monitor;
    
    @GetMapping("/progress/{name}")
    public MigrationStats getMigrationProgress(@PathVariable String name) {
        return monitor.getProgress(name);
    }
    
    // 응답 예시:
    // {
    //   "totalProcessed": 2450000,
    //   "lastIteration": 245,
    //   "totalIterations": 5000,
    //   "status": "RUNNING",
    //   "percentage": 4.9
    // }
}
```

---

## 🔬 내부 동작 원리

### 1. 배치 처리가 필요한 이유

```
단일 UPDATE 시나리오:
┌──────────────────────────────────────────┐
│ UPDATE orders SET status = 'COMPLETED'   │
│ WHERE status = 'DONE' LIMIT 5000000;     │
└──────────────────────────────────────────┘
  │
  ├─ Table Lock (EXCLUSIVE) 획득
  │   → 다른 모든 쿼리 차단
  │
  ├─ 5000만 건 스캔
  │   → Where 조건 매칭
  │
  ├─ 5000만 건 Undo Log 생성
  │   → 메모리: 5M × 100 bytes = 500MB
  │   → Disk: 5M × 100 bytes × 복사본 = 수 GB
  │
  ├─ 5000만 건 업데이트 및 WAL 기록
  │   → Replication 지연 발생
  │
  └─ COMMIT
      → 모든 변경 영구화
      
문제: 테이블 잠금 기간 = 스캔 + 업데이트 + 커밋 시간
     = 수 분 → 서비스 다운

배치 처리 시나리오:
┌──────────────────────────────────────────┐
│ UPDATE orders SET status = 'COMPLETED'   │
│ WHERE status = 'DONE'                    │
│ LIMIT 1000;  ← Batch 1                   │
└──────────────────────────────────────────┘
  │
  ├─ Table Lock (EXCLUSIVE) 획득
  ├─ 1000 건 스캔 및 업데이트
  ├─ Undo Log: 1000 × 100 bytes = 100KB
  ├─ COMMIT
  └─ Lock 해제 ← 다른 쿼리 실행 가능!
  
  ↓ (1000ms 대기)
  
┌──────────────────────────────────────────┐
│ UPDATE orders SET status = 'COMPLETED'   │
│ LIMIT 1000;  ← Batch 2                   │
└──────────────────────────────────────────┘
  │
  ├─ (락 다시 획득 → 빠름)
  ├─ 1000 건 업데이트
  ├─ COMMIT
  └─ Lock 해제 ← 다시 기회 제공
  
  ... (5000 반복)

장점:
- 락 보유 시간: 수 분 → 수 ms × 5000
- Undo Log: 500MB → 100KB
- 다른 쿼리: 차단됨 → 병렬 실행 가능
- Replication: 지연 → 실시간 동기화
```

### 2. Cursor-Based Pagination (재시작 가능한 배치)

```
ID 기반 배치 (문제):
┌─────────────────────────────────────────────┐
│ SELECT * FROM orders                        │
│ WHERE status = 'DONE'                       │
│ OFFSET 1000000 LIMIT 1000;  ← 첫 100만 행 │
└─────────────────────────────────────────────┘
  │
  ├─ DB가 0번부터 100만 번째까지 스캔
  │   (그 중 처음 100만 버림) ← 낭비!
  │
  └─ 시간 복잡도: O(n²)
  
Cursor 기반 배치 (권장):
┌─────────────────────────────────────────────┐
│ SELECT * FROM orders                        │
│ WHERE status = 'DONE' AND id > 1000000      │
│ ORDER BY id LIMIT 1000;                     │
└─────────────────────────────────────────────┘
  │
  ├─ id > 1000000인 행부터 바로 시작
  │   (스캔 불필요) ← 효율적
  │
  └─ 시간 복잡도: O(n)
  
재시작 시:
마지막 진행률: iteration 1245, last_id = 1245000

다음 배치:
SELECT * FROM orders
WHERE status = 'DONE' AND id > 1245000
ORDER BY id LIMIT 1000;

→ 정확한 지점부터 재시작 가능
```

### 3. Dark Launch의 호환성 전략

```
배포 1: 새 컬럼 추가 + 양쪽 쓰기
┌────────────────────────────────────┐
│ CREATE TABLE users (                │
│   id BIGINT PRIMARY KEY,            │
│   name VARCHAR(255),     ← 기존     │
│   full_name VARCHAR(255) ← 신규     │
│ );                                  │
└────────────────────────────────────┘

User user = getUser(1);
// user.name = "John"
// user.full_name = null (초기값)

user.setFullName("John Doe");
// setter가 양쪽 업데이트:
// user.name = "John Doe"
// user.full_name = "John Doe"

배포 2: 읽기 전환
user.getFullName()  // full_name 우선 사용

배포 3: 구 컬럼 제거
ALTER TABLE users DROP COLUMN name;

이 3단계를 통해:
- Step 1: 데이터 무결성 점검
- Step 2: 코드 호환성 점검  
- Step 3: 정리

즉시 전환 (❌ 위험):
1단계에서 바로 컬럼 제거
→ 과도기 코드와 호환성 없음
```

---

## 💻 실전 실험

### 실험 1: 배치 처리 벤치마크

**테이블 설정**:
```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    status VARCHAR(50),
    total DECIMAL(10,2),
    created_at TIMESTAMP
);

-- 500만 건 테스트 데이터 생성
INSERT INTO orders (user_id, status, total, created_at) 
SELECT 
    FLOOR(RAND() * 10000) + 1,
    'DONE',
    ROUND(RAND() * 1000, 2),
    DATE_SUB(NOW(), INTERVAL FLOOR(RAND() * 365) DAY)
FROM 
    (SELECT 1 UNION SELECT 2 UNION SELECT 3 ... UNION SELECT 5000000) t
LIMIT 5000000;
```

**Test 1: 단일 UPDATE**

```sql
-- 시작 시간: 14:00:00
UPDATE orders SET status = 'COMPLETED' 
WHERE status = 'DONE';

-- 종료 시간: 14:03:45
-- 소요 시간: 3분 45초
-- 문제: 이 시간 동안 다른 쿼리 모두 대기
```

**Test 2: 배치 처리 (1000건씩)**

```java
@Test
public void testBatchMigration() {
    JdbcTemplate jdbc = new JdbcTemplate(dataSource);
    
    int batchSize = 1000;
    int totalBatches = 5000;
    
    long startTime = System.currentTimeMillis();
    
    for (int i = 0; i < totalBatches; i++) {
        long batchStart = System.currentTimeMillis();
        
        int updated = jdbc.update(
            "UPDATE orders SET status = 'COMPLETED' " +
            "WHERE status = 'DONE' LIMIT ?",
            batchSize
        );
        
        long batchTime = System.currentTimeMillis() - batchStart;
        
        if (i % 100 == 0) {
            System.out.printf(
                "Batch %d/%d: %d rows, %dms%n",
                i + 1, totalBatches, updated, batchTime
            );
        }
        
        // 레플리카 따라잡을 시간 제공
        Thread.sleep(10);
    }
    
    long totalTime = System.currentTimeMillis() - startTime;
    System.out.printf("Total time: %d ms (%.2f minutes)%n", 
        totalTime, totalTime / 60000.0);
}

// 예상 결과:
// Batch 1/5000: 1000 rows, 15ms
// Batch 101/5000: 1000 rows, 12ms
// Batch 201/5000: 1000 rows, 14ms
// ...
// Total time: 75000 ms (1.25 minutes)
// 
// vs 단일 UPDATE: 225초
// 배치: 75초 = 3배 빠름!
```

---

### 실험 2: Dark Launch 구현

**Entity (양쪽 필드 지원)**:

```java
@Entity
@Table(name = "products")
@Data
@NoArgsConstructor
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "title")
    private String titleOld;  // 기존 필드
    
    @Column(name = "product_title")
    private String productTitle;  // 신규 필드
    
    // Setter: 양쪽에 동시에 쓰기
    public void setProductTitle(String title) {
        this.productTitle = title;
        this.titleOld = title;  // 호환성 유지
    }
    
    // Getter: 신규 필드 우선, 없으면 구 필드
    public String getProductTitle() {
        if (productTitle != null && !productTitle.isEmpty()) {
            return productTitle;
        }
        return titleOld;
    }
}
```

**마이그레이션**:

```sql
-- V101__add_product_title_column.sql
ALTER TABLE products 
ADD COLUMN product_title VARCHAR(255) DEFAULT NULL;

-- 기존 데이터 초기화 (나중에 배치로 채움)
```

**배치 마이그레이션**:

```java
@Component
@Slf4j
public class ProductTitleMigrationBatch {
    
    @Autowired
    private JdbcTemplate jdbc;
    
    private static final int BATCH_SIZE = 10000;
    
    @Scheduled(fixedRate = 60000)  // 1분마다 실행
    public void migrateProductTitles() {
        int updated = jdbc.update(
            "UPDATE products " +
            "SET product_title = title " +
            "WHERE product_title IS NULL " +
            "LIMIT ?",
            BATCH_SIZE
        );
        
        if (updated > 0) {
            log.info("Migrated {} product titles", updated);
            
            // 진행률 기록
            Integer remaining = jdbc.queryForObject(
                "SELECT COUNT(*) FROM products WHERE product_title IS NULL",
                Integer.class
            );
            
            log.info("Remaining: {} products", remaining);
        }
    }
}
```

**서비스**:

```java
@Service
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    public ProductDTO getProduct(Long id) {
        Product product = productRepository.findById(id).orElseThrow();
        
        return ProductDTO.builder()
            .id(product.getId())
            .title(product.getProductTitle())  // 신규 필드 우선
            .build();
    }
    
    public void updateProduct(Long id, String newTitle) {
        Product product = productRepository.findById(id).orElseThrow();
        
        // Setter가 양쪽 업데이트
        product.setProductTitle(newTitle);
        
        productRepository.save(product);
        // INSERT: title=newTitle, product_title=newTitle
        // UPDATE: title=newTitle, product_title=newTitle
    }
}
```

**테스트**:

```java
@SpringBootTest
public class DarkLaunchTest {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Test
    public void testDarkLaunchMigration() {
        // 1. 기존 데이터 (신규 필드 없음)
        Product oldProduct = new Product();
        oldProduct.setTitleOld("Laptop");
        // productTitle은 null
        productRepository.save(oldProduct);
        
        // 2. 조회 시 getProductTitle() 사용
        ProductDTO dto = productService.getProduct(oldProduct.getId());
        assertEquals("Laptop", dto.getTitle());  // 구 필드에서 읽음
        
        // 3. 업데이트 시 양쪽 필드 업데이트
        productService.updateProduct(oldProduct.getId(), "Gaming Laptop");
        
        // 4. 다시 조회
        Product updated = productRepository.findById(oldProduct.getId()).orElse(null);
        assertNotNull(updated);
        assertEquals("Gaming Laptop", updated.getProductTitle());  // 신규 필드
        assertEquals("Gaming Laptop", updated.getTitleOld());      // 구 필드
        
        // 5. 배치 마이그레이션이 나머지 처리
        // (별도 배치가 product_title IS NULL인 행들을 업데이트)
    }
}
```

---

### 실험 3: 진행률 모니터링 (완전 구현)

**마이그레이션 클래스**:

```java
public class V102__MigrateOrdersBatch extends BaseJavaMigration {
    
    private static final int BATCH_SIZE = 5000;
    private static final int TOTAL_ITEMS = 5000000;
    private static final int TOTAL_BATCHES = TOTAL_ITEMS / BATCH_SIZE;
    private static final long SLEEP_MS = 100;
    
    @Override
    public void migrate(Context context) throws Exception {
        Connection conn = context.getConnection();
        JdbcTemplate jdbc = new JdbcTemplate(
            new SingleConnectionDataSource(conn, false)
        );
        
        long startTime = System.currentTimeMillis();
        int totalAffected = 0;
        
        // 진행 상태 초기화
        jdbc.update(
            "INSERT INTO migration_progress " +
            "(migration_name, total_iterations, status) " +
            "VALUES (?, ?, ?)",
            "V102__MigrateOrdersBatch", TOTAL_BATCHES, "RUNNING"
        );
        
        for (int iteration = 0; iteration < TOTAL_BATCHES; iteration++) {
            try {
                long iterStart = System.currentTimeMillis();
                
                int affected = jdbc.update(
                    "UPDATE orders SET status = 'COMPLETED' " +
                    "WHERE status = 'DONE' LIMIT ?",
                    BATCH_SIZE
                );
                
                totalAffected += affected;
                long iterTime = System.currentTimeMillis() - iterStart;
                
                // 진행률 로깅
                double percentage = (double) (iteration + 1) / TOTAL_BATCHES * 100;
                long eta = calculateETA(startTime, iteration + 1, TOTAL_BATCHES);
                
                if ((iteration + 1) % 100 == 0) {
                    System.out.printf(
                        "Progress: %d/%d (%.2f%%) - %d rows - ETA: %ds%n",
                        iteration + 1, TOTAL_BATCHES, percentage,
                        totalAffected, eta
                    );
                }
                
                // 진행률 DB에 기록
                jdbc.update(
                    "UPDATE migration_progress " +
                    "SET iteration = ?, affected_rows = ?, executed_at = NOW() " +
                    "WHERE migration_name = ?",
                    iteration + 1, totalAffected, "V102__MigrateOrdersBatch"
                );
                
                if (affected > 0) {
                    Thread.sleep(SLEEP_MS);
                }
            } catch (Exception e) {
                jdbc.update(
                    "UPDATE migration_progress " +
                    "SET status = 'FAILED', error_message = ? " +
                    "WHERE migration_name = ?",
                    e.getMessage(), "V102__MigrateOrdersBatch"
                );
                throw e;
            }
        }
        
        // 완료
        jdbc.update(
            "UPDATE migration_progress " +
            "SET status = 'COMPLETED' " +
            "WHERE migration_name = ?",
            "V102__MigrateOrdersBatch"
        );
        
        long totalTime = System.currentTimeMillis() - startTime;
        System.out.printf("Migration completed: %d rows in %dms%n",
            totalAffected, totalTime);
    }
    
    private long calculateETA(long startTime, int completed, int total) {
        long elapsed = System.currentTimeMillis() - startTime;
        long rate = elapsed / completed;  // ms per item
        long remaining = total - completed;
        return (remaining * rate) / 1000;  // seconds
    }
}
```

**모니터링 REST API**:

```java
@RestController
@RequestMapping("/admin")
public class MigrationMonitorController {
    
    @Autowired
    private JdbcTemplate jdbc;
    
    @GetMapping("/migration-progress")
    public ResponseEntity<?> getMigrationProgress() {
        Map<String, Object> progress = jdbc.queryForMap(
            "SELECT " +
            "  migration_name, " +
            "  iteration, " +
            "  total_iterations, " +
            "  affected_rows, " +
            "  status, " +
            "  ROUND(iteration * 100.0 / total_iterations, 2) as percentage, " +
            "  TIMESTAMPDIFF(MINUTE, started_at, NOW()) as elapsed_minutes, " +
            "  CASE WHEN iteration > 0 " +
            "    THEN CEIL(TIMESTAMPDIFF(SECOND, started_at, NOW()) / iteration * (total_iterations - iteration) / 60) " +
            "    ELSE NULL " +
            "  END as eta_minutes " +
            "FROM migration_progress " +
            "WHERE status = 'RUNNING' " +
            "ORDER BY started_at DESC " +
            "LIMIT 1"
        );
        
        return ResponseEntity.ok(progress);
    }
    
    // 응답 예시:
    // {
    //   "migration_name": "V102__MigrateOrdersBatch",
    //   "iteration": 1250,
    //   "total_iterations": 1000,
    //   "affected_rows": 6250000,
    //   "status": "RUNNING",
    //   "percentage": 12.5,
    //   "elapsed_minutes": 5,
    //   "eta_minutes": 43
    // }
}
```

---

## 📊 성능/비용 비교

| 방식 | 처리 시간 | 락 보유 시간 | 메모리 사용 | Replication Lag | 재시작 가능 |
|------|---------|-----------|----------|----------------|-----------|
| **단일 UPDATE** | 240초 | 240초 | 500MB | 심함 | 불가능 |
| **배치 (1000건)** | 75초 | 75ms × 5K | 10MB | 낮음 | 가능 |
| **배치 (5000건)** | 35초 | 100ms × 1K | 50MB | 중간 | 가능 |
| **Dark Launch** | 지속적 | 10ms × N | 5MB | 없음 | N/A |

---

## ⚖️ 트레이드오프

### 1. 배치 크기 선택
- **작은 배치 (100)**: 락 시간 짧음, 처리 시간 김 (오버헤드 증가)
- **큰 배치 (50000)**: 처리 시간 짧음, 락 시간 김 (다른 쿼리 영향)
- **권장**: 1000~5000 (성능과 영향의 균형)

### 2. Dark Launch의 복잡성
- **장점**: 무중단 마이그레이션, 롤백 쉬움
- **단점**: 코드 복잡도 증가 (양쪽 필드 관리)
- **비용**: 개발 시간 증가, 테스트 복잡

### 3. 모니터링 오버헤드
- **장점**: 진행률 실시간 확인, 문제 조기 감지
- **단점**: 진행률 기록 로직이 성능 저하 가능
- **해결**: 배치마다 아니라 100배치마다 기록

---

## 📌 핵심 정리

1. **수백만 건 마이그레이션은 배치 처리 필수**
   - 단일 UPDATE ❌ → 배치 처리 (1000~5000건씩) ✅
   - 락 보유 시간: 분 단위 → 밀리초 단위

2. **Cursor-Based Pagination으로 재시작 가능하게**
   - OFFSET 사용 ❌ (O(n²) 복잡도)
   - ID > last_id 사용 ✅ (O(n) 복잡도)

3. **Dark Launch로 무중단 전환**
   - 즉시 전환 ❌ → 점진적 마이그레이션 ✅
   - 데이터 불일치 시간 확보

4. **진행률 모니터링으로 운영 신뢰성 확보**
   - migration_progress 테이블 생성
   - REST API로 진행률 조회 가능
   - ETA 계산으로 완료 시간 예측

5. **배포 전 테스트 환경에서 충분히 검증**
   - 배치 크기 최적화
   - Sleep 시간 조정 (Replication lag 고려)
   - 실제 크기의 10% 정도 테스트

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 배치 크기를 어떻게 정해야 할까?</strong></summary>

**A**: 데이터 크기, DB 리소스, Replication lag를 고려합니다.

**기본 공식**:
```
배치 크기 = (Row 크기 × 원하는 락 시간) / 행당 처리 시간

예시:
- Row 크기: 500 bytes
- 원하는 락 시간: 50ms
- 행당 처리 시간: 0.01ms

배치 크기 = (500 × 50) / 0.01 = 2,500,000 행

하지만 이건 너무 큼. 현실적으로:
- 200ms 락 시간: 1,000~5,000 행
- 500ms 락 시간: 5,000~10,000 행
```

**선택 기준**:
| 상황 | 배치 크기 |
|-----|---------|
| High-traffic 서비스 | 1000 |
| 일반 서비스 | 5000 |
| 야간 배치 | 10000 |
| 대용량 Read-Heavy | 20000 |

**테스트 방법**:
```java
for (int batchSize : Arrays.asList(1000, 5000, 10000)) {
    long start = System.currentTimeMillis();
    int updated = jdbc.update(
        "UPDATE orders SET status = 'COMPLETED' " +
        "WHERE status = 'DONE' LIMIT ?",
        batchSize
    );
    long time = System.currentTimeMillis() - start;
    System.out.printf("%d rows: %dms%n", batchSize, time);
}
```

결과:
```
1000 rows: 12ms
5000 rows: 45ms
10000 rows: 95ms
```

→ 서비스 영향 최소화하려면 5000 권장
</details>

<details>
<summary><strong>Q2: Dark Launch 중에 데이터 불일치가 발생하면?</strong></summary>

**A**: 재점검 배치와 데이터 검증이 필요합니다.

**문제 시나리오**:
```
배포 1: 신규 컬럼 추가 (product_title)
배포 2: 양쪽 쓰기 시작
배포 3: 배치 마이그레이션 (낮은 우선순위, 1일 소요)

중간에 버그 발생:
- product_title 업데이트 실패
- title은 업데이트됨
→ 데이터 불일치

해결:
1. 버그 수정 후 재배포
2. 불일치 데이터 재점검:

SELECT COUNT(*) FROM products
WHERE title != product_title AND product_title IS NOT NULL;

3. 재점검 배치:

UPDATE products 
SET product_title = title
WHERE title != product_title OR product_title IS NULL;

4. 검증:

SELECT * FROM products
WHERE product_title IS NULL OR title != product_title;
```

**권장사항**:
- 신규 컬럼에 버전/타임스탬프 추가
  ```sql
  ALTER TABLE products ADD COLUMN product_title_version INT DEFAULT 0;
  ```
- 배치마다 버전 증가
  ```sql
  UPDATE products SET product_title = title, product_title_version = product_title_version + 1
  WHERE product_title IS NULL;
  ```
- 검증 쿼리로 모니터링
</details>

<details>
<summary><strong>Q3: Replication lag를 어떻게 모니터링할까?</strong></summary>

**A**: MySQL Replication 메타데이터를 조회합니다.

```sql
-- Replication 상태 확인
SHOW SLAVE STATUS\G

-- 주요 필드:
-- Seconds_Behind_Master: Slave가 Master보다 몇 초 뒤처져 있나?
-- Relay_Log_Space: Relay log가 얼마나 쌓여있나?

예시 결과:
Seconds_Behind_Master: 45  ← Master보다 45초 뒤처짐

문제: 마이그레이션이 너무 빨라서 Slave가 따라가지 못함
해결: SLEEP(1) 또는 SLEEP(2) 추가 (배치 사이)
```

**진행 상황 확인 배치**:

```java
@Component
public class ReplicationLagMonitor {
    
    @Autowired
    private JdbcTemplate jdbc;
    
    @Scheduled(fixedRate = 10000)  // 10초마다
    public void monitorLag() {
        Integer secondsBehind = jdbc.queryForObject(
            "SHOW SLAVE STATUS LIMIT 1",
            "Seconds_Behind_Master",
            Integer.class
        );
        
        log.info("Replication lag: {} seconds", secondsBehind);
        
        if (secondsBehind > 60) {
            log.warn("High replication lag detected, pause migration");
            // 마이그레이션 일시 정지
        }
    }
}
```

**권장 lag 수준**:
- 0~5초: 안전
- 5~30초: 주의
- 30초 이상: 마이그레이션 일시 정지
</details>

---

<div align="center">

**[⬅️ 이전: JPA Entity와 마이그레이션 동기화](./03-jpa-entity-sync.md)** | **[홈으로 🏠](../README.md)** | **[다음: 실전 케이스 스터디 ➡️](./05-real-world-case-study.md)**

</div>
