# 마이그레이션 감시(Audit)와 컴플라이언스

---

## 🎯 핵심 질문

- `flyway_schema_history` 테이블의 각 필드가 정확히 무엇을 의미할까?
- 마이그레이션 감사 이력을 어떻게 활용하여 "언제 누가 어떤 스키마 변경을 했는가"를 추적할까?
- 환경별(dev/staging/prod)로 서로 다른 DB 사용자를 사용하면 누가 마이그레이션을 실행했는지 추적할 수 있을까?
- Git 이력과 flyway_schema_history를 조합하여 완전한 감시 추적을 만들려면?
- PCI-DSS, SOC2 같은 규제 요구사항을 마이그레이션 감시로 어떻게 충족할까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이그레이션 감사는 단순한 기술적 관심사가 아니라 **법적 필수 요구사항**입니다:

1. **규제 준수(Compliance)**:
   - PCI-DSS: 카드 정보 처리 시스템의 DB 변경 이력 기록 필수
   - SOC2: 정보 보안 감사 로그 요구
   - HIPAA: 의료 데이터 DB 변경 추적
   - GDPR: 개인 정보 처리 변경 이력

2. **운영 추적성(Traceability)**:
   - "3개월 전에 무엇을 배포했나?" → SQL 쿼리로 즉시 확인
   - "이 버그가 어느 마이그레이션에서 생겼나?" → 감시 이력으로 추적
   - "누가 이 칼럼을 삭제했나?" → installed_by 필드로 담당자 확인

3. **보안 감시**:
   - 비정상적인 스키마 변경 감지 (예: 야간 DELETE 작업)
   - 무단 마이그레이션 시도 감지
   - 권한 없는 사용자의 DB 접근 기록

4. **장애 대응**:
   - "이 버그는 V5 마이그레이션 이후 발생" → 원인 범위 좁혀짐
   - 특정 마이그레이션의 영향받은 데이터 파악
   - 빠른 롤백 의사결정

따라서 **마이그레이션 감사는 기술 + 법무 + 보안이 함께 요구하는 필수 사항**입니다.

---

## 😱 흔한 실수 (Before — ...)

### 실수 1: "flyway_schema_history를 장기 보존하지 않음"

```sql
-- 어느 날 DBA가 깨끗이 정리한다고 생각하며:
DELETE FROM flyway_schema_history;
-- 또는
TRUNCATE TABLE flyway_schema_history;
```

**결과**:
- 마이그레이션 이력 완전 소실
- "언제 스키마가 변경되었나?" 확인 불가
- 규제 감사에서 "이력이 없다" = "컴플라이언스 위반"
- 벌금 (GDPR: 최대 회사 연 매출 4%)

---

### 실시 2: "installed_by를 앱 계정으로 기록하여 실행자 구분 불가"

```yaml
# 모든 환경(dev, staging, prod)에서 동일한 DB 사용자 사용
spring:
  datasource:
    username: app_user  # 모든 마이그레이션이 "app_user"로 기록됨
    password: secret
```

**결과**:
```sql
SELECT installed_by, installed_on, description
FROM flyway_schema_history
ORDER BY installed_on DESC;

-- 모두 동일:
-- installed_by: app_user (누구인지, 어떤 환경인지 불명)
-- installed_on: 2024-01-15 10:00:00
```

**문제**: 프로덕션 버그가 발생해도 "누가 배포했는가"를 알 수 없음.

---

### 실수 3: "Git과 flyway_schema_history의 연관성을 놓침"

```
Git에는:
- V5__add_status.sql 커밋 기록 있음 (누가 커밋했나, 언제 커밋했나)
- flyway_schema_history에는:
  - V5 적용 기록 있음 (언제 프로덕션에 배포했나)
  
두 정보를 연결하지 않으면:
- "V5를 커밋한 사람" ≠ "V5를 배포한 사람"
- 책임 추적 불가
```

---

### 실수 4: "마이그레이션 파일을 삭제하거나 변경했는데 git log로 추적 안 함"

```bash
# 개발자가 실수로 파일 삭제
rm db/migrations/V3__add_status.sql
git add -A
git commit -m "Clean up"

# 나중에 프로덕션에서 문제 발생
# "V3가 적용되어 있는데 로컬에 없다"

# flyway_schema_history에는 기록 있음
# Git에는 기록 없음 (삭제됨)

# 추적 불가능한 상태!
```

