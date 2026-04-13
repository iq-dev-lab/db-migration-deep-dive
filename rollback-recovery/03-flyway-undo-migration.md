# 03. Flyway Undo 마이그레이션

---

## 🎯 핵심 질문

```
❓ Flyway Teams(유료)의 Undo 마이그레이션이란?
   SQL 롤백이 불가능한데, Undo는 뭐가 다른가?
```

---

## 🔍 왜 이 개념이 실무에서 중요한가

**Undo ≠ 완전한 롤백**

이를 모르면:

- ❌ "Undo 파일 준비하면 마이그레이션 실패도 안심이겠네" → 위험한 착각
- ❌ Undo를 실행했는데 데이터가 사라짐 → 원인 모를 장애
- ❌ Flyway Teams 비용 지출했는데 기대만큼 안전하지 않음

**Undo의 진짜 역할**:
- ✅ 개발/스테이징 환경에서 빠른 반복 개발
- ✅ 잘못된 스키마를 즉시 정리
- ✅ 프로덕션에서는 Forward-Only와 병행

---

## 😱 흔한 실수 (Before — Undo가 마법의 롤백이라 생각)

### 실수 1: Undo 파일만 작성하고 Forward-Only 무시

```sql
-- V001__add_phone_column.sql (프로덕션 배포)
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
INSERT INTO users ... (phone으로 초기화);

-- U001__add_phone_column.sql (있으면 안심)
ALTER TABLE users DROP COLUMN phone;

-- 😱 프로덕션에서:
-- V001 실행 후 문제 발견
-- "Undo 파일이 있으니까 flyway undo 실행하면 되겠지"
-- 실행: flyway undo

-- 결과:
-- ✅ phone 컬럼 삭제됨
-- ❌ 하지만 앱은 이미 phone을 사용하는 상태
-- ❌ 다음 INSERT에서 "phone 컬럼 없음" 오류!
-- ❌ 프로덕션 장애 확산!
```

### 실수 2: 데이터 마이그레이션 후 Undo 실행

```sql
-- V001__add_status_column.sql
ALTER TABLE orders ADD COLUMN status VARCHAR(50) DEFAULT 'PENDING';

-- V002__populate_status_data.sql
UPDATE orders SET status = 'COMPLETED' WHERE created_at < '2024-01-01';
-- 1,000만 건 데이터 업데이트 완료

-- 며칠 뒤 "status 컬럼이 필요 없다"는 요구사항
-- Undo를 단순히 실행:
-- flyway undo  (V002 제거)
-- flyway undo  (V001 제거)

-- 결과:
// ✅ 컬럼 삭제됨
// ❌ 데이터 손실! (1000만 건의 status 정보 영구 손실)
// ❌ "왜 어제 업데이트한 데이터가 없지?" → 원인 모를 장애
```

### 실수 3: 프로덕션에서 Undo 시도

```
Flyway Community (무료):
├─ V 파일만 지원 (마이그레이션)
└─ Undo 불가능

Flyway Teams (유료):
├─ V + U 파일 지원
├─ flyway undo 명령어 사용 가능
└─ 하지만 프로덕션에서 사용 권장하지 않음!
```

**왜 프로덕션에서 안 되나?**

```
프로덕션 데이터는 "살아있는 데이터":
├─ 고객이 접근 중
├─ 외부 API가 참조 중
├─ 로그 시스템이 기록 중
└─ Undo로 되돌리면 데이터 불일치 발생!
```

---

## ✨ 올바른 접근 (After — Undo의 올바른 사용처)

### 올바른 사용처 1: 개발 로컬에서 빠른 반복

```bash
# 로컬 개발 중
# 마이그레이션 V003 작성 후 실행
$ flyway migrate
Successfully applied 1 migration

# "아, V003은 잘못 설계했다. 처음부터 다시."
$ flyway undo
Successfully undid 1 migration

# 스키마 초기화 후 V003 파일 수정
$ rm sql/V003__...sql
$ cat > sql/V003__correct_design.sql
ALTER TABLE ...  # 수정된 내용

# 다시 실행
$ flyway migrate
```

**장점**:
- ✅ 빠른 개발 반복
- ✅ 테이블 복사할 필요 없음
- ✅ 스키마 히스토리 정리

### 올바른 사용처 2: 스테이징에서 검증

