# 실전 케이스 스터디

---

## 🎯 핵심 질문

실제 프로덕션 환경에서 마주치는 세 가지 전형적인 시나리오(컬럼 이름 변경, 복합 인덱스 추가, 테이블 분리)를 어떻게 안전하게 처리할까? 각 단계별 마이그레이션 파일, Entity 변경, 배포 순서, 롤백 전략은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

이론을 배웠어도 실제 상황에서는:

1. **한 번에 모든 걸 바꾸기 유혹**: "그냥 다 한 번에 바꾸면 안 될까?" → 배포 실패
2. **롤백 계획 부족**: 중간에 오류 발생했을 때 어떻게 복구할 것인가?
3. **데이터 검증 누락**: 스키마는 바뀌었지만 데이터가 정합성 있는가?
4. **성능 악화**: 인덱스 추가 중 다른 쿼리는 어떻게 처리하나?
5. **테스트 부족**: 로컬에서는 되는데 프로덕션에서는 왜 실패하나?

---

## 케이스 1: 컬럼 이름 변경 (users.name → users.full_name)

### Before: 현재 상태

**Database**:
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (id, name, email) VALUES
(1, 'John Doe', 'john@example.com'),
(2, 'Jane Smith', 'jane@example.com');
-- ... 500만 건 이상
```

**Entity**:
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;  // ← 변경해야 함
    
    @Column(unique = true)
    private String email;
}
```

**API 응답**:
```json
{
  "id": 1,
  "name": "John Doe",
  "email": "john@example.com"
}
```

### Target: 변경 후

**Database**:
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    full_name VARCHAR(255) NOT NULL,  -- ← 변경
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Entity**:
```java
@Column(name = "full_name", nullable = false)
private String fullName;  // ← 변경
```

**API 응답**:
```json
{
  "id": 1,
  "fullName": "John Doe",
  "email": "john@example.com"
}
```

### 전체 마이그레이션 전략 (3 단계 배포)

```
배포 1: 신규 컬럼 추가 (Expand)
├─ V001__add_full_name_column.sql
├─ User.java: name + fullName 동시 관리
└─ API 응답: name만 반환 (호환성)

배포 2: 읽기/쓰기 전환 (Switch)
├─ (마이그레이션 없음)
├─ User.java: fullName 우선 사용
├─ API 응답: fullName 반환
└─ 데이터 검증

배포 3: 구 컬럼 제거 (Contract)
├─ V002__drop_name_column.sql
├─ User.java: fullName만 사용
└─ API 응답: fullName만 반환
```

### Step 1: Expand (신규 컬럼 추가)

**마이그레이션 파일 (V001__add_full_name_column.sql)**:

```sql
-- Step 1: 신규 컬럼 추가 (NULL 허용)
ALTER TABLE users ADD COLUMN full_name VARCHAR(255) DEFAULT NULL;

-- Step 2: 기존 데이터로 채우기 (배치)
-- 500만 건을 1만 건씩 처리 (1시간 소요)
UPDATE users SET full_name = name 
WHERE full_name IS NULL LIMIT 10000;

-- 이후 배치 배포로 처리
-- (마이그레이션 파일에서 모두 처리하지 않음)

-- Step 3: 인덱스 추가 (선택)
CREATE INDEX idx_full_name ON users(full_name);
```

**Entity.java**:

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    // 기존 필드 (읽기만, 곧 제거)
    @Column(name = "name")
    private String nameOld;
    
    // 신규 필드 (읽기/쓰기)
    @Column(name = "full_name", nullable = false)
    private String fullName;
    
    @Column(unique = true)
    private String email;
    
    // Setter: 양쪽에 동시에 쓰기
    public void setFullName(String fullName) {
        this.fullName = fullName;
        this.nameOld = fullName;
    }
    
    // Getter: 기존 코드와의 호환성 (name)
    public String getName() {
        return nameOld;  // 아직 name 필드 사용
    }
    
    // Getter: 신규 필드 (fullName)
    public String getFullName() {
        return fullName != null ? fullName : nameOld;
    }
}
```

**Controller**:

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
        User user = userService.getUser(id);
        
        return ResponseEntity.ok(UserDTO.builder()
            .id(user.getId())
            .name(user.getName())  // 여전히 name 필드 반환 (호환성)
            .email(user.getEmail())
            .build());
    }
    
    @PostMapping
    public ResponseEntity<UserDTO> createUser(@RequestBody CreateUserRequest req) {
        User user = new User();
        user.setFullName(req.getName());  // setter가 양쪽 업데이트
        user.setEmail(req.getEmail());
        
        userService.save(user);
        
        return ResponseEntity.status(201).body(UserDTO.builder()
            .id(user.getId())
            .name(user.getName())
            .email(user.getEmail())
            .build());
    }
}
```

