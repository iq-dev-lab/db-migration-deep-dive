# GitHub Actions 통합

---

## 🎯 핵심 질문

- GitHub Actions에서 마이그레이션 파일 변경을 자동으로 감지하고 실행하려면?
- 테스트 환경에서는 자동 마이그레이션하고, 프로덕션은 수동 승인 게이트로 보호하려면?
- 마이그레이션 실패 시 Slack으로 즉시 알림을 받으려면?
- 여러 환경(dev, staging, prod)에서 서로 다른 DB 접속 정보를 안전하게 관리하려면?

---

## 🔍 왜 이 개념이 실무에서 중요한가

GitHub Actions는 대부분의 팀이 사용하는 기본 CI/CD 도구입니다. 마이그레이션을 GitHub Actions에 통합하면:

1. **마이그레이션 자동화**: 코드 푸시 → 자동으로 마이그레이션 검증 및 실행
2. **환경별 분리**: 개발/스테이징은 자동, 프로덕션은 수동 승인 (가트)
3. **가시성**: 마이그레이션 실행 이력이 GitHub 에서 직접 조회 가능
4. **팀 협업**: PR에서 마이그레이션 파일 변경 내용 검토, Comment로 위험 알림
5. **실패 알림**: Slack 연동으로 실패 시 팀원 즉시 통보

특히 **프로덕션 마이그레이션은 자동 실행되면 안 되므로**, GitHub Actions의 `environment` + `required_reviewers`로 수동 승인 게이트를 설정하는 것이 필수입니다.

---

## 😱 흔한 실수 (Before — ...)

### 실수 1: "모든 환경에 자동으로 마이그레이션 실행"

```yaml
name: DB Migration

on: [push]

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Migration
        run: |
          docker run --rm \
            -e FLYWAY_URL=jdbc:mysql://${{ secrets.DB_URL }} \
            -e FLYWAY_USER=${{ secrets.DB_USER }} \
            -e FLYWAY_PASSWORD=${{ secrets.DB_PASS }} \
            flyway/flyway:9 migrate
```

**문제**:
- Dev, Staging, Prod 모두에 자동으로 마이그레이션 실행
- 실수로 잘못된 마이그레이션 파일을 푸시하면 프로덕션 DB가 손상될 수 있음
- 프로덕션 데이터 손실 → 심각한 장애

---

### 실수 2: "Slack 알림 없이 마이그레이션 실패를 놓침"

```yaml
- name: Run Migration
  run: |
    docker run --rm \
      -e FLYWAY_URL=jdbc:mysql://${{ secrets.PROD_DB_URL }} \
      flyway/flyway:9 migrate
```

**문제**:
- 마이그레이션 실패해도 GitHub Actions 로그를 일일이 확인해야 함
- 팀원들이 장애를 모를 수 있음
- 디버깅이 늦어짐

---

### 실수 3: "DB 접속 정보를 Workflow 파일에 직접 작성"

```yaml
steps:
  - name: Run Migration
    env:
      FLYWAY_URL: jdbc:mysql://mysql.c2klqnxq1234.us-east-1.rds.amazonaws.com:3306/mydb
      FLYWAY_USER: admin_user
      FLYWAY_PASSWORD: MyS3cr3tP@ssw0rd  # 위험!
```

**문제**:
- 비밀 정보가 GitHub 리포지토리에 노출
- 누구든 이 정보로 프로덕션 DB 접근 가능
- 보안 사고 → GDPR 벌금

---

### 실수 4: "마이그레이션 검증 없이 바로 실행"

```yaml
steps:
  - name: Run Migration
    run: flyway migrate  # validate 먼저 하지 않음!
```

**문제**:
- 마이그레이션 파일이 손상되거나 이전 파일이 변경된 경우 감지 불가
- 부분적으로만 적용되고 롤백 불가능
- DB 불일치 상태 → 서비스 장애

---

## ✨ 올바른 접근 (After — ...)

### 올바른 패턴: 환경별 다단계 파이프라인