```bash
# 스테이징: 프로덕션 데이터 사본으로 테스트

$ flyway migrate
# V001~V005 모두 적용됨

# "V005는 성능이 안 좋다. 설계 변경 필요"
$ flyway undo
# V005만 제거

# 새로운 V005__performance_optimized.sql 준비
$ flyway migrate
# V005 재적용 (새 설계로)

# 성능 벤치마크
$ ab -n 1000 http://localhost:8080/api/data
```

**중요 조건**:
- ✅ 아직 프로덕션 배포 안 됨
- ✅ 데이터는 사본이므로 손실해도 무방

### 올바른 사용처 3: 아직 배포 안 된 마이그레이션만 Undo

```sql
-- 마이그레이션 히스토리 현황:
-- V001: ✅ 프로덕션 배포됨
-- V002: ✅ 프로덕션 배포됨
-- V003: ❌ 아직 스테이징에만 있음 (프로덕션 미배포)

-- V003만 Undo 가능 (안전)
$ flyway undo

-- V001, V002는 Undo 금지 (프로덕션 영향)
```

### Undo 파일 작성 (U 파일)

```sql
-- V002__add_phone_column.sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
CREATE INDEX idx_phone ON users(phone);

-- U002__add_phone_column.sql (Undo 파일)
DROP INDEX idx_phone ON users;
ALTER TABLE users DROP COLUMN phone;
```

**Undo 파일 작성 팁**:

```sql
-- ❌ 단순 역순은 항상 안전한 게 아님
-- V001: CREATE TABLE users (id INT PRIMARY KEY, ...);
-- U001: DROP TABLE users;
-- 만약 users 테이블에 데이터가 있으면? → 데이터 손실!

-- ✅ Undo 전 조건 확인
-- U001: DROP TABLE IF EXISTS users;

-- ✅ 더 안전: 데이터 백업 후 삭제
-- U001: 
-- CREATE TABLE users_backup_20240115 AS SELECT * FROM users;
-- DROP TABLE users;

-- ✅ 외래 키 있는 경우: 순서 주의
-- V002: 
--   CREATE TABLE orders (id INT, user_id INT, FOREIGN KEY(user_id) REFERENCES users(id));
-- U002:
--   DROP TABLE orders;  -- 외래 키가 있는 쪽부터 삭제!
--   (users는 아직 남음)
```

---

## 🔬 내부 동작 원리

### 1. Flyway Teams의 Undo 메커니즘

```
┌──────────────────────────────────────────────┐
│ flyway_schema_history 테이블                 │
├──────────────────────────────────────────────┤
│ version │ description  │ type  │ success      │
├─────────┼──────────────┼───────┼──────────────┤
│ 1       │ init         │ SQL   │ true         │
│ 2       │ add_phone    │ SQL   │ true         │
│ 3       │ add_phone    │ UNDO  │ true ← 있음! │
│ 4       │ ...          │ SQL   │ ...          │
└──────────────────────────────────────────────┘

flyway undo 실행:
1. 마지막 SQL 마이그레이션(V2) 찾기
2. 해당하는 UNDO 파일(U002) 찾기
3. U002 실행
4. flyway_schema_history에 "UNDO" 레코드 추가
5. 히스토리 상 V2는 제거됨 (건너뛰게 됨)
```

### 2. Undo vs Rollback의 차이

```
롤백 (ROLLBACK / Transactional DDL):
├─ 자동 실행 (개발자가 할 일 없음)
├─ 모든 변경 사항 100% 복원
├─ 실패 불가능 (DB가 보장)
└─ PostgreSQL 지원

Undo (Flyway Teams):
├─ 수동 실행 (개발자가 flyway undo 명령)
├─ 개발자가 작성한 역 SQL 실행
├─ Undo 파일 버그 → 부분 복원 또는 실패
└─ Flyway Teams 지원 (유료)
```

### 3. Undo 실행 흐름

