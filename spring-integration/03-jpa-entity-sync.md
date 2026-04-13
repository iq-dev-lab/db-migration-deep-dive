# JPA Entity와 마이그레이션 동기화

---

## 🎯 핵심 질문

Entity 클래스 변경과 Flyway 마이그레이션을 어떻게 동기화해야 배포 시 스키마와 코드 불일치로 인한 오류를 방지할 수 있을까? 그리고 `ddl-auto=validate`가 정확히 어떤 불일치를 잡아내고, 어떤 것은 놓칠까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Entity와 마이그레이션이 동기화되지 않으면:

1. **배포 중 서비스 중단**: Entity에 새 필드 추가 → 마이그레이션 누락 → 서버 시작 실패
2. **데이터 손상**: 마이그레이션은 했는데 Entity가 구 버전 → Null 저장 또는 예외 발생
3. **Pull Request 리뷰 혼란**: 스키마 변경과 코드 변경이 분리되어 리뷰자가 전체 그림을 못 봄
4. **롤백 불가능**: 마이그레이션은 스키마 변경했는데 코드 롤백하면 호환성 깨짐
5. **테스트 실패**: 로컬에서는 성공해도 프로덕션 마이그레이션 순서 차이로 실패

---

## 😱 흔한 실수 (Before)

### 문제 1: Entity 변경 후 마이그레이션 생략

```java
// User.java (변경됨)
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String username;
    
    @Column(nullable = false)
    private String email;
    
    // ❌ 새 필드 추가
    @Column(nullable = false)
    private String phone;  // ← 마이그레이션 파일은 없음!
}
```

**배포 시나리오**:
```
1. 배포 시작
   ↓
2. Flyway 마이그레이션 실행 (변경 없음)
   ↓
3. Spring Boot 시작
   ↓
4. JPA가 Entity 로드 → User.phone 필드 발견
   ↓
5. Validation: 'phone' 컬럼이 DB에 없음
   ↓
6. @Column(nullable=false) → validate 실패
   ↓
7. org.hibernate.HibernateException: 
   "Missing column in database: phone in table users"
   ↓
8. 애플리케이션 시작 실패 💥
```

### 문제 2: ddl-auto=validate가 잡지 못하는 불일치

```java
@Entity
@Table(name = "users")
public class User {
    @Column(name = "email")
    private String userEmail;  // ✅ validate가 잡음 (컬럼 존재 확인)
    
    @Column(name = "age", nullable = false)  // ❌ validate가 놓침
    private Integer age;  // nullable=false이지만 DB는 nullable=true
    
    @Column(name = "zipcode", length = 5)  // ❌ validate가 놓침
    private String zipcode;  // Entity는 length 5, DB는 VARCHAR(255)
    
    @Version
    private Long version;  // ❌ validate가 놓침 (OCC 버전 필드)
}
```

**데이터베이스 상태**:
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(100),
    age INT NULL,           -- nullable=true (Entity와 다름)
    zipcode VARCHAR(255),   -- length 255 (Entity의 5와 다름)
    -- version 컬럼 없음 (OCC 사용 불가)
);
```

**결과**:
- ✅ validate 통과 (컬럼은 존재하니까)
- ❌ 런타임 에러 (nullable 제약 위반 또는 예상 밖의 데이터)

### 문제 3: Entity-First 워크플로우로 스키마 생성

```java
// application.yml
spring:
  jpa:
    hibernate:
      ddl-auto: update  // ❌ 위험! 프로덕션에서는 절대 금지

// Entity 변경 후 로컬에서 테스트
@Entity
public class Product {
    @Column(nullable = false)
    private String sku;
    
    @Column
    private String description;
}

// 로컬 DB에서 자동 생성됨:
// ALTER TABLE products ADD COLUMN sku VARCHAR(255) NOT NULL;
// ALTER TABLE products ADD COLUMN description VARCHAR(255);