**배치 작업** (마이그레이션 후 데이터 채우기):

```java
@Component
@Slf4j
public class FullNameMigrationBatch {
    
    @Autowired
    private JdbcTemplate jdbc;
    
    @Scheduled(fixedRate = 60000)  // 1분마다
    public void migrateFullName() {
        int updated = jdbc.update(
            "UPDATE users SET full_name = name " +
            "WHERE full_name IS NULL LIMIT 10000"
        );
        
        if (updated > 0) {
            log.info("Migrated {} users", updated);
            
            Integer remaining = jdbc.queryForObject(
                "SELECT COUNT(*) FROM users WHERE full_name IS NULL",
                Integer.class
            );
            log.info("Remaining: {} users", remaining);
        }
    }
}
```

**배포 1 체크리스트**:
```
[ ] 마이그레이션 파일 작성
[ ] 로컬에서 마이그레이션 테스트
[ ] Entity 업데이트 (양쪽 필드 관리)
[ ] API는 name만 반환 (호환성)
[ ] PR 리뷰: SQL + Entity + Controller 함께
[ ] 테스트: 신규 사용자 생성/조회 테스트
[ ] 프로덕션 배포
[ ] 배치 배포로 전체 데이터 마이그레이션 (시간 소요)
```

---

### Step 2: Switch (읽기/쓰기 전환)

**전제 조건**:
- full_name 컬럼이 거의 다 채워짐 (99% 이상)
- 신규 사용자는 모두 full_name으로 생성됨

**Entity.java**:

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name")
    @Deprecated  // 곧 제거될 필드임을 명시
    private String nameOld;
    
    @Column(name = "full_name", nullable = false)
    private String fullName;
    
    @Column(unique = true)
    private String email;
    
    // Getter: 신규 필드 우선
    public String getFullName() {
        return fullName != null ? fullName : nameOld;
    }
    
    // 기존 필드는 호환성을 위해 유지
    public String getName() {
        return getFullName();  // fullName에서 읽음
    }
}
```

**Controller**:

```java
@GetMapping("/{id}")
public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
    User user = userService.getUser(id);
    
    return ResponseEntity.ok(UserDTO.builder()
        .id(user.getId())
        .fullName(user.getFullName())  // ← fullName 반환 (변경)
        .email(user.getEmail())
        .build());
}
```

**데이터 검증**:

```java
@Component
@Slf4j
public class FullNameValidation {
    
    @Autowired
    private JdbcTemplate jdbc;
    
    @PostConstruct
    public void validateData() {
        // 검증 1: NULL 값 확인
        Integer nullCount = jdbc.queryForObject(
            "SELECT COUNT(*) FROM users WHERE full_name IS NULL",
            Integer.class
        );
        
        if (nullCount > 100) {  // 100건 이상 NULL이면 경고
            log.warn("Found {} users with NULL full_name", nullCount);
            throw new DataInconsistencyException(
                "Too many NULL values in full_name column"
            );
        }
        
        // 검증 2: name과 full_name 불일치
        Integer mismatchCount = jdbc.queryForObject(
            "SELECT COUNT(*) FROM users " +
            "WHERE name != full_name AND full_name IS NOT NULL",
            Integer.class
        );
        
        if (mismatchCount > 0) {
            log.warn("Found {} users with name != full_name", mismatchCount);
        }
        
        log.info("Data validation passed: " +
            "NULL count = {}, Mismatch count = {}", nullCount, mismatchCount);
    }
}
```

**API 호환성** (필드명 변경 시 고려사항):

```java
// Option 1: JSON 필드명 매핑 (Jackson)
public class UserDTO {
    
    @JsonProperty("fullName")  // JSON에서 fullName
    private String fullName;
    
    @JsonAlias("name")  // 또는 name도 수용 (구 클라이언트 호환)
    private String nameForCompatibility;
}

// Option 2: API 버전 분리
@GetMapping("/v1/users/{id}")  // 구 API: name 반환
public ResponseEntity<UserDTOv1> getUserV1(@PathVariable Long id) { ... }