```python
def flyway_undo():
    # Step 1: 마지막 성공한 마이그레이션 찾기
    last_migration = get_last_successful_migration()
    # version=3, description="add_phone", type="SQL"
    
    # Step 2: 해당하는 Undo 파일 찾기
    undo_file = find_undo_file(last_migration.version)
    # U003__add_phone.sql
    
    if undo_file is None:
        raise MissingUndoFileError(f"U{last_migration.version} not found")
    
    # Step 3: Undo 파일 실행
    execute_sql(undo_file)
    # ALTER TABLE users DROP COLUMN phone;
    
    # Step 4: 히스토리 기록
    record_in_history(
        version=last_migration.version,
        type="UNDO",
        success=True,
        installed_on=datetime.now()
    )
    
    # Step 5: 메타데이터 정리
    # (V3는 다시 마이그레이션 대상이 됨)
```

### 4. 프로덕션 안전 장치

```sql
-- Flyway Community (무료)
$ flyway migrate  # ✅ 가능

-- Flyway Teams (유료)
$ flyway undo  # ⚠️ 가능하지만...

-- 하지만 권장하는 방식:
-- 1. 프로덕션: Forward-Only (새 V 파일로 수정)
--    V005__fix_v004_issue.sql
--    (V004의 설계 오류를 V005에서 보정)
--
-- 2. 스테이징/로컬: Undo 활용
--    빠른 개발 반복, 스키마 정리
```

---

## 💻 실전 실험

### 실험 1: Flyway Teams로 Undo 실행

```bash
# Flyway Teams 다운로드 및 설치 (유료)
# https://flywaydb.org/try

# 설정 파일
cat > flyway.conf << 'EOF'
flyway.driver=com.mysql.cj.jdbc.Driver
flyway.url=jdbc:mysql://localhost:3306/appdb
flyway.user=root
flyway.password=password
flyway.locations=filesystem:./sql
EOF

mkdir -p sql
```

```sql
-- V001__create_users.sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE
);

INSERT INTO users (email) VALUES ('user1@example.com');
INSERT INTO users (email) VALUES ('user2@example.com');

-- U001__create_users.sql (Undo 파일)
-- DROP TABLE users;
-- 위 주석을 풀면 안 됨! (데이터 손실 위험)
```

```bash
# 마이그레이션 실행
$ flyway migrate
# Successfully applied 1 migration

# 히스토리 확인
$ mysql -u root -ppassword appdb -e \
  "SELECT version, description, type, success FROM flyway_schema_history;"
# 1 | create users | SQL | true

# 데이터 확인
$ mysql -u root -ppassword appdb -e "SELECT * FROM users;"
# 1 | user1@example.com
# 2 | user2@example.com

# Undo 실행
$ flyway undo
# Successfully undid 1 migration

# 히스토리 확인 (UNDO 타입 추가)
# 1 | create users | SQL  | true
# 1 | create users | UNDO | true  ← 새로 추가됨

# 테이블 상태 확인
$ mysql -u root -ppassword appdb -e "SHOW TABLES;"
# (users 테이블 없음 - 삭제됨)
```

**문제점**: U001 파일에 문제가 있으면 어떻게 되나?

```sql
-- U001__create_users.sql (버그 있는 Undo)
DROP TABLE IF EXISTS users CASCADE;  -- CASCADE는 MySQL에서 미지원
ALTER TABLE products DROP CONSTRAINT fk_user;  -- 문법 오류

-- Undo 실행 시:
$ flyway undo
# ❌ Error: Unknown keyword: CASCADE
# → Undo 실패!
# → users 테이블은 남음 (부분 실패)
```

### 실험 2: 스테이징에서 Undo를 활용한 빠른 반복

```bash
# 스테이징 환경 설정
docker run -d --name mysql-staging \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=appdb \
  mysql:8.0

sleep 10

# 마이그레이션 디렉토리
mkdir -p migrations

cat > migrations/V001__users_schema.sql << 'EOF'
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
EOF

cat > migrations/U001__users_schema.sql << 'EOF'
DROP TABLE IF EXISTS users;
EOF

# V002 시작
cat > migrations/V002__add_status.sql << 'EOF'
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'ACTIVE';
CREATE INDEX idx_status ON users(status);
EOF

cat > migrations/U002__add_status.sql << 'EOF'
DROP INDEX idx_status ON users;
ALTER TABLE users DROP COLUMN status;
EOF

# 개발 과정
# Step 1: V001, V002 마이그레이션 실행
$ flyway migrate
# Successfully applied 2 migrations

# Step 2: V002 성능 테스트 후 "설계 변경 필요" 판단
$ flyway undo
# Successfully undid 1 migration

# Step 3: V002 파일 수정 (성능 개선)
cat > migrations/V002__add_status_optimized.sql << 'EOF'
ALTER TABLE users 
ADD COLUMN status VARCHAR(20) DEFAULT 'ACTIVE';
-- VARCHAR 길이 단축 (성능 개선)

-- 복합 인덱스 추가 (더 효율적)
CREATE INDEX idx_users_status_created 
ON users(status, created_at);
EOF

cat > migrations/U002__add_status_optimized.sql << 'EOF'
DROP INDEX idx_users_status_created ON users;
ALTER TABLE users DROP COLUMN status;
EOF

# Step 4: 수정된 V002 재실행
$ flyway migrate
# Successfully applied 1 migration

# 성능 비교
$ mysql -u root -ppassword appdb -e \
  "SHOW CREATE TABLE users\G"
```

