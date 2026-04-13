# 02. Forward-Only 마이그레이션 전략

---

## 🎯 핵심 질문

```
❓ 마이그레이션 실패 시 "되돌리지 않고 앞으로만 간다"는 게 뭔가?
   그럼 데이터는 어떻게 보호하고, 실수는 누가 책임지나?
```

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Forward-Only는 단순한 기술이 아니라 철학**입니다.

- ❌ "오류 나면 ROLLBACK" → 불가능 (MySQL DDL)
- ✅ "오류 나면 빠르게 수정해서 다시 실행" → Forward-Only

이를 모르면:

- 스키마를 "더럽게" 만든 후 복구 방법이 없음
- 배포 후 롤백할 수 없다는 것을 뒤늦게 깨달음
- 마이그레이션 버전 충돌로 팀 전체가 박힘

**Forward-Only를 이해하면**:
- 실패를 두려워하지 않고 설계 가능
- 프로덕션과 스테이징이 다른 버전이어도 처리 가능
- Blue-Green 배포의 핵심 전략

---

## 😱 흔한 실수 (Before — 롤백이 있을 거라고 가정)

### 실수 1: 스키마 일관성 무시

```sql
-- V001__create_users_table.sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    age INT
);

-- V002__add_phone_column.sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- 😱 프로덕션 배포 후 버그 발견:
-- phone 컬럼이 필요 없었다!
-- "ROLLBACK해야 하는데 DDL은 롤백 불가..."
```

### 실수 2: 구 버전 앱과의 호환성 무시

```sql
-- V001__users_schema.sql (V1.0 앱이 사용)
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(255)
);

-- V002__add_age_mandatory.sql (V2.0 배포 전)
ALTER TABLE users 
ADD COLUMN age INT NOT NULL DEFAULT 0;
-- ❌ age 컬럼이 NOT NULL이므로, V1.0 앱에서
--    INSERT INTO users (id, email) ... 실행 시 오류!
```

### 실수 3: 데이터 손실 가능성 방치

```sql
-- V001__create_products.sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    category_id INT  -- 외래 키 없음
);

INSERT INTO products VALUES (1, 'Laptop', 10);
INSERT INTO products VALUES (2, 'Phone', 999);  -- 존재하지 않는 category

-- V002__add_category_constraint.sql
ALTER TABLE products 
ADD CONSTRAINT fk_category 
FOREIGN KEY (category_id) REFERENCES categories(id);
-- ❌ 오류: category_id = 999는 존재하지 않음
--    마이그레이션 실패! 데이터 정제는?
```

---

## ✨ 올바른 접근 (After — Forward-Only로 설계)

### 원칙 1: 하위 호환성(Backward Compatibility) 유지

```sql
-- ✅ V001__create_users_schema.sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE
);

-- ✅ V002__add_phone_soft_optional.sql
-- 새 컬럼은 NULL 허용 또는 DEFAULT 값 있어야 함
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
--                           ↑ NULL 허용 → V1.0 앱도 INSERT 가능

-- ✅ V003__add_age_with_default.sql
ALTER TABLE users ADD COLUMN age INT DEFAULT 18;
--                                 ↑ DEFAULT 값 → V1.0 앱의 INSERT도 정상 작동
```

**왜 중요한가?**

Blue-Green 배포 중:
```
시간 t=0: V1.0 앱만 실행
         INSERT INTO users (id, email) VALUES (...);
         
시간 t=1: V002 마이그레이션 실행
         ALTER TABLE users ADD COLUMN phone VARCHAR(20);
         
시간 t=2~3: V1.0과 V2.0 앱이 동시 실행 (롤링 배포)
         V1.0: INSERT INTO users (id, email) ... ✅ (phone은 NULL)
         V2.0: INSERT INTO users (id, email, phone) ... ✅
         
시간 t=4: V1.0 앱 중단, V2.0만 실행
```

**만약 phone이 NOT NULL이었다면?**
```
시간 t=2~3에서 V1.0 앱의 INSERT가 실패!
→ 프로덕션 장애!
```

### 원칙 2: 스키마 개선을 여러 버전에 걸쳐 수행

