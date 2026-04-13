# 04. 마이그레이션 실패 시 복구 절차

---

## 🎯 핵심 질문

```
❓ 마이그레이션 중간에 실패하면?
   어디까지 실행됐는지 어떻게 알고,
   남은 부분은 어떻게 처리하나?
```

---

## 🔍 왜 이 개념이 실무에서 중요한가

**프로덕션 장애 3단계**:

```
1️⃣ 발견 (마이그레이션 실패)
2️⃣ 분석 (어디서 실패했는가?)
3️⃣ 복구 (다음은 뭘 해야 하나?)
```

이를 모르면:

- ❌ "마이그레이션 실패했는데 뭘 해야 할지..." → 패닉
- ❌ 일부만 적용된 스키마 상태 불명확 → 데이터 불일치
- ❌ 다시 실행하면 "이미 적용된 부분" 또는 "원본 상태"인지 모름
- ❌ `flyway repair`를 잘못 사용 → 히스토리 손상

**실제 대응 절차를 알면**:
- ✅ 실패 지점을 정확히 파악
- ✅ 부분 적용된 스키마 정리 방법 알기
- ✅ 마이그레이션을 안전하게 재시작

---

## 😱 흔한 실수 (Before — 부분 실패를 모르고 재시도)

### 실수 1: success=false를 무시하고 다시 실행

```sql
-- V002__add_constraints.sql
ALTER TABLE orders ADD CONSTRAINT check_amount CHECK (amount > 0);
UPDATE orders SET amount = 100 WHERE amount <= 0;
-- ❌ 오류: CHECK 제약 조건 위반 (이미 amount <= 0인 행이 있음)

-- 마이그레이션 히스토리:
-- V001 | true ✅
-- V002 | false ❌ (부분 적용: 제약 추가는 실패, UPDATE는 어떻게 됐나?)
```

```bash
# 😱 개발자 대응: "다시 실행해보면?"
$ flyway migrate
# ❌ Error: V002 failed in previous run. Fix the database 
#     state manually and run 'flyway repair'

# "좋아, repair 하자"
$ flyway repair  # 이건 success=false를 제거만 함!
$ flyway migrate
# ❌ 또 같은 오류 (근본 원인 미해결)
```

### 실수 2: flyway repair를 잘못 이해

```bash
# 상황: V002 부분 실패 (DDL은 커밋, DML 실패)

# ❌ 잘못된 이해
# "repair는 DB를 복구하겠지"
$ flyway repair
# → ❌ DB를 수정하지 않음!
#   → success=false 레코드만 삭제
#   → 실제 스키마 문제는 그대로 남음

# 그 다음 다시 실행
$ flyway migrate
# ❌ 여전히 오류 (repair는 DB 문제를 고치지 않음)
```

### 실수 3: 부분 실패의 원인을 모르는 상태에서 진행

```
마이그레이션 시작: BEGIN;
├─ DDL 1: ALTER TABLE users ADD COLUMN age INT;
│         ✅ 암묵적 COMMIT (MySQL의 특성)
│
├─ DDL 2: ALTER TABLE orders ADD CONSTRAINT ...
│         ❌ 오류 발생 (데이터 위반)
│         ⏸️ 마이그레이션 중단
│
└─ 결과:
    users.age 컬럼: 추가됨 ✅
    orders 제약: 추가 안 됨 ❌
    → 스키마가 "더럽혀진" 상태
```

"이 상태에서 뭘 해야 하나?" → 명확한 절차 필요

---

## ✨ 올바른 접근 (After — 구조화된 복구 절차)

### 복구 절차 체크리스트

**Step 1: 오류 원인 파악**

```bash
# flyway_schema_history에서 실패 기록 조회
$ mysql -u root appdb -e \
  "SELECT version, description, success, installed_on 
   FROM flyway_schema_history 
   WHERE success = false ORDER BY installed_on DESC LIMIT 1;"

# 결과:
# version | description           | success | installed_on
# 2       | add_order_constraint  | false   | 2024-01-15 14:23:45

# 마이그레이션 로그 확인 (에러 메시지)
$ cat flyway-error.log  # 또는 애플리케이션 로그

# 또는 DB에서 직접 오류 원인 검증
$ mysql -u root appdb -e \
  "SELECT * FROM orders WHERE amount <= 0 LIMIT 5;"
# → amount <= 0인 행이 있다! (제약 조건 위반의 원인)
```