// 문제: DDL 변경 내용을 마이그레이션 파일로 추출하지 않음
// → 다른 환경에서 같은 스키마 변경 없음
```

---

## ✨ 올바른 접근 (After)

### 올바른 Approach 1: Schema-First 워크플로우

**원칙**: Entity는 Database 스키마를 따른다 (반대가 아님)

**프로세스**:

```
1. SQL 마이그레이션 파일 작성 (V002__add_phone_column.sql)
   ↓
2. 마이그레이션 파일 테스트 (로컬 DB에서 실행)
   ↓
3. Entity 클래스 업데이트 (마이그레이션 파일과 일치하도록)
   ↓
4. JPA validate 실행 (ddl-auto=validate)
   ↓
5. 한 커밋에 마이그레이션 파일 + Entity 코드 포함
```

**예시 - 컬럼 추가**:

```sql
-- V002__add_phone_column.sql
ALTER TABLE users 
ADD COLUMN phone VARCHAR(20) NOT NULL DEFAULT '';

-- 기존 행에 기본값 설정
UPDATE users SET phone = 'N/A' WHERE phone = '';

-- 기본값 제거 (새로운 행은 반드시 값이 필요)
ALTER TABLE users MODIFY COLUMN phone VARCHAR(20) NOT NULL;
```

```java
// User.java (마이그레이션 후에 수정)
@Entity
@Table(name = "users")
public class User {
    @Column(nullable = false, length = 20)
    private String phone;
}
```

**한 커밋**:
```
Commit: "Add phone field to users table"
├─ V002__add_phone_column.sql (스키마)
└─ User.java (코드)
```

### 올바른 Approach 2: ddl-auto=validate로 불일치 감지

```yaml
# application.yml
spring:
  jpa:
    hibernate:
      ddl-auto: validate  # Entity와 DB 스키마 검증만 수행
```

**Entity 정의** (올바른 예):

```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, length = 100, unique = true)
    private String sku;  // DB: VARCHAR(100) NOT NULL UNIQUE
    
    @Column(length = 1000)
    private String description;  // DB: VARCHAR(1000)
    
    @Column(nullable = false)
    private BigDecimal price;  // DB: DECIMAL(10,2) NOT NULL
    
    @Column(nullable = false)
    private LocalDateTime createdAt;  // DB: TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
}
```

**데이터베이스** (마이그레이션으로 생성):

```sql
-- V001__create_products_table.sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    sku VARCHAR(100) NOT NULL UNIQUE,
    description VARCHAR(1000),
    price DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sku ON products(sku);
```

**validate 실행 시**:
- ✅ 통과: 모든 Entity 필드가 DB에 존재하고 타입 일치

### 올바른 Approach 3: validate가 놓치는 제약 사항 사전 계획

**validate가 검증하는 것**:
- ✅ 컬럼 존재 여부
- ✅ 컬럼 타입 기본 호환성 (VARCHAR vs INT 등)
- ✅ NOT NULL 제약 (부분적)

**validate가 검증하지 않는 것**:
- ❌ 정확한 VARCHAR 길이 (VARCHAR(255) vs VARCHAR(100))
- ❌ DECIMAL 자릿수 (DECIMAL(10,2) vs DECIMAL(10,3))
- ❌ 기본값 (DEFAULT)
- ❌ 인덱스 존재 여부
- ❌ 외래 키 제약
- ❌ 체크 제약
- ❌ @Version 필드와 버전 컬럼 동기화

**따라서 다음을 위해 마이그레이션에 명시적으로 작성**:

```sql
-- V002__add_version_column_for_optimistic_locking.sql
ALTER TABLE products 
ADD COLUMN version BIGINT DEFAULT 0 NOT NULL;

