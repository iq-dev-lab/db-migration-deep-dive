# 마이그레이션 버전 충돌

---

## 🎯 핵심 질문

두 개발자가 동시에 마이그레이션 파일을 작성해서 같은 버전 번호로 커밋하면 무엇이 먼저 실행될까요? 왜 충돌이 발생하고, 어떻게 예방할까요?

## 🔍 왜 이 개념이 실무에서 중요한가

마이그레이션은 **순서가 매우 중요**합니다. V3이 V2보다 먼저 실행되면 의존성이 깨져서 전체 배포가 실패합니다. 특히 팀에서 여러 개발자가 마이그레이션을 작성할 때 버전 충돌은 빈번하게 발생합니다. 충돌을 감지하지 못하면 스키마 불일치로 인한 데이터 손상이나 앱 오류가 발생할 수 있습니다.

---

## 😱 흔한 실수 (Before)

### 시나리오: 두 개발자가 동시에 V3 생성

**개발자 A (Feature A - users 테이블 추가)**

```bash
$ git checkout -b feature/add-users main
# V3__Add_users_table.sql 작성
$ git add db/migration/
$ git commit -m "Add users table migration"
```

**개발자 B (Feature B - orders 테이블 추가)**

```bash
$ git checkout -b feature/add-orders main
# V3__Add_orders_table.sql 작성
$ git add db/migration/
$ git commit -m "Add orders table migration"
```

**main 브랜치에 A 먼저 머지됨**

```bash
$ git checkout main
$ git merge feature/add-users
# Flyway: V3__Add_users_table.sql 실행 ✓
```

**B도 main에 머지하려고 함**

```bash
$ git merge feature/add-orders
# 파일 수준의 충돌은 없음 (다른 파일이므로)
# 하지만 Flyway 버전 충돌 발생!
```

**배포 시 Flyway 오류**

```
ERROR: Schema version 3 has been already applied.
Duplicate schema version: 3
SQL State: [ERROR_1]
```

### 왜 이 실수가 심각한가?

1. **Git 레벨에선 충돌 없음** - 두 개의 다른 파일이므로 merge conflict가 없음
2. **런타임에 발견됨** - 배포 단계에서 느리게 감지됨
3. **배포 실패** - 전체 배포 파이프라인이 중단됨
4. **타이밍 문제** - 어느 파일이 먼저 실행될지 예측 불가

---

## ✨ 올바른 접근 (After)

### 1. 팀 규칙 수립: 타임스탬프 기반 버전 관리

마이그레이션 파일명에 **타임스탬프**를 포함시켜 자동으로 순서가 결정되도록 함:

```bash
# ❌ 나쁜 예
V3__Add_users_table.sql
V3__Add_orders_table.sql

# ✅ 좋은 예
V3__20240415_143000__Add_users_table.sql
V3__20240415_144530__Add_orders_table.sql
```

### 2. 타임스탬프 기반 버전 정의

Flyway 설정 (`application.yml`):

```yaml
spring:
  flyway:
    baseline-on-migrate: false
    out-of-order: false
    validate-on-migrate: true
    clean-disabled: true
    placeholders:
      app-name: payment-service
```

마이그레이션 파일명 규칙:

```
V{major}__{yyyyMMdd_HHmmss}__{description}.sql
```

**예시:**
- `V1__20240101_100000__Initial_schema.sql` (1월 1일 10시)
- `V2__20240415_143000__Add_users_table.sql` (4월 15일 14시 30분)
- `V2__20240415_144530__Add_orders_table.sql` (4월 15일 14시 45분)

### 3. 마이그레이션 작성 프로세스 (팀 가이드)

```bash
# 1. main 브랜치에서 최신 상태 확인
git checkout main
git pull origin main

# 2. 로컬 마이그레이션 파일 확인
ls -la db/migration/ | tail -5

# 3. Feature 브랜치 생성 (마이그레이션 후 생성!)
git checkout -b feature/add-users-table

# 4. 현재 시간 기반 버전 생성
# 2024-04-15 14:30:30 → V2__20240415_143030__Add_users_table.sql
cat > db/migration/V2__20240415_143030__Add_users_table.sql << 'EOF'
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
EOF

# 5. 커밋 및 PR 생성
git add db/migration/
git commit -m "feat: add users table"
git push origin feature/add-users-table

# 6. PR 머지 전에 다시 한 번 main 확인!
git fetch origin
git rebase origin/main
# → 충돌 시 여기서 발견
```

---

## 🔬 내부 동작 원리

### 1. Flyway의 버전 비교 메커니즘

Flyway는 `flyway_schema_history` 테이블에서 **버전 번호**로 중복을 감지합니다:

```sql
-- flyway_schema_history 구조
SELECT * FROM flyway_schema_history ORDER BY installed_rank;

+---------------+---------+----------------------------+--------+-----------+
| version       | type    | script                     | status | checksum  |
+---------------+---------+----------------------------+--------+-----------+
| 1             | SQL     | V1__Initial_schema.sql     | OK     | 123...    |
| 2             | SQL     | V2__20240415_143000__...   | OK     | 456...    |
| 2             | SQL     | V2__20240415_144530__...   | PENDING| 789...    | ← 충돌!
+---------------+---------+----------------------------+--------+-----------+
```

**Flyway 검증 로직:**

```java
// org.flywaydb.core.internal.schemahistory.SchemaHistory
public void validate() {
    // 데이터베이스에 적용된 마이그레이션들
    List<ResolvedMigration> appliedMigrations = getAppliedMigrations();
    
    // 파일 시스템에서 발견한 마이그레이션들
    List<ResolvedMigration> availableMigrations = scanner.scan();
    
    // 같은 버전이 중복되었는지 확인
    for (ResolvedMigration available : availableMigrations) {
        for (ResolvedMigration applied : appliedMigrations) {
            if (available.getVersion().equals(applied.getVersion())) {
                // 체크섬이 다르면 오류
                if (!available.getChecksum().equals(applied.getChecksum())) {
                    throw new FlywayException("Migration version mismatch");
                }
                // 둘 다 같은 버전인데 파일이 다르면 충돌
                if (!available.getScript().equals(applied.getScript())) {
                    throw new FlywayException("Duplicate version");
                }
            }
        }
    }
}
```

### 2. 타임스탬프 방식이 충돌을 방지하는 원리

타임스탬프를 포함하면 **버전 문자열이 자동으로 정렬됩니다:**

```
V2__20240415_143000__Add_users_table.sql    ← 14:30:00 에 생성
V2__20240415_144530__Add_orders_table.sql   ← 14:45:30 에 생성

정렬 순서: 143000 < 144530
→ 자동으로 올바른 순서 보장!
```

Flyway는 마이그레이션 파일을 **버전 번호로 정렬**한 후 실행:

```java
List<ResolvedMigration> migrations = scanner.scan();
Collections.sort(migrations, (a, b) -> {
    // MigrationVersion 비교: 숫자 + 문자열
    return a.getVersion().compareTo(b.getVersion());
});
```

**MigrationVersion 비교 규칙:**

```
V1 < V2 < V3 < V10
V2__A < V2__B < V2__Z (알파벳 순)
V2__20240415_143000 < V2__20240415_144530 (타임스탬프 순)
```

### 3. installed_rank vs version

DB에 저장되는 실제 순서:

```sql
-- installed_rank: 실제 실행 순서 (auto increment)
-- version: 마이그레이션 버전 번호

SELECT installed_rank, version, script FROM flyway_schema_history;

+----------------+---------+-----------------------------------+
| installed_rank | version | script                            |
+----------------+---------+-----------------------------------+
| 1              | 1       | V1__Initial_schema.sql            |
| 2              | 2       | V2__20240415_143000__Add_users... |
| 3              | 2       | V2__20240415_144530__Add_orders.. |
+----------------+---------+-----------------------------------+

-- ❌ 같은 version 값을 가진 두 개의 레코드 → 충돌!
```

---

## 💻 실전 실험

### 실험 1: 버전 충돌 재현

**시나리오 설정 (Docker 컨테이너):**

```bash
# MySQL 시작
docker run -d --name mysql-test \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=testdb \
  mysql:8.0

# 잠깐 기다린 후 접속
mysql -h localhost -u root -ppassword testdb
```

**초기 마이그레이션 파일들:**

```bash
mkdir -p db/migration

# V1 (모든 개발자가 가진 버전)
cat > db/migration/V1__Initial_schema.sql << 'EOF'
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL
);
EOF

# V2 (공통)
cat > db/migration/V2__Add_email.sql << 'EOF'
ALTER TABLE users ADD COLUMN email VARCHAR(255);
EOF
```

**Flyway 실행 (첫 번째):**

```bash
mvn flyway:migrate -Dflyway.url=jdbc:mysql://localhost:3306/testdb \
  -Dflyway.user=root -Dflyway.password=password

# 결과: V1, V2 적용됨
```

**충돌 시뮬레이션 (Feature A):**

```bash
cat > db/migration/V3__Add_users_table.sql << 'EOF'
ALTER TABLE users ADD COLUMN status VARCHAR(50);
EOF
```

**충돌 시뮬레이션 (Feature B - 같은 버전):**

```bash
cat > db/migration/V3__Add_orders_table.sql << 'EOF'
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
EOF

# 이제 두 개의 V3__*.sql 파일이 있음!
ls db/migration/V3*.sql
```

