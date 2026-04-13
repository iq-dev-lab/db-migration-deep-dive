# 마이그레이션 검증 자동화

---

## 🎯 핵심 질문

- `flyway validate` 명령어는 정확히 무엇을 검증하고, 언제 사용해야 할까?
- Dry Run으로 실제 실행 없이 마이그레이션을 미리보기할 수 있을까?
- 마이그레이션 파일이 변경되었는지 자동으로 감지할 수 있을까?
- 큰 테이블을 변경하는 위험한 마이그레이션을 사전에 감지하려면?
- 스테이징 환경에서 안전하게 테스트한 후 프로덕션에 적용하려면?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이그레이션 검증은 **배포 전 문제를 조기에 발견하는 마지막 방어선**입니다:

1. **Checksum 검증**: 이전에 적용된 마이그레이션이 변경되었는지 감지 (데이터 불일치 방지)
2. **구문 검증**: SQL 문법 오류 조기 발견 (DB 실행 전)
3. **순서 검증**: Out-of-order 마이그레이션 감지 (예측 불가능한 스키마 방지)
4. **위험 DDL 감지**: 대용량 테이블 변경, DROP 명령어 등 사전 경고
5. **성능 영향 분석**: 마이그레이션 실행 시 예상 소요 시간, 테이블 잠금 시간 추정

특히 **프로덕션 환경에서는 검증 없이 마이그레이션을 실행하면 안 됩니다**. 데이터 손실이나 서비스 장애로 이어질 수 있습니다.

---

## 😱 흔한 실수 (Before — ...)

### 실수 1: "검증 없이 바로 마이그레이션 실행"

```yaml
# GitHub Actions Workflow (잘못된 예)
deploy-prod:
  steps:
    - name: Run Migration
      run: |
        flyway \
          -url=$PROD_DB_URL \
          migrate  # validate 생략!
```

**결과**:
- 마이그레이션 파일에 문법 오류가 있어도 미리 감지 불가
- 파일이 변경된 줄 모르고 적용
- 데이터 부분 손상 → 서비스 장애
- 롤백 불가능 (이미 DB에 반영됨)

---

### 실수 2: "이전 마이그레이션이 수정되었는데 감지 못함"

```sql
-- V1__initial.sql (처음 배포, 이미 프로덕션 적용됨)
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  email VARCHAR(255)
);

-- 2주 후, 개발자가 실수로 파일 수정
-- (email 길이 변경)
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  email VARCHAR(512)  # 255 -> 512 (수정됨)
);

-- Flyway validate 없이 재배포
# 결과: 일부 서버는 구 스키마, 일부는 신 스키마 → 데이터 불일치
```

---

### 실수 3: "위험한 DDL이 포함되었는데 미리 알지 못함"

```sql
-- V5__cleanup.sql
ALTER TABLE users DROP COLUMN temp_status;  # 대용량 테이블에서 DROP
```

**문제**:
- MySQL에서 DROP COLUMN은 전체 테이블을 다시 쓰므로 많은 시간 소요
- 프로덕션 장시간 Lock → 서비스 중단
- 미리 몰랐으므로 시간 계획 불가
- 온라인 마이그레이션 도구(pt-online-schema-change) 사용 검토 불가

---

### 실수 4: "Out-of-order 마이그레이션이 섞여서 실행 순서 엉망"

```
실행 순서:
V1__create_users.sql (적용됨)
V2__add_posts.sql (적용됨)
V4__add_status.sql (실수로 먼저 배포됨!)
V3__add_comments.sql (나중에 배포)

결과:
V4는 V3에서 생성되는 테이블을 참조 → 외래키 오류!
```

---

## ✨ 올바른 접근 (After — ...)

### 올바른 패턴 1: Validate로 사전 검증

```bash
# Step 1: Validate (실제 실행 없이 검증)
flyway \
  -url=jdbc:mysql://localhost:3306/mydb \
  -user=migration \
  -password=secret \
  -locations=filesystem:./db/migrations \
  validate

# 출력:
# Successfully validated 5 migrations (checksum not required)
# Schema has no migrations applied yet.

# 만약 파일이 변경되었다면:
# ERROR: Detected applied migration not resolved locally: 1__initial
# The applied migration 1__initial was found in the DB but not in the local file
```