-- 기존 행에는 버전 0 설정
UPDATE products SET version = 0 WHERE version IS NULL;
```

```java
@Entity
public class Product {
    @Version
    private Long version;  // Optimistic Locking
}
```

### 올바른 Approach 4: Entity와 마이그레이션을 같은 PR에 포함

**나쁜 PR 분리**:
```
PR #100: "Add phone field to database"
├─ Commit: V002__add_phone_column.sql
└─ Description: "Adds phone column to users table"

PR #101: "Update User entity with phone field"
├─ Commit: User.java
└─ Description: "Maps phone field in JPA entity"

문제: 두 PR이 다른 시점에 머지될 수 있음
→ PR #100만 배포되면 스키마만 변경, Entity는 구버전
→ PR #101만 배포되면 Entity만 변경, 스키마 없음
```

**좋은 PR 통합**:
```
PR #100: "Add phone field to users"
├─ Commit 1: V002__add_phone_column.sql
├─ Commit 2: User.java
└─ Description:
   "Add phone field
    - Database migration in V002__add_phone_column.sql
    - JPA entity mapping in User.java
    - Validation: ddl-auto=validate passes"

이점: PR을 함께 리뷰 → 스키마와 코드 불일치 방지
```

---

## 🔬 내부 동작 원리

### 1. JPA Validation 프로세스

```
Spring Boot 시작
  ↓
LocalContainerEntityManagerFactoryBean.createNativeEntityManager()
  ↓
Hibernate SessionFactoryImpl 생성
  ↓
Dialect 로드 (MySQL8Dialect)
  ↓
Metamodel 생성
  ├─ @Entity 클래스 스캔
  ├─ @Column 어노테이션 파싱
  └─ 필드 타입 정보 수집
  ↓
DDL 모드 확인 (ddl-auto=validate)
  ↓
Database에서 메타데이터 조회
  ├─ information_schema.COLUMNS
  ├─ information_schema.TABLES
  └─ information_schema.KEY_COLUMN_USAGE
  ↓
Entity 정의 vs Database 메타데이터 비교
  ├─ 테이블 이름
  ├─ 컬럼 이름
  ├─ 컬럼 타입
  ├─ Nullable 여부 (부분적)
  └─ 고유 키 여부 (부분적)
  ↓
불일치 발견 → SchemaManagementException
  ↓
Application Startup Failure
```

### 2. ddl-auto 옵션별 동작

| 옵션 | 스키마 검증 | DDL 생성 | DDL 실행 | 안전성 | 프로덕션 적합성 |
|------|-----------|--------|--------|------|-------------|
| **validate** | ✅ | ✅ | ❌ | 높음 | ✅ 권장 |
| **update** | ✅ | ✅ | ✅ | 낮음 | ❌ 위험 |
| **create** | ✅ | ✅ | ✅ | 낮음 | ❌ 위험 |
| **create-drop** | ✅ | ✅ | ✅ | 낮음 | ❌ 위험 |
| **none** | ❌ | ❌ | ❌ | 중간 | ✅ 가능 |

**프로덕션 권장**: `validate` (검증만, 변경 없음)

### 3. Entity-First vs Schema-First 비교

```
Entity-First (❌ 권장하지 않음)
┌─────────────────────────────────────┐
│ 1. Entity 클래스 정의               │
│    (스키마 없이 개발)               │
│                                     │
│    @Entity                          │
│    public class User {              │
│        private String phone;        │
│    }                                │
│                                     │
│ 2. ddl-auto=update/create 실행      │
│    → SQL 자동 생성 및 실행          │
│    CREATE TABLE / ALTER TABLE       │
│                                     │
│ 3. 수동으로 SQL 추출 (번거로움)     │
│    → Flyway 마이그레이션 파일 작성  │
│                                     │
│ 문제점:                             │
│ - SQL 최적화 불가                  │
│ - 자동 생성된 DDL이 비효율적        │
│ - 초기화 시간 오래 걸림             │
│ - 버전 관리 어려움                  │
└─────────────────────────────────────┘