```yaml
name: Database Migration Pipeline

on:
  push:
    paths:
      - 'db/migrations/**'
      - '.github/workflows/db-migration.yml'

env:
  FLYWAY_VERSION: 9.17.0

jobs:
  # Job 1: 마이그레이션 파일 검증
  validate:
    name: Validate Migrations
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Check migration files exist
        run: |
          if [ ! -d "db/migrations" ]; then
            echo "❌ db/migrations 디렉토리가 없습니다"
            exit 1
          fi
          echo "✓ 마이그레이션 파일 ${#MIGRATIONS[@]} 개 발견"
        shell: bash
      
      - name: Lint SQL files
        run: |
          echo "마이그레이션 파일 검사..."
          for file in db/migrations/*.sql; do
            echo "검사: $file"
            # 기본적인 SQL 문법 검사 (더 나은 도구: sqlfluff)
            if ! grep -q "^--" "$file"; then
              echo "⚠️  경고: $file에 설명 주석이 없습니다"
            fi
          done

  # Job 2: 테스트 DB에 마이그레이션 적용
  test:
    name: Test on Dev Database
    runs-on: ubuntu-latest
    needs: validate
    
    services:
      mysql:
        image: mysql:5.7
        env:
          MYSQL_ROOT_PASSWORD: testpass
          MYSQL_DATABASE: testdb
        options: >-
          --health-cmd="mysqladmin ping -h localhost"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
        ports:
          - 3306:3306
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Wait for MySQL
        run: |
          for i in {1..30}; do
            if mysql -h 127.0.0.1 -u root -ptestpass -e "SELECT 1" &>/dev/null; then
              echo "✓ MySQL 준비 완료"
              break
            fi
            echo "MySQL 대기 중... ($i/30)"
            sleep 1
          done
      
      - name: Set up Flyway
        run: |
          cd /tmp
          wget -q https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/${FLYWAY_VERSION}/flyway-commandline-${FLYWAY_VERSION}-linux-x64.tar.gz
          tar -xzf flyway-commandline-${FLYWAY_VERSION}-linux-x64.tar.gz
          echo "/tmp/flyway-${FLYWAY_VERSION}" >> $GITHUB_PATH
      
      - name: Validate migrations
        run: |
          flyway \
            -url=jdbc:mysql://127.0.0.1:3306/testdb \
            -user=root \
            -password=testpass \
            -locations=filesystem:$(pwd)/db/migrations \
            validate
      
      - name: Run migrations
        run: |
          flyway \
            -url=jdbc:mysql://127.0.0.1:3306/testdb \
            -user=root \
            -password=testpass \
            -locations=filesystem:$(pwd)/db/migrations \
            migrate
      
      - name: Check schema
        run: |
          mysql -h 127.0.0.1 -u root -ptestpass testdb -e "
            SELECT table_name, engine, row_format
            FROM information_schema.tables
            WHERE table_schema = 'testdb'
            ORDER BY table_name;
          "

  # Job 3: Staging 자동 적용 (DEV 환경)
  deploy-dev:
    name: Deploy to Dev
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Flyway
        run: |
          cd /tmp
          wget -q https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/${FLYWAY_VERSION}/flyway-commandline-${FLYWAY_VERSION}-linux-x64.tar.gz
          tar -xzf flyway-commandline-${FLYWAY_VERSION}-linux-x64.tar.gz
          echo "/tmp/flyway-${FLYWAY_VERSION}" >> $GITHUB_PATH
      
      - name: Run migrations on Dev DB
        run: |
          flyway \
            -url="${{ secrets.DEV_DB_URL }}" \
            -user="${{ secrets.DEV_DB_USER }}" \
            -password="${{ secrets.DEV_DB_PASSWORD }}" \
            -locations=filesystem:$(pwd)/db/migrations \
            migrate
      
      - name: Verify Dev migration
        run: |
          echo "✓ Dev 마이그레이션 완료"

  # Job 4: Staging 자동 적용
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Flyway
        run: |
          cd /tmp
          wget -q https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/${FLYWAY_VERSION}/flyway-commandline-${FLYWAY_VERSION}-linux-x64.tar.gz
          tar -xzf flyway-commandline-${FLYWAY_VERSION}-linux-x64.tar.gz
          echo "/tmp/flyway-${FLYWAY_VERSION}" >> $GITHUB_PATH
      
      - name: Run migrations on Staging DB
        run: |
          flyway \
            -url="${{ secrets.STAGING_DB_URL }}" \
            -user="${{ secrets.STAGING_DB_USER }}" \
            -password="${{ secrets.STAGING_DB_PASSWORD }}" \
            -locations=filesystem:$(pwd)/db/migrations \
            migrate
      
      - name: Notify Staging Complete
        uses: 8398a7/action-slack@v3
        if: success()
        with:
          status: custom
          custom_payload: |
            {
              text: "✓ Staging 마이그레이션 완료",
              attachments: [{
                color: 'good',
                text: `커밋: ${process.env.AS_COMMIT}\n작성자: ${process.env.AS_AUTHOR}`
              }]
            }
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}

  # Job 5: 프로덕션 (수동 승인)
  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://api.production.example.com
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Flyway
        run: |
          cd /tmp
          wget -q https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/${FLYWAY_VERSION}/flyway-commandline-${FLYWAY_VERSION}-linux-x64.tar.gz
          tar -xzf flyway-commandline-${FLYWAY_VERSION}-linux-x64.tar.gz
          echo "/tmp/flyway-${FLYWAY_VERSION}" >> $GITHUB_PATH
      
      - name: Dry Run on Production DB
        run: |
          flyway \
            -url="${{ secrets.PROD_DB_URL }}" \
            -user="${{ secrets.PROD_DB_USER }}" \
            -password="${{ secrets.PROD_DB_PASSWORD }}" \
            -locations=filesystem:$(pwd)/db/migrations \
            -dryRunOutput=/tmp/migration-dryrun.sql \
            lflyMigrate
        continue-on-error: true
      
      - name: Display Dry Run SQL
        run: |
          echo "=== 프로덕션에 적용될 SQL ==="
          cat /tmp/migration-dryrun.sql
      
      - name: Run migrations on Production DB
        run: |
          flyway \
            -url="${{ secrets.PROD_DB_URL }}" \
            -user="${{ secrets.PROD_DB_USER }}" \
            -password="${{ secrets.PROD_DB_PASSWORD }}" \
            -locations=filesystem:$(pwd)/db/migrations \
            migrate
      
      - name: Notify Production Complete
        uses: 8398a7/action-slack@v3
        if: success()
        with:
          status: custom
          custom_payload: |
            {
              text: "✓ 프로덕션 마이그레이션 완료",
              attachments: [{
                color: 'good',
                text: `커밋: ${process.env.AS_COMMIT}\n작성자: ${process.env.AS_AUTHOR}`
              }]
            }
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}

  # Job 6: 실패 알림
  notify-failure:
    name: Notify Failure
    runs-on: ubuntu-latest
    needs: [validate, test, deploy-dev, deploy-staging, deploy-prod]
    if: failure()
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Send Slack Alert
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              text: "❌ DB 마이그레이션 실패!",
              attachments: [{
                color: 'danger',
                text: `커밋: ${process.env.AS_COMMIT}\n작성자: ${process.env.AS_AUTHOR}\n\n GitHub Actions 로그를 확인하세요: ${process.env.AS_WORKFLOW_URL}`
              }]
            }
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 🔬 내부 동작 원리

### 1. GitHub Actions 환경(environment) 및 필수 검토자 설정

GitHub 리포지토리 설정:

```yaml
# .github/environments/production.yml (UI에서 설정해야 함)
# GitHub Settings > Environments > production

