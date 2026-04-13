# Flyway 설정 옵션 완전 분석

---

## 🎯 핵심 질문

1. `baseline-on-migrate`가 정확히 무엇을 하며, 기존 데이터베이스에 Flyway를 적용할 때 왜 필수인가?
2. `out-of-order` 옵션이 활성화되면 마이그레이션 순서가 깨지는데, 이것이 실무에서 어떤 버그를 만드는가?
3. `ignore-missing-migrations`를 활성화하면 안 되는 이유는 무엇이고, 삭제된 마이그레이션 파일은 어떻게 처리해야 하는가?
4. `clean-disabled: false`인 환경에서 실수로 `flyway clean`을 실행하면 어떻게 되고, 복구는 가능한가?
5. `mixed: true` 설정이 MySQL에서 위험한 이유는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Flyway의 기본 동작만으로는 부족하며, 환경에 맞는 설정이 필수입니다. 예를 들어, 기존 프로덕션 DB에 Flyway를 처음 도입할 때 `baseline-on-migrate`를 모르면 모든 마이그레이션을 다시 실행하려고 시도하게 됩니다. 또한 `clean-disabled: false`로 설정된 프로덕션 환경에서 실수로 clean을 실행하면 모든 데이터가 삭제됩니다.

더 문제인 것은 불명확한 설정입니다. `ignore-missing-migrations: true`를 설정하면 누군가 실수로 마이그레이션 파일을 삭제해도 아무 경고가 없습니다. 이는 나중에 환경 간 스키마 불일치로 이어집니다. 이 장에서는 각 옵션의 의도와 위험성, 그리고 올바른 프로덕션 설정을 배웁니다.

---

## 😱 흔한 실수 (Before — ...)

```yaml
# ❌ 흔한 실수 1: baseline-on-migrate 없이 기존 DB에 Flyway 적용
spring:
  flyway:
    locations: classpath:db/migration
    # baseline-on-migrate 설정 안 함

# 상황: 프로덕션 DB는 이미 3년간 운영되며 100+ 테이블이 있음
# Spring Boot 시작 시 Flyway의 동작:
# 1. flyway_schema_history 테이블이 없음
# 2. V1__initial.sql부터 모든 마이그레이션 실행 시도
# 3. "Table 'users' already exists" 오류 → 앱 시작 실패!
# 4. 관리자는 혼동: "분명 이미 있는 테이블인데 왜 다시 만들려고?"
```

```yaml
# ❌ 흔한 실수 2: 프로덕션에서 clean-disabled를 false로 설정
spring:
  flyway:
    clean-disabled: false  # 위험!

# 프로덕션 운영 중, 개발자가 실수로 다음을 실행:
# $ flyway clean -url=jdbc:mysql://prod-db:3306/db -user=root -password=***
# 
# 결과:
# 1. 모든 테이블 삭제
# 2. 모든 뷰 삭제
# 3. 모든 저장 프로시저 삭제
# 4. 모든 인덱스 삭제
# 5. 수년간의 데이터 손실
# 6. 서비스 다운타임
# 7. 백업에서 복구 시작... (몇 시간 소요)
```

```yaml
# ❌ 흔한 실수 3: out-of-order 활성화로 순서 변경
spring:
  flyway:
    out-of-order: true  # 위험!

# 시나리오:
# [기존] V1, V2, V3 이미 적용됨
# [새로 추가] V1.5__new_feature.sql (버전 1과 2 사이)
#
# out-of-order: false (기본값)
# → 오류: Version 1.5 < 2 (out of order!)
#
# out-of-order: true
# → V1.5 실행 (V2, V3 이후에!)
# → 하지만 V1.5는 V2가 만든 테이블에 의존할 수 있음
# → 버그 또는 충돌 가능성 높음
```

