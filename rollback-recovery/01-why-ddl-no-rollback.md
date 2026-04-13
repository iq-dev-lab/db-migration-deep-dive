# 01. DDL 롤백이 없는 이유

---

## 🎯 핵심 질문

```
❓ 왜 BEGIN; ALTER TABLE users ADD COLUMN age INT; ROLLBACK; 명령어는 
   컬럼을 롤백하지 않고, DELETE 쿼리는 롤백될까?
```

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이그레이션 전략의 **가장 기초적인 사실**입니다. 이를 모르면:

- ❌ "오류 나면 ROLLBACK하면 되겠지" → **스키마 변경 사항이 DB에 남음**
- ❌ 마이그레이션 실패 후 복구 절차를 모름 → **중복 실행 방지 메커니즘 미작동**
- ❌ 설계 단계부터 롤백 불가를 고려하지 않음 → **Forward-Only 전략 불가능**

**MySQL과 PostgreSQL의 근본적 차이**를 이해하면, 각 DB별 마이그레이션 전략이 명확해집니다.

---

## 😱 흔한 실수 (Before — DDL 롤백을 시도한다)

```sql
-- 스테이징 환경에서 테스트 중
BEGIN;

ALTER TABLE orders ADD COLUMN shipping_address VARCHAR(255);
-- 100만 건 데이터 마이그레이션 로직...
UPDATE orders SET shipping_address = '...' WHERE ...;
-- ❌ 오류 발생: 컨스트레인트 위반

ROLLBACK;  -- "롤백하면 되지!"

-- 🚨 결과: 
-- ✅ UPDATE는 롤백됨 (데이터 변경 없음)
-- ❌ 하지만 shipping_address 컬럼은 여전히 존재!
-- ❌ BEGIN; 시점의 스키마로 돌아가지 않음
```

**내 생각**: "DDL도 트랜잭션에 포함되니까 ROLLBACK될 거야"

---

## ✨ 올바른 접근 (After — MySQL의 암묵적 COMMIT을 이해한다)

```sql
-- MySQL 8.0 기본 동작
BEGIN;
-- 암묵적 COMMIT! (DDL 자동 커밋)
ALTER TABLE orders ADD COLUMN shipping_address VARCHAR(255);

UPDATE orders SET shipping_address = '...' WHERE ...;
-- ❌ 오류 발생

ROLLBACK;  -- UPDATE만 롤백됨

-- 결과: shipping_address 컬럼은 영구적으로 추가됨!
```

**올바른 패턴**:

```sql
-- 1️⃣ DDL이 성공했는지 먼저 확인하는 로직 필요
DESCRIBE orders;  -- shipping_address가 있는가?

-- 2️⃣ DDL 오류 시 대응 SQL을 미리 준비
-- (마이그레이션 V002에서 롤백 대신 Forward-Only로 수정)

-- 3️⃣ DDL과 DML을 분리
-- DDL: V001__add_shipping.sql (스키마 변경)
-- DML: V002__migrate_shipping.sql (데이터 변환)
```

---

## 🔬 내부 동작 원리

### 1. MySQL의 암묵적 COMMIT (Implicit Commit)

MySQL은 **DDL 실행 시 자동으로 COMMIT을 수행**합니다.

```
┌─────────────────────────────────────────────────┐
│ BEGIN;                                          │
│   (트랜잭션 시작)                               │
├─────────────────────────────────────────────────┤
│ ALTER TABLE ... ADD COLUMN ...                  │
│   ✅ [자동 COMMIT] ← DDL 실행 완료              │
├─────────────────────────────────────────────────┤
│ INSERT/UPDATE (새로운 암묵적 트랜잭션 시작)     │
│ ROLLBACK;                                       │
│   ✅ 이것만 롤백됨 (스키마는 보존)              │
└─────────────────────────────────────────────────┘
```

**MySQL 공식 문서에서의 명시**:
> "Some statements cause an implicit commit, terminating any preceding transaction:
> - DDL statements: ALTER, CREATE, DROP, RENAME, TRUNCATE"

### 2. PostgreSQL의 Transactional DDL

PostgreSQL은 **DDL도 트랜잭션에 포함**됩니다.