**Flyway 실행 시도:**

```bash
mvn flyway:migrate -Dflyway.url=jdbc:mysql://localhost:3306/testdb \
  -Dflyway.user=root -Dflyway.password=password

# 오류 메시지:
# [ERROR] ERROR: Migration version 3 has been already applied
# [ERROR] Duplicate schema version: 3
```

### 실험 2: out-of-order=true로 해결

**위험한 해결책 (권장하지 않음):**

```yaml
spring:
  flyway:
    out-of-order: true  # 순서 상관없이 실행
```

```bash
mvn flyway:migrate -Dflyway.outOfOrder=true

# 결과: 두 V3 파일이 모두 실행됨
# 하지만 어떤 순서로 실행될지 예측 불가!
```

**확인:**

```sql
SELECT installed_rank, version, script FROM flyway_schema_history 
ORDER BY installed_rank DESC LIMIT 3;

+----------------+---------+-----------------------------------+
| installed_rank | version | script                            |
+----------------+---------+-----------------------------------+
| 4              | 3       | V3__Add_orders_table.sql (or...)  | ← 순서 불명
| 3              | 3       | V3__Add_users_table.sql (or...)   | ← 순서 불명
| 2              | 2       | V2__Add_email.sql                 |
+----------------+---------+-----------------------------------+
```

### 실험 3: 타임스탬프 방식으로 올바르게 해결

**파일 이름 변경:**

```bash
# 기존 파일 제거
rm db/migration/V3__Add_users_table.sql
rm db/migration/V3__Add_orders_table.sql

# 타임스탬프로 재생성 (Feature A - 14시 30분)
cat > db/migration/V3__20240415_143000__Add_status_column.sql << 'EOF'
ALTER TABLE users ADD COLUMN status VARCHAR(50);
EOF

# 타임스탐프로 재생성 (Feature B - 14시 45분)
cat > db/migration/V3__20240415_144530__Add_orders_table.sql << 'EOF'
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
EOF

# DB 초기화 (테스트용)
mvn flyway:clean
```

**Flyway 재실행:**

```bash
mvn flyway:migrate -Dflyway.url=jdbc:mysql://localhost:3306/testdb

# 성공! 순서대로 실행됨:
# V1__Initial_schema.sql
# V2__Add_email.sql
# V3__20240415_143000__Add_status_column.sql
# V3__20240415_144530__Add_orders_table.sql
```

**DB 확인:**

```sql
SELECT installed_rank, version, script FROM flyway_schema_history;

+----------------+---------+-------------------------------------------+
| installed_rank | version | script                                    |
+----------------+---------+-------------------------------------------+
| 1              | 1       | V1__Initial_schema.sql                    |
| 2              | 2       | V2__Add_email.sql                         |
| 3              | 3       | V3__20240415_143000__Add_status_column.sql|
| 4              | 3       | V3__20240415_144530__Add_orders_table.sql |
+----------------+---------+-------------------------------------------+

-- ✅ 다른 버전 번호라고 취급할 수 있게 정렬됨!
```

---

## 📊 성능/비용 비교

| 방식 | 장점 | 단점 | 비용 |
|------|------|------|------|
| **순차 버전** (V1, V2, V3...) | 간단함, 읽기 쉬움 | 충돌 가능성 높음, 빠른 감지 어려움 | 높음 (충돌 재작업) |
| **타임스탐프** (V1__yyyyMMdd...) | 자동 순서 보장, 충돌 없음 | 파일명이 길어짐 | 낮음 (충돌 zero) |
| **out-of-order=true** | 충돌 무시 | 실행 순서 예측 불가, 데이터 손상 위험 | 매우 높음 (버그 디버깅) |
| **UUID 기반** (V1__uuid__desc) | 완전히 고유함 | 순서 추적 어려움 | 중간 (관리 복잡성) |

**권장:** 타임스탐프 기반 (장점 대비 비용 최소)

---

## ⚖️ 트레이드오프

### 1. 단순성 vs 안전성

```
순차 버전:    V1, V2, V3 (매우 읽기 쉬움)
            ↓
            충돌 발생 → 수정 필요 → 재작업

타임스탐프:  V1__20240415_143000... (약간 복잡)
            ↓
            충돌 zero → 자동 처리 완료
```

**권장:** 타임스탐프 (초기 학습 비용 << 충돌 재작업)

### 2. 구버전 앱과의 호환성

타임스탐프 도입 시 기존 마이그레이션 파일을 변경하면 안 됩니다:

```sql
-- ❌ 기존 DB에 이미 적용된 상태
V1__Initial_schema.sql        (checksum: 123abc)
V2__Add_email.sql             (checksum: 456def)

-- ❌ 파일명을 변경하면
V1__20240101_100000__Initial_schema.sql  (checksum: 123abc)
    ↑ 새로운 파일명, 구 DB에서 찾을 수 없음!
```