```yaml
# ❌ 흔한 실수 4: ignore-missing-migrations 활성화
spring:
  flyway:
    ignore-missing-migrations: true

# 시나리오:
# [기존] flyway_schema_history에 V1, V2, V3 기록
# [실제] V1, V2, V3 파일이 모두 있음
# 
# 누군가 실수로 V2__add_column.sql 파일 삭제:
# $ rm src/main/resources/db/migration/V2__add_column.sql
#
# ignore-missing-migrations: false (기본값)
# → 오류: Missing migration V2 (파일이 없음!)
# → 개발자가 눈치챔: "어? V2 파일이 없네?"
# → 즉시 복구 가능
#
# ignore-missing-migrations: true
# → 아무 경고도 없음!
# → 다른 DB는 V2 적용, 이 DB는 V2 스킵
# → 환경 간 스키마 불일치 발생
# → 며칠 후 프로덕션 버그 발생
```

```sql
-- ❌ 흔한 실수 5: mixed: true에서 DDL + DML 혼합 (MySQL)
-- V1__migration.sql
BEGIN;
CREATE TABLE users (id INT PRIMARY KEY);
INSERT INTO users VALUES (1);
COMMIT;

-- MySQL의 동작:
-- InnoDB 기본값 + DDL 트랜잭션 불안정
-- CREATE TABLE은 implicit commit이 일어남
-- → INSERT가 실행되지 않거나 트랜잭션 분리
-- → DDL 후 DML이 별도 트랜잭션이 됨
```

---

## ✨ 올바른 접근 (After — ...)

```yaml
# ✅ 올바른 접근 1: 기존 DB에 Flyway 처음 도입
spring:
  flyway:
    baseline-on-migrate: true  # 기존 DB에만 필요
    baseline-version: "1"      # 선택사항
    baseline-description: "Initial baseline"  # 선택사항
    locations: classpath:db/migration

# 동작:
# 1. flyway_schema_history 테이블 없음 → 자동 생성
# 2. V1__initial.sql 존재 → baseline 버전 설정
# 3. flyway_schema_history에 baseline 레코드 삽입
#    (installed_rank=1, version=1, description="Initial baseline", success=true)
# 4. V1__initial.sql은 실행하지 않음 (이미 적용된 것으로 간주)
# 5. V2__add_column.sql부터 실행
# 
# 결과: 안전하게 Flyway 도입 가능
```

```yaml
# ✅ 올바른 접근 2: 프로덕션 설정 (안전성 최우선)
spring:
  datasource:
    url: jdbc:mysql://prod-db:3306/myapp
    username: flyway_user  # 전용 계정
    password: ${FLYWAY_PASSWORD}
  
  flyway:
    # 안전 설정
    clean-disabled: true              # CLEAN 명령어 금지
    validate-on-migrate: true         # 시작 시 검증
    out-of-order: false               # 순서 변경 금지
    ignore-missing-migrations: false  # 삭제된 파일 감지
    
    # 성능 설정
    baseline-on-migrate: false        # 프로덕션에는 불필요
    locations: classpath:db/migration
    
    # 에러 처리
    fail-on-validation-error: true    # 검증 실패 시 앱 시작 거부
    
    # 권장: 자체 설정 파일 분리
    # application-prod.yml에 위 내용 포함
    # 다른 환경에서는 다른 설정 적용
```

```yaml
# ✅ 올바른 접근 3: 개발/스테이징/프로덕션 설정 분리
# application-dev.yml
spring:
  flyway:
    baseline-on-migrate: true   # 개발은 자유
    clean-disabled: false       # 청소 가능
    out-of-order: true         # 순서 변경 가능

# application-stg.yml
spring:
  flyway:
    baseline-on-migrate: false  # 이미 초기화된 환경
    clean-disabled: true        # 청소 금지
    out-of-order: false         # 순서 엄격
    validate-on-migrate: true

# application-prod.yml
spring:
  flyway:
    baseline-on-migrate: false  # 절대 baseline 사용 금지
    clean-disabled: true        # 절대 clean 금지
    out-of-order: false         # 순서 엄격
    ignore-missing-migrations: false  # 파일 삭제 감지
    validate-on-migrate: true
    fail-on-validation-error: true
```