```sql
-- PostgreSQL 15
BEGIN;

ALTER TABLE orders ADD COLUMN shipping_address VARCHAR(255);
-- 📝 트랜잭션 로그에 기록됨 (아직 커밋되지 않음)

UPDATE orders SET shipping_address = '...' WHERE ...;
-- 오류 발생

ROLLBACK;  
-- ✅ DDL과 DML 모두 롤백됨!
-- ✅ shipping_address 컬럼이 생성되지 않음
```

**PostgreSQL이 이를 지원하는 이유**:

1. **MVCC (Multi-Version Concurrency Control)**: 
   - 커밋 전까지 다른 세션에 컬럼이 보이지 않음
   - 스키마도 "버전"으로 관리됨

2. **DDL 로그**:
   - 모든 DDL을 `pg_catalog` 테이블에 기록
   - 롤백 시 역방향 작업 가능

3. **설계 철학**:
   - ACID 트랜잭션 원칙을 DDL에까지 확대

### 3. MySQL이 DDL을 Transactional로 만들지 않은 이유

**성능 최적화**:
- InnoDB가 DDL을 빠르게 처리하기 위해 별도 메커니즘 사용
- 트랜잭션 로그에 DDL을 기록하면 오버헤드 증가
- MySQL 5.1 이전: 테이블 복사 방식 (매우 느림)
- MySQL 5.1+: In-place 알고리즘 (빠름, 트랜잭션 미지원)

**호환성**:
- MySQL의 전통적 설계 (1995년부터)
- 많은 애플리케이션이 암묵적 COMMIT 동작 가정

### 4. InnoDB DDL 로그 (`ddl_log`)

MySQL도 완전한 롤백은 아니지만, **부분 복구 메커니즘**이 있습니다.

```sql
-- MySQL 내부 시스템 테이블 (사용자가 직접 수정하지 않음)
SELECT * FROM mysql.innodb_ddl_log;
-- id | page_no | log_type | table_id | index_id | ...
-- 1  | 0       | FREE     | 123      | 0        | ...
-- 2  | 512     | DELETE   | 123      | 456      | ...
```

**동작 원리**:

1. DDL 시작 → `ddl_log` 테이블에 작업 기록
2. DDL 중간에 충돌 발생
3. MySQL 재시작 후 → `ddl_log` 읽고 미완료 작업 복구
4. 하지만 "완전한 롤백"은 아님 (이미 커밋된 부분도 있음)

---

## 💻 실전 실험

### 실험 1: MySQL에서 DDL 롤백 시도

```bash
# MySQL 8.0 컨테이너 실행
docker run -d --name mysql8 \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=testdb \
  mysql:8.0 \
  --default-authentication-plugin=mysql_native_password

sleep 10
docker exec -it mysql8 mysql -u root -ppassword testdb
```

```sql
-- 1️⃣ 테스트 테이블 생성
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL
);

INSERT INTO products (name) VALUES ('Product A');
INSERT INTO products (name) VALUES ('Product B');

-- 2️⃣ 트랜잭션 시작
START TRANSACTION;

-- 3️⃣ DDL 실행
ALTER TABLE products ADD COLUMN price DECIMAL(10,2);

-- 4️⃣ DML 시도 (오류 유발)
UPDATE products SET price = -100;  -- 체크 제약 없으니 성공
INSERT INTO products (name, price) VALUES ('Product C', 99.99);

-- 5️⃣ 롤백
ROLLBACK;

-- 6️⃣ 확인
DESCRIBE products;
-- ✅ price 컬럼이 여전히 있음! (DDL은 롤백 안 됨)

SELECT * FROM products;
-- ✅ 데이터는 원래대로 (DML은 롤백됨)
```

**예상 결과**:
```
mysql> DESCRIBE products;
+-------+---------------+------+-----+---------+----------------+
| Field | Type          | Null | Key | Default | Extra          |
+-------+---------------+------+-----+---------+----------------+
| id    | int           | NO   | PRI | NULL    | auto_increment |
| name  | varchar(255)  | NO   |     | NULL    |                |
| price | decimal(10,2) | YES  |     | NULL    |                |  ← 롤백 안 됨!
+-------+---------------+------+-----+---------+----------------+
```

### 실험 2: PostgreSQL에서 DDL 롤백

```bash
# PostgreSQL 15 컨테이너 실행
docker run -d --name postgres15 \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=testdb \
  postgres:15

sleep 10
docker exec -it postgres15 psql -U postgres -d testdb
```