Schema-First (✅ 권장)
┌─────────────────────────────────────┐
│ 1. SQL 마이그레이션 파일 작성       │
│    (V002__add_phone_column.sql)     │
│                                     │
│    ALTER TABLE users                │
│    ADD COLUMN phone VARCHAR(20);    │
│                                     │
│ 2. 마이그레이션 파일 테스트/리뷰    │
│    (SQL 문법 검증, 성능 확인)       │
│                                     │
│ 3. Entity 클래스 업데이트           │
│    (스키마에 맞게 매핑)             │
│                                     │
│    @Column(name = "phone",          │
│             length = 20)            │
│    private String phone;            │
│                                     │
│ 4. ddl-auto=validate로 검증         │
│    (일치성 확인)                    │
│                                     │
│ 장점:                               │
│ - SQL 최적화 가능                   │
│ - 버전 관리 명확                    │
│ - 재현성 보장                       │
│ - 리뷰 용이                         │
└─────────────────────────────────────┘
```

---

## 💻 실전 실험

### 실험 1: validate가 감지하는 불일치

**시나리오**: Entity에서 필드를 추가했는데 마이그레이션 파일을 작성하지 않음

**User.java** (변경 후):
```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, length = 100)
    private String username;
    
    @Column(nullable = false, length = 100)
    private String email;
    
    // ❌ 새 필드: DB에 없음
    @Column(nullable = false, length = 20)
    private String phone;
}
```

**database.sql** (변경 없음):
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL
);
```

**application.yml**:
```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```

**테스트**:
```java
@SpringBootTest
public class ValidateTest {
    @Test
    public void testMissingColumn() {
        // Spring Boot 시작 시 오류 발생
    }
}
```

**에러 메시지**:
```
org.hibernate.HibernateException: 
Missing column in database: phone in table users

  at org.hibernate.tool.schema.internal.SchemaValidator.validateTable
  at org.hibernate.tool.schema.internal.SchemaValidator.performValidation
  at org.hibernate.tool.schema.internal.SchemaValidator.validate
  at org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean
  
Caused by: 
Table users does not have column phone
Expected column: phone VARCHAR(20) NOT NULL
```

**해결**:
```sql
-- V002__add_phone_column.sql
ALTER TABLE users 
ADD COLUMN phone VARCHAR(20) NOT NULL DEFAULT '';

UPDATE users SET phone = 'N/A';

ALTER TABLE users MODIFY COLUMN phone VARCHAR(20) NOT NULL;
```

---

### 실험 2: validate가 놓치는 불일치

**시나리오**: NULL 제약과 VARCHAR 길이가 Entity와 일치하지 않음

**User.java**:
```java
@Entity
@Table(name = "users")
public class User {
    @Column(nullable = false)  // NOT NULL
    private Integer age;
    
    @Column(length = 50)  // VARCHAR(50)
    private String zipcode;
}
```

**database.sql**:
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    age INT NULL,  -- ❌ nullable=true (Entity: NOT NULL)
    zipcode VARCHAR(255)  -- ❌ length 255 (Entity: 50)
);
```

**application.yml**:
```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```

**테스트**:
```java
@SpringBootTest
public class ValidateGapsTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    public void testValidatePassesDespiteMismatch() {
        // ✅ validate 통과! (컬럼이 존재하니까)
        // 하지만 다음 문제들이 숨어있음:
        
        User user = new User();
        user.setUsername("john");
        user.setEmail("john@example.com");
        // user.age를 설정하지 않음
        
        userRepository.save(user);
        // age가 NULL로 저장됨 (Entity: nullable=false이지만)
        