```sql
-- ✅ 올바른 접근 4: 삭제된 마이그레이션 처리
-- 상황: V2__add_column.sql 파일이 실수로 삭제됨
-- 
-- Step 1: 파일 복구 (버전 관리 시스템에서)
git checkout V2__add_column.sql
-- 또는 수동으로 코드 복사

-- Step 2: 검증
flyway validate
-- "Successfully validated 5 migrations"

-- Step 3: 다시 실행 (이미 적용됐으므로 스킵)
flyway migrate
-- "Successfully applied 0 migrations" (이미 다 적용됨)

-- 절대 하면 안 되는 것:
-- ignore-missing-migrations: true로 설정해서 무시하기
-- → 나중에 환경 간 불일치 발생
```

```yaml
# ✅ 올바른 접근 5: mixed 옵션 설정
# DDL + DML을 함께 사용해야 한다면:

spring:
  flyway:
    mixed: true  # 필수
    
# 하지만 SQL 작성 시 주의:
# - DDL과 DML을 분리하는 것이 최선
# - 꼭 섞어야 한다면 별도 파일로 관리
```

---

## 🔬 내부 동작 원리

### 1. baseline-on-migrate 메커니즘

```java
// Flyway의 내부 처리 로직 (의사 코드)
public void migrate() {
    // 1단계: flyway_schema_history 테이블 존재 여부 확인
    if (!schemaHistoryTableExists()) {
        createSchemaHistoryTable();
        
        // 2단계: baseline-on-migrate 검사
        if (baselineOnMigrate && schemaExists()) {
            // 기존 DB에 이미 스키마가 있음
            
            // baseline 레코드 삽입 (마이그레이션 실행 아님!)
            insertBaselineRecord(baselineVersion, baselineDescription);
            
            // 결과:
            // flyway_schema_history에
            // version='1', description='Initial baseline', success=true
            // 레코드가 추가됨
            
            // 이후 V1__initial.sql는 이미 적용된 것으로 간주
            return;  // 여기서 끝!
        }
    }
    
    // 3단계: 마이그레이션 실행 (기존 로직)
    List<Migration> pending = getPendingMigrations();
    for (Migration m : pending) {
        m.execute();
    }
}

// baseline-on-migrate의 실제 효과:
// Before:
//   flyway_schema_history: (비어있음)
//   실제 DB: users, products, orders 테이블 이미 존재
//   
// After baseline 삽입:
//   flyway_schema_history: version=1, success=true (실행 없음!)
//   실제 DB: users, products, orders 테이블 (변경 없음)
//   
// 그 다음 마이그레이션:
//   V1__initial.sql: 스킵 (이미 적용됨)
//   V2__add_column.sql: 실행 (미적용)
```

```sql
-- Baseline 레코드의 정확한 형태
INSERT INTO flyway_schema_history
(installed_rank, version, description, type, script, checksum, installed_by, installed_on, execution_time, success)
VALUES
(1, '1', 'Initial baseline', 'SQL', '<< Baseline >>', NULL, 'root', NOW(), 0, true);

-- 특징:
-- - installed_rank = 1 (첫 번째)
-- - script = '<< Baseline >>' (특별한 마커)
-- - execution_time = 0 (실행되지 않았으므로)
-- - checksum = NULL (파일이 없으므로)
-- - success = true (성공으로 간주)
```

### 2. out-of-order 옵션의 순서 검증

```java
public void validateMigrations() {
    // 적용된 모든 마이그레이션의 버전을 정렬
    List<String> appliedVersions = getAllAppliedVersions();
    // 예: ["1", "2", "3"]
    
    // 적용할 마이그레이션의 버전을 정렬
    List<String> pendingVersions = getAllPendingVersions();
    // 예: ["1.5", "4"]
    
    if (!out_of_order) {
        // 기본값: 순서 엄격
        // 마지막 적용된 버전: "3"
        // 새 마이그레이션: "1.5", "4"
        
        // 검사:
        // "1.5" < "3" → 오류! (out of order)
        // "4" > "3" → 허용
        
        if (pendingVersions.get(0).compareTo(appliedVersions.get(last)) < 0) {
            throw new FlywayException("Out of order migration");
        }
    } else {
        // out-of-order: true
        // 순서 제약 없음
        // 1.5 → 4 순서로 모두 실행
        // 하지만 "1.5"가 "3"에서 생성한 테이블을 사용하면?
        // → 의존성 오류 발생 가능!
    }
}
```