# 필수 검토자: (조직 멤버 2명 이상)
# - tech-lead@company.com
# - devops-lead@company.com

# 배포 브랜치 제약: 
# - main 브랜치만 허용

# 시크릿:
# - PROD_DB_URL
# - PROD_DB_USER
# - PROD_DB_PASSWORD
```

**작동**:
```
프로덕션 Job 실행 시도
  ↓
environment: production 선언 감지
  ↓
GitHub 자동으로 필수 검토자(2명)에게 알림
  ↓
모두가 "승인"을 클릭할 때까지 대기
  ↓
마이그레이션 실행
```

---

### 2. Slack 통합 (webhook)

```bash
# Slack Workspace 설정
1. https://api.slack.com/apps 접속
2. Create New App > From scratch
3. App name: "GitHub Migrations"
4. Workspace 선택
5. Incoming Webhooks > Add New Webhook to Workspace
6. 채널 선택: #database-alerts
7. Copy Webhook URL: https://hooks.slack.com/services/T00.../B00.../xxx...

# GitHub 리포지토리 시크릿 설정
Settings > Secrets > New repository secret
Name: SLACK_WEBHOOK
Value: https://hooks.slack.com/services/...
```

**작동**:
```
Workflow 실행
  ↓
Job 성공/실패
  ↓
action-slack 액션이 webhook 호출
  ↓