---

## ✨ 올바른 접근 (After — ...)

### 올바른 패턴 1: 환경별 다른 DB 사용자로 installed_by 구분

```yaml
# dev 환경
spring:
  datasource:
    username: dev_migration_user   # Dev 마이그레이션 사용자
    password: ${DEV_DB_PASSWORD}

---
# staging 환경
spring:
  datasource:
    username: staging_migration_user   # Staging 마이그레이션 사용자
    password: ${STAGING_DB_PASSWORD}

---
# prod 환경
spring:
  datasource:
    username: prod_migration_user   # Prod 마이그레이션 사용자
    password: ${PROD_DB_PASSWORD}
```

**결과**:
```sql
SELECT installed_by, installed_on, version, description
FROM flyway_schema_history
ORDER BY installed_on DESC;

-- installed_by로 환경 구분 가능:
-- prod_migration_user  | 2024-01-15 10:00:00 | V5 | add_status
-- staging_migration_user | 2024-01-14 20:00:00 | V5 | add_status
-- dev_migration_user | 2024-01-14 15:00:00 | V5 | add_status
```

**추가 개선**: 실제 배포 담당자 이름 추적

```sql
-- Flyway 설정에서 플레이스홀더 사용
-- flyway.placeholders.deployed_by=alice@company.com

-- 마이그레이션 파일:
-- V5__add_status.sql
-- COMMENT ON TABLE users IS 'Deployed by ${deployed_by}';
```

---

### 올바른 패턴 2: flyway_schema_history 보존 정책

```sql
-- 1. 권한 보호: 아무도 flyway_schema_history를 삭제할 수 없도록
-- CREATE USER 'readonly_user'@'localhost' IDENTIFIED BY 'password';
-- GRANT SELECT, INSERT ON flyway_schema_history TO 'readonly_user'@'localhost';
-- (DELETE, UPDATE, TRUNCATE 권한 없음)

---

-- 2. 자동 백업 정책
-- 매주 flyway_schema_history를 별도 테이블에 복사
CREATE TABLE flyway_schema_history_archive AS
  SELECT * FROM flyway_schema_history WHERE installed_on < DATE_SUB(NOW(), INTERVAL 3 MONTHS);

---

-- 3. 보존 기간
-- 5년 이상 보존 권장 (규제 요구사항)
SELECT installed_on, COUNT(*) FROM flyway_schema_history
WHERE installed_on >= DATE_SUB(NOW(), INTERVAL 5 YEAR)
GROUP BY YEAR(installed_on), MONTH(installed_on);
```

---

### 올바른 패턴 3: 감사 쿼리 모음

```sql
-- 쿼리 1: 지난 3개월 마이그레이션 요약
SELECT
  DATE(installed_on) as migration_date,
  COUNT(*) as migration_count,
  GROUP_CONCAT(version ORDER BY version) as versions,
  GROUP_CONCAT(DISTINCT installed_by) as deployed_by
FROM flyway_schema_history
WHERE installed_on >= DATE_SUB(NOW(), INTERVAL 3 MONTH)
GROUP BY DATE(installed_on)
ORDER BY migration_date DESC;

-- 결과:
-- migration_date | count | versions | deployed_by
-- 2024-01-15     | 2     | V5, V6   | prod_migration_user
-- 2024-01-08     | 1     | V4       | staging_migration_user

---

-- 쿼리 2: 특정 테이블의 변경 이력
SELECT
  version,
  description,
  installed_on,
  installed_by,
  execution_time,
  success
FROM flyway_schema_history
WHERE description LIKE '%users%'  -- users 테이블 관련 마이그레이션
ORDER BY installed_on DESC;

-- 결과:
-- V2 | alter_users_add_status | 2024-01-15 10:02:00 | prod_migration_user | 1500 | true
-- V1 | create_users | 2024-01-10 09:00:00 | prod_migration_user | 500 | true

---

-- 쿼리 3: 마이그레이션 실행 시간이 오래 걸린 항목 (성능 이슈 감지)
SELECT
  version,
  description,
  execution_time,
  installed_on,
  ROUND(execution_time / 1000.0, 2) as duration_seconds
FROM flyway_schema_history
WHERE execution_time > 5000  -- 5초 이상
ORDER BY execution_time DESC;

-- 결과:
-- V8 | load_bulk_data | 45000 | 2024-01-15 10:05:00 | 45.00초

---

-- 쿼리 4: 마이그레이션 실패 이력
SELECT
  version,
  description,
  installed_on,
  installed_by,
  success
FROM flyway_schema_history
WHERE success = false
ORDER BY installed_on DESC;

-- (성공한 마이그레이션만 기록되므로 보통 비어있음)
-- Flyway는 실패하면 schema_history에 기록하지 않음

---

-- 쿼리 5: 환경별 마이그레이션 진행 상황
SELECT
  installed_by as environment,
  COUNT(*) as total_migrations,
  MAX(installed_on) as last_migration,
  MAX(version) as latest_version
FROM flyway_schema_history
GROUP BY installed_by
ORDER BY MAX(installed_on) DESC;

-- 결과:
-- prod_migration_user | 8 | 2024-01-15 10:02:00 | V8
-- staging_migration_user | 8 | 2024-01-15 09:00:00 | V8
-- dev_migration_user | 10 | 2024-01-15 15:00:00 | V10 (dev는 더 진행됨)
```