```
out-of-order: false (기본값)의 강력한 보호:
┌─────────────────────────────────────────────────────────────┐
│ Applied:   V1 → V2 → V3                                      │
│ Pending:   V1.5, V4                                          │
│                                                              │
│ V1.5은 V1과 V2 사이에 와야 하는데,                            │
│ V3이 이미 적용됐으므로 위험!                                 │
│                                                              │
│ out-of-order: false                                          │
│ → 오류 발생, 앱 시작 거부                                    │
│                                                              │
│ out-of-order: true                                           │
│ → V1.5 실행 (V3 이후에!)                                    │
│ → V1.5가 V3에서 생성한 컬럼에 의존하면 안전                  │
│ → 하지만 V2에만 의존한다면 실행 후에도 문제 없음             │
│ → "우연히 작동하는" 버그 발생 가능                           │
└─────────────────────────────────────────────────────────────┘
```

### 3. validate-on-migrate의 자동 검증

```java
public void migrateWithValidation() {
    if (validateOnMigrate) {
        // 1단계: 먼저 validation 실행
        validate();  // 모든 적용된 마이그레이션 체크섬 확인
        
        // 2단계: 모든 파일 존재 확인
        checkAllFilesExist();
        
        // 3단계: 버전 순서 확인
        validateVersionOrder();
        
        // 모든 검증 통과 후에만:
        migrate();  // 마이그레이션 실행
    } else {
        // validation 스킵하고 바로 실행 (위험!)
        migrate();
    }
}

// validate-on-migrate의 효과:
// Before: 앱 시작 → migrate() → 도중에 오류 발생 → 부분 적용 위험
// After:  앱 시작 → validate() (모든 검증) → 오류 없을 때만 migrate() → 안전
```

### 4. clean-disabled의 destroy 방지

```java
public void clean() {
    if (!cleanDisabled) {
        // clean 실행 허용
        dropAllObjects();  // 모든 테이블, 뷰, 함수 삭제
        dropSchema();      // 스키마 자체도 삭제
        // → 프로덕션에서 실행하면 재앙!
    } else {
        // clean-disabled: true (권장)
        throw new FlywayException("Clean is disabled");
    }
}

// 프로덕션 설정:
spring:
  flyway:
    clean-disabled: true  // 실수 방지

// 예외 시에만 실행:
// $ flyway -url=... -user=... -password=... \
//          -cleanDisabled=false clean  # 명시적으로 비활성화
```

---

## 💻 실전 실험

### 실험 1: baseline-on-migrate 실습

```bash
# Step 1: 기존 DB 생성 (마이그레이션 없이)
mysql -uroot -proot << 'EOF'
CREATE DATABASE legacy_db;
USE legacy_db;
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100)
);
INSERT INTO users VALUES (1, 'Alice');
EOF

# Step 2: Flyway 설정 파일 준비
cat > application.yml << 'EOF'
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/legacy_db
    username: root
    password: root
  flyway:
    baseline-on-migrate: true
    baseline-version: "1"
    baseline-description: "Legacy schema"
    locations: classpath:db/migration
EOF

# Step 3: 마이그레이션 파일 준비
mkdir -p src/main/resources/db/migration

cat > src/main/resources/db/migration/V1__initial.sql << 'EOF'
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100)
);
EOF

cat > src/main/resources/db/migration/V2__add_email.sql << 'EOF'
ALTER TABLE users ADD COLUMN email VARCHAR(100);
EOF

# Step 4: Spring Boot 앱 실행
# mvn spring-boot:run

# Step 5: 결과 확인
mysql -uroot -proot legacy_db << 'EOF'
SELECT * FROM flyway_schema_history;
-- 출력:
-- installed_rank | version | description | type
-- 1 | 1 | Legacy schema | SQL
-- 2 | 2 | add email | SQL

SELECT * FROM users;
-- 출력:
-- id | name | email
-- 1 | Alice | NULL
EOF
```