**validate가 검증하는 항목**:
1. SQL 문법 (기본적인 검사)
2. Checksum (이전 파일과 비교)
3. 순서 (V001, V002, V003... 순차적 증가)
4. 스크립트 이름 형식 (V001__description.sql)

---

### 올바른 패턴 2: Staging에서 리허설, 프로덕션에서 실행

```bash
#!/bin/bash
# migrate-with-validation.sh

set -e  # 오류 발생 시 즉시 중단

ENV=$1  # staging 또는 production
DB_URL=$2
DB_USER=$3
DB_PASS=$4

echo "=== Step 1: Validate ==="
flyway \
  -url=$DB_URL \
  -user=$DB_USER \
  -password=$DB_PASS \
  -locations=filesystem:./db/migrations \
  validate

if [ $? -ne 0 ]; then
  echo "❌ 검증 실패, 마이그레이션 중단"
  exit 1
fi

echo ""
echo "=== Step 2: Dry Run (SQL 미리보기) ==="
# Flyway Teams에서만 지원
flyway \
  -url=$DB_URL \
  -user=$DB_USER \
  -password=$DB_PASS \
  -locations=filesystem:./db/migrations \
  -dryRunOutput=/tmp/migration-${ENV}.sql \
  migrate

echo "생성될 SQL:"
cat /tmp/migration-${ENV}.sql

echo ""
echo "=== Step 3: 실행 (프로덕션) ==="
if [ "$ENV" = "production" ]; then
  read -p "정말 프로덕션에 적용하시겠습니까? (yes/no): " confirm
  if [ "$confirm" != "yes" ]; then
    echo "중단되었습니다"
    exit 1
  fi
fi

flyway \
  -url=$DB_URL \
  -user=$DB_USER \
  -password=$DB_PASS \
  -locations=filesystem:./db/migrations \
  migrate

echo "✓ 마이그레이션 완료"
```

---

### 올바른 패턴 3: 위험한 DDL 자동 감지

```bash
#!/bin/bash
# detect-dangerous-ddl.sh

DANGER_PATTERNS=(
  "DROP TABLE"
  "DROP DATABASE"
  "DROP COLUMN"
  "TRUNCATE"
  "ALTER.*RENAME"
  "ALTER.*DROP"
  "DELETE FROM"
  "UPDATE.*WHERE 1=1"
)

MIGRATION_DIR="db/migrations"
HAS_DANGER=false

for pattern in "${DANGER_PATTERNS[@]}"; do
  files=$(grep -l "$pattern" $MIGRATION_DIR/*.sql 2>/dev/null || true)
  if [ -n "$files" ]; then
    echo "❌ 위험한 DDL 감지:"
    grep -n "$pattern" $files
    HAS_DANGER=true
  fi
done

if [ "$HAS_DANGER" = true ]; then
  echo ""
  echo "위험한 DDL이 포함된 마이그레이션입니다."
  echo "다음을 확인하세요:"
  echo "1. 정말 DELETE/DROP/TRUNCATE가 필요한가?"
  echo "2. 데이터 백업은 했는가?"
  echo "3. 영향받는 테이블의 크기는?"
  echo "4. 예상 소요 시간은?"
  echo ""
  exit 1
fi

echo "✓ 위험한 DDL 없음"
```

**마이그레이션 파일 예**:
```sql
-- V5__clean_audit_logs.sql

-- ❌ 위험: TRUNCATE (0.1초, 하지만 모든 데이터 손실)
-- TRUNCATE TABLE audit_logs;

-- ✓ 안전: DELETE + 보존 기간 (부분 삭제, 롤백 가능)
DELETE FROM audit_logs
WHERE created_at < DATE_SUB(NOW(), INTERVAL 1 YEAR);

-- 또는 아예 파티셔닝으로 처리
-- ALTER TABLE audit_logs DROP PARTITION p_2023;
```

---

### 올바른 패턴 4: PR에서 마이그레이션 변경 자동 경고