**해결책:**

```yaml
spring:
  flyway:
    # 기존 migration 무시, 새것부터 타임스탐프 시작
    baseline-version: "2"
    baseline-on-migrate: true
```

---

## 📌 핵심 정리

1. **버전 충돌의 근본 원인**: 두 마이그레이션이 같은 version 번호를 가지면 Flyway가 감지
2. **Git은 도움이 안 됨**: 파일이 다르면 merge conflict가 없어서 런타임에 발견됨
3. **타임스탐프 기반 이름**: `V{major}__{yyyyMMdd_HHmmss}__{description}.sql`로 자동 정렬
4. **팀 규칙 수립**: main pull 후 feature 브랜치 생성, PR 머지 전 재검증
5. **out-of-order=true는 최후의 수단**: 순서 보장 불가, 데이터 손상 위험
6. **CI 검증**: `flyway validate`를 PR 단계에 추가해서 조기 감지

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 타임스탐프 형식이 고정되지 않으면 어떻게 될까요?</strong></summary>

**문제 상황:**
```
V2__20240415_143000__Add_users.sql       (개발자 A)
V2__20240415_143000__Add_users_v2.sql    (개발자 B, 같은 시간!)
```

**정렬 결과:**
```
V2__20240415_143000 vs V2__20240415_143000
→ 다음 문자열로 비교: "__Add_users" vs "__Add_users_v2"
→ "__Add_users" < "__Add_users_v2"
```

**영향:**
- 데이터베이스에는 정렬된 순서로 저장
- 초 단위 충돌 시 파일명 알파벳 순으로 결정 (예측 불가)

**해결책:**
- 타임스탐프 정밀도를 **분 단위**로 유지: `yyyyMMdd_HHmm`
- 또는 **나노초** 포함: `yyyyMMdd_HHmmss_SSS`
- 팀에서 **절대 같은 시간에 마이그레이션 생성하지 않기** 규칙

</details>

<details>
<summary><strong>Q2: 만약 실수로 V5를 건너뛰고 V6을 생성했다면?</strong></summary>

**상황:**
```
flyway_schema_history:
V1 ✓
V2 ✓
V3 ✓
V4 ✓
(V5 누락!)
V6 생성하려고 함
```

**Flyway 동작:**
```bash
mvn flyway:validate

# 오류: Schema version 5 is missing
# Version sequence issue detected!
```

**해결책:**
```bash
# V5 파일 생성
cat > db/migration/V5__20240415_143000__Missing_migration.sql << 'EOF'
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
EOF

# 재시도
mvn flyway:validate  # ✓ 통과
mvn flyway:migrate   # ✓ 실행
```

**교훈:** Flyway는 **연속된 버전 번호**를 강제함. 번호를 건너뛰면 안 됨.

</details>

<details>
<summary><strong>Q3: 체크섬 불일치는 언제 발생하고 어떻게 대응할까요?</strong></summary>

**시나리오:**
```
# 기존 적용된 V2
SELECT checksum FROM flyway_schema_history WHERE version = 2;
→ checksum: 1a2b3c4d5e6f

# 그 후 파일을 실수로 수정함
cat db/migration/V2__Add_email.sql
-- 내용 변경됨! (예: ALTER → CREATE)

# 재실행 시도
mvn flyway:validate

# 오류: Checksum mismatch for migration version 2
```

**원인:**
- 파일 내용이 변경되면 checksum이 달라짐
- Flyway는 이미 실행된 마이그레이션 파일을 수정하는 것을 방지

**대응책:**

**1️⃣ 파일을 원래대로 복원 (권장)**
```bash
git checkout db/migration/V2__Add_email.sql
```

**2️⃣ 새로운 마이그레이션으로 변경 (수정 필요 시)**
```bash
# 이전 V2 유지, V3으로 보정 마이그레이션 생성
cat > db/migration/V3__20240415_143000__Fix_email_column.sql << 'EOF'
ALTER TABLE users MODIFY email VARCHAR(500);
EOF
```

**3️⃣ 재설정 (DB 초기화, 개발 환경만)**
```bash
mvn flyway:clean  # 모든 테이블 삭제!
mvn flyway:migrate  # 처음부터 다시 실행
```

**교훈:** 이미 적용된 마이그레이션은 **절대 수정하면 안 됨**!

</details>

---

<div align="center">

**[⬅️ 이전: Chapter 4 — 백업과 마이그레이션](../rollback-recovery/05-backup-before-migration.md)** | **[홈으로 🏠](../README.md)** | **[다음: 브랜치 전략과 마이그레이션 ➡️](./02-branch-strategy.md)**

</div>