### 실험 2: out-of-order 오류 재현

```bash
# Step 1: 기본 마이그레이션 적용
cat > db/migration/V1__create.sql << 'EOF'
CREATE TABLE products (id INT PRIMARY KEY);
EOF

cat > db/migration/V2__add.sql << 'EOF'
ALTER TABLE products ADD COLUMN name VARCHAR(100);
EOF

# Flyway 실행
flyway migrate

# Step 2: 이제 V1.5 추가 (V2 이후!)
cat > db/migration/V1.5__insert_data.sql << 'EOF'
INSERT INTO products VALUES (1, 'Widget');
EOF

# Step 3: out-of-order: false (기본값)
flyway -outOfOrder=false migrate

# 출력:
# ERROR: Validate failed: Migrations have failed validation
# Migration checksum mismatch for migration version 1

# Step 4: out-of-order: true로 변경
flyway -outOfOrder=true migrate

# 출력:
# Migrating schema public to version 1.5 - insert data
# Successfully applied 1 migration
```

### 실험 3: clean-disabled 테스트

```bash
# Step 1: 데이터 있는 DB 준비
mysql -uroot -proot testdb << 'EOF'
CREATE TABLE important_data (id INT PRIMARY KEY, value VARCHAR(100));
INSERT INTO important_data VALUES (1, 'Critical Data');
EOF

# Step 2: clean-disabled: false (위험!)
cat > application.yml << 'EOF'
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/testdb
    username: root
    password: root
  flyway:
    clean-disabled: false  # ❌ 위험!
EOF

# Step 3: 실수로 clean 명령어 실행
flyway clean

# 결과:
# Cleaning schema public
# Schema public has been successfully cleaned

# Step 4: 데이터 확인
mysql -uroot -proot testdb -e "SHOW TABLES;"

# 출력:
# (비어있음! 모든 테이블 삭제됨!)

# ✅ 대책: clean-disabled: true로 설정
cat > application.yml << 'EOF'
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/testdb
    username: root
    password: root
  flyway:
    clean-disabled: true  # ✅ 안전!
EOF

# 재시도:
flyway clean

# 출력:
# ERROR: Clean is disabled
```

### 실험 4: validate-on-migrate 검증

```bash
# Step 1: 마이그레이션 파일 준비
cat > db/migration/V1__initial.sql << 'EOF'
CREATE TABLE users (id INT PRIMARY KEY);
EOF

# Step 2: 첫 실행 (성공)
flyway migrate
# Successfully applied 1 migration

# Step 3: V1 파일 수정 (오류 유도)
cat > db/migration/V1__initial.sql << 'EOF'
CREATE TABLE users (id INT PRIMARY KEY, extra_column INT);  -- 컬럼 추가
EOF

# Step 4: validate-on-migrate: true (기본값)
flyway -validateOnMigrate=true migrate

# 출력:
# ERROR: Validate failed: Migrations have failed validation
# Migration checksum mismatch for migration version 1
# (앱 시작 전에 오류 감지!)

# Step 5: validate-on-migrate: false (권장하지 않음)
flyway -validateOnMigrate=false migrate

# 출력:
# (아무것도 실행되지 않음, 이미 적용됨)
# (하지만 체크섬 불일치 문제는 여전히 존재)
```

### 실험 5: ignore-missing-migrations의 위험