---

### 올바른 패턴 4: Git + Flyway 통합 감사

```bash
#!/bin/bash
# audit-migration.sh
# Git 커밋 이력과 Flyway 배포 이력을 조합하여 완전한 감사 보고서 생성

set -e

PROD_DB_URL="jdbc:mysql://prod-db:3306/mydb"
PROD_DB_USER="audit_user"
PROD_DB_PASS="password"

echo "=== 마이그레이션 감시 보고서 ==="
echo "생성일: $(date)"
echo ""

# Step 1: Flyway에서 최근 3개월 마이그레이션 조회
echo "## 프로덕션 배포 이력"
echo ""

mysql -h prod-db -u $PROD_DB_USER -p$PROD_DB_PASS mydb << 'EOF' > /tmp/prod-migrations.txt
SELECT
  DATE_FORMAT(installed_on, '%Y-%m-%d %H:%i:%s') as deployment_date,
  version,
  description,
  installed_by,
  ROUND(execution_time / 1000.0, 2) as duration_seconds
FROM flyway_schema_history
WHERE installed_on >= DATE_SUB(NOW(), INTERVAL 3 MONTH)
ORDER BY installed_on DESC;
EOF

cat /tmp/prod-migrations.txt

# Step 2: Git에서 해당 기간의 커밋 조회
echo ""
echo "## Git 커밋 이력"
echo ""

git log --since="3 months ago" \
  --pretty=format:"%ad | %an | %s" \
  --date=short \
  -- db/migrations/*.sql | \
  sort -r > /tmp/git-commits.txt

cat /tmp/git-commits.txt

# Step 3: Git과 Flyway 매핑
echo ""
echo "## Git-Flyway 매핑 (누가 작성, 누가 배포)"
echo ""

echo "| Git 커밋 날짜 | 작성자 | 마이그레이션 파일 | Flyway 배포 날짜 | 배포 담당 환경 |"
echo "|---|---|---|---|---|"

# git log에서 V*.sql 패턴 추출 및 Flyway 기록과 매칭
git log --since="3 months ago" \
  --pretty=format:"%ad | %an | %s" \
  --date=short \
  -- db/migrations/*.sql | \
while IFS='|' read git_date author subject; do
  # subject에서 파일명 추출 (예: "V5__add_status.sql")
  if [[ $subject =~ V[0-9]+__ ]]; then
    version=$(echo $subject | grep -oE 'V[0-9]+' | head -1)
    
    # Flyway 기록에서 해당 버전 찾기
    deployment_info=$(grep "^$version " /tmp/prod-migrations.txt | head -1 || echo "미배포")
    
    echo "| $git_date | $author | $version | $deployment_info |"
  fi
done

echo ""
echo "✓ 감시 보고서 생성 완료"
```

---

### 올바른 패턴 5: PCI-DSS 및 SOC2 컴플라이언스