@GetMapping("/v2/users/{id}")  // 신 API: fullName 반환
public ResponseEntity<UserDTOv2> getUserV2(@PathVariable Long id) { ... }
```

**배포 2 체크리스트**:
```
[ ] 모든 사용자가 full_name으로 마이그레이션되었는지 확인
[ ] Entity: getFullName() 우선 사용으로 변경
[ ] API: 응답에 fullName 포함 (name도 유지 권장)
[ ] 테스트: 기존 필드와 신 필드 모두 동작 확인
[ ] Canary 배포: 10%의 트래픽부터 시작
[ ] 모니터링: 에러 로그 감시
[ ] 점진적 롤아웃: 100%까지 확대
```

---

### Step 3: Contract (구 컬럼 제거)

**전제 조건**:
- Switch 배포 후 최소 1주일 이상 운영 (버그 없음)
- 클라이언트들이 신 API 응답 형식에 적응
- 모든 직원이 fullName 사용법 숙지

**마이그레이션 파일 (V002__drop_name_column.sql)**:

```sql
-- 최종 검증
SELECT COUNT(*) as null_count FROM users WHERE full_name IS NULL;

-- 마지막 체크: name과 full_name 일치
SELECT COUNT(*) as mismatch_count FROM users 
WHERE name IS NOT NULL AND full_name != name;

-- 컬럼 제거
ALTER TABLE users DROP COLUMN name;

-- 인덱스 확인 (선택)
SHOW INDEXES FROM users;

-- 로그
-- Removed name column from users table
-- Full name migration completed
```

**Entity.java**:

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "full_name", nullable = false)
    private String fullName;
    
    @Column(unique = true)
    private String email;
    
    @Column(nullable = false)
    private LocalDateTime createdAt;
    
    // nameOld 필드 제거
    
    public String getFullName() {
        return fullName;
    }
}
```

**배포 3 체크리스트**:
```
[ ] 1주일 이상 Switch 배포 운영 확인
[ ] 마이그레이션 파일 작성 및 검증 쿼리 포함
[ ] Entity: nameOld 필드 완전 제거
[ ] 전체 코드베이스에서 .getName() 제거 또는 .getFullName()으로 변경
[ ] IDE 정적 분석: @Deprecated 경고 없음
[ ] 모든 테스트 통과
[ ] 프로덕션 배포
[ ] 모니터링: 예상 밖의 데이터 조회 없음
```

**완전 마이그레이션 타임라인**:

```
일차 1: 배포 1 (Expand)
├─ 14:00 마이그레이션 + Entity 변경 배포
├─ 14:05 배치 시작 (1000건/분)
└─ 07:00 배치 완료 (500만 건, ~13시간)

일차 2-7: 검증 기간 (7일)
├─ 배치 진행 상황 모니터링
├─ 데이터 검증 (NULL, 불일치)
└─ 버그 리포트 없음

일차 8: 배포 2 (Switch)
├─ 09:00 Entity 변경 + API 응답 변경 배포
├─ 09:05 모니터링 (에러율, 응답시간)
└─ 23:00 모니터링 완료 (정상)

일차 9-15: 안정성 검증 (7일)
├─ 클라이언트 업데이트 준비
├─ API 응답 형식 검증
└─ 레거시 클라이언트 호환성 확인

일차 16: 배포 3 (Contract)
├─ 10:00 마이그레이션 + Entity 변경 배포
├─ 10:05 모니터링
└─ 17:00 마이그레이션 완료 및 검증
```

---

## 케이스 2: 복합 인덱스 추가 (orders 테이블)

### 문제 상황

**쿼리**:
```sql
-- 자주 사용되는 쿼리
SELECT * FROM orders 
WHERE user_id = 123 
  AND status = 'COMPLETED' 
  AND created_at >= '2025-01-01'
ORDER BY created_at DESC;

-- 현재 인덱스:
-- KEY idx_user_id (user_id)
-- KEY idx_status (status)
-- KEY idx_created_at (created_at)

-- 문제: 3개 인덱스 각각 사용하거나, index merge 발생
-- 실행 계획:
-- | Using where; Using intersect(idx_user_id,idx_status,idx_created_at)
-- → 비효율적
```

### 목표

```sql
-- 목표 인덱스:
CREATE INDEX idx_user_status_created 
ON orders(user_id, status, created_at DESC);

-- 실행 계획 (인덱스 사용 후):
-- | Using index (Covering index 가능)
-- → 매우 효율적
```

### 마이그레이션 전략 (2 단계)

**문제**: 인덱스 추가 시 테이블이 잠김 (Lock은 짧지만, 대용량 테이블이면 시간 소요)

**해결**: MySQL Online DDL 또는 gh-ost 사용

### Step 1: 인덱스 추가 (Online DDL)

**마이그레이션 파일 (V003__add_composite_index.sql)**:

```sql
-- MySQL 8.0 Online DDL: ALGORITHM=INSTANT 또는 INPLACE
ALTER TABLE orders 
ADD INDEX idx_user_status_created (user_id, status, created_at DESC)
ALGORITHM=INPLACE, LOCK=NONE;

-- ALGORITHM 옵션:
-- COPY: 오래된 방식, 테이블 복사 (Lock 발생, 느림) ❌
-- INPLACE: 제자리 수정 (Lock 짧음) ✅
-- INSTANT: 극초단 (JSON 변경만 가능)

-- LOCK 옵션:
-- DEFAULT: 가능한 한 Lock 최소화
-- NONE: Lock 없음 (INPLACE만 가능)
-- SHARED: 읽기만 가능 (쓰기 불가)
-- EXCLUSIVE: 완전 Lock

-- 검증
SELECT * FROM information_schema.STATISTICS
WHERE TABLE_NAME = 'orders' 
  AND INDEX_NAME = 'idx_user_status_created';
```

**성능 비교**:

```
테이블 크기: 10억 건

❌ ALGORITHM=COPY (구 방식):
- 정렬: 10분
- 테이블 복사: 20분
- 테이블 전환: 1분
- 총 시간: 31분
- Lock: 전체 시간 (테이블 사용 불가)

✅ ALGORITHM=INPLACE (신 방식):
- 인덱스 생성: 15분
- Lock: 수 ms (시작/종료 시만)
- 총 시간: 15분
- 사용자 영향: 무시할 수준
```

**테스트**:

```java
@Test
public void testCompositeIndexUsage() {
    JdbcTemplate jdbc = new JdbcTemplate(dataSource);
    
    // 인덱스 추가 전 실행 계획
    String explainBefore = jdbc.queryForObject(
        "EXPLAIN SELECT * FROM orders " +
        "WHERE user_id = 123 " +
        "  AND status = 'COMPLETED' " +
        "  AND created_at >= '2025-01-01' " +
        "ORDER BY created_at DESC",
        (rs, rowNum) -> rs.getString("Extra")
    );
    System.out.println("Before: " + explainBefore);
    
    // 인덱스 추가
    jdbc.execute(
        "ALTER TABLE orders " +
        "ADD INDEX idx_user_status_created " +
        "(user_id, status, created_at DESC) " +
        "ALGORITHM=INPLACE, LOCK=NONE"
    );
    
    // 인덱스 추가 후 실행 계획
    String explainAfter = jdbc.queryForObject(
        "EXPLAIN SELECT * FROM orders " +
        "WHERE user_id = 123 " +
        "  AND status = 'COMPLETED' " +
        "  AND created_at >= '2025-01-01' " +
        "ORDER BY created_at DESC",
        (rs, rowNum) -> rs.getString("Extra")
    );
    System.out.println("After: " + explainAfter);
    
    // 성능 비교
    long timeBefore = timeQuery(jdbc, 
        "SELECT * FROM orders " +
        "WHERE user_id = 123 AND status = 'COMPLETED'");
    
    long timeAfter = timeQuery(jdbc,
        "SELECT * FROM orders " +
        "WHERE user_id = 123 AND status = 'COMPLETED'");
    
    System.out.printf("Performance improvement: %.1f%%", 
        (timeBefore - timeAfter) * 100.0 / timeBefore);
}
```

### Step 2: 검증 및 모니터링

**마이그레이션 검증**:

```sql
-- 1. 인덱스 존재 확인
SELECT * FROM information_schema.STATISTICS
WHERE TABLE_NAME = 'orders'
  AND INDEX_NAME = 'idx_user_status_created';

-- 2. 인덱스 통계 확인
ANALYZE TABLE orders;
SELECT STAT_VALUE FROM mysql.innodb_index_stats
WHERE TABLE_NAME = 'orders'
  AND INDEX_NAME = 'idx_user_status_created'
  AND STAT_NAME = 'n_diff_pfx01';  -- 첫 컬럼(user_id)의 distinct 값

-- 3. 인덱스 크기 확인
SELECT 
    INDEX_NAME,
    SEQ_IN_INDEX,
    COLUMN_NAME,
    (STAT_VALUE * 8 / 1024 / 1024) as size_mb
FROM mysql.innodb_index_stats
WHERE TABLE_NAME = 'orders'
  AND INDEX_NAME = 'idx_user_status_created';
```

**쿼리 성능 모니터링**:

```java
@Component
@Slf4j
public class IndexPerformanceMonitor {
    
    @Autowired
    private JdbcTemplate jdbc;
    
    @Scheduled(fixedRate = 300000)  // 5분마다
    public void monitorIndexUsage() {
        // Performance Schema에서 인덱스 사용 통계
        String query = 
            "SELECT OBJECT_NAME, COUNT_STAR as usage_count, " +
            "       COUNT_READ, COUNT_WRITE " +
            "FROM performance_schema.table_io_waits_summary_by_table " +
            "WHERE OBJECT_NAME = 'orders'";
        
        Map<String, Object> stats = jdbc.queryForMap(query);
        log.info("Index usage for orders table: {}", stats);
        
        // 느려진 쿼리 감지
        String slowQueryCheck =
            "SELECT SQL_TEXT, COUNT(*) as freq, " +
            "       AVG(TIMER_WAIT) / 1000000000 as avg_ms " +
            "FROM performance_schema.events_statements_summary_by_digest " +
            "WHERE DIGEST_TEXT LIKE '%FROM orders%' " +
            "GROUP BY SQL_TEXT " +
            "HAVING avg_ms > 100 " +
            "ORDER BY avg_ms DESC";
        
        List<Map<String, Object>> slowQueries = 
            jdbc.queryForList(slowQueryCheck);
        
        if (!slowQueries.isEmpty()) {
            log.warn("Slow queries detected: {}", slowQueries);
        }
    }
}
```

---

## 케이스 3: 테이블 분리 마이그레이션 (1:N 분리)

### Before: 정규화 전

**orders 테이블** (모든 주문 정보):

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(50),
    item_1_product_id BIGINT,
    item_1_quantity INT,
    item_1_price DECIMAL(10,2),
    item_2_product_id BIGINT,
    item_2_quantity INT,
    item_2_price DECIMAL(10,2),
    item_3_product_id BIGINT,
    item_3_quantity INT,
    item_3_price DECIMAL(10,2),
    -- ... item_10까지 (너무 많음)
    total DECIMAL(10,2)
);

-- 문제:
-- 1. item_11을 추가하려면? → NULL 컬럼 추가 필요
-- 2. 대부분의 행이 item_3, 4, 5를 사용하지 않음 → NULL 낭비
-- 3. 쿼리: "특정 상품의 모든 주문" → JOIN 복잡
```

### After: 정규화 후

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    order_date TIMESTAMP,
    status VARCHAR(50)
);

CREATE TABLE order_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    KEY idx_order_id (order_id)
);

-- 장점:
-- 1. 유연한 아이템 개수 지원
-- 2. NULL 낭비 없음
-- 3. 간단한 JOIN으로 쿼리 가능
```

### 마이그레이션 전략 (4 단계)

**핵심 원칙**: 기존 서비스 중단 없이 점진적 전환

### Step 1: 신규 테이블 생성

**마이그레이션 파일 (V004__create_order_items_table.sql)**:

```sql
-- 1단계: 신규 테이블 생성
CREATE TABLE order_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL DEFAULT 1,
    price DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    KEY idx_order_id (order_id),
    KEY idx_product_id (product_id)
);

-- 2단계: 기존 orders 데이터에서 items 추출 및 삽입
-- 배치로 처리 (마이그레이션 파일에서 전체 처리하지 않음)
INSERT INTO order_items (order_id, product_id, quantity, price)
SELECT 
    id,
    item_1_product_id,
    item_1_quantity,
    item_1_price
FROM orders 
WHERE item_1_product_id IS NOT NULL;

-- 주의: 5백만 건의 items를 한 번에 INSERT하면 안 됨!
-- 대신 배치 배포로 처리
```

**배치 마이그레이션** (별도 배포):

```java
@Component
@Slf4j
public class OrderItemsMigrationBatch {
    
    @Autowired
    private JdbcTemplate jdbc;
    
    @Scheduled(fixedRate = 60000)
    public void migrateOrderItems() {
        // 배치 1: item_1 마이그레이션
        int item1 = jdbc.update(
            "INSERT INTO order_items (order_id, product_id, quantity, price) " +
            "SELECT id, item_1_product_id, item_1_quantity, item_1_price " +
            "FROM orders " +
            "WHERE item_1_product_id IS NOT NULL " +
            "  AND id NOT IN (SELECT DISTINCT order_id FROM order_items) " +
            "LIMIT 10000"
        );
        
        // 배치 2: item_2 마이그레이션
        int item2 = jdbc.update(
            "INSERT INTO order_items (order_id, product_id, quantity, price) " +
            "SELECT id, item_2_product_id, item_2_quantity, item_2_price " +
            "FROM orders " +
            "WHERE item_2_product_id IS NOT NULL " +
            "  AND id NOT IN (...)"
            // 복잡해지므로 Java로 처리
        );
        
        log.info("Migrated items: {} from item_1, {} from item_2", 
            item1, item2);
    }
}
```