```bash
# Step 1: 초기 상태
cat > db/migration/V1__create.sql << 'EOF'
CREATE TABLE users (id INT PRIMARY KEY);
EOF

cat > db/migration/V2__add_index.sql << 'EOF'
CREATE INDEX idx_users_id ON users(id);
EOF

# Flyway 실행
flyway migrate
# Successfully applied 2 migrations

# Step 2: V1 파일 삭제 (실수!)
rm db/migration/V1__create.sql

# Step 3: ignore-missing-migrations: false (기본값)
flyway -ignoreMissingMigrations=false migrate

# 출력:
# ERROR: Validate failed: Migrations have failed validation
# Missing migration V1 (file not found)
# → 개발자가 즉시 눈치챔!

# Step 4: ignore-missing-migrations: true (위험!)
flyway -ignoreMissingMigrations=true migrate

# 출력:
# (아무 경고 없음!)
# 
# 문제: 다른 환경에서는 V1이 있을 수 있음
# → 환경 간 스키마 불일치 발생
```

---

## 📊 성능/비용 비교

```
┌────────────────────────────────────────────────────────────────┐
│ 각 옵션의 성능 영향                                              │
├──────────────────────┬──────────┬──────────────────────────────┤
│ 옵션                 │ 성능영향 │ 특징                         │
├──────────────────────┼──────────┼──────────────────────────────┤
│ baseline-on-migrate  │ 무관     │ 한번만 사용, 이후 영향 없음  │
│ (true)               │ (+10ms)  │ 레코드 1개 삽입              │
├──────────────────────┼──────────┼──────────────────────────────┤
│ validate-on-migrate  │ 중간     │ 매 시작마다 모든 파일 검사   │
│ (true)               │ (+50ms)  │ 파일 100개 시 수백ms 가능   │
├──────────────────────┼──────────┼──────────────────────────────┤
│ out-of-order         │ 무관     │ 검증 로직만 추가, 실행 영향 │
│ (false)              │ (+5ms)   │ 없음                         │
├──────────────────────┼──────────┼──────────────────────────────┤
│ clean-disabled       │ 무관     │ 플래그만 체크, 성능 무관     │
│ (true)               │ (0ms)    │                              │
├──────────────────────┼──────────┼──────────────────────────────┤
│ mixed                │ 낮음     │ DDL + DML 처리 복잡도 증가  │
│ (true)               │ (+20ms)  │ 트랜잭션 관리 추가           │
└──────────────────────┴──────────┴──────────────────────────────┘

시작 시간 영향 (마이그레이션 100개 기준):
┌──────────────────────────────────────────────────────────────┐
│ 설정                        │ 예상 시간 │ 비고              │
├──────────────────────────────┼──────────┼─────────────────┤
│ validate-on-migrate: false   │ 100ms    │ 위험 (권장 아님) │
│ validate-on-migrate: true    │ 150ms    │ 권장 (안전)     │
│ validate-on-migrate: true    │ 200ms    │ 네트워크 느림   │
│ + ignore-missing: false      │          │ (DB 원거리)     │
└──────────────────────────────┴──────────┴─────────────────┘
```

---

## ⚖️ 트레이드오프

```
┌────────────────────────────────────────────────────────────────┐
│ baseline-on-migrate                                            │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 기존 DB에 Flyway 도입 가능                    │
│              │ - 레거시 시스템 마이그레이션 용이               │
│              │ - 한 번만 사용하면 안전                         │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 설정을 잘못하면 데이터 손실 위험              │
│              │ - 기존 스키마를 정확히 파악해야 함              │
│              │ - 문서화 필수                                   │
└──────────────┴─────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ validate-on-migrate                                            │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 시작 전 모든 문제 감지                        │
│              │ - 부분 적용 방지                                │
│              │ - 프로덕션 환경 안정성                          │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 시작 시간 10% ~ 20% 증가                     │
│              │ - 마이그레이션 파일 많을수록 느림               │
│              │ - 개발 환경에서는 과도할 수 있음                │
└──────────────┴─────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ clean-disabled                                                 │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 실수 방지 (100% 효과적)                       │
│              │ - 프로덕션 데이터 보호                          │
│              │ - 성능 오버헤드 없음                            │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 개발 환경에서 유연성 감소                     │
│              │ - DB 초기화 필요 시 수동 작업 필요              │
│              │ - 명시적 설정 필요                              │
└──────────────┴─────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ ignore-missing-migrations                                      │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 실수로 삭제된 파일 무시 가능                  │
│ (거의 없음)  │                                                 │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 환경 간 스키마 불일치 위험 (매우 높음)       │
│ (심각)       │ - 버전 관리 추적 불가능                        │
│              │ - 프로덕션 버그로 이어질 가능성 높음            │
│              │ - 절대 활성화하지 말 것!                       │
└──────────────┴─────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ out-of-order                                                   │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 중간 버전 삽입 가능                           │
│ (드물음)     │ - 레거시 시스템에서 유용                        │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 의존성 추적 어려움 (마이그레이션 순서 혼란)   │
│ (심각)       │ - 버그 재현 어려움                              │
│              │ - 팀 내 규칙 위반                               │
│              │ - 프로덕션에서 절대 금지                        │
└──────────────┴─────────────────────────────────────────────────┘
```