        User found = userRepository.findById(user.getId()).orElse(null);
        assertNotNull(found);
        assertNull(found.getAge());  // ✅ NULL (Entity와 모순)
    }
    
    @Test
    public void testZipcodeLength() {
        User user = new User();
        user.setUsername("jane");
        user.setEmail("jane@example.com");
        user.setZipcode("A".repeat(255));  // 255 문자
        
        userRepository.save(user);
        // ✅ 저장 성공 (DB는 VARCHAR(255))
        // 하지만 Entity는 length=50을 기대
        
        // 프로덕션: 데이터 검증 로직 기대하지 않는 장 텍스트
    }
}
```

**해결**:
```sql
-- V002__fix_age_not_null_and_zipcode_length.sql
ALTER TABLE users 
MODIFY COLUMN age INT NOT NULL;

ALTER TABLE users 
MODIFY COLUMN zipcode VARCHAR(50);

-- 기존에 길이를 초과하는 데이터가 있으면 미리 정리
UPDATE users SET zipcode = LEFT(zipcode, 50) WHERE LENGTH(zipcode) > 50;
```

---

### 실험 3: Schema-First 워크플로우 전체 예시

**요구사항**: Product 엔티티에 "category" 필드 추가

**Step 1: 마이그레이션 파일 작성**

```sql
-- src/main/resources/db/migration/V003__add_category_to_products.sql

-- 1단계: 새 category 테이블 생성
CREATE TABLE categories (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL UNIQUE,
    description VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2단계: products 테이블에 category_id 컬럼 추가
ALTER TABLE products 
ADD COLUMN category_id BIGINT DEFAULT NULL;

-- 3단계: 외래 키 제약 추가
ALTER TABLE products 
ADD CONSTRAINT fk_products_category_id 
FOREIGN KEY (category_id) REFERENCES categories(id);

-- 4단계: 기존 상품에 기본 카테고리 할당
INSERT INTO categories (name, description) 
VALUES ('Uncategorized', 'Default category for existing products');

UPDATE products SET category_id = 1 
WHERE category_id IS NULL;

-- 5단계: 이제 NULL 제약 추가 가능
ALTER TABLE products 
MODIFY COLUMN category_id BIGINT NOT NULL;

CREATE INDEX idx_products_category_id ON products(category_id);
```

**Step 2: 마이그레이션 테스트**

```bash
# 로컬 MySQL에서 테스트
mysql> USE testdb;
mysql> source V003__add_category_to_products.sql;
Query OK, 1 row affected (0.05 sec)
```

**Step 3: Entity 업데이트**

```java
// Category.java (새 엔티티)
@Entity
@Table(name = "categories")
@Data
@NoArgsConstructor
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true, length = 100)
    private String name;
    
    @Column(length = 500)
    private String description;
    
    @Column(nullable = false)
    private LocalDateTime createdAt = LocalDateTime.now();
    
    @OneToMany(mappedBy = "category", cascade = CascadeType.ALL)
    private List<Product> products = new ArrayList<>();
}

// Product.java (수정)
@Entity
@Table(name = "products")
@Data
@NoArgsConstructor
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true, length = 100)
    private String sku;
    
    @Column(length = 1000)
    private String description;
    
    @Column(nullable = false)
    private BigDecimal price;
    
    // ✅ 새 필드: 마이그레이션 완료 후 추가
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private Category category;
    
    @Column(nullable = false)
    private LocalDateTime createdAt = LocalDateTime.now();
}
```

**Step 4: validate 확인**

```yaml
# application.yml
spring:
  jpa:
    hibernate:
      ddl-auto: validate  # Entity와 DB 스키마 일치성 검증
```

```java
@SpringBootTest
public class CategoryMigrationTest {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private CategoryRepository categoryRepository;
    