### Step 2: 코드에서 양쪽 모두 사용

**Entity**:

```java
// Order.java (기존)
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private Long userId;
    
    @Column(name = "status")
    private String status;
    
    // 기존 방식: orders 테이블에서 읽음
    @Transient
    private List<OrderItemLegacy> itemsLegacy = new ArrayList<>();
    
    // 신규 방식: order_items 테이블에서 읽음
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items = new ArrayList<>();
    
    // 통합 메서드
    public List<OrderItem> getAllItems() {
        if (!items.isEmpty()) {
            return items;  // 신규 테이블에서 우선 읽음
        }
        // 신규 테이블이 비어있으면 구 테이블에서 읽음
        return convertLegacyItems();
    }
    
    public void addItem(Product product, int quantity, BigDecimal price) {
        OrderItem item = new OrderItem();
        item.setProduct(product);
        item.setQuantity(quantity);
        item.setPrice(price);
        items.add(item);
        
        // 호환성: 구 테이블에도 저장
        setLegacyItem(product.getId(), quantity, price);
    }
}

// OrderItem.java (신규)
@Entity
@Table(name = "order_items")
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;
    
    @Column(nullable = false)
    private Long productId;
    
    @Column(nullable = false)
    private Integer quantity;
    
    @Column(nullable = false)
    private BigDecimal price;
}
```

**Service**:

```java
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private OrderItemRepository orderItemRepository;
    
    public Order createOrder(CreateOrderRequest req) {
        Order order = new Order();
        order.setUserId(req.getUserId());
        order.setStatus("PENDING");
        
        orderRepository.save(order);  // orders 테이블에 저장
        
        // 아이템 추가: 신규 테이블에만 저장
        for (OrderItemRequest itemReq : req.getItems()) {
            OrderItem item = new OrderItem();
            item.setOrder(order);
            item.setProductId(itemReq.getProductId());
            item.setQuantity(itemReq.getQuantity());
            item.setPrice(itemReq.getPrice());
            
            orderItemRepository.save(item);  // order_items에 저장
        }
        
        return order;
    }
    
    public OrderDTO getOrder(Long id) {
        Order order = orderRepository.findById(id).orElseThrow();
        
        // 통합 메서드: 신규 or 구 테이블에서 읽음
        List<OrderItem> items = order.getAllItems();
        
        return OrderDTO.builder()
            .id(order.getId())
            .userId(order.getUserId())
            .items(items.stream()
                .map(item -> OrderItemDTO.builder()
                    .productId(item.getProductId())
                    .quantity(item.getQuantity())
                    .price(item.getPrice())
                    .build())
                .collect(Collectors.toList()))
            .build();
    }
}
```

### Step 3: 구 테이블 정리

**마이그레이션 파일 (V005__drop_legacy_columns.sql)**:

```sql
-- 사전 검증: 모든 주문의 아이템이 order_items로 마이그레이션되었나?
SELECT COUNT(*) as unmigrated_count
FROM orders o
WHERE EXISTS (
    SELECT 1 FROM orders
    WHERE item_1_product_id IS NOT NULL 
      AND id NOT IN (SELECT DISTINCT order_id FROM order_items)
)
LIMIT 1;

-- 위 쿼리 결과가 0이면 안전하게 컬럼 제거 가능

-- 단계 1: 기존 컬럼 제거 (한 번에 모두 제거)
ALTER TABLE orders 
DROP COLUMN item_1_product_id,
DROP COLUMN item_1_quantity,
DROP COLUMN item_1_price,
DROP COLUMN item_2_product_id,
DROP COLUMN item_2_quantity,
DROP COLUMN item_2_price,
-- ... item_10까지 ...
DROP COLUMN total;

-- 단계 2: total 컬럼이 필요하면 generated column으로 대체
ALTER TABLE orders 
ADD COLUMN total DECIMAL(10,2) GENERATED ALWAYS AS (
    (SELECT SUM(price * quantity) FROM order_items 
     WHERE order_items.order_id = orders.id)
) STORED;
```

### Step 4: 최종 정리