### 실험 3: Undo 파일 없을 때의 오류

```bash
# 마이그레이션만 있고 Undo 파일이 없는 경우
cat > migrations/V003__add_phone.sql << 'EOF'
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
EOF

# U003__add_phone.sql는 생성하지 않음

$ flyway migrate
# Successfully applied 1 migration

$ flyway undo
# ❌ Error: Undo migration U003 not found
# 또는: Error: U migration not found (Flyway 버전에 따라)

# 해결책: U003 파일 생성
cat > migrations/U003__add_phone.sql << 'EOF'
ALTER TABLE users DROP COLUMN phone;
EOF

# 다시 시도
$ flyway undo
# ✅ Successfully undid 1 migration
```

---

## 📊 성능/비용 비교

| 항목 | Rollback | Forward-Only | Undo |
|------|----------|------------|------|
| **도구** | PostgreSQL | 모든 DB | Flyway Teams |
| **실행 속도** | 빠름 | 중간 | 중간 |
| **안전성** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **비용** | $0 | $0 | Flyway Teams 비용 |
| **프로덕션** | ✅ 권장 | ✅ 권장 | ❌ 비권장 |
| **개발** | ✅ 권장 | △ 괜찮음 | ✅ 권장 |

---

## ⚖️ 트레이드오프

### Undo의 장점
- ✅ 개발 속도 빠름 (빠른 반복)
- ✅ 스키마 정리 간단 (테이블 복사 불필요)
- ✅ 히스토리 정리 가능

### Undo의 단점
- ❌ Flyway Teams 유료 (비용)
- ❌ 개발자가 Undo 파일을 제대로 작성해야 함
- ❌ 프로덕션에서는 사용 불가 (Forward-Only만 가능)
- ❌ 데이터 마이그레이션이 포함되면 Undo 불안전

---

## 📌 핵심 정리

1. **Undo = "마지막 마이그레이션만 제거"**
   ```
   V001 ✅ → V002 ✅ → V003 ✅ → flyway undo → V003 ❌
   ```

2. **완전한 롤백이 아니라 "스키마 정리"**
   - DDL 부분은 되돌릴 수 있지만 (U 파일 실행)
   - DML은 보장하지 않음 (개발자가 U 파일 작성)

3. **개발/스테이징에서만 사용**
   - 로컬: 빠른 반복 개발
   - 스테이징: 설계 검증
   - 프로덕션: Forward-Only 적용

4. **Undo 파일 작성이 더 중요**
   ```sql
   V 파일: 마이그레이션 실행
   U 파일: 역방향 작업 (개발자가 손수 작성)
   ```

5. **프로덕션 안전 전략: Undo 금지, Forward-Only 권장**
   ```
   프로덕션: V005__fix_v004_issue.sql (수정)
   로컬: flyway undo (빠른 반복)
   ```

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: U 파일 작성 중 실수가 있으면 어떻게 되나? 예를 들어 DROP 구문 실패.</strong></summary>

**답**:

**Undo 실패 시나리오**:

```sql
-- V001: 초기 테이블
CREATE TABLE orders (id INT, user_id INT);
ALTER TABLE orders ADD CONSTRAINT fk_user 
  FOREIGN KEY (user_id) REFERENCES users(id);

-- U001: 잘못된 Undo (외래 키 먼저 삭제해야 함)
DROP TABLE orders;  -- ❌ 외래 키 제약이 있으면 실패 가능
```

**Undo 실행 결과**:

```
$ flyway undo
❌ Error: Cannot drop table 'orders' referenced by foreign key

flyway_schema_history 상태:
├─ V001 | SQL | true  ← 여전히 있음
├─ U001 | UNDO | false ← 실패!
```

**복구 절차**:

```sql
-- Step 1: U001 파일 수정 (외래 키 먼저 삭제)
-- U001__create_orders.sql (수정)
ALTER TABLE orders DROP FOREIGN KEY fk_user;
DROP TABLE orders;

-- Step 2: 수동으로 상태 정리
-- (또는 다시 마이그레이션 설정)

-- Step 3: 다시 시도
$ flyway undo
✅ Successfully undid 1 migration
```

**교훈**:
- U 파일도 테스트 필요 (로컬/스테이징에서)
- 외래 키, 인덱스 등 종속성 고려
- 복잡한 경우 Forward-Only가 나을 수도 있음

</details>

<details>
<summary><strong>Q2: Undo 파일에는 데이터 손실을 어떻게 방지하나?</strong></summary>

**답**:

Undo 파일 설계 시 **데이터 보존 패턴** 적용:

```sql
-- 패턴 1: 컬럼 삭제 전 백업 테이블 생성
-- V001__add_status.sql
ALTER TABLE orders ADD COLUMN status VARCHAR(50);

-- U001__add_status.sql (안전한 방식)
CREATE TABLE orders_status_backup_20240115 AS 
SELECT id, status FROM orders;
ALTER TABLE orders DROP COLUMN status;
```

```sql
-- 패턴 2: 테이블 삭제 전 백업
-- V002__create_temp_table.sql
CREATE TABLE temp_processing (
    id INT PRIMARY KEY AUTO_INCREMENT,
    data JSON
);

-- U002__create_temp_table.sql
CREATE TABLE temp_processing_backup_20240115 AS 
SELECT * FROM temp_processing;
DROP TABLE temp_processing;
```

```sql
-- 패턴 3: 중요 데이터는 다른 위치에 복사
-- V003__archive_users.sql
ALTER TABLE users ADD COLUMN archived_at TIMESTAMP NULL;

-- U003__archive_users.sql
CREATE TABLE users_with_archived_backup AS 
SELECT * FROM users WHERE archived_at IS NOT NULL;
ALTER TABLE users DROP COLUMN archived_at;
```

**하지만 완벽한 방법은?**

```
마이그레이션 전 백업 (별도 도구)
├─ RDS Snapshot
├─ mysqldump
└─ 물리 백업 (xtrabackup)

→ Undo보다 백업이 더 중요!
```

</details>

<details>
<summary><strong>Q3: Flyway Community를 사용 중인데, Undo 기능이 필요하면?</strong></summary>

**답**:

**옵션 1: Flyway Teams로 업그레이드** (비용 $)
- 공식 지원
- 일관된 버전 관리

**옵션 2: 수동으로 Forward-Only 구현**

```bash
# Undo 대신 Forward-Only로 대응

# 실수한 마이그레이션:
# V001__add_phone.sql (문제 발생)

# Flyway Community에서:
# 1. V002 마이그레이션 작성 (수정)
# V002__remove_phone.sql
# ALTER TABLE users DROP COLUMN phone;

# 2. 마이그레이션 수동 롤백 (DB만 되돌리기)
# mysql> DELETE FROM flyway_schema_history 
#         WHERE version = 1;
# mysql> ALTER TABLE users DROP COLUMN phone;

# 3. V002 마이그레이션 실행
$ flyway migrate
# Successfully applied 1 migration
```

**옵션 3: 로컬에서만 스키마 초기화**

```bash
# 로컬 개발 (문제 발생)
$ flyway clean  # ⚠️ 모든 마이그레이션 제거 (주의!)
$ rm sql/V001__problematic.sql
$ cat > sql/V001__correct_design.sql
$ flyway migrate
```

**권장 순서**:
1. 로컬: `flyway clean` + 파일 수정
2. 스테이징: Forward-Only (새 V002 작성)
3. 프로덕션: Forward-Only 적용

</details>

---

<div align="center">

**[⬅️ 이전: Forward-Only 마이그레이션 전략](./02-forward-only-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: 마이그레이션 실패 시 복구 절차 ➡️](./04-failure-recovery.md)**

</div>