    @Test
    public void testMigrationAndEntitySync() {
        // ✅ validate 통과: Entity와 DB 스키마 일치
        
        Category electronics = new Category();
        electronics.setName("Electronics");
        electronics.setDescription("Electronic devices");
        categoryRepository.save(electronics);
        
        Product laptop = new Product();
        laptop.setSku("LAPTOP-001");
        laptop.setPrice(new BigDecimal("999.99"));
        laptop.setCategory(electronics);
        productRepository.save(laptop);
        
        Product found = productRepository.findById(laptop.getId()).orElse(null);
        assertNotNull(found);
        assertNotNull(found.getCategory());
        assertEquals("Electronics", found.getCategory().getName());
    }
}
```

**Step 5: 한 커밋으로 통합**

```
Commit: "Add category field to products"

Files:
├─ src/main/resources/db/migration/V003__add_category_to_products.sql
├─ src/main/java/com/example/domain/Category.java
└─ src/main/java/com/example/domain/Product.java

Message:
Add category field to products

- Create categories table with id, name, description
- Add category_id foreign key to products table
- Create Category JPA entity
- Add category field to Product entity
- Validation: ddl-auto=validate passes
```

---

## 📊 성능/비용 비교

| 측면 | Entity-First | Schema-First |
|------|------------|------------|
| **개발 속도** | 빠름 (자동 생성) | 느림 (수동 작성) |
| **SQL 최적화** | 불가능 | 가능 |
| **마이그레이션 품질** | 낮음 (비효율적 DDL) | 높음 (최적화된 DDL) |
| **버전 관리** | 어려움 | 명확함 |
| **리뷰 용이성** | 낮음 (자동 생성) | 높음 (의도 명확) |
| **초기화 시간** | 오래 걸림 | 빠름 |
| **데이터 손상 위험** | 높음 | 낮음 |
| **CI/CD 안정성** | 낮음 | 높음 |

---

## ⚖️ 트레이드오프

### 1. Schema-First의 수동 작업 부담 vs Entity-First의 위험성
- **Schema-First**: 초기 개발 속도는 느리지만 안정성 높음 (권장)
- **Entity-First**: 빠르지만 실수하면 데이터 손상 위험

### 2. validate의 제약 사항
- **검증 가능**: 컬럼 이름, 기본 타입
- **검증 불가능**: VARCHAR 길이, DECIMAL 자릿수, 인덱스

### 3. 마이그레이션과 Entity의 동기화 비용
- **같은 커밋**: 리뷰 복잡도 증가하지만 불일치 불가능
- **분리된 커밋**: 리뷰 간단하지만 동기화 실수 가능 (권장: 같은 커밋)

---

## 📌 핵심 정리

1. **Entity와 마이그레이션은 같은 커밋에 포함되어야 함**
   - PR 리뷰 시 스키마와 코드를 함께 검토
   - 배포 시 코드와 스키마가 동시에 적용

2. **Schema-First 워크플로우 권장**
   - SQL 마이그레이션 파일 작성 → 테스트 → Entity 업데이트 → validate
   - Entity-First는 초기는 빠르지만 장기적으로 기술 부채 증가

3. **ddl-auto=validate는 완벽하지 않음**
   - ✅ 컬럼 존재, 기본 타입 검증
   - ❌ NULL 제약, VARCHAR 길이, 인덱스, 외래 키 등 세부 사항
   - 따라서 마이그레이션 파일을 명확하게 작성해야 함

4. **Entity 변경 시 체크리스트**
   ```
   [ ] SQL 마이그레이션 파일 작성
   [ ] 마이그레이션 파일 로컬에서 테스트
   [ ] Entity 클래스 업데이트
   [ ] ddl-auto=validate 통과 확인
   [ ] 마이그레이션 + Entity 같은 커밋에 포함
   [ ] PR에서 SQL + Entity 함께 리뷰
   ```

5. **버전 관리와 배포 순서 중요**
   - 마이그레이션 파일은 스키마 버전 관리
   - 코드와 함께 배포되어 버전 동기화
   - 이전 버전 코드는 새 스키마와 호환 불가능

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: Entity-First로 빠르게 개발하고 나중에 마이그레이션 파일로 변환해도 될까?</strong></summary>

**A**: 이론상 가능하지만 실무에서는 위험합니다.

**프로세스**:
```
1. ddl-auto=update로 빠르게 개발
   @Entity 변경 → 자동 DDL 생성 및 실행
   