```sql
-- ❌ 한 번에 많이 (위험)
-- V001__do_everything.sql
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(10,2) NOT NULL DEFAULT 0;
ALTER TABLE orders ADD COLUMN shipping_address VARCHAR(255);
ALTER TABLE orders ADD INDEX idx_status (status);
UPDATE orders SET total_amount = quantity * unit_price;
-- 한 곳 실패 → 전체 마이그레이션 롤백 불가!

-- ✅ 단계적으로 분리 (안전)
-- V001__add_total_amount_column.sql
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(10,2) DEFAULT 0;

-- V002__populate_total_amount.sql
UPDATE orders SET total_amount = quantity * unit_price 
WHERE total_amount = 0;

-- V003__add_shipping_address_column.sql
ALTER TABLE orders ADD COLUMN shipping_address VARCHAR(255);

-- V004__add_status_index.sql
ALTER TABLE orders ADD INDEX idx_status (status);
```

### 원칙 3: 마이그레이션 실패를 Forward-Only로 해결

**시나리오**: V002 마이그레이션이 실패했다

```sql
-- 마이그레이션 히스토리
-- V001: ✅ success
-- V002: ❌ failed
-- V003: ⏭️ pending

-- 문제: V002의 오류 원인
-- → 데이터 무결성 제약 위반 (예: NULL 값이 예상 밖으로 많음)

-- Forward-Only 해결책 (ROLLBACK 불가):
-- 1. V002 마이그레이션 SQL 파일 수정
-- 2. flyway repair (success=false 제거)
-- 3. V002를 다시 실행 또는 V003에서 보정
```

**예시**:

```sql
-- V002__migrate_status_original.sql (❌ 실패)
-- UPDATE products SET status = 'ACTIVE' WHERE status IS NULL;
-- 오류: 1032 Can't find record in 'products'

-- → flyway repair 후, 옵션 1: V002 파일 수정
-- V002__migrate_status_original.sql (✅ 수정)
UPDATE products SET status = 'ACTIVE' 
WHERE status IS NULL AND id > 0;  -- 조건 추가

-- 또는 옵션 2: V003에서 보정
-- V003__fix_status_migration.sql
UPDATE products SET status = 'ACTIVE' 
WHERE status IS NULL;
```

### 원칙 4: 실수한 컬럼은 다음 버전에서 DROP

```sql
-- V001: ✅ 배포됨
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- 결론: phone 컬럼이 불필요하다는 것을 알게 됨
-- 하지만 ROLLBACK 불가!

-- Forward-Only 해결책:
-- V002: 컬럼을 다음 마이그레이션에서 제거
ALTER TABLE users DROP COLUMN phone;
```

**중요**: 
- DROP은 즉시하지 않음 (다른 곳에서 사용 중일 수 있음)
- 보통 1~2 버전 후 제거 (사용자가 코드 업데이트할 시간 제공)

---

## 🔬 내부 동작 원리

### 1. Blue-Green 배포에서 Forward-Only

```
┌─────────────────────────────────────────────────────────────┐
│ Phase 1: Green(신버전) 준비                                  │
├─────────────────────────────────────────────────────────────┤
│ Blue(구버전 앱)          │ 공유 DB (single)                  │
│ ↓ INSERT/SELECT          │ ← 모든 트래픽 처리                │
│ (기존 스키마 사용)       │                                   │
├─────────────────────────────────────────────────────────────┤
│ Phase 2: 마이그레이션 실행                                   │
├─────────────────────────────────────────────────────────────┤
│ 스키마 변경: 기존 앱과 신 앱 모두 호환                       │
│ ALTER TABLE users ADD COLUMN phone VARCHAR(20);             │
│ (phone은 NULL 허용 → 호환성 보장)                           │
├─────────────────────────────────────────────────────────────┤
│ Phase 3: Green(신버전) 배포                                  │
├─────────────────────────────────────────────────────────────┤
│ Blue(구버전)                                                 │
│ ↓ INSERT (id, email, age) ... ✅                            │
│                                                              │
│ Green(신버전)                                                │
│ ↓ INSERT (id, email, age, phone) ... ✅                     │
│                                                              │
│ → 둘 다 정상 작동! (phone은 어느 쪽이든 nullable)           │
├─────────────────────────────────────────────────────────────┤
│ Phase 4: Blue 제거                                          │
├─────────────────────────────────────────────────────────────┤
│ Green(신버전만)                                              │
│ ↓ 모든 트래픽 처리 (phone 컬럼 자유롭게 사용)               │
└─────────────────────────────────────────────────────────────┘
```

**key**: 마이그레이션이 "양쪽 버전 모두와 호환"되어야 함