---

## 📌 핵심 정리

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. baseline-on-migrate: 기존 DB에만 사용                       │
│    - true: "이 DB는 이미 스키마가 있다" 표시                    │
│    - false: (기본값) "처음부터 마이그레이션 실행"               │
│    - 프로덕션: 항상 false                                      │
│                                                                 │
│ 2. validate-on-migrate: 항상 true (프로덕션 필수)              │
│    - 시작 전 모든 마이그레이션 검증                             │
│    - 체크섬, 파일 존재, 순서 확인                               │
│    - 개발: true, 프로덕션: true (필수)                         │
│                                                                 │
│ 3. clean-disabled: 프로덕션에서 항상 true                      │
│    - false: "clean 명령어 사용 가능" (위험!)                   │
│    - true: "clean 명령어 금지" (안전)                          │
│    - 한 명령어로 모든 데이터 삭제 방지                           │
│                                                                 │
│ 4. out-of-order: 프로덕션에서 항상 false                      │
│    - false: "버전 순서 엄격" (기본값, 권장)                     │
│    - true: "버전 순서 무관" (위험)                             │
│    - 의존성 추적 불가능 → 버그 가능성 높음                     │
│                                                                 │
│ 5. ignore-missing-migrations: 절대 true하지 말 것             │
│    - false: "삭제된 파일 감지" (기본값, 필수)                   │
│    - true: "삭제된 파일 무시" (절대 금지!)                     │
│    - 환경 간 스키마 불일치로 프로덕션 버그 발생                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🤔 생각해볼 문제

**Q1.** 프로덕션 DB에 처음 Flyway를 도입하는데, 기존 테이블이 100개, 저장 프로시저 50개가 있다. `baseline-on-migrate: true`를 사용하면 안전한가?

<details>
<summary>해설 보기</summary>

**답: 기술적으로는 안전하지만, 절차적으로는 매우 위험**

```
기술적 측면:
- baseline-on-migrate: true
- 현재 스키마를 "기존 상태"로 표시
- V1__initial.sql은 실행하지 않음
- 안전해 보임

실무 위험:
1. 문서화 부족
   - "왜 V1을 건너뛰나?" 궁금해하는 개발자
   - 3개월 후, "원래 V1은 뭐였지?" 혼동
   - 새로운 팀원이 V1을 실행해야 한다고 생각

2. 검증 불가능
   - "정말로 기존 스키마가 V1과 일치하나?"
   - 확인할 방법 없음 (기록이 없음)
   - 나중에 마이그레이션 시 불일치 발생

3. 버전 단절
   - V1: ??? (무엇이 있었는지 알 수 없음)
   - V2: add_column (기준점이 없음)
   - V3: create_index (V2가 필요한가? 확인 불가)

✅ 올바른 절차:
1. 현재 스키마를 SQL 파일로 내보내기 (SHOW CREATE TABLE 등)
2. V1__baseline_schema.sql 생성 (모든 CREATE 문 포함)
3. baseline-on-migrate: false (기본값)
4. flyway migrate 실행
   - V1 "기존 스키마" 레코드 삽입
   - V2부터 새 마이그레이션 적용
5. 기록 보존: V1의 내용이 SQL 파일로 영구 보존
```