**Entity 정리**:

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private Long userId;
    
    @Column(name = "status")
    private String status;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items = new ArrayList<>();
    
    // 구 필드(itemsLegacy 등) 완전 제거
    // 구 메서드(convertLegacyItems 등) 완전 제거
}
```

**마이그레이션 타임라인**:

```
배포 1: 신규 테이블 생성
├─ V004: order_items 테이블 생성
├─ Entity: 양쪽 모두 지원
└─ Service: 신규 테이블 우선 사용

배포 2-7: 데이터 마이그레이션 (6일)
├─ 배치: 매일 10만 건씩 마이그레이션
├─ 검증: 데이터 일치성 확인
└─ 모니터링: 비정상 쿼리 감시

배포 8: 구 테이블 정리
├─ V005: 구 컬럼 제거
├─ Entity: 신규 필드만 사용
└─ Service: 신규 테이블만 참조

완료: 정규화 완료
```

---

## 📊 세 케이스의 공통 패턴

| 단계 | 컬럼명 변경 | 인덱스 추가 | 테이블 분리 |
|------|----------|----------|----------|
| **1. 준비** | 신규 컬럼 추가 | 인덱스 추가 | 신규 테이블 생성 |
| **2. 마이그레이션** | 데이터 복사 (배치) | (해당 없음) | 데이터 추출 (배치) |
| **3. 양쪽 지원** | Entity + 양쪽 쓰기 | 쿼리 최적화 | Entity + 양쪽 읽기 |
| **4. 검증** | NULL, 불일치 확인 | 인덱스 사용 확인 | 데이터 무결성 확인 |
| **5. 전환** | 읽기 전환 | (자동) | 읽기 전환 |
| **6. 정리** | 구 컬럼 제거 | (필요시 정리) | 구 테이블 제거 |
| **소요 기간** | 2-3주 | 1-2일 | 2-3주 |

---

## 📌 핵심 정리

1. **복잡한 마이그레이션은 여러 배포로 나누기**
   - 한 번에 모두 하려고 하면 실패 가능성 높음
   - Expand → Switch → Contract 패턴 사용

2. **데이터 마이그레이션은 배치로 처리**
   - 500만 건 이상이면 수 ms씩 여러 번
   - 락 시간 제어로 서비스 영향 최소화

3. **각 단계마다 검증과 모니터링**
   - NULL 값, 데이터 불일치 확인
   - 쿼리 성능, 에러율 모니터링
   - 최소 1주일 안정화 기간 필수

4. **롤백 계획 항상 준비**
   - 각 배포 단계마다 롤백 방법 문서화
   - 실제 프로덕션에서 롤백 드릴 수행

5. **테스트 환경에서 충분히 연습**
   - 프로덕션과 동일한 크기의 테스트 데이터
   - 실제 시간 측정으로 배포 일정 예측

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 배포 중간에 오류가 발생하면 롤백을 어떻게 진행할까?</strong></summary>

**A**: 배포 단계별로 다른 전략이 필요합니다.

**Case 1: Step 1 (Expand) 중 오류**
```
상황: V001 마이그레이션 실패, 신규 컬럼 생성 중단

롤백 전략:
1. ALTER 취소 (자동 롤백)
2. Entity에서 fullName 필드 제거
3. 배포 취소

영향: 최소 (마이그레이션만 실패, 데이터 변경 없음)
```

**Case 2: Step 2 (Switch) 중 오류**
```
상황: Entity 변경 배포 후 API 응답 형식 오류

롤백 전략:
1. API를 이전 버전으로 롤백 (name 필드 반환)
   - UserDTO 필드 재수정
   - Controller 원복
   
2. Entity는 유지 (fullName 필드 존재해도 문제 없음)
3. 2일 뒤 재배포 (버그 수정 후)

영향: 일시적 API 응답 형식 변경 (클라이언트 혼동)
```

**Case 3: Step 3 (Contract) 중 오류**
```
상황: name 컬럼 제거 중 데이터 손상 우려

롤백 불가능! (이미 컬럼 삭제됨)

대신 즉시:
1. Entity에 name 필드 재추가
2. ALTER를 통해 name 컬럼 복구
   - 백업에서 복구 (이전 바이너리 로그 사용)
   
3. 향후 예방:
   - Step 1/2 안정화 기간 충분히 (3주 이상)
   - Contract 전 모든 클라이언트 업데이트 완료 확인
```

**권장 롤백 계획**:
```
배포 1 (Expand): 저위험
├─ 롤백 방법: 마이그레이션 파일 제거, Entity 원복
└─ 소요 시간: 10분

배포 2 (Switch): 중위험
├─ 롤백 방법: API 응답 포맷 원복
└─ 소요 시간: 5분