### 2. 버전 호환성 행렬

```
           | V1.0 앱 | V2.0 앱
-----------|---------|----------
스키마 V1  | ✅      | ❌ (새 컬럼 없음)
스키마 V2  | ✅      | ✅ (모두 호환)
           (NULL    (phone 필드
            허용)    자유로움)
```

**V1.0 앱이 V2 스키마에서 작동하려면**:
- V1.0이 알고 있는 컬럼들(`id`, `email`)은 변화 없어야 함
- 새 컬럼(`phone`)은 NULL 허용이거나 DEFAULT 있어야 함
- 기존 컬럼을 삭제하지 않거나, 1~2 버전 후에만 삭제

### 3. 마이그레이션 버전의 의미

```
V001: 초기 스키마 (V1.0 앱과 호환)
 ↓
V002: 신 컬럼 추가 (V1.0 + V2.0 앱 모두 호환)
 ↓
V003: 데이터 마이그레이션 (선택적, 백그라운드에서 가능)
 ↓
V004: 기존 컬럼 정리 (V1.0 앱 완전히 제거된 후)
```

---

## 💻 실전 실험

### 실험 1: Blue-Green 배포 시뮬레이션

```bash
# MySQL 8.0 실행
docker run -d --name mysql-forward \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=appdb \
  mysql:8.0 \
  --default-authentication-plugin=mysql_native_password

sleep 10
docker exec -it mysql-forward mysql -u root -ppassword appdb
```

```sql
-- Phase 1: 초기 스키마 (V1.0 앱용)
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (email) VALUES ('user1@example.com');
INSERT INTO users (email) VALUES ('user2@example.com');

DESCRIBE users;
-- V1.0 앱이 사용하는 컬럼들

-- Phase 2: 마이그레이션 (V2로 준비)
-- ✅ 올바른 방식: NULL 허용
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
-- ❌ 틀린 방식: ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;

-- Phase 3: Blue(V1.0)와 Green(V2.0) 동시 작동 시뮬레이션
-- V1.0 앱: phone 컬럼을 알지 못함 (INSERT할 때 phone 명시하지 않음)
INSERT INTO users (email) VALUES ('v1-user@example.com');
-- ✅ 성공 (phone은 NULL으로 자동 저장)

-- V2.0 앱: phone 컬럼 포함
INSERT INTO users (email, phone) VALUES ('v2-user@example.com', '010-1234-5678');
-- ✅ 성공

-- 확인: 모두 정상
SELECT id, email, phone FROM users;
-- id | email                 | phone
-- 1  | user1@example.com     | NULL
-- 2  | user2@example.com     | NULL
-- 3  | v1-user@example.com   | NULL
-- 4  | v2-user@example.com   | 010-1234-5678

-- Phase 4: V2.0으로 완전 전환 후, 구 컬럼 정리 (V3 이후)
-- 아직은 phone을 유지 (사용 가능)
```

### 실험 2: 마이그레이션 실패 후 Forward-Only 복구

```sql
-- V001: 초기 스키마 (성공)
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL
);

INSERT INTO products (name, price) VALUES ('Laptop', 999.99);
INSERT INTO products (name, price) VALUES ('Mouse', 25.50);
INSERT INTO products (name, price) VALUES ('Keyboard', 0);  -- 가격 0?

-- V002: 가격 유효성 검사 추가 시도 (실패할 코드)
-- ALTER TABLE products ADD CONSTRAINT check_price CHECK (price > 0);
-- ❌ 오류: Keyboard의 가격이 0이므로 제약 위반

-- Forward-Only 해결책:
-- Step 1: 데이터 정제
UPDATE products SET price = 79.99 WHERE id = 3;  -- Keyboard 가격 수정

-- Step 2: 이제 제약 추가 가능
ALTER TABLE products ADD CONSTRAINT check_price CHECK (price > 0);

-- Step 3: 확인
SHOW CREATE TABLE products;
-- CONSTRAINT check_price CHECK (price > 0)

-- V003: 필요 시 추가 개선
ALTER TABLE products ADD COLUMN stock INT DEFAULT 0;
```

### 실험 3: Flyway를 사용한 Forward-Only 마이그레이션