```yaml
# .github/workflows/migration-review.yml

name: Migration Change Review

on:
  pull_request:
    paths:
      - 'db/migrations/**'

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # 전체 히스토리 필요
      
      - name: Detect modified migrations
        run: |
          echo "=== 수정된 마이그레이션 파일 ==="
          git diff origin/main...HEAD --name-only -- 'db/migrations/*' | while read file; do
            # 이미 applied된 마이그레이션인지 확인
            if git log --oneline --all | grep -q "$(basename $file)"; then
              echo "⚠️  경고: 이미 적용된 마이그레이션이 수정되었습니다!"
              echo "파일: $file"
              git diff origin/main...HEAD -- "$file"
            fi
          done
      
      - name: Check for dangerous operations
        run: |
          echo "=== 위험한 DDL 검사 ==="
          git diff origin/main...HEAD -- 'db/migrations/*' | \
          grep -E "^\+.*DROP|^\+.*TRUNCATE|^\+.*RENAME" || echo "✓ 위험한 DDL 없음"
      
      - name: Estimate impact on large tables
        run: |
          echo "=== 영향받는 테이블 크기 ==="
          # 새로운 마이그레이션에서 ALTER하는 테이블명 추출
          grep -h "ALTER TABLE" db/migrations/*.sql | \
          sed 's/ALTER TABLE \([^ ]*\).*/\1/' | sort -u | \
          while read table; do
            echo "테이블: $table (DB에서 크기 확인 필요)"
          done
      
      - name: Post PR comment
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const output = fs.readFileSync('/tmp/migration-analysis.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 마이그레이션 검토\n\n${output}`
            });
```

---

### 올바른 패턴 5: 스테이징 → 프로덕션 파이프라인

```yaml
# GitHub Actions Workflow

name: Validate and Migrate

on:
  push:
    branches: [main]
    paths:
      - 'db/migrations/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    outputs:
      migration-files: ${{ steps.files.outputs.modified }}
    steps:
      - uses: actions/checkout@v3
      
      - name: Find migration files
        id: files
        run: |
          git diff HEAD~1 HEAD --name-only -- 'db/migrations/*.sql' > /tmp/files.txt
          echo "modified=$(cat /tmp/files.txt | tr '\n' ' ')" >> $GITHUB_OUTPUT
      
      - name: Setup Flyway
        run: |
          cd /tmp
          wget -q https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/9.17.0/flyway-commandline-9.17.0-linux-x64.tar.gz
          tar -xzf flyway-commandline-9.17.0-linux-x64.tar.gz
          echo "/tmp/flyway-9.17.0" >> $GITHUB_PATH
      
      - name: Validate syntax
        run: |
          echo "검증: ${{ steps.files.outputs.modified }}"
          for file in ${{ steps.files.outputs.modified }}; do
            echo "파일: $file"
            # 기본 문법 검사 (더 나은 도구: sqlfluff)
            grep -E "^(CREATE|ALTER|DROP|INSERT|UPDATE|DELETE|TRUNCATE)" "$file" || true
          done

  test-on-staging:
    name: Test on Staging
    needs: validate
    runs-on: ubuntu-latest
    environment:
      name: staging
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Flyway
        run: |
          cd /tmp
          wget -q https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/9.17.0/flyway-commandline-9.17.0-linux-x64.tar.gz
          tar -xzf flyway-commandline-9.17.0-linux-x64.tar.gz
          echo "/tmp/flyway-9.17.0" >> $GITHUB_PATH
      
      - name: Validate on Staging
        run: |
          flyway \
            -url="${{ secrets.STAGING_DB_URL }}" \
            -user="${{ secrets.STAGING_DB_USER }}" \
            -password="${{ secrets.STAGING_DB_PASSWORD }}" \
            -locations=filesystem:$(pwd)/db/migrations \
            validate
      
      - name: Run on Staging (Dry Run)
        run: |
          flyway \
            -url="${{ secrets.STAGING_DB_URL }}" \
            -user="${{ secrets.STAGING_DB_USER }}" \
            -password="${{ secrets.STAGING_DB_PASSWORD }}" \
            -locations=filesystem:$(pwd)/db/migrations \
            -dryRunOutput=/tmp/staging-dryrun.sql \
            migrate
        continue-on-error: true
      
      - name: Analyze migration impact
        run: |
          echo "=== 스테이징 마이그레이션 분석 ==="
          cat /tmp/staging-dryrun.sql
          
          # 영향받는 테이블 통계
          echo ""
          echo "=== 테이블 변경 ==="
          # 실제로는 MySQL 쿼리로 스테이징 DB에서 확인
          mysql -h staging-db -u user -ppass mydb -e "
            SELECT table_name, table_rows, data_length/1024/1024 as size_mb
            FROM information_schema.tables
            WHERE table_schema = 'mydb'
            AND table_name IN (
              SELECT DISTINCT
              REGEXP_SUBSTR(line, 'TABLE \`?([^ \`]+)\`?', 1, 1, NULL, 1) as tbl
              FROM /tmp/staging-dryrun.sql
            )
            ORDER BY table_rows DESC;
          "
      
      - name: Run actual migration on Staging
        run: |
          flyway \
            -url="${{ secrets.STAGING_DB_URL }}" \
            -user="${{ secrets.STAGING_DB_USER }}" \
            -password="${{ secrets.STAGING_DB_PASSWORD }}" \
            -locations=filesystem:$(pwd)/db/migrations \
            migrate
      
      - name: Test application on Staging
        run: |
          # Staging 환경의 앱이 신 스키마 대응하는지 테스트
          curl -s https://staging-api.example.com/health
          # 적절한 테스트 스위트 실행
          npm test -- staging
      
      - name: Create deployment approval issue
        if: success()
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `[PROD] Approve migration - ${context.sha.substring(0, 7)}`,
              body: `스테이징 마이그레이션이 성공했습니다.\n\n프로덕션 배포 승인: https://github.com/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`,
              labels: ['deployment', 'database']
            });

  deploy-production:
    name: Deploy to Production
    needs: test-on-staging
    runs-on: ubuntu-latest
    environment:
      name: production
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Flyway
        run: |
          cd /tmp
          wget -q https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/9.17.0/flyway-commandline-9.17.0-linux-x64.tar.gz
          tar -xzf flyway-commandline-9.17.0-linux-x64.tar.gz
          echo "/tmp/flyway-9.17.0" >> $GITHUB_PATH
      
      - name: Final validation on Production
        run: |
          flyway \
            -url="${{ secrets.PROD_DB_URL }}" \
            -user="${{ secrets.PROD_DB_USER }}" \
            -password="${{ secrets.PROD_DB_PASSWORD }}" \
            -locations=filesystem:$(pwd)/db/migrations \
            validate
      
      - name: Run migration on Production
        run: |
          flyway \
            -url="${{ secrets.PROD_DB_URL }}" \
            -user="${{ secrets.PROD_DB_USER }}" \
            -password="${{ secrets.PROD_DB_PASSWORD }}" \
            -locations=filesystem:$(pwd)/db/migrations \
            migrate