```sql
-- 1️⃣ 테스트 테이블 생성
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

INSERT INTO products (name) VALUES ('Product A');
INSERT INTO products (name) VALUES ('Product B');

-- 2️⃣ 트랜잭션 시작
BEGIN;

-- 3️⃣ DDL 실행
ALTER TABLE products ADD COLUMN price DECIMAL(10,2);

-- 4️⃣ DML 시도
UPDATE products SET price = 99.99;

-- 5️⃣ 롤백
ROLLBACK;

-- 6️⃣ 확인
\d products
-- ❌ price 컬럼이 없음! (DDL이 롤백됨)

SELECT * FROM products;
-- ✅ 데이터는 원래대로
```

**예상 결과**:
```
Table "public.products"
 Column |          Type          | Collation | Nullable |  Default
--------+------------------------+-----------+----------+--------------------
 id     | integer                |           | not null | nextval(...)
 name   | character varying(255) |           | not null |
-- price 컬럼이 없음! ← PostgreSQL은 DDL도 롤백됨
```

### 실험 3: BEGIN 전후 동작 비교

```sql
-- MySQL: DDL이 자동 COMMIT되므로 BEGIN의 의미 없음
BEGIN;
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
-- 이 순간 자동 COMMIT!
SHOW OPEN TABLES;  -- 빈 결과 (트랜잭션 없음)

-- PostgreSQL: DDL도 트랜잭션에 포함
BEGIN;
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
-- 아직 커밋되지 않음
SELECT current_transaction_isolation;  -- read committed
ROLLBACK;  -- DDL도 롤백됨
```

---

## 📊 성능/비용 비교

| 항목 | MySQL | PostgreSQL | 설명 |
|------|-------|-----------|------|
| **DDL 롤백** | ❌ 불가능 | ✅ 가능 | MySQL의 근본적 제약 |
| **스키마 변경 속도** | ⚡ 빠름 (In-place) | 중간 | MySQL이 더 빠른 이유: 트랜잭션 미지원 |
| **마이그레이션 전략** | Forward-Only | Rollback-capable | 설계 단계부터 다름 |
| **복구 복잡도** | 🔴 높음 | 🟢 낮음 | 실패 후 수동 복구 필요 |
| **충돌 복구** | InnoDB DDL 로그 | 자동 롤백 | MySQL은 부분 복구만 가능 |

---

## ⚖️ 트레이드오프

### MySQL의 선택 (성능 > 트랜잭션)
**장점**:
- ✅ DDL 속도가 빠름 (대용량 테이블에서도 몇 초)
- ✅ InnoDB 저수준 최적화 가능

**단점**:
- ❌ 마이그레이션 실패 시 스키마 상태 불명확
- ❌ 롤백 전략 없음 (Forward-Only만 가능)
- ❌ 감염된 스키마 정리에 추가 마이그레이션 필요

### PostgreSQL의 선택 (ACID > 성능)
**장점**:
- ✅ ACID 원칙 일관성
- ✅ 실패 시 완전 복구 가능
- ✅ 마이그레이션 전략 유연함

**단점**:
- ❌ DDL 속도가 상대적으로 느림
- ❌ MVCC 오버헤드

---

## 📌 핵심 정리

1. **MySQL은 DDL 실행 시 암묵적으로 COMMIT**
   ```
   BEGIN; ALTER TABLE...; → [자동 COMMIT] → DML 시작
   ```

2. **PostgreSQL은 DDL도 트랜잭션에 포함**
   ```
   BEGIN; ALTER TABLE...; → ROLLBACK; ✅ (롤백 가능)
   ```

3. **이 차이가 마이그레이션 전략을 결정**
   - MySQL: Forward-Only (실패 시 앞으로만 나감)
   - PostgreSQL: Rollback-capable (필요 시 이전 상태로 복원)

4. **MySQL 환경에서는 설계 단계부터 Forward-Only를 고려**
   - 롤백 불가를 전제로 마이그레이션 작성
   - 실패 시 별도 마이그레이션으로 수정