</details>

---

**Q2.** 개발 환경에서는 `clean-disabled: false`를 사용하는데, 실수로 프로덕션 설정을 복사해서 사용했다. 어떤 위험이 있는가?

<details>
<summary>해설 보기</summary>

**답: 프로덕션 데이터 전체 삭제의 위험**

```
시나리오:
1. 개발자가 실수로 프로덕션 credentials를 dev 환경에 설정
   spring:
     datasource:
       url: jdbc:mysql://prod-db-ip:3306/prod_db  # ❌ 프로덕션 DB!
       username: prod_user
       password: ${PROD_PASSWORD}
     flyway:
       clean-disabled: false  # ❌ 위험!

2. CI/CD 자동화 스크립트가 실행:
   flyway clean

3. 결과:
   - 모든 테이블 삭제
   - 모든 뷰 삭제
   - 모든 저장 프로시저 삭제
   - 모든 데이터 삭제
   - 서비스 완전 다운

4. 복구:
   - 백업에서 복구 (시간: 몇 시간 ~ 몇 일)
   - 영업손실: 서비스 이용 불가
   - 이미지 손상: 신뢰도 하락

✅ 방지 방법:
1. clean-disabled: true (프로덕션 필수!)
   spring:
     datasource:
       url: jdbc:mysql://prod-db-ip:3306/prod_db
     flyway:
       clean-disabled: true  # ✅ 안전!

2. clean 명령어 실행 불가:
   $ flyway clean
   > ERROR: Clean is disabled
   > (명시적으로 비활성화해야만 가능)

3. 명시적 비활성화 (마지막 방어선):
   $ flyway -cleanDisabled=false clean
   > (여전히 실행 불가, DB 계정 권한 문제 등으로 추가 보호)

4. 프로덕션 DB 계정 권한 제한:
   $ GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, DROP 
     ON prod_db.* TO 'flyway_user'@'%';
   > (GRANT 권한 제거해서 DDL 실행 불가)
```

</details>

---

**Q3.** `validate-on-migrate: true`로 설정했는데, 마이그레이션 파일 500개가 있다. 매번 시작 시 모든 파일을 검증하려니 앱 시작 시간이 5초 증가했다. 성능을 개선하려면?

<details>
<summary>해설 보기</summary>

**답: 환경별 전략 수립 필요**

```
원인 분석:
- 500개 파일 × 각 5~10ms = 2.5~5초
- 네트워크 지연이 크면 더 증가
- DB 쿼리 (체크섬 비교) 반복

해결책 1: 환경별 설정 분리
development:
  flyway:
    validate-on-migrate: false  # 개발은 빠르게
    
staging:
  flyway:
    validate-on-migrate: true   # 스테이징은 안전하게
    
production:
  flyway:
    validate-on-migrate: true   # 프로덕션은 반드시

해결책 2: 마이그레이션 파일 정리
- 너무 오래된 마이그레이션 아카이빙
  - V1~V100: 2020년 (더 이상 변경 안 함)
  - V101~V500: 2021~현재
- 별도 아카이브 폴더로 이동 (validate 제외)

해결책 3: 병렬 검증 (Flyway Pro 기능)
- 여러 파일을 동시에 검증
- 하지만 커뮤니티 버전 미지원

해결책 4: 조건부 검증
- 첫 시작: validate-on-migrate: true
- 이후 재시작: validate-on-migrate: false (선택사항)
- 하지만 안정성 감소

✅ 실무 권장:
- 프로덕션: 5초 증가는 문제 아님 (한 번 시작)
- 개발: 개발자 환경에서만 validate 비활성화
- CI/CD: 모든 파일 검증 필수 (배포 전)
```

</details>

---

<div align="center">

**[⬅️ 이전: 마이그레이션 유형](./02-migration-types.md)** | **[홈으로 🏠](../README.md)** | **[다음: Java 기반 마이그레이션 ➡️](./04-java-migration.md)**

</div>