```sql
-- PCI-DSS 3.0 요구사항: 12.3.1 - 모든 물리적, 로직적 DB 접근 변경 이력 기록

-- 1. 감사 테이블 생성 (flyway_schema_history와 별도)
CREATE TABLE audit_log (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  table_name VARCHAR(255),
  operation VARCHAR(50),  -- CREATE, ALTER, DROP, INSERT, UPDATE, DELETE
  operation_by VARCHAR(100),
  operation_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  description LONGTEXT,
  affected_rows INT,
  duration_ms INT,
  INDEX idx_table_name (table_name),
  INDEX idx_operation_at (operation_at)
);

-- 2. 마이그레이션 파일에서 감사 로그 기록
-- V5__add_status.sql
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'ACTIVE';

INSERT INTO audit_log (
  table_name, operation, operation_by, description, affected_rows, duration_ms
) VALUES (
  'users', 'ALTER', CURRENT_USER(), 'Add status column', NULL, NULL
);

-- 3. 정기 감사 리포트 (월 1회)
SELECT
  DATE_TRUNC(operation_at, MONTH) as audit_month,
  operation,
  COUNT(*) as count,
  GROUP_CONCAT(DISTINCT operation_by) as operators
FROM audit_log
GROUP BY audit_month, operation
ORDER BY audit_month DESC;

-- 4. 비정상 접근 감지
-- (예: 야간 DELETE 작업)
SELECT
  operation_at,
  operation,
  operation_by,
  table_name,
  description,
  affected_rows
FROM audit_log
WHERE HOUR(operation_at) BETWEEN 20 AND 6  -- 20:00 ~ 06:00
  AND operation IN ('DELETE', 'TRUNCATE', 'UPDATE')
ORDER BY operation_at DESC;
```

---

## 🔬 내부 동작 원리

### 1. Flyway_schema_history의 필드 의미

```sql
CREATE TABLE flyway_schema_history (
  -- 순서 번호 (마이그레이션 실행 순서)
  installed_rank INT PRIMARY KEY AUTO_INCREMENT,
  
  -- 마이그레이션 버전 (파일명에서 추출: V001, V002, ...)
  version VARCHAR(50),
  
  -- 마이그레이션 설명 (파일명의 __뒤 부분)
  description VARCHAR(255),
  
  -- 마이그레이션 타입 (SQL, JDBC, Undo)
  type VARCHAR(20),
  
  -- 실행된 파일명
  script VARCHAR(1000),
  
  -- 파일 내용의 MD5 해시 (변경 감지용)
  checksum INT,
  
  -- 마이그레이션을 실행한 DB 사용자명
  installed_by VARCHAR(100),
  
  -- 마이그레이션 실행 시각 (매우 중요한 감사 정보!)
  installed_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  -- 실행 소요 시간 (밀리초)
  execution_time INT,
  
  -- 성공 여부 (true만 기록됨, false는 기록 안 됨)
  success BOOLEAN
);
```

**각 필드의 감사 가치**:
- `version + description`: 무엇이 변경되었나
- `installed_by`: 누가 변경했나 (환경별 사용자 사용 시)
- `installed_on`: 언제 변경했나
- `execution_time`: 얼마나 걸렸나 (성능 모니터링)

---

### 2. installed_by 자동 채우기 메커니즘

```
Flyway 마이그레이션 실행:
  ↓
1. DB 연결
   datasource.username = "migration_user"
  ↓
2. 마이그레이션 실행 (SQL)
   EXECUTE migration_v5.sql;
  ↓
3. flyway_schema_history에 기록
   INSERT INTO flyway_schema_history (
     version='V5',
     installed_by=CURRENT_USER(),  ← DB 연결 사용자
     installed_on=NOW()
   )
  ↓
4. 결과: installed_by = "migration_user"
```

**따라서 감사를 위해서는 환경별로 다른 DB 사용자를 사용해야 합니다**:
- dev_migration_user: dev 환경
- staging_migration_user: staging 환경
- prod_migration_user: prod 환경

---

### 3. Git과 Flyway 통합 추적