5. **InnoDB DDL 로그는 "완전한 롤백"이 아니라 "부분 복구"**
   - 충돌 복구용 (MySQL 재시작 후 미완료 작업 복구)
   - 사용자 차원에서 롤백할 수 없음

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: Flyway가 MySQL을 지원하는데, DDL 롤백이 없으면 마이그레이션 실패는 어떻게 처리하나?</strong></summary>

**답**:

Flyway는 **롤백 대신 `flyway_schema_history` 테이블의 `success` 플래그를 사용**합니다.

```sql
-- 마이그레이션 실패 후 히스토리
SELECT version, description, success, installed_on FROM flyway_schema_history;
-- 2    | add_shipping | false | 2024-01-15 10:23:45  ← 실패함
```

**다음 migrate 명령어 실행 시**:
```
❌ 오류: "V002 failed in previous run. Fix the database 
          state manually and run 'flyway repair'"
```

**해결 절차**:
1. 실패한 V002 마이그레이션의 오류 분석
2. DB에서 이미 실행된 부분 확인 (DESCRIBE, SELECT)
3. 실패한 부분을 수동으로 완료 또는 V003에서 수정
4. `flyway repair` 명령어로 success=false 제거
5. 다시 `flyway migrate` 실행

즉, **MySQL은 DDL 롤백이 없으므로 Flyway도 롤백을 하지 않고, 대신 수동 개입을 요구**합니다.

</details>

<details>
<summary><strong>Q2: MySQL에서 DDL과 DML을 같은 마이그레이션 파일에 넣으면 안 되나?</strong></summary>

**답**:

이론상 가능하지만 **실무에서는 피해야 할 안티패턴**입니다.

```sql
-- ❌ 안 좋은 예: V001__add_column_and_migrate.sql
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(10,2);
-- [여기서 암묵적 COMMIT 발생]
UPDATE orders SET total_amount = quantity * unit_price;
-- 오류 발생 (예: 데이터 무결성 제약 위반)

-- 결과: 
-- ✅ total_amount 컬럼은 추가됨 (DDL은 커밋됨)
-- ❌ UPDATE는 실패 (DML은 미완료)
-- → flyway repair 후 수동 개입 필요
```

**올바른 패턴** (DDL/DML 분리):

```
V001__add_total_amount_column.sql
├─ ALTER TABLE orders ADD COLUMN total_amount DECIMAL(10,2);

V002__populate_total_amount.sql
├─ UPDATE orders SET total_amount = quantity * unit_price;
```

**왜 분리하는가**:
1. DDL은 실패해도 스키마 변경 없음 (Forward-Only로 처리)
2. DML은 재시도 가능 (동일한 데이터 변환 반복)
3. 각 단계의 성공/실패를 명확히 추적

</details>

<details>
<summary><strong>Q3: PostgreSQL에서 DDL 롤백이 가능하면, 마이그레이션 전략이 MySQL과 완전히 다른가?</strong></summary>

**답**:

가능하지만 **실무에서는 여전히 Forward-Only를 권장**합니다.

```sql
-- PostgreSQL에서 가능한 것:
BEGIN;
ALTER TABLE users ADD COLUMN age INT;
UPDATE users SET age = 30 WHERE id = 1;  -- 부분 실패
ROLLBACK;  -- ✅ DDL도 롤백됨

-- 하지만 프로덕션에서는?
```

**문제점**:
1. **이미 배포된 앱이 새 컬럼을 사용 중**이면 롤백 불가
   - ROLLBACK 시 앱이 `age` 컬럼을 찾을 수 없음 → 런타임 오류

2. **Blue-Green 배포에서**는 양쪽 버전이 동시 동작
   - 구 버전은 `age` 컬럼을 모름
   - ROLLBACK 시 신 버전은 오류 발생

3. **다른 트랜잭션이 새 컬럼 사용 중이면**
   - ROLLBACK 실패 (Lock 발생)

**결론**:
```
PostgreSQL = DDL 롤백 기술적으로 가능
Forward-Only = 실무 권장 사항 (모든 DB 공통)

즉, "가능하다 != 해야 한다"
```

</details>

---

<div align="center">

**[⬅️ 이전: Chapter 3 — 외래 키 제약 관리](../zero-downtime-migration/07-foreign-key-management.md)** | **[홈으로 🏠](../README.md)** | **[다음: Forward-Only 마이그레이션 전략 ➡️](./02-forward-only-strategy.md)**

</div>