**Step 2: 현재 스키마 상태 확인**

```sql
-- V002가 부분 실패한 상태에서
-- 어느 부분은 성공했나?

DESCRIBE users;
-- age 컬럼: 있음 ✅ (Step 1에서 성공)

DESCRIBE orders;
-- CHECK 제약: 없음 ❌ (Step 2에서 실패)

SHOW CREATE TABLE orders\G
-- CONSTRAINT check_amount가 없음
```

**Step 3: 실패한 부분의 원인 제거**

```sql
-- V002의 실패 원인: amount <= 0인 행이 있음

-- 옵션 A: 데이터 정제
UPDATE orders SET amount = NULL WHERE amount <= 0;
-- 또는 최소값 설정
UPDATE orders SET amount = 0.01 WHERE amount <= 0;

-- 옵션 B: 제약 조건 완화
-- (CHECK (amount >= 0) 대신 CHECK (amount >= 0))

-- 옵션 C: 마이그레이션 파일 수정
-- V002 파일 수정:
-- ALTER TABLE orders 
--   ADD CONSTRAINT check_amount CHECK (amount > 0 OR amount IS NULL);
-- 이렇게 하면 NULL도 허용
```

**Step 4: flyway repair 실행**

```bash
# flyway repair: success=false 레코드 제거
$ flyway repair

# 이제 flyway_schema_history에서 V002의 success = false 제거됨
# 대신 새로운 repair 레코드 추가
```

**Step 5: 수정된 마이그레이션 재실행**

```bash
# V002 파일을 수정했으면
$ flyway migrate

# 성공 여부 확인
$ mysql -u root appdb -e \
  "SELECT version, success FROM flyway_schema_history WHERE version = 2;"

# 이제 CHECK 제약이 추가되어 있어야 함
```

### 부분 실패 상황별 복구 전략

#### 시나리오 1: DDL은 성공, DML은 실패

```sql
-- V002__migrate_data.sql
ALTER TABLE products ADD COLUMN category_id INT;  -- ✅ 성공
UPDATE products SET category_id = 1 WHERE ...;    -- ❌ 실패

-- 현재 상태:
-- products.category_id: 추가됨 (DEFAULT NULL)
-- 데이터: 부분 업데이트만 됨

-- 복구:
-- Option A: UPDATE 조건 완화 후 재실행
UPDATE products SET category_id = 1 WHERE category_id IS NULL;

-- Option B: 마이그레이션 파일 수정 (기존 조건이 너무 까다로움)
-- 파일 수정 후 flyway migrate
```

#### 시나리오 2: DDL 중간에 실패

```sql
-- V003__schema_changes.sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);       -- ✅
CREATE INDEX idx_phone ON users(phone);               -- ✅
ALTER TABLE users MODIFY COLUMN email VARCHAR(500);  -- ❌ 실패
-- (예: foreign key constraint로 인해 실패)

-- 현재 상태:
-- users.phone: 추가됨 ✅
-- idx_phone: 생성됨 ✅
-- email 길이: 변경 안 됨 ❌

-- 복구:
-- 1. 실패 원인 확인 (외래 키 제약)
SELECT CONSTRAINT_NAME FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE 
WHERE TABLE_NAME = 'users' AND COLUMN_NAME = 'email';

-- 2. 임시로 외래 키 비활성화
SET FOREIGN_KEY_CHECKS = 0;
ALTER TABLE users MODIFY COLUMN email VARCHAR(500);
SET FOREIGN_KEY_CHECKS = 1;

-- 3. flyway repair
-- 4. 마이그레이션 파일 수정 (외래 키 고려)
```

#### 시나리오 3: 제약 조건 위반