```
Git 리포지토리:
  ↓
V5__add_status.sql 커밋
  Commit: abc1234
  Author: alice@company.com
  Date: 2024-01-15 10:00:00
  Message: "Add user status field"
  ↓
Git에 기록됨:
  - 파일 내용 (어떤 SQL이 실행될지)
  - 작성자 (alice)
  - 커밋 시각 (10:00:00)

---

프로덕션 배포:
  ↓
마이그레이션 실행 (V5__add_status.sql)
  ↓
Flyway_schema_history에 기록:
  - version: V5
  - installed_by: prod_migration_user
  - installed_on: 2024-01-15 10:02:00
  - execution_time: 1500ms

---

연결:
  Git: V5 파일을 누가(alice) 언제(10:00:00) 작성했나
  Flyway: V5를 어느 환경(prod_migration_user) 에서 언제(10:02:00) 배포했나
  
  → 완전한 감사 추적:
    alice가 작성한 V5를 프로덕션에 배포했다 (2분 후)
```

---

## 💻 실전 실험

### 실험 1: 마이그레이션 감사 쿼리 실행

```bash
#!/bin/bash
# test-audit-queries.sh

set -e

DB_HOST=localhost
DB_USER=root
DB_PASS=root
TEST_DB=audit_test

# Step 1: 테스트 DB 설정
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS << EOF
DROP DATABASE IF EXISTS $TEST_DB;
CREATE DATABASE $TEST_DB;
EOF

# Step 2: Flyway로 마이그레이션 적용 (여러 환경)
for env in dev staging prod; do
  echo "Migrating $env environment..."
  
  mkdir -p /tmp/migrations-$env
  
  cat > /tmp/migrations-$env/V1__initial.sql << 'EOF'
CREATE TABLE users (id BIGINT PRIMARY KEY);
EOF
  
  docker run --rm \
    -v /tmp/migrations-$env:/migrations \
    -e FLYWAY_URL=jdbc:mysql://$DB_HOST:3306/$TEST_DB \
    -e FLYWAY_USER=${env}_migration_user \
    -e FLYWAY_PASSWORD=secret \
    -e MYSQL_USER=${env}_migration_user \
    -e MYSQL_PASSWORD=secret \
    flyway/flyway:9 migrate || true  # 사용자 없어도 진행
done

# Step 3: Manual setup (데모용)
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $TEST_DB << 'EOF'
-- 마이그레이션 이력 수동 생성 (Flyway 결과 시뮬레이션)
CREATE TABLE IF NOT EXISTS flyway_schema_history (
  installed_rank INT PRIMARY KEY AUTO_INCREMENT,
  version VARCHAR(50),
  description VARCHAR(255),
  type VARCHAR(20),
  script VARCHAR(1000),
  checksum INT,
  installed_by VARCHAR(100),
  installed_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  execution_time INT,
  success BOOLEAN
);

INSERT INTO flyway_schema_history (version, description, installed_by, execution_time, success)
VALUES
('V1', 'initial', 'dev_migration_user', 500, 1),
('V1', 'initial', 'staging_migration_user', 600, 1),
('V1', 'initial', 'prod_migration_user', 550, 1),
('V2', 'add_users', 'dev_migration_user', 1000, 1),
('V2', 'add_users', 'staging_migration_user', 1100, 1),
('V2', 'add_users', 'prod_migration_user', 950, 1),
('V3', 'add_status', 'dev_migration_user', 2000, 1),
('V3', 'add_status', 'staging_migration_user', 2200, 1);
-- V3은 아직 prod에 배포 안 됨
EOF

# Step 4: 감사 쿼리 실행
echo ""
echo "=== 환경별 마이그레이션 진행 상황 ==="
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $TEST_DB << 'EOF'
SELECT
  installed_by as environment,
  COUNT(*) as total_migrations,
  MAX(version) as latest_version,
  MAX(installed_on) as last_deployment
FROM flyway_schema_history
GROUP BY installed_by
ORDER BY MAX(installed_on) DESC;
EOF

echo ""
echo "=== 마이그레이션 성능 (소요 시간) ==="
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $TEST_DB << 'EOF'
SELECT
  version,
  description,
  AVG(execution_time) as avg_ms,
  MIN(execution_time) as min_ms,
  MAX(execution_time) as max_ms
FROM flyway_schema_history
GROUP BY version
ORDER BY version;
EOF

echo ""
echo "=== 미배포 마이그레이션 (Dev는 적용, Prod는 미적용) ==="
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $TEST_DB << 'EOF'
SELECT DISTINCT version
FROM flyway_schema_history
WHERE installed_by = 'dev_migration_user'
  AND version NOT IN (
    SELECT version FROM flyway_schema_history
    WHERE installed_by = 'prod_migration_user'
  );
EOF

echo ""
echo "✓ 감사 쿼리 실행 완료"
```