```

---

## 🔬 내부 동작 원리

### 1. Flyway Validate의 동작 과정

```
flyway validate 실행
  ↓
1. 로컬 마이그레이션 파일 스캔
   - db/migrations/*.sql 읽기
   - 각 파일의 checksum 계산 (MD5)
   ↓
2. flyway_schema_history 조회
   - 이미 적용된 마이그레이션 버전과 checksum 비교
   ↓
3. 검증 로직
   - 파일과 DB의 checksum이 일치하는가?
   - 순서가 연속적인가? (V001, V002, V003...)
   - Out-of-order는 허용되는가?
   ↓
4. 결과 반환
   - 성공: "Successfully validated N migrations"
   - 실패: "Detected applied migration not resolved locally"
```

**Checksum 계산 (MD5)**:
```
V1__initial.sql의 내용:
CREATE TABLE users (id BIGINT PRIMARY KEY);

Checksum = MD5("CREATE TABLE users (id BIGINT PRIMARY KEY);")
         = 1234567890abcdef...

이 값이 flyway_schema_history에 저장되고,
다음 validate에서 현재 파일의 checksum과 비교됨.
```

---

### 2. Dry Run의 동작 원리 (Teams Edition)

```
flyway migrate -dryRunOutput=/tmp/output.sql
  ↓
1. 마이그레이션 파일 로드 (validate와 동일)
  ↓
2. SQL 파싱
   - 각 마이그레이션의 SQL을 파싱하여 변환
   - 트랜잭션 래핑 추가
   ↓
3. 실제 DB 연결 없이 SQL 출력
   - /tmp/output.sql에 합쳐진 SQL 저장
   ↓
4. 파일 수동 검토 가능
   - SQL 구문 확인
   - 테이블 변경 내용 확인
   - 예상 성능 영향 추정
```

**Dry Run 출력 예**:
```sql
-- Flyway metadata table
CREATE TABLE IF NOT EXISTS flyway_schema_history (
  -- columns...
);

-- Transactions
SET autocommit=0;

-- Migration V1
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  email VARCHAR(255)
);

-- Migration V2
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'ACTIVE';

-- Rollback to savepoint if needed
COMMIT;
```

---

### 3. information_schema를 이용한 영향 분석

```sql
-- 마이그레이션 전 테이블 통계 수집
SELECT
  table_name,
  table_rows,
  data_length / 1024 / 1024 AS size_mb,
  ENGINE,
  ROW_FORMAT
FROM information_schema.tables
WHERE table_schema = 'mydb'
ORDER BY table_rows DESC;

-- 결과:
-- users      | 5000000 | 256.5 MB | InnoDB | DYNAMIC
-- posts      | 2000000 | 156.2 MB | InnoDB | DYNAMIC
-- comments   | 8000000 | 512.3 MB | InnoDB | DYNAMIC
```

**용도**:
- ALTER TABLE이 큰 테이블을 변경할 때 예상 잠금 시간 추정
- 프로덕션 마이그레이션 일정 조율
- 온라인 마이그레이션 도구 선택 판단

---

## 💻 실전 실험

### 실험 1: flyway validate 실패 케이스 재현

```bash
#!/bin/bash
# test-validate.sh

set -e

DB_HOST=localhost
DB_USER=root
DB_PASS=root
TEST_DB=test_validate

# Step 1: 초기 DB 설정
echo "=== Step 1: 초기 DB 설정 ==="
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS << EOF
DROP DATABASE IF EXISTS $TEST_DB;
CREATE DATABASE $TEST_DB;
EOF

mkdir -p migrations

# Step 2: 첫 번째 마이그레이션 적용
echo "=== Step 2: V1 마이그레이션 적용 ==="
cat > migrations/V1__initial.sql << 'EOF'
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255)
);
EOF

docker run --rm \
  -v $(pwd)/migrations:/migrations \
  -e FLYWAY_URL=jdbc:mysql://$DB_HOST:3306/$TEST_DB \
  -e FLYWAY_USER=$DB_USER \
  -e FLYWAY_PASSWORD=$DB_PASS \
  flyway/flyway:9 migrate

echo "✓ V1 적용 완료"

# Step 3: V1 파일 검증 (성공해야 함)
echo ""
echo "=== Step 3: V1 검증 (성공 예상) ==="
docker run --rm \
  -v $(pwd)/migrations:/migrations \
  -e FLYWAY_URL=jdbc:mysql://$DB_HOST:3306/$TEST_DB \
  -e FLYWAY_USER=$DB_USER \
  -e FLYWAY_PASSWORD=$DB_PASS \
  flyway/flyway:9 validate

echo "✓ V1 검증 성공"

# Step 4: V1 파일 의도적으로 변경
echo ""
echo "=== Step 4: V1 파일 변경 ==="
cat > migrations/V1__initial.sql << 'EOF'
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(512)  # 255 -> 512 (변경됨)
);
EOF

echo "파일 변경됨 (VARCHAR 255 -> 512)"

# Step 5: 검증 시도 (실패해야 함)
echo ""
echo "=== Step 5: V1 검증 (실패 예상) ==="
docker run --rm \
  -v $(pwd)/migrations:/migrations \
  -e FLYWAY_URL=jdbc:mysql://$DB_HOST:3306/$TEST_DB \
  -e FLYWAY_USER=$DB_USER \
  -e FLYWAY_PASSWORD=$DB_PASS \
  flyway/flyway:9 validate || {
    echo "❌ 예상대로 checksum 불일치 오류 발생"
    echo ""
    echo "=== 해결 방법 ==="
    echo "1. 파일을 원래대로 복구하거나"
    echo "2. 새 버전의 마이그레이션으로 변경사항 적용"
    echo ""
    echo "예: V2__alter_users.sql 생성"
    
    cat > migrations/V2__alter_users.sql << 'EOF'
ALTER TABLE users MODIFY COLUMN email VARCHAR(512);
EOF

    echo "✓ V2 추가 마이그레이션 생성"
    
    # V1 원복
    cat > migrations/V1__initial.sql << 'EOF'
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255)
);
EOF

    echo "✓ V1 원복"
    
    # 재검증
    docker run --rm \
      -v $(pwd)/migrations:/migrations \
      -e FLYWAY_URL=jdbc:mysql://$DB_HOST:3306/$TEST_DB \
      -e FLYWAY_USER=$DB_USER \
      -e FLYWAY_PASSWORD=$DB_PASS \
      flyway/flyway:9 validate
    
    echo "✓ 이제 검증 통과"
  }
```

**실행 결과**:
```
=== Step 3: V1 검증 (성공 예상) ===
✓ V1 검증 성공

=== Step 5: V1 검증 (실패 예상) ===
❌ Detected applied migration not resolved locally: 1__initial
Detected resolved migration not removed from database: 1__initial
Checksum mismatch of migration V1__initial
```

---

### 실험 2: 위험한 DDL 감지 스크립트

```bash
#!/bin/bash
# detect-dangerous-ddl.sh

MIGRATION_DIR="db/migrations"
TEMP_REPORT="/tmp/ddl-report.txt"

> $TEMP_REPORT

analyze_migration() {
  local file=$1
  local basename=$(basename "$file")
  
  # 큰 테이블에 대한 변경인지 확인
  local altered_tables=$(grep -oP '(?<=ALTER TABLE )[^ ;]*' "$file" || true)
  
  for table in $altered_tables; do
    # information_schema에서 테이블 크기 조회 (MySQL 필요)
    local row_count=$(mysql -h localhost -u root -e "
      SELECT table_rows
      FROM information_schema.tables
      WHERE table_name = '$table' AND table_schema = 'mydb'
    " 2>/dev/null | tail -1 || echo 0)
    
    echo "$basename: ALTER TABLE $table (약 $row_count 행)" >> $TEMP_REPORT
  done
  
  # 위험한 연산 감지
  if grep -q "DROP TABLE\|DROP COLUMN\|DROP DATABASE" "$file"; then
    echo "⚠️  $basename: DROP 연산 감지" >> $TEMP_REPORT
  fi
  
  if grep -q "TRUNCATE" "$file"; then
    echo "⚠️  $basename: TRUNCATE 감지 (데이터 손실 위험)" >> $TEMP_REPORT
  fi
  
  if grep -q "DELETE.*WHERE\|UPDATE.*WHERE" "$file"; then
    # WHERE 절이 있으면 괜찮음
    if grep -qE "DELETE|UPDATE" "$file" && ! grep -q "WHERE" "$file"; then
      echo "❌ $basename: WHERE 절 없는 DELETE/UPDATE (데이터 손실!)" >> $TEMP_REPORT
    fi
  fi
  
  if grep -qE "RENAME.*COLUMN|RENAME.*TABLE" "$file"; then
    echo "⚠️  $basename: RENAME 연산 (호환성 주의)" >> $TEMP_REPORT
  fi
}

echo "분석 중..."
for file in $MIGRATION_DIR/*.sql; do
  [ -f "$file" ] && analyze_migration "$file"
done

if [ -s $TEMP_REPORT ]; then
  echo ""
  echo "=== 마이그레이션 위험도 분석 ==="
  cat $TEMP_REPORT
else
  echo "✓ 위험한 DDL 없음"
fi
```

---

### 실험 3: 스테이징 → 프로덕션 검증 파이프라인

```bash
#!/bin/bash
# staging-to-prod-pipeline.sh

set -e

STAGING_DB_URL="jdbc:mysql://staging-db:3306/mydb"
PROD_DB_URL="jdbc:mysql://prod-db:3306/mydb"
DB_USER="migration"
DB_PASS="secret"

echo "=== 스테이징 → 프로덕션 마이그레이션 파이프라인 ==="

# Step 1: 로컬 검증
echo ""
echo "Step 1: 로컬 검증"
docker run --rm \
  -v $(pwd)/db/migrations:/migrations \
  -e FLYWAY_URL="$STAGING_DB_URL" \
  -e FLYWAY_USER="$DB_USER" \
  -e FLYWAY_PASSWORD="$DB_PASS" \
  flyway/flyway:9 validate

# Step 2: 스테이징에 Dry Run
echo ""
echo "Step 2: 스테이징 Dry Run"
docker run --rm \
  -v $(pwd)/db/migrations:/migrations \
  -e FLYWAY_URL="$STAGING_DB_URL" \
  -e FLYWAY_USER="$DB_USER" \
  -e FLYWAY_PASSWORD="$DB_PASS" \
  -e FLYWAY_OUT_OF_ORDER=true \
  flyway/flyway:9 \
  -dryRunOutput=/tmp/staging-dryrun.sql \
  migrate || true

echo "Dry Run 결과:"
if [ -f /tmp/staging-dryrun.sql ]; then
  cat /tmp/staging-dryrun.sql
fi

# Step 3: 스테이징 실제 마이그레이션
echo ""
echo "Step 3: 스테이징 실제 마이그레이션"
docker run --rm \
  -v $(pwd)/db/migrations:/migrations \
  -e FLYWAY_URL="$STAGING_DB_URL" \
  -e FLYWAY_USER="$DB_USER" \
  -e FLYWAY_PASSWORD="$DB_PASS" \
  flyway/flyway:9 migrate

echo "✓ 스테이징 마이그레이션 완료"

# Step 4: 스테이징에서 테스트
echo ""
echo "Step 4: 스테이징 테스트"
echo "- 테이블 조회"
docker exec staging-db mysql -u $DB_USER -p$DB_PASS mydb -e "
  SHOW TABLES;
"

# Step 5: 사용자 승인 (수동)
echo ""
echo "Step 5: 프로덕션 승인 대기"
read -p "프로덕션에 마이그레이션을 적용하시겠습니까? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
  echo "중단되었습니다"
  exit 1
fi

# Step 6: 프로덕션 최종 검증
echo ""
echo "Step 6: 프로덕션 최종 검증"
docker run --rm \
  -v $(pwd)/db/migrations:/migrations \
  -e FLYWAY_URL="$PROD_DB_URL" \
  -e FLYWAY_USER="$DB_USER" \
  -e FLYWAY_PASSWORD="$DB_PASS" \
  flyway/flyway:9 validate

# Step 7: 프로덕션 Dry Run
echo ""
echo "Step 7: 프로덕션 Dry Run"
docker run --rm \
  -v $(pwd)/db/migrations:/migrations \
  -e FLYWAY_URL="$PROD_DB_URL" \
  -e FLYWAY_USER="$DB_USER" \
  -e FLYWAY_PASSWORD="$DB_PASS" \
  flyway/flyway:9 \
  -dryRunOutput=/tmp/prod-dryrun.sql \
  migrate || true

echo "프로덕션 Dry Run 결과:"
if [ -f /tmp/prod-dryrun.sql ]; then
  head -20 /tmp/prod-dryrun.sql
  echo "... (전체는 로그 참고)"
fi

# Step 8: 프로덕션 마이그레이션
echo ""
echo "Step 8: 프로덕션 마이그레이션 실행"
docker run --rm \
  -v $(pwd)/db/migrations:/migrations \
  -e FLYWAY_URL="$PROD_DB_URL" \
  -e FLYWAY_USER="$DB_USER" \
  -e FLYWAY_PASSWORD="$DB_PASS" \
  flyway/flyway:9 migrate

echo ""
echo "✓ 프로덕션 마이그레이션 완료!"
```

---

## 📊 성능/비용 비교

| 단계 | 검증 내용 | 소요 시간 | 비용 | 효과 |
|------|---------|---------|------|------|
| validate | Checksum, 순서 | ~1초 | 무료 | 가장 빠른 감지 |
| Dry Run (Teams) | SQL 미리보기 | ~2초 | 유료 | 정확한 SQL 검토 |
| Staging 실행 | 실제 실행, 테스트 | ~5분 | 낮음 | 프로덕션 환경 시뮬레이션 |
| 온라인 마이그레이션 도구 | pt-osc, 대용량 테이블 처리 | ~10~30분 | 높음 | 무중단 마이그레이션 |

---

## ⚖️ 트레이드오프

### 검증만 수행 (validate)
**장점**: 빠르고 간단
**단점**: SQL 내용 확인 불가

### Staging에서 전체 테스트
**장점**: 실제 환경과 동일하게 테스트
**단점**: 시간 오래 걸림, 비용 증가

### 자동 검증 + 수동 승인
**장점**: 안전과 속도의 균형
**단점**: 승인자 필요, 프로세스 복잡

---

## 📌 핵심 정리

1. **Validate는 필수 단계** (모든 마이그레이션 배포 전)
   - Checksum 검증으로 파일 변경 감지
   - 순서 검증으로 Out-of-order 방지

2. **Staging에서 리허설** (프로덕션 전)
   - 실제 데이터로 테스트
   - 성능 영향 사전 확인
   - 앱 호환성 검증

3. **위험한 DDL 자동 감지**
   - DROP, TRUNCATE, DELETE, UPDATE 등
   - 프로덕션 장애 사전 방지

4. **Dry Run으로 SQL 사전 검토** (Flyway Teams)
   - 승인자가 정확히 무엇이 실행될지 확인
   - 마이그레이션 파일과 실제 SQL의 차이 감지

5. **파이프라인**: Validate → Test → Staging → Production
   - 각 단계에서 실패 시 다음 단계 중단

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 마이그레이션이 validate를 통과했는데 실제 실행에서 오류가 난다면?</strong></summary>

**예**:
```sql
-- V5__add_constraint.sql
ALTER TABLE posts ADD UNIQUE KEY uk_title (title);

-- validate: 성공 (문법 OK)
-- 실행: 실패 (duplicate title이 이미 존재함)
```

**validate의 한계**:
- SQL 문법만 검증 (의미 검증 아님)
- 데이터 제약 조건 검증 불가
- 개체 참조 검증 불가

**해결**:
```sql
-- 1. 데이터 먼저 정리
DELETE FROM posts WHERE title IS NULL OR title = '';

-- 2. 중복 제거
DELETE FROM posts WHERE id NOT IN (
  SELECT MIN(id) FROM posts GROUP BY title
);

-- 3. 그 후 UNIQUE 제약 추가
ALTER TABLE posts ADD UNIQUE KEY uk_title (title);
```

**교훈**: Validate는 **문법만 검증**하므로, Staging 환경에서 **실제 데이터로 테스트**해야 합니다.
</details>

<details>
<summary><strong>Q2: 프로덕션에서 마이그레이션 중 중단되었다면?</strong></summary>

**예**: 마이그레이션이 50% 진행 중일 때 네트워크 끊김

**Flyway의 대응**:
```
마이그레이션 중:
1. 트랜잭션 시작
   BEGIN TRANSACTION;

2. SQL 실행 중... 50% 완료
   ALTER TABLE users ADD COLUMN status VARCHAR(50);
   -- 네트워크 끊김!

3. 트랜잭션 자동 롤백
   ROLLBACK;
   (모든 변경사항 취소)

4. flyway_schema_history에 기록 안 함
   (적용 실패이므로 미기록)

5. 재시도 시 다시 처음부터
   마이그레이션 완전 재실행
```

**InnoDB의 자동 롤백**:
- InnoDB는 트랜잭션을 지원하므로 부분 적용 불가능
- 모두 성공 또는 모두 실패

**해결**:
```bash
# 재접속하여 마이그레이션 재시도
flyway migrate

# Flyway가 자동으로:
# 1. 이전 마이그레이션 checksum 검증
# 2. 마지막으로 적용된 버전 확인 (flyway_schema_history)
# 3. 미적용된 마이그레이션만 실행
```

**교훈**: 마이그레이션은 **멱등성이 보장**되어야 합니다 (여러 번 실행해도 같은 결과).
</details>

<details>
<summary><strong>Q3: 스테이징과 프로덕션의 데이터 용량이 다르면 마이그레이션 시간도 다르지 않을까?</strong></summary>

**예**:
```
Staging: users 테이블 100,000행
Production: users 테이블 50,000,000행

같은 ALTER 마이그레이션:
Staging: 2초
Production: 200초 (100배)
```

**이유**:
```
ALTER TABLE users ADD COLUMN status VARCHAR(50);

MySQL 5.7 이하:
1. 전체 테이블 메모리 복사 (50M행)
2. 새 테이블 생성 + 데이터 복사
3. Swap (원본과 신규 테이블 이름 변경)

시간: 행 개수에 정비례
```

**대책**:
```sql
-- 1. 온라인 마이그레이션 도구 (pt-online-schema-change)
-- Original 테이블 유지하면서 신규 테이블에 데이터 복사
-- Trigger로 변경사항 동기화

-- 2. MySQL 8.0 이상의 INSTANT 알고리즘
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'ACTIVE';
-- Algorithm=INSTANT (메타데이터만 변경, 데이터 복사 없음, ~0.1초)

-- 3. 배치 마이그레이션
-- 사용자 배치를 나누어 마이그레이션
-- 예: 월 1일씩 1000개 레코드 마이그레이션
```

**교훈**: 대용량 데이터의 마이그레이션은 **Staging 환경의 데이터 양이 프로덕션과 유사**해야 정확한 성능 예측이 가능합니다.
</details>

---

<div align="center">

**[⬅️ 이전: GitHub Actions 통합](./02-github-actions-integration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Kubernetes 배포와 마이그레이션 ➡️](./04-kubernetes-migration.md)**

</div>