```bash
# Flyway CLI 다운로드 (없으면)
# https://flywaydb.org/download/community

# 마이그레이션 파일 구조
cat > flyway.conf << 'EOF'
flyway.driver=com.mysql.cj.jdbc.Driver
flyway.url=jdbc:mysql://localhost:3306/appdb
flyway.user=root
flyway.password=password
flyway.locations=filesystem:./sql
EOF

mkdir -p sql

# V001: 초기 스키마
cat > sql/V001__Create_users_table.sql << 'EOF'
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE
);
EOF

# V002: 새 컬럼 추가 (NULL 허용)
cat > sql/V002__Add_phone_column.sql << 'EOF'
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
EOF

# V003: 새 컬럼 추가 (DEFAULT 있음)
cat > sql/V003__Add_age_column.sql << 'EOF'
ALTER TABLE users ADD COLUMN age INT DEFAULT 0;
EOF

# Flyway 실행
# flyway migrate
# (아래는 예상 출력)
# Successfully applied 3 migrations

# 히스토리 확인
# SELECT version, description, success FROM flyway_schema_history;
# 1 | Create users table | true
# 2 | Add phone column   | true
# 3 | Add age column     | true
```

---

## 📊 성능/비용 비교

| 전략 | 성공 시 | 실패 시 | 복구 시간 | 데이터 손실 위험 |
|------|--------|--------|---------|-----------------|
| **Rollback-Capable** | 빠름 | 자동 ROLLBACK | 즉시 | ❌ 낮음 |
| **Forward-Only** | 빠름 | 수동 수정 필요 | 중간 | ⚠️ 있을 수 있음 |

---

## ⚖️ 트레이드오프

### Forward-Only의 장점
- ✅ MySQL 호환 (모든 DB에서 가능)
- ✅ 마이그레이션 파일 단순 (Undo 파일 불필요)
- ✅ 팀 전체가 같은 철학으로 작업

### Forward-Only의 단점
- ❌ 실수한 마이그레이션을 즉시 되돌릴 수 없음
- ❌ 수동 복구 절차 필요
- ❌ 실패 감지와 대응이 빨라야 함

---

## 📌 핵심 정리

1. **Forward-Only = "마이그레이션은 일방향"**
   - 실패 시 ROLLBACK 불가
   - 대신 앞으로 나가면서 수정 (V002, V003...)

2. **하위 호환성이 핵심**
   ```sql
   새 컬럼 = NULL 허용 OR DEFAULT 값 있음
   → V1.0 앱이 V2 스키마에서도 동작
   ```

3. **Blue-Green 배포와 필수 짝꿍**
   - 마이그레이션은 "양쪽 버전 모두 호환"되도록 설계
   - 마이그레이션 자체는 한 번만 실행
   - 앱은 여러 번 배포 (V1.0 → V2.0)

4. **실수 복구는 새 마이그레이션으로**
   ```sql
   V001: 실수 (컬럼 잘못 추가)
   V002: 수정 (컬럼 타입 변경 또는 삭제)
   ```

5. **컬럼 삭제는 여유 있게**
   - 불필요한 컬럼은 즉시 삭제하지 않음
   - 1~2 버전 후에 DROP (사용자 코드 업데이트 시간 제공)

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: Forward-Only 전략에서 "실수"를 어떻게 방지하나?</strong></summary>

**답**:

완벽하게 방지할 수 없지만, **다단계 검증으로 최소화**합니다.

```
1. 개발 로컬: 마이그레이션 테스트
   ├─ 스키마 변경 확인 (DESCRIBE)
   ├─ 앱 코드 호환성 확인 (컴파일, 기본 테스트)
   
2. 스테이징: 실제 데이터로 테스트
   ├─ 마이그레이션 성공 여부
   ├─ 성능 영향도 측정 (큰 테이블 ALTER 시 시간)
   ├─ 롤링 배포 시뮬레이션 (V1.0과 V2.0 동시 실행)
   
3. 프로덕션: 저트래픽 시간대 배포
   ├─ 모니터링 강화
   ├─ 빠른 대응 팀 대기
   ├─ 롤백 대신 Forward 패치 준비 (V002 in ready)
```

**핵심 방지 기법**:
```java
// 마이그레이션 실행 전 자동 검증
public class MigrationValidator {
    @BeforeEach
    void validateMigration() {
        // V{n} 마이그레이션이 V{n-1} 버전 앱과 호환?
        // 1. 새 컬럼이 NOT NULL? → 오류
        // 2. 기존 컬럼 삭제? → 경고
        // 3. 제약 추가 시 데이터 만족? → 확인
    }
}
```