배포 3 (Contract): 고위험
├─ 롤백 방법: 없음 (이미 삭제된 데이터)
├─ 예방: Step 1/2 이후 최소 3주 대기
└─ 재해 대비: 매일 백업 + WAL 보존
```
</details>

<details>
<summary><strong>Q2: 대기 중인 SELECT 쿼리가 많을 때 인덱스를 추가하면?</strong></summary>

**A**: ALGORITHM=INPLACE를 사용하면 문제가 거의 없습니다.

**시나리오**:
```
17:00 피크타임, 동시 연결 수: 5000
SELECT 쿼리 초당: 10000 QPS

인덱스 추가 명령 실행:
ALTER TABLE orders ADD INDEX idx_user_status_created...

영향:
1. 인덱스 생성 시작 (15분)
2. 대기 중인 SELECT 쿼리: 계속 실행 (영향 없음)
3. 새로운 INSERT/UPDATE: 약간 느려짐 (인덱스 생성 때문)
4. Lock 시점: 시작 수 ms + 완료 수 ms (무시할 수준)
```

**성능 모니터링**:
```sql
-- 인덱스 생성 진행률 확인
SELECT * FROM PERFORMANCE_SCHEMA.EVENTS_STAGES_CURRENT
WHERE EVENT_NAME LIKE '%ALTER%'
  AND OBJECT_NAME = 'orders';

-- 응답 시간 모니터링
SELECT 
    TIMER_WAIT / 1000000000 as duration_s,
    SQL_TEXT
FROM PERFORMANCE_SCHEMA.EVENTS_STATEMENTS_CURRENT
WHERE SQL_TEXT LIKE '%SELECT%orders%'
ORDER BY TIMER_WAIT DESC
LIMIT 10;
```

**주의사항**:
```
❌ ALGORITHM=COPY 사용 금지 (피크타임)
→ 전체 테이블 복사로 30분 소요 (Lock 발생)

✅ ALGORITHM=INPLACE 권장 (피크타임도 안전)
→ 제자리 수정으로 15분 (Lock 거의 없음)

만약 ALGORITHM=INPLACE 불가능하면:
- 피크타임 아닌 시간 선택 (자정~새벽)
- 또는 온라인 마이그레이션 도구(gh-ost) 사용
```
</details>

<details>
<summary><strong>Q3: 1:N 테이블 분리 후 기존 쿼리의 성능은 어떻게 변할까?</strong></summary>

**A**: 대부분 개선되지만 특정 쿼리는 더 느려질 수 있습니다.

**개선되는 쿼리**:
```sql
-- Before: orders 테이블에서 item_1~10 모두 로드
SELECT * FROM orders WHERE user_id = 123;
-- → 전체 행 크기: 600 bytes (item 컬럼들 포함)

-- After: orders만 로드 (item 컬럼 없음)
SELECT * FROM orders WHERE user_id = 123;
-- → 행 크기: 100 bytes
-- 성능: 6배 빠름! (버퍼풀 효율 개선)

-- 또한 필요한 데이터만 로드
SELECT o.id, o.user_id, o.status FROM orders o WHERE user_id = 123;
-- → 50 bytes (매우 빠름)
```

**더 느려지는 쿼리**:
```sql
-- Before: 조건 없이 전체 아이템 조회
SELECT item_1_product_id, item_1_quantity, item_1_price 
FROM orders 
WHERE user_id = 123;
-- → 한 번의 테이블 스캔

-- After: JOIN 필요
SELECT oi.product_id, oi.quantity, oi.price 
FROM orders o 
JOIN order_items oi ON o.id = oi.order_id 
WHERE o.user_id = 123;
-- → 두 개 테이블 스캔 + JOIN 오버헤드 (약간 느림)

하지만 order_items에 idx_order_id가 있으면 오히려 더 빠를 수 있음
```

**종합 분석**:

| 쿼리 | Before | After | 개선도 |
|-----|--------|-------|------|
| orders 전체 조회 | 1000ms | 100ms | 10배 빠름 ✅ |
| orders + items JOIN | N/A | 150ms | N/A |
| 특정 주문의 items | 구현 안 됨 | 50ms | 새로운 기능 ✅ |
| 전체 데이터 로드 | 200ms | 250ms | 25% 느림 ❌ |

**결론**: 전체적으로 개선. JOIN으로 인한 약간의 성능 저하는 정규화의 이점에 비해 무시할 수준.
</details>

---

<div align="center">

**[⬅️ 이전: 대용량 데이터 마이그레이션](./04-large-data-migration.md)** | **[홈으로 🏠](../README.md)**

</div>