---

### 실험 2: Git과 Flyway 통합 감사 보고서

```bash
#!/bin/bash
# git-flyway-audit.sh

set -e

cd /path/to/myapp-repo

echo "=== Git + Flyway 통합 감시 보고서 ==="
echo "생성일: $(date)"
echo ""

# Git 커밋 이력 추출
git log --since="1 month ago" \
  --format='%ad|%an|%s' \
  --date=short \
  -- db/migrations/*.sql > /tmp/git-log.txt

echo "## Git 커밋 이력 (지난 1개월)"
echo "| 날짜 | 작성자 | 내용 |"
echo "|------|--------|------|"

while IFS='|' read date author subject; do
  echo "| $date | $author | $subject |"
done < /tmp/git-log.txt

echo ""
echo "## Flyway 배포 이력 (지난 1개월)"
echo "| 배포 일시 | 버전 | 설명 | 배포 담당 | 소요시간 |"
echo "|----------|------|------|----------|---------|"

mysql -h prod-db -u audit_user -ppassword mydb << 'EOF' > /tmp/flyway-log.txt
SELECT
  DATE_FORMAT(installed_on, '%Y-%m-%d %H:%i'),
  version,
  description,
  installed_by,
  CONCAT(ROUND(execution_time/1000.0, 2), 's')
FROM flyway_schema_history
WHERE installed_on >= DATE_SUB(NOW(), INTERVAL 1 MONTH)
ORDER BY installed_on DESC;
EOF

# 포맷팅
awk -F'\t' '{print "| " $1 " | " $2 " | " $3 " | " $4 " | " $5 " |"}' /tmp/flyway-log.txt

echo ""
echo "## 감시 요약"
total_commits=$(wc -l < /tmp/git-log.txt)
echo "- Git 커밋: $total_commits개"
echo "- Flyway 배포: $(cat /tmp/flyway-log.txt | wc -l)개"
echo "- 배포 대기 중: $(( total_commits - $(cat /tmp/flyway-log.txt | wc -l) ))개"
```

---

### 실험 3: 비정상 감지 (야간 DELETE 작업)

```sql
-- anomaly-detection.sql

-- 위험한 패턴: 야간 대량 DELETE
SELECT
  DATE_FORMAT(installed_on, '%H:%i:%s') as time_of_day,
  version,
  description,
  installed_by
FROM flyway_schema_history
WHERE HOUR(installed_on) BETWEEN 20 AND 6  -- 20:00 ~ 06:00 (야간)
  AND (
    description LIKE '%delete%' OR
    description LIKE '%drop%' OR
    description LIKE '%truncate%' OR
    description LIKE '%clean%'
  )
ORDER BY installed_on DESC;

-- 비정상: 프로덕션 사용자가 아닌 다른 사용자의 마이그레이션
SELECT
  installed_on,
  version,
  installed_by,
  description
FROM flyway_schema_history
WHERE installed_by NOT IN ('dev_migration_user', 'staging_migration_user', 'prod_migration_user')
ORDER BY installed_on DESC;

-- 비정상: 아주 긴 실행 시간 (성능 이슈 또는 락 문제)
SELECT
  version,
  description,
  execution_time,
  ROUND(execution_time / 1000.0, 2) as duration_seconds,
  installed_on
FROM flyway_schema_history
WHERE execution_time > 60000  -- 60초 이상
ORDER BY execution_time DESC;
```

---

## 📊 성능/비용 비교