2. 개발 완료 후 마이그레이션으로 변환
   show create table으로 생성 DDL 조회
   → SQL 파일로 수작업 변환
```

**문제점**:
1. **비효율적 DDL**: 자동 생성 DDL은 성능 고려 없음
   - 불필요한 NULL 제약
   - 인덱스 누락
   - 잘못된 데이터 타입

2. **재현성 부족**: 자동 생성 DDL이 항상 같지 않음
   - Hibernate 버전 차이
   - Dialect 차이
   - 실행 순서 차이

3. **기술 부채**: 나중에 SQL 정리하는 시간이 더 오래 걸림

**권장**: 개발 초기부터 Schema-First 습관 (처음에는 느리지만 장기적으로 빠름)
</details>

<details>
<summary><strong>Q2: 기존 레거시 시스템에서 Entity와 스키마가 다를 때는?</strong></summary>

**A**: 점진적으로 동기화하면서 validate 도입합니다.

**Step 1: 현재 상태 파악**
```java
@Column(name = "email")  // Entity: email
// DB: user_email (이름 다름)

@Column(nullable = false)
private String phone;  // Entity: NOT NULL
// DB: VARCHAR(20) NULL (NULL 가능)
```

**Step 2: @Column으로 매핑 수정**
```java
@Column(name = "user_email")  // DB 컬럼명에 맞게 변경
private String email;

@Column(nullable = true)  // Entity도 현실에 맞게
private String phone;
```

**Step 3: 마이그레이션으로 스키마 정규화 (선택)**
```sql
-- V100__refactor_schema_for_jpa_validation.sql
-- 컬럼명 변경 (user_email → email)
ALTER TABLE users CHANGE COLUMN user_email email VARCHAR(100);

-- NULL 제약 추가
ALTER TABLE users MODIFY COLUMN phone VARCHAR(20) NOT NULL DEFAULT 'N/A';
```

**Step 4: validate 도입**
```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate  # 이제 안전
```

**결론**: ddl-auto=none에서 validate로 단계적 마이그레이션 가능
</details>

<details>
<summary><strong>Q3: ORM으로 관리할 수 없는 컬럼(트리거, 함수 등)은 어떻게 처리할까?</strong></summary>

**A**: 마이그레이션 파일에만 작성하고 Entity는 무시합니다.

**예시 - 자동 업데이트 타임스탬프**:

```sql
-- V001__create_users_with_trigger.sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(100) NOT NULL,
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

```java
@Entity
@Table(name = "users")
public class User {
    @Column(updatable = false)
    private LocalDateTime createdAt;
    
    @Column(insertable = false, updatable = false)
    private LocalDateTime updatedAt;  // DB에서만 관리 (trigger)
}
```

**또 다른 예 - 계산된 컬럼 (MySQL 5.7+)**:

```sql
CREATE TABLE orders (
    id BIGINT,
    price DECIMAL(10,2),
    tax_rate DECIMAL(3,2),
    total_price DECIMAL(10,2) GENERATED ALWAYS AS (price * (1 + tax_rate)) STORED
);
```

```java
@Entity
public class Order {
    @Column(insertable = false, updatable = false)
    private BigDecimal totalPrice;  // 읽기 전용
}
```

**권장**: 
- `insertable = false, updatable = false` 속성 사용
- 마이그레이션에는 명시적으로 주석 작성
- 리뷰자가 이해할 수 있도록
</details>

---

<div align="center">

**[⬅️ 이전: 테스트 데이터 관리](./02-test-data-management.md)** | **[홈으로 🏠](../README.md)** | **[다음: 대용량 데이터 마이그레이션 ➡️](./04-large-data-migration.md)**

</div>