</details>

<details>
<summary><strong>Q2: 프로덕션에서 마이그레이션 실패 시 즉시 대응 (롤백 불가)하는 절차는?</strong></summary>

**답**:

**Incident Response Plan**:

```
T+0: 마이그레이션 실패 감지
├─ Slack 알림: "#db-incidents" 채널
├─ 온콜 DBA/Dev 호출

T+2: 원인 파악
├─ 마이그레이션 로그 분석
│  ├─ "Duplicate key error" → 데이터 중복
│  ├─ "Constraint violation" → 데이터 부분 정제 필요
│  ├─ "Lock wait timeout" → 다른 세션이 테이블 잠금
│
├─ flyway_schema_history 확인
│  └─ version, success, error_message 검토

T+5: Forward-Only 해결책 선택
├─ 옵션 A: 수동으로 실패한 부분 완료
│  ├─ 문제 데이터 정제
│  ├─ 마이그레이션 파일 수정
│  ├─ flyway repair
│  ├─ 마이그레이션 재실행
│
├─ 옵션 B: 새 버전에서 보정
│  ├─ V{n+1}__Fix_v{n}_issue.sql 작성
│  ├─ 실패한 V{n} 상태를 V{n+1}에서 수정
│  ├─ flyway repair
│  ├─ 마이그레이션 재실행

T+10: 검증
├─ 스키마 상태 확인 (DESCRIBE)
├─ 데이터 무결성 검증 (체크섬, 행 수)
├─ 앱 헬스체크 (로그 모니터링)

T+15: 사후 분석 (Post-mortem)
├─ 왜 실패했는가?
├─ 스테이징에서 감지 가능했는가?
├─ 재발 방지 대책?
```

**예시 상황**:
```sql
-- V002 마이그레이션 실패: "Duplicate entry '010-2000-0000'"
-- 문제: phone 컬럼의 UNIQUE 제약이 기존 데이터와 충돌

-- 해결 옵션 A: 수동 정제 후 재실행
UPDATE users SET phone = NULL WHERE phone = '010-2000-0000' AND id > 10;
flyway repair  -- success=false 제거
flyway migrate -- V002 재실행

-- 또는 옵션 B: V003에서 보정
-- V003__Fix_duplicate_phone.sql
ALTER TABLE users DROP CONSTRAINT unique_phone;  -- 제약 제거
-- V002 재실행 후 성공
-- 그 다음 V004에서 다시 UNIQUE 추가 (중복 제거 후)
```

</details>

<details>
<summary><strong>Q3: 컬럼을 삭제할 때 "1~2 버전 후에 DELETE"라고 했는데, 너무 보수적 아닌가?</strong></summary>

**답**:

보수적이지만 **실무에서 필수**입니다. 이유:

```
V1 앱: 코드에 phone 필드 사용 중
├─ serialization: phone 포함
├─ API response: phone 반환
├─ 캐시: phone 저장
└─ 메모리: phone 객체에 유지

배포 일정:
├─ 월요일: V1 → V2로 마이그레이션 (phone 컬럼 추가)
├─ 화요일: V2 앱 배포 (모든 서버에 배포 완료)
├─ 목요일: V3 마이그레이션 실행 (phone 컬럼 삭제)
│        ❌ 만약 V2 배포가 지연되면?
│        → 일부 V1 앱이 여전히 실행 중
│        → V3 실행 시 V1 앱 오류!
```

**더 나은 전략**:

```sql
-- V001: 초기 스키마
CREATE TABLE users (...);

-- V002: phone 추가 (선택적)
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- V003~V004: phone 선택적으로 사용
-- (모든 앱이 phone을 처리 가능하도록 이행)

-- V005: phone이 정말 필요 없다고 확정
ALTER TABLE users DROP COLUMN phone;
```

**타이밍**:
- 마이그레이션 이후 최소 1주 이상 경과
- 배포 히스토리 확인 (모든 앱 버전 업데이트)
- 로그 확인 (phone 필드 사용 사실 없음)

</details>

---

<div align="center">

**[⬅️ 이전: DDL 롤백이 없는 이유](./01-why-ddl-no-rollback.md)** | **[홈으로 🏠](../README.md)** | **[다음: Flyway Undo 마이그레이션 ➡️](./03-flyway-undo-migration.md)**

</div>