| 항목 | 감시 수준 | 보존 기간 | 저장 비용 | 규제 준수 | 추천 상황 |
|------|---------|---------|---------|---------|----------|
| 기본 | flyway_schema_history만 | 1년 | 거의 0 | 최소 | 초기 스타트업 |
| 중간 | + audit_log 테이블 | 3년 | 낮음 | SOC2 | 성장기 스타트업 |
| 높음 | + 별도 감사 DB | 5년+ | 중간 | PCI-DSS, HIPAA | 엔터프라이즈 |

---

## ⚖️ 트레이드오프

### 최소 감시 (flyway_schema_history만)
**장점**: 별도 설정 불필요, 자동으로 기록
**단점**: 기본 정보만 기록, 규제 요구사항 부족

### 상세 감시 (Git + Flyway + Audit Log)
**장점**: 완전한 추적, 규제 준수, 보안 감시
**단점**: 복잡도 증가, 저장소 비용, 쿼리 성능 고려 필요

---

## 📌 핵심 정리

1. **flyway_schema_history는 자동 감시 저장소**
   - 모든 마이그레이션이 자동으로 기록됨
   - 절대 삭제하면 안 됨

2. **installed_by로 환경과 담당자 추적**
   - 환경별 다른 DB 사용자 사용
   - 누가 어느 환경에서 배포했는지 추적 가능

3. **Git + Flyway의 이원 추적이 가장 강력**
   - Git: 누가 작성했나, 언제 작성했나, 코드 리뷰
   - Flyway: 언제 배포했나, 얼마나 걸렸나, 성공했나

4. **규제 준수는 마이그레이션 감시로 시작**
   - PCI-DSS, SOC2, HIPAA 등은 감사 이력 필수
   - 마이그레이션 감시 = 법적 필수 요구사항

5. **정기 감사 보고서 자동화**
   - 월 1회 감사 쿼리 자동 실행
   - 비정상 패턴 자동 감지
   - 경영진 보고서 생성

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: flyway_schema_history를 실수로 DELETE했다면 복구할 수 있을까?</strong></summary>

**상황**:
```sql
DELETE FROM flyway_schema_history;
COMMIT;
```

**결과**:
- 마이그레이션 이력 완전 소실
- Flyway는 이 테이블이 비어있다고 판단
- 다음 마이그레이션 실행 시 "V1이 미적용된 것처럼" 판단 → 중복 실행 위험!

**복구 방법 1: 백업에서 복구**
```sql
-- 자동 백업 (1시간 전)이 있다면
RESTORE FROM backup_2024-01-15_10-00-00;
```

**복구 방법 2: Git 히스토리로 재구성**
```bash
# Git에서 모든 마이그레이션 파일 조회
git log --name-only --pretty=format: -- db/migrations/*.sql | \
sort -u | grep '\.sql$'

# 각 파일에 대해 수동으로 flyway_schema_history 복구
# 문제: execution_time, checksum 등은 추론만 가능
```

**예방 방법 1: 권한 제한**
```sql
-- 일반 사용자는 DELETE 불가
GRANT SELECT, INSERT ON flyway_schema_history TO 'app_user'@'localhost';
-- DELETE, UPDATE, TRUNCATE 권한 없음

-- 오직 DBA만 SELECT
GRANT SELECT ON flyway_schema_history TO 'dba_user'@'localhost';
```

**예방 방법 2: 자동 백업**
```sql
-- 매일 자동 백업
CREATE EVENT backup_schema_history
ON SCHEDULE EVERY 1 DAY
DO BEGIN
  CREATE TABLE IF NOT EXISTS flyway_schema_history_backup_$(DATE_FORMAT(NOW(), '%Y%m%d'))
  AS SELECT * FROM flyway_schema_history;
END;
```

**교훈**: **flyway_schema_history는 감시 데이터이므로 DELETE 권한을 매우 제한**해야 합니다.
</details>

<details>
<summary><strong>Q2: 프로덕션에서는 "prod_migration_user"라는 사용자 계정을 사용하는데, 실제로 누가 배포를 실행했는지 알 수 없지 않을까?</strong></summary>

**문제**:
```sql
SELECT installed_by, description, installed_on
FROM flyway_schema_history
WHERE installed_on >= DATE_SUB(NOW(), INTERVAL 1 DAY);

-- 결과:
-- prod_migration_user | V5 | 2024-01-15 10:00:00

-- "prod_migration_user가 배포했다"는 알지만,
-- "누가 이 배포를 지시했나"는 모름
```