```sql
-- V004__add_constraint.sql
-- 기존 데이터가 제약을 위반하는 경우

ALTER TABLE orders ADD CONSTRAINT check_price CHECK (price > 0);
-- ❌ 오류: price = 0 또는 음수인 행이 있음

-- 복구:
-- 1. 문제 데이터 확인
SELECT * FROM orders WHERE price <= 0;

-- 2. 데이터 정제
UPDATE orders SET price = NULL WHERE price <= 0;
-- 또는
DELETE FROM orders WHERE price <= 0;  -- 삭제 (원본 데이터 손실!)

-- 3. 또는 제약 조건 수정
-- V004 파일 수정:
ALTER TABLE orders 
ADD CONSTRAINT check_price CHECK (price > 0 OR price IS NULL);

-- 4. flyway repair
-- 5. 마이그레이션 재실행
```

---

## 🔬 내부 동작 원리

### 1. flyway_schema_history의 success 플래그

```
마이그레이션 실행 순서:
┌──────────────────────────────────────────────┐
│ BEGIN;  (명시적 또는 암묵적)                 │
├──────────────────────────────────────────────┤
│ V002__migration.sql 실행                     │
│  ├─ Statement 1: ✅ 성공                    │
│  ├─ Statement 2: ✅ 성공                    │
│  ├─ Statement 3: ❌ 오류!                   │
│  └─ (이후 Statement 실행 안 됨)              │
├──────────────────────────────────────────────┤
│ 마이그레이션 롤백 (또는 커밋?)               │
│  MySQL: 일부 DDL은 자동 커밋됨!            │
├──────────────────────────────────────────────┤
│ flyway_schema_history 기록:                 │
│  success = false ← 마이그레이션 실패 표시   │
│  error_message = "..."                      │
└──────────────────────────────────────────────┘
```

### 2. success=false 상태에서의 동작

```
flyway_schema_history:
┌─────────┬──────────────────────┬──────────┐
│ version │ description          │ success  │
├─────────┼──────────────────────┼──────────┤
│ 1       │ init                 │ true     │
│ 2       │ add_constraints      │ false ❌ │  ← 실패
│ 3       │ (아직 실행 안 됨)   │ -        │
└─────────┴──────────────────────┴──────────┘

다음 flyway migrate 시도:
├─ V1: ✅ 이미 성공, 건너뜀
├─ V2: ❌ success=false, "이미 실패한 마이그레이션"
│       → Flyway는 V2 재실행 대신 오류 발생
│       → "Fix the database state manually and run 'flyway repair'"
└─ V3: ⏸️ V2가 성공할 때까지 실행 안 됨
```

### 3. flyway repair의 동작

```python
def flyway_repair():
    # Step 1: success=false 레코드 삭제
    delete_failed_migrations()
    # DELETE FROM flyway_schema_history WHERE success = false;
    
    # Step 2: 파일 체크섬 재계산 (변조 감지)
    recalculate_checksums()
    # UPDATE flyway_schema_history 
    # SET checksum = calculate_new_checksum(file)
    # WHERE version IN (successful_versions);
    
    # Step 3: Repeatable 마이그레이션 체크섬 재계산
    recalculate_repeatable_checksums()
    
    # 결과: 히스토리 정리, 다음 migrate 가능
```

**repair 후 상태**:

```
flyway_schema_history:
┌─────────┬──────────────────────┬──────────┐
│ version │ description          │ success  │
├─────────┼──────────────────────┼──────────┤
│ 1       │ init                 │ true     │
│ 2       │ add_constraints      │ true ← repair로 표시 변경
│ 3       │ (아직 실행 안 됨)   │ -        │
└─────────┴──────────────────────┴──────────┘

이제 flyway migrate 시도:
├─ V1: ✅ 이미 성공, 건너뜀
├─ V2: ❌ success=true이지만 DB 상태는 부분 적용
│       repair는 DB를 고치지 않음!
│       → 여전히 오류 발생 가능
└─ 따라서 repair 전에 "수동으로 DB 문제를 해결"해야 함
```

---

## 💻 실전 실험

### 실험 1: 의도적으로 마이그레이션 실패 유도