Slack 채널에 메시지 표시
```

---

### 3. Dry Run (프로덕션 전 미리보기)

```bash
# Flyway 9.0+ 에서만 지원 (Teams 버전)
flyway -dryRunOutput=/tmp/output.sql migrate

# 출력 예:
# SET autocommit=0;
# use mydb;
# CREATE TABLE users (...);
# ALTER TABLE posts ADD COLUMN author_id ...;
# COMMIT;
```

**이점**:
- 실제 실행 없이 SQL 미리보기
- 승인자가 정확히 어떤 SQL이 실행될지 확인 가능
- 위험한 DDL 사전 감지

---

### 4. 마이그레이션 파일 변경 감지

```yaml
on:
  push:
    paths:
      - 'db/migrations/**'  # 이 경로의 파일 변경 시만 트리거
      - '.github/workflows/db-migration.yml'  # Workflow 파일 변경 시도 트리거
```

**작동**:
```
Git push
  ↓
GitHub Actions 평가
  ↓
'db/migrations/**' 또는 Workflow 파일 변경 있나?
  ↓
있음 -> Workflow 실행
없음 -> 실행 안 함 (트리거 무시)
```

---

## 💻 실전 실험

### 실험 1: 실제 동작하는 GitHub Actions Workflow

**리포지토리 구조**:
```
my-app/
├── db/
│   └── migrations/
│       ├── V1__initial_schema.sql
│       ├── V2__add_users_table.sql
│       └── V3__add_status_column.sql
├── .github/
│   └── workflows/
│       └── db-migration.yml
└── README.md
```

**V1__initial_schema.sql**:
```sql
-- Create initial schema
CREATE TABLE IF NOT EXISTS flyway_schema_history (
  installed_rank INT,
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

CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) NOT NULL UNIQUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE posts (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  title VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

**V2__add_users_table.sql**:
```sql
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'ACTIVE';
ALTER TABLE users ADD COLUMN last_login TIMESTAMP NULL;
```

**V3__add_status_column.sql**:
```sql
ALTER TABLE posts ADD COLUMN status VARCHAR(50) DEFAULT 'PUBLISHED';
```

**실행**:
```bash
cd my-app
git add db/migrations/ .github/workflows/db-migration.yml
git commit -m "Add database migrations"
git push origin main

# GitHub Actions 자동 트리거
# 1. validate: 마이그레이션 파일 검증
# 2. test: 테스트 DB에 적용
# 3. deploy-dev: Dev DB에 자동 적용
# 4. deploy-staging: Staging DB에 자동 적용
# 5. deploy-prod: 프로덕션 (수동 승인 대기)
```

---

### 실험 2: 프로덕션 수동 승인 게이트 테스트

```bash
# 1. PR 또는 커밋 푸시
git push origin main

# 2. GitHub Actions 실행
# validate, test, deploy-dev, deploy-staging 자동 완료

# 3. deploy-prod Job이 대기
# GitHub UI에 "Review deployments" 버튼 표시

# 4. PR/커밋 작성자가 승인 요청
# https://github.com/myorg/myrepo/actions/runs/12345

# 5. 필수 검토자(2명) 승인
# 각자 "Review pending deployments" 클릭
# "Approve and deploy" 선택

# 6. 모두 승인하면 deploy-prod Job 실행
# 프로덕션 마이그레이션 실행
# Slack으로 완료 알림 전송
```

---

### 실험 3: 실패 시나리오 및 롤백

**시나리오 1: 마이그레이션 파일에 문법 오류**

```sql
-- V4__bad_migration.sql (문법 오류)
ALTER TABLE users ADD COLUMN birthdate DATE;
SELECT * FROM users;  -- SELECT는 마이그레이션에 부적합
```

**결과**:
```
validate Job 실패
  ↓
test Job 건너뜀
  ↓
deploy-dev, deploy-staging, deploy-prod 모두 건너뜀
  ↓
notify-failure Job 실행
  ↓
Slack 알림: "❌ DB 마이그레이션 실패!"
```

**해결**:
```bash
# 1. 파일 수정
cat > db/migrations/V4__bad_migration.sql << 'EOF'
ALTER TABLE users ADD COLUMN birthdate DATE;
EOF

# 2. 다시 커밋 및 푸시
git add db/migrations/V4__bad_migration.sql
git commit -m "Fix migration syntax error"
git push origin main

# 3. GitHub Actions 자동으로 다시 실행
# 이번엔 성공
```

**시나리오 2: 프로덕션에 의도하지 않은 DROP TABLE**

```sql
-- V5__cleanup.sql
DROP TABLE posts;  # 실수로 작성됨!
```

**방어 메커니즘**:
```yaml
- name: Detect dangerous DDL
  run: |
    if grep -q "DROP TABLE\|DROP COLUMN\|DROP DATABASE\|DELETE\|TRUNCATE" db/migrations/*.sql; then
      echo "❌ 위험한 DDL 감지되었습니다"
      echo "이 마이그레이션을 적용하시겠습니까?"
      exit 1  # 수동 검토 필수
    fi
```

**결과**: deploy-prod Job에서 수동 검토 요청, 승인자가 신중히 검토한 후 승인.

---

## 📊 성능/비용 비교

| 항목 | 수동 마이그레이션 | 자동 (Dev) | 자동 (Staging) | 수동 승인 (Prod) |
|------|------------------|-----------|--------------|-----------------|
| 소요 시간 | 30분+ (수동 작업) | ~2분 (자동) | ~2분 (자동) | ~5분 (승인 대기) |
| 인적 오류 위험 | 높음 | 낮음 | 낮음 | 매우 낮음 |
| 추적 가능성 | 없음 | GitHub에 기록 | GitHub에 기록 | GitHub + Slack |
| 롤백 난이도 | 높음 | 낮음 | 낮음 | 높음 (데이터 복구 필요) |
| GitHub Actions 비용 | 0 | ~1분/실행 | ~1분/실행 | ~1분/실행 |

---

## ⚖️ 트레이드오프

### 전체 자동화 (Dev + Staging + Prod)
**장점**:
- 가장 빠른 배포 (무중단 배포 시간 단축)
- 팀원 개입 최소화

**단점**:
- 치명적 오류 시 프로덕션 DB 손상
- 보안: 시크릿 노출 위험
- 규정(컴플라이언스) 미충족 (감사 추적 불충분)

### 일부 자동 + 프로덕션 수동 승인 (권장)
**장점**:
- Dev/Staging은 빠른 피드백
- 프로덕션은 신중한 검토
- 팀의 신뢰 구축
- 규정 준수

**단점**:
- 프로덕션 배포 시간 증가 (승인 대기)
- 필수 검토자 부재 시 배포 불가

### 전체 수동 (GitHub Actions 최소화)
**장점**:
- 모든 마이그레이션을 명시적으로 검토
- 가장 안전

**단점**:
- 가장 느린 배포
- 팀의 병목
- 자동화 이점 상실

---

## 📌 핵심 정리

1. **GitHub Actions는 마이그레이션 자동화의 필수 도구**
   - PR 단계에서 검증 (validate)
   - 테스트 DB에서 검증 (test)
   - Dev/Staging 자동 적용
   - 프로덕션은 수동 승인

2. **환경별 다단계 파이프라인 구성**
   - validate → test → dev → staging → prod (수동)
   - 각 단계 실패 시 다음 단계 자동 중단

3. **Slack 통합으로 팀원 즉시 통보**
   - 성공: 초록색 알림
   - 실패: 빨간색 알림 + 로그 링크

4. **프로덕션 수동 승인은 필수**
   - `environment` + `required_reviewers` 설정
   - 2명 이상 동시 승인 요구
   - Dry Run으로 미리보기 가능

5. **시크릿 관리는 철저하게**
   - GitHub Secrets 사용 (암호화)
   - 환경별 다른 계정 사용 (least privilege)
   - 정기적 로테이션

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 만약 프로덕션 마이그레이션 중 필수 검토자 2명이 모두 오프라인 상태라면 어떻게 할까?</strong></summary>

**상황**:
- 긴급 마이그레이션이 필요 (예: 보안 패치)
- 필수 검토자 2명이 사용 불가능 (휴가, 야간 등)
- 배포가 지연되고 있음

**해결 방법 1: 임시 검토자 추가**
```
Settings > Environments > production > Deployment branches
Required reviewers 추가 시 3명 이상으로 설정
비상 연락처(온콜, 시니어 엔지니어) 포함
```

**해결 방법 2: 긴급 배포 프로세스**
```
일반: dev -> staging -> production (수동 승인)
긴급: dev -> production_emergency (1명 승인, 감사 기록)
```

**해결 방법 3: 최소 승인자 동적 설정**
```yaml
# Workflow에서 조건부 검토자 설정
deploy-prod:
  environment:
    name: production
    required_reviewers:
      - tech-lead  # 기본
      - devops-lead
      # 평일 19:00~07:00: oncall-engineer도 승인 가능
```

**교훈**: 비상 시나리오에 대한 사전 계획이 필수입니다.
</details>

<details>
<summary><strong>Q2: 마이그레이션 Dry Run 출력에 민감한 정보(암호화 키, 개인정보)가 노출된다면?</strong></summary>

**예**:
```sql
-- V10__add_encryption_keys.sql
ALTER TABLE secrets ADD COLUMN api_key VARCHAR(255);
INSERT INTO secrets (api_key) VALUES ('sk-1234567890abcdef');  # Dry Run에 노출!
```

**해결 방법 1: Dry Run 출력 마스킹**
```bash
- name: Run Dry Run
  run: |
    flyway -dryRunOutput=/tmp/dryrun.sql migrate
    
    # 민감한 패턴 마스킹
    sed -i "s/VALUES ('[^']*')/VALUES ('***')/g" /tmp/dryrun.sql
    cat /tmp/dryrun.sql
```

**해결 방법 2: Dry Run 비활성화 (프로덕션)**
```yaml
deploy-prod:
  steps:
    # Dry Run 대신 validate만 수행
    - run: |
        flyway \
          -url="${{ secrets.PROD_DB_URL }}" \
          -password="${{ secrets.PROD_DB_PASSWORD }}" \
          validate  # SQL 출력 없음
    
    # GitHub 시크릿으로 실제 SQL 파일 암호화 제공
    - run: flyway migrate  # 비밀 정보는 로그에 안 보임
```

**해결 방법 3: GitHub Actions 로그 암호화**
```yaml
steps:
  - name: Mask Sensitive Output
    run: |
      echo "::add-mask::$(cat /tmp/dryrun.sql | grep -o "'[^']*'" | head -5)"
```

**교훈**: Dry Run은 프로덕션에서 신중하게 사용해야 하며, 민감한 정보 마스킹이 필수입니다.
</details>

<details>
<summary><strong>Q3: 마이그레이션 파일을 실수로 수정(다시 작성)했을 때 Flyway가 감지할까?</strong></summary>

**예**:
```sql
-- V1__create_users.sql (원본)
CREATE TABLE users (id BIGINT PRIMARY KEY, email VARCHAR(255));

-- flyway_schema_history에 기록됨
-- checksum: 1234567890

---

-- 몇 주 후, V1__create_users.sql 수정 (실수)
CREATE TABLE users (id BIGINT PRIMARY KEY, email VARCHAR(512));  -- 255 -> 512

-- Checksum: 0987654321 (다름!)
```

**Flyway의 감지**:
```
마이그레이션 실행 시:
1. V1__create_users.sql의 현재 checksum 계산 (0987654321)
2. flyway_schema_history에서 이전 checksum 조회 (1234567890)
3. 불일치 감지!

에러:
"Detected applied migration not resolved locally: 1__create_users
Detected resolved migration not applied to database: 1__create_users"
```

**방지 방법 1: Flyway 설정으로 검증**
```yaml
flyway:
  validateOnMigrate: true  # 기본값 true
  outOfOrder: false  # 순서 변경 방지
```

**방법 2: Git 히스토리로 보호**
```bash
# 이미 적용된 마이그레이션은 수정 금지
# Git hook으로 방지:

#!/bin/bash
# .git/hooks/pre-commit
APPLIED=$(mysql ... -e "SELECT script FROM flyway_schema_history")
MODIFIED=$(git diff --cached --name-only)

for file in $MODIFIED; do
  if echo "$APPLIED" | grep -q "$file"; then
    echo "❌ 이미 적용된 마이그레이션 수정 불가: $file"
    exit 1
  fi
done
```

**교훈**: 마이그레이션 파일은 **수정이 아니라 새 버전으로 추가**해야 합니다. Flyway의 checksum 검증은 이를 자동으로 감지합니다.
</details>

---

<div align="center">

**[⬅️ 이전: 배포 파이프라인에서의 마이그레이션 시점](./01-migration-timing-in-pipeline.md)** | **[홈으로 🏠](../README.md)** | **[다음: 마이그레이션 검증 자동화 ➡️](./03-migration-validation.md)**

</div>