**해결 방법 1: 배포 이력 별도 기록**
```sql
-- 배포 로그 테이블
CREATE TABLE deployment_log (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  deployment_by VARCHAR(100),  -- 실제 사람 이름
  deployment_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  migration_version VARCHAR(50),
  environment VARCHAR(50),
  status VARCHAR(50)  -- SUCCESS, FAILED
);

-- 마이그레이션 전 기록
INSERT INTO deployment_log (deployment_by, migration_version, environment, status)
VALUES ('alice@company.com', 'V5', 'production', 'IN_PROGRESS');

-- 마이그레이션 완료 후 업데이트
UPDATE deployment_log SET status = 'SUCCESS'
WHERE deployment_by = 'alice@company.com' AND migration_version = 'V5';
```

**해결 방법 2: GitHub Actions 통합**
```yaml
# GitHub Actions Workflow
- name: Deploy Migration
  env:
    DEPLOYER: ${{ github.actor }}  # GitHub 사용자명
  run: |
    echo "Deploying as: $DEPLOYER"
    # 마이그레이션 실행 전 로그 기록
    mysql ... -e "INSERT INTO deployment_log (deployment_by, migration_version) VALUES ('$DEPLOYER', 'V5')"
    
    # 마이그레이션 실행
    flyway migrate
```

**해결 방법 3: Kubernetes RBAC 통합**
```yaml
# Kubernetes Job
- name: Record deployer
  env:
    DEPLOYER: ${{ github.actor }}
  run: |
    kubectl annotate job db-migration deployer=$DEPLOYER
```

**교훈**: **DB 사용자와 실제 배포 담당자를 분리하여 기록**해야 합니다.
</details>

<details>
<summary><strong>Q3: 마이그레이션 실패 시 flyway_schema_history에 기록되지 않는데, 이를 어떻게 감시할까?</strong></summary>

**문제**:
```sql
-- 마이그레이션 V5 실패 (예: 문법 오류)
-- flyway_schema_history에 기록 안 됨
-- (성공한 것만 기록)

SELECT * FROM flyway_schema_history WHERE version = 'V5';
-- (결과 없음)
```

**영향**: 실패 이력이 DB에 남지 않아 감시 불가능

**해결 방법 1: 별도 실패 로그 테이블**
```sql
CREATE TABLE migration_failure_log (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  version VARCHAR(50),
  description VARCHAR(255),
  attempted_by VARCHAR(100),
  attempted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  error_message LONGTEXT,
  error_line INT,
  status VARCHAR(50)
);

-- Flyway를 래핑한 스크립트에서 실패 캐치
# wrapper-script.sh
flyway migrate || {
  error_msg=$(flyway migrate 2>&1)
  mysql -u $USER -p$PASS mydb -e \
    "INSERT INTO migration_failure_log (version, attempted_by, error_message) VALUES ('$VERSION', '$USER', '$error_msg')"
  exit 1
}
```

**해결 방법 2: GitHub Actions 통합**
```yaml
- name: Run Migration
  continue-on-error: true
  id: migration
  run: flyway migrate

- name: Log Failure
  if: failure()
  run: |
    mysql ... -e "INSERT INTO migration_failure_log (version, attempted_by, error_message) VALUES ('V5', '$GITHUB_ACTOR', '${{ failure() }}')"
```

**해결 방법 3: Kubernetes 이벤트 기록**
```yaml
# Job 실패 시 Kubernetes Event 생성
apiVersion: batch/v1
kind: Job
spec:
  template:
    spec:
      containers:
      - name: migration
        command:
          - sh
          - -c
          - |
            flyway migrate || {
              # Kubernetes Event 생성
              kubectl annotate job db-migration \
                failure-reason="$(flyway migrate 2>&1)"
              exit 1
            }
```

**교훈**: **마이그레이션 실패도 명시적으로 기록**해야 완전한 감시가 가능합니다.
</details>

---

<div align="center">

**[⬅️ 이전: Kubernetes 배포와 마이그레이션](./04-kubernetes-migration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — Spring Boot + Flyway 자동 설정 ➡️](../spring-integration/01-spring-boot-flyway-autoconfigure.md)**

</div>