```bash
# MySQL 8.0 실행
docker run -d --name mysql-repair \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=appdb \
  mysql:8.0

sleep 10

# Flyway 설정
mkdir -p migrations
cat > flyway.conf << 'EOF'
flyway.driver=com.mysql.cj.jdbc.Driver
flyway.url=jdbc:mysql://localhost:3306/appdb
flyway.user=root
flyway.password=password
flyway.locations=filesystem:./migrations
EOF
```

```sql
-- migrations/V001__init.sql
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2)
);

INSERT INTO products (name, price) VALUES ('Laptop', 999.99);
INSERT INTO products (name, price) VALUES ('Mouse', 25.50);
INSERT INTO products (name, price) VALUES ('Monitor', 0);  -- 문제: 가격 0
```

```sql
-- migrations/V002__add_constraint.sql
-- 이 마이그레이션이 실패할 것
ALTER TABLE products ADD CONSTRAINT check_price CHECK (price > 0);
UPDATE products SET price = 100 WHERE price = 0;
```

```bash
# Flyway 실행 (V002가 실패할 것)
$ flyway migrate

# 출력:
# Successfully applied 1 migration
# ❌ Migration of schema "appdb" to version "2" failed.
# Error: Constraint check_price failed
```

```bash
# 현재 상태 확인
$ mysql -u root -ppassword appdb -e \
  "SELECT version, description, success FROM flyway_schema_history;"

# 결과:
# version | description     | success
# 1       | init            | true
# 2       | add_constraint  | false ← 실패!

# DB 상태 확인
$ mysql -u root -ppassword appdb -e "DESCRIBE products;"
# price 컬럼에는 CHECK 제약이 없음 (DDL은 실패했음)

$ mysql -u root -ppassword appdb -e "SELECT * FROM products;"
# Monitor의 가격은 여전히 0 (UPDATE도 실패함)
```

### 실험 2: 복구 절차 실행

```bash
# Step 1: 오류 원인 파악
$ mysql -u root -ppassword appdb -e \
  "SELECT id, name, price FROM products WHERE price <= 0;"
# id | name    | price
# 3  | Monitor | 0     ← 제약 위반 원인

# Step 2: 데이터 정제
$ mysql -u root -ppassword appdb -e \
  "UPDATE products SET price = 99.99 WHERE id = 3;"

# Step 3: flyway repair
$ flyway repair
# ✅ Repair of schema "appdb" was successful

# 상태 확인
$ mysql -u root -ppassword appdb -e \
  "SELECT version, success FROM flyway_schema_history;"
# version | success
# 1       | true
# 2       | true  ← repair로 인해 true로 변경됨 (하지만 DB는 아직 부분 적용)

# Step 4: 다시 마이그레이션 실행
$ flyway migrate
# ✅ Successfully applied 1 migration

# 최종 확인
$ mysql -u root -ppassword appdb -e "DESCRIBE products;"
# price 컬럼에 CHECK 제약이 있음 ✅

$ mysql -u root -ppassword appdb -e "SELECT * FROM products;"
# id | name    | price
# 1  | Laptop  | 999.99
# 2  | Mouse   | 25.50
# 3  | Monitor | 99.99  ← 수정됨
```

### 실험 3: 마이그레이션 파일 수정 후 repair

```sql
-- migrations/V002__add_constraint_fixed.sql (수정)
-- 원본: ALTER TABLE products ADD CONSTRAINT check_price CHECK (price > 0);
-- 수정: 제약 조건을 완화 (NULL도 허용)
ALTER TABLE products 
ADD CONSTRAINT check_price CHECK (price > 0 OR price IS NULL);

UPDATE products SET price = 100 WHERE price = 0;
```

```bash
# 기존 V002 실패 상태에서
# 1. 파일 수정 (위의 SQL로 변경)

# 2. flyway repair
$ flyway repair
# Repair of schema "appdb" was successful

# 3. 마이그레이션 재실행
$ flyway migrate
# ✅ Successfully applied 1 migration (with adjusted constraints)
```

---

## 📊 성능/비용 비교

| 단계 | 작업 | 시간 | 위험 |
|------|------|------|------|
| **원인 파악** | 로그 분석, DB 상태 확인 | 1~5분 | 낮음 |
| **데이터 정제** | UPDATE/DELETE 실행 | 1분~1시간 | 중간 (데이터 손실 가능) |
| **flyway repair** | 히스토리 정리 | <1초 | 낮음 |
| **마이그레이션 재실행** | 전체 SQL 재실행 | 1분~1시간 | 중간 |

---

## ⚖️ 트레이드오프

### 신속한 대응 vs 신중한 검증

**신속** (1분):
- flyway repair + migrate 바로 실행
- 위험: DB 상태 불명확 → 데이터 손실 가능

**신중** (10~30분):
- 원인 파악 → 데이터 검증 → 정제 → repair → 재실행
- 안전: 데이터 보존, 스키마 일관성 보장

---

## 📌 핵심 정리

1. **마이그레이션 실패의 주요 원인**
   ```
   - SQL 문법 오류
   - 제약 조건 위반 (CHECK, FOREIGN KEY, UNIQUE)
   - 타임아웃 (대용량 UPDATE)
   - 리소스 부족
   - 동시성 제약 (다른 세션이 테이블 잠금)
   ```

2. **flyway_schema_history의 success=false는 경고**
   ```
   → flyway repair로 제거되지 않은 것
   → 반드시 원인을 먼저 해결해야 함
   ```

3. **flyway repair는 DB를 고치지 않고 히스토리만 정리**
   ```
   repair 전: 반드시 수동으로 DB 상태 정정
   repair 후: 마이그레이션 재실행 가능
   ```

4. **부분 실패 복구 절차**
   ```
   1. 오류 메시지 분석
   2. 현재 스키마 상태 확인 (DESCRIBE, SHOW CREATE)
   3. 실패 원인 제거 (데이터 정제, 조건 완화 등)
   4. flyway repair
   5. 마이그레이션 재실행
   ```

5. **MySQL의 DDL 특성**
   ```
   - 일부 DDL은 즉시 커밋 (ALTER TABLE, CREATE, DROP)
   - 따라서 부분 적용 상태 발생 가능
   - DML은 같은 트랜잭션에 포함되지만 DDL 후 자동 커밋
   ```

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: DDL 실패 후 자동 롤백되지 않는데, 부분 적용된 상태를 어떻게 정리하나?</strong></summary>

**답**:

**MySQL의 자동 COMMIT 특성**:

```sql
BEGIN;
ALTER TABLE users ADD COLUMN age INT;  -- ✅ 자동 COMMIT (DDL)
ALTER TABLE users ADD COLUMN phone VARCHAR(20);  -- ❌ 오류

-- 결과: age는 추가됨, phone은 안 추가됨
-- → 이미 커밋된 age를 취소할 수 없음
```

**정리 방법**:

```
Option 1: 새 마이그레이션에서 역방향 작업
V003__remove_partial_columns.sql:
ALTER TABLE users DROP COLUMN age;
ALTER TABLE users DROP COLUMN phone IF EXISTS;

Option 2: 다음 마이그레이션에서 설계 변경
V003__redesign_schema.sql:
-- age는 유지, phone은 다른 이름으로 재추가
ALTER TABLE users ADD COLUMN mobile_number VARCHAR(20);
-- 그 다음에 age를 정리할지 결정

Option 3: 수동 개입
mysql> ALTER TABLE users DROP COLUMN age;
flyway repair
flyway migrate
```

**패턴**:
```
실패한 DDL → 부분 적용 → 다음 마이그레이션에서 정리 (Forward-Only)
또는
실패한 DDL → 수동으로 정리 → flyway repair → 재실행
```

</details>

<details>
<summary><strong>Q2: flyway repair를 실행하면 히스토리가 손상되지 않나?</strong></summary>

**답**:

**repair는 히스토리를 "정리"하는 작업**:

```
repair 전:
┌─────────┬──────────────────┬─────────┐
│ version │ description      │ success │
├─────────┼──────────────────┼─────────┤
│ 1       │ init             │ true    │
│ 2       │ add_constraint   │ false ❌│
└─────────┴──────────────────┴─────────┘

repair 후:
┌─────────┬──────────────────┬─────────┐
│ version │ description      │ success │
├─────────┼──────────────────┼─────────┤
│ 1       │ init             │ true    │
│ 2       │ add_constraint   │ true ← repair로 인해 변경
└─────────┴──────────────────┴─────────┘

추가 변경 사항:
├─ 체크섬 재계산 (파일 변조 감지용)
├─ Repeatable 마이그레이션 체크섬 업데이트
└─ executed_at 갱신
```

**손상이 아니라 "정상화"**:
```
repair = "success=false를 제거하고, 체크섬 재계산"
→ 다음 migrate에서 V2를 이미 적용된 것으로 취급
→ V3부터 실행 가능
```

**repair 후 주의사항**:
```
1. repair 전에 반드시 DB 상태를 정상화해야 함
   (데이터 정제, 파일 수정 등)

2. repair는 DB를 고치지 않음!
   → repair 만 실행하고 마이그레이션 안 하면
   → 스키마는 부분 적용 상태로 남음

3. 팀원들에게 알리기
   → "repair 후 모두 migrate 필요"
```

</details>

<details>
<summary><strong>Q3: 대용량 마이그레이션(1000만 행 UPDATE) 중 타임아웃이 발생하면?</strong></summary>

**답**:

**대용량 작업의 부분 실패**:

```sql
-- V005__populate_large_table.sql
-- users 테이블에 1000만 건 행이 있음

UPDATE users SET status = 'ACTIVE' WHERE created_at < '2024-01-01';
-- ❌ 타임아웃 발생 (예: 30분 후)

-- 결과: 일부만 업데이트됨
-- 몇 건이 업데이트됐는가? → 모름!
-- → 다시 실행하면 이미 업데이트된 행도 다시 처리?
```

**대용량 UPDATE의 올바른 방식**:

```sql
-- V005__populate_status_batch.sql
-- 배치 처리: 한 번에 100만 건씩
UPDATE users SET status = 'ACTIVE' 
WHERE created_at < '2024-01-01' AND status != 'ACTIVE'
LIMIT 1000000;

-- V006__populate_status_batch_2.sql
UPDATE users SET status = 'ACTIVE' 
WHERE created_at < '2024-01-01' AND status != 'ACTIVE'
LIMIT 1000000;

-- V007, V008... (반복)
```

**또는 애플리케이션 로직으로 처리**:

```java
// Spring Boot 마이그레이션
@Component
public class UserStatusMigration {
    
    @Transactional
    public void populateStatus() {
        int batchSize = 100000;
        List<User> batch;
        
        do {
            batch = userRepository.findUnarchivedWithoutStatus(batchSize);
            batch.forEach(user -> user.setStatus("ACTIVE"));
            userRepository.saveAll(batch);
            entityManager.flush();
            entityManager.clear();
        } while (!batch.isEmpty());
    }
}

// Flyway Callback으로 실행
@Component
public class V006__PopulateStatusCallback implements FlywayCallback {
    
    @Override
    public void handle(FlywayEvent event) {
        if (event instanceof BeforeEachMigrateEvent) {
            migration.populateStatus();
        }
    }
}
```

**타임아웃 복구**:

```
1. 마이그레이션 로그에서 몇 건이 업데이트됐는지 확인
   mysql> SELECT COUNT(*) FROM users WHERE status = 'ACTIVE';

2. 남은 행 개수 확인
   mysql> SELECT COUNT(*) FROM users WHERE status IS NULL;

3. V006에서 남은 부분 처리
   UPDATE users SET status = 'ACTIVE' 
   WHERE status IS NULL AND created_at < '2024-01-01';

4. flyway repair
5. 마이그레이션 재실행
```

</details>

---

<div align="center">

**[⬅️ 이전: Flyway Undo 마이그레이션](./03-flyway-undo-migration.md)** | **[홈으로 🏠](../README.md)** | **[다음: 백업과 마이그레이션 ➡️](./05-backup-before-migration.md)**

</div>
