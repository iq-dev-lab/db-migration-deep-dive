# 체크섬 불일치 오류 해결

---

## 🎯 핵심 질문

1. "Migrations have failed validation. Migration checksum mismatch" 오류가 발생하는 정확한 원인 3가지는 무엇인가?
2. CRC32 체크섬 계산에서 CRLF(줄바꿈)와 BOM(Byte Order Mark) 때문에 불일치가 발생하는 이유는?
3. `flyway repair` 명령어가 정확히 무엇을 하고, 언제 안전하며 언제 위험한가?
4. 팀에서 체크섬 충돌을 예방하는 규칙은 무엇이고, `.gitattributes` 설정은 어떻게 하는가?
5. PR merge 전에 `flyway validate`를 자동으로 실행하는 CI/CD 설정은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이그레이션이 적용된 후 파일을 수정하는 것은 가장 흔한 실수입니다. 세미콜론 하나를 빼먹었거나, 주석을 추가했거나, 줄바꿈 방식을 변경했거나... 그러면 체크섬이 변경되어 Flyway가 "수정되었다!"고 감지합니다. 이때 무작정 `flyway repair`를 실행하면 프로덕션 DB가 손상될 수 있습니다.

또한 Windows에서 작업한 개발자(CRLF)와 Linux에서 작업한 개발자(LF)가 같은 파일을 수정하면 깃에서는 같은 파일인데 Flyway에서는 체크섬이 다릅니다. 이런 환경별 불일치를 미리 방지하는 `.gitattributes` 설정을 모르면 매번 체크섬 문제를 마주칠 것입니다. 실무에서는 이런 문제를 사전에 예방하는 것이 핵심입니다.

---

## 😱 흔한 실수 (Before — ...)

```sql
-- ❌ 흔한 실수 1: 적용된 마이그레이션 파일을 실수로 수정
-- V1__create_users.sql (원본, 이미 프로덕션 적용됨)
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(100)
);

-- 누군가가 주석을 추가
-- V1__create_users.sql (수정본)
-- Initial schema creation
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(100)
);

-- Spring Boot 시작 시:
-- $ mvn spring-boot:run
-- 
-- 로그:
-- ERROR: Validate failed: Migrations have failed validation
-- Migration checksum mismatch for migration version 1
--   Got:      -1234567890 (수정본)
--   Expected:  1234567891 (원본)
```

```
-- ❌ 흔한 실수 2: 줄바꿈 차이로 인한 체크섬 불일치
-- Windows 개발자: .gitattributes 없음
-- $ git config core.autocrlf false
-- 파일 내용: "CREATE TABLE users\r\n..."  (CRLF)
-- 체크섬: 111111

-- Linux CI/CD: .gitattributes 없음
-- $ git config core.autocrlf input
-- 파일 내용: "CREATE TABLE users\n..."    (LF)
-- 체크섬: 222222

-- 결과:
-- Flyway validate 실패
-- "Migration checksum mismatch"
-- 개발자 PC에서는 성공, CI에서는 실패
```

```bash
# ❌ 흔한 실수 3: 체크섬 불일치 시 무작정 repair 사용
$ flyway repair

# 아, Flyway repair가 뭐하는 건지 모르고 실행했다...
# flyway_schema_history의 체크섬이 현재 파일로 업데이트됨
# 
# 문제:
# 1. 실제로는 다른 버전의 마이그레이션이 적용됐을 수도
# 2. 파일과 DB의 스키마가 불일치할 수도
# 3. repair 이후 진짜 데이터 손상 시 복구 불가능
```

```sql
-- ❌ 흔한 실수 4: 프로덕션 DB에서 마이그레이션 파일 직접 수정
-- 프로덕션에서 급하게 쿼리 수정 (수동)
UPDATE users SET last_name = NULL WHERE last_name = '';

-- 그 다음 V2__fix_names.sql 파일 (개발)
-- 같은 수정을 SQL로 작성했지만 다른 방식
UPDATE users SET last_name = NULL WHERE last_name IS NOT NULL AND last_name = '';

-- 결과:
-- 프로덕션: 이미 실행되지 않음 (flyway_schema_history에 없음)
-- 개발: V2는 새로운 마이그레이션
-- 프로덕션과 개발의 스키마 불일치!
```

```
-- ❌ 흔한 실수 5: BOM(Byte Order Mark) 때문의 불일치
-- 파일을 UTF-8 with BOM으로 저장
-- (Windows 메모장이 기본값)
-- 
-- 파일 내용 (바이너리):
-- EF BB BF (BOM) + "CREATE TABLE ..."
-- 
-- 체크섬 계산:
-- BOM 포함: 111111
-- BOM 제외: 222222
-- 
-- flyway_schema_history에 저장된 체크섬과 다름!
```

---

## ✨ 올바른 접근 (After — ...)

```yaml
# ✅ 올바른 접근 1: 적용된 마이그레이션은 절대 수정하지 말 것
# .gitattributes
# (모든 마이그레이션 파일 공통 설정)
*.sql eol=lf  # LF로 통일
*.sql text    # 텍스트 파일로 취급

# application.yml
spring:
  flyway:
    validate-on-migrate: true  # 시작 시 검증 (반드시!)
    fail-on-validation-error: true  # 검증 실패 시 앱 시작 거부

# 규칙:
# 1. V1이 프로덕션에 적용되면 절대 수정 금지
# 2. 수정하려면 새 V2 마이그레이션 작성
# 3. Git에서 apply된 파일 수정 감지 (PR review)
```

```bash
# ✅ 올바른 접근 2: .gitattributes로 줄바꿈 통일
# .gitattributes 파일을 프로젝트 루트에 생성
cat > .gitattributes << 'EOF'
# 모든 SQL 파일: LF로 통일
*.sql text eol=lf

# 모든 Java 파일: LF로 통일
*.java text eol=lf

# 기본: 자동 변환
* text=auto

# Windows 실행 파일
*.bat binary
*.exe binary

# 바이너리 파일
*.class binary
*.jar binary
*.png binary
*.jpg binary
EOF

# 이미 CRLF로 저장된 파일 수정
# Step 1: Git에서 CRLF 제거
$ git rm -rf .
$ git reset --hard

# Step 2: 자동 변환
$ git add .

# Step 3: 커밋
$ git commit -m "Normalize line endings"

# 이제 모든 팀원이 LF로 작업
```

```bash
# ✅ 올바른 접근 3: flyway repair의 안전한 사용
# repair는 체크섬을 재계산해서 flyway_schema_history 업데이트

# 안전한 경우:
# 1. 개발 환경에서 파일을 실수로 수정했을 때
# 2. clean-disabled를 사용 중일 때 (프로덕션 아님)
# 3. 파일 내용이 DB 스키마와 일치함을 확인했을 때

# 위험한 경우:
# 1. 프로덕션 환경
# 2. 파일이 정말 수정된 게 아닐 때 (CRLF, BOM 같은 메타 변경)
# 3. DB 스키마가 파일과 다를 가능성 있을 때

# 절대 금지:
$ flyway -cleanDisabled=false clean  # 프로덕션에서 절대!
$ flyway repair  # 원인 파악 전에 절대!

# 올바른 사용:
# Step 1: 원인 파악
git diff HEAD~1 src/main/resources/db/migration/V2__*.sql

# Step 2: 파일 복구
git checkout src/main/resources/db/migration/V2__*.sql

# Step 3: validate로 확인
flyway validate

# Step 4: 필요하면 migrate
flyway migrate

# 만약 진짜 파일을 수정했다면?
# V2를 복구하고, 새로운 V3 마이그레이션으로 수정사항 적용
```

```bash
# ✅ 올바른 접근 4: flyway validate를 CI/CD에 통합
# .github/workflows/validate-migrations.yml (GitHub Actions)
name: Validate Flyway Migrations

on:
  pull_request:
    paths:
      - 'src/main/resources/db/migration/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
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
      
      - name: Check for modified migration files
        run: |
          # 이미 적용된 마이그레이션 파일이 수정됐나?
          git diff origin/main..HEAD --name-only \
            | grep "src/main/resources/db/migration/V" \
            | while read file; do
              git diff origin/main HEAD -- "$file" | grep -q "^-" && {
                echo "ERROR: Applied migration file was modified: $file"
                exit 1
              }
            done
      
      - name: Run Flyway validation
        run: |
          mvn clean flyway:validate
```

```yaml
# ✅ 올바른 접근 5: GitLab CI 설정
stages:
  - test
  - validate

validate-migrations:
  stage: validate
  image: maven:3.8-jdk-11
  services:
    - mysql:8.0
      variables:
        MYSQL_ROOT_PASSWORD: root
        MYSQL_DATABASE: testdb
  script:
    # 마이그레이션 검증
    - mvn flyway:validate -Dflyway.url="jdbc:mysql://mysql:3306/testdb" -Dflyway.user=root -Dflyway.password=root
    
    # 마이그레이션 적용 (테스트용 DB)
    - mvn flyway:migrate -Dflyway.url="jdbc:mysql://mysql:3306/testdb" -Dflyway.user=root -Dflyway.password=root
  
  only:
    changes:
      - src/main/resources/db/migration/**
```

---

## 🔬 내부 동작 원리

### 1. CRC32 체크섬 계산 및 CRLF/BOM 영향

```java
// CRC32 체크섬 계산 원리
import java.util.zip.CRC32;
import java.nio.charset.StandardCharsets;

public class ChecksumCalculationDemo {
    
    public static void main(String[] args) {
        String sql = "CREATE TABLE users (\n" +
                     "  id INT PRIMARY KEY\n" +
                     ");";
        
        // 정상: LF (Unix 줄바꿈)
        int checksum1 = calculateChecksum(sql);
        System.out.println("LF:   " + checksum1);  // 1234567890
        
        // 줄바꿈 변경: CRLF (Windows 줄바꿈)
        String sqlCRLF = sql.replace("\n", "\r\n");
        int checksum2 = calculateChecksum(sqlCRLF);
        System.out.println("CRLF: " + checksum2);  // 다른 값!
        
        // BOM 포함 (UTF-8 with BOM)
        byte[] bomBytes = new byte[] {(byte)0xEF, (byte)0xBB, (byte)0xBF};
        String sqlWithBOM = new String(bomBytes) + sql;
        int checksum3 = calculateChecksum(sqlWithBOM);
        System.out.println("BOM:  " + checksum3);  // 또 다른 값!
        
        // 결과:
        // checksum1 ≠ checksum2 ≠ checksum3
        // 모두 다름!
    }
    
    private static int calculateChecksum(String content) {
        CRC32 crc = new CRC32();
        // 중요: 인코딩도 영향을 줌
        crc.update(content.getBytes(StandardCharsets.UTF_8));
        return (int) crc.getValue();
    }
}
```

```
CRC32 불일치의 원인:

┌─────────────────────────────────────────────────────────┐
│ 1. CRLF vs LF (줄바꿈)                                  │
├─────────────────────────────────────────────────────────┤
│ 파일 내용 (같아 보임):                                   │
│ CREATE TABLE users (                                   │
│   id INT PRIMARY KEY                                   │
│ );                                                    │
│                                                       │
│ 바이너리 (다름):                                        │
│ LF 버전:   ...41 42 43 0A 44 45 46...  (0A = \n)      │
│ CRLF 버전: ...41 42 43 0D 0A 44 45 46... (0D 0A = \r\n)
│                                                       │
│ CRC32 입력: 바이너리 데이터                             │
│ → 결과: 완전히 다른 체크섬!                             │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ 2. BOM (Byte Order Mark)                               │
├─────────────────────────────────────────────────────────┤
│ UTF-8 without BOM:                                     │
│ 43 52 45 41 54 45 20 54 41 42 4C 45... (파일 시작)     │
│                                                       │
│ UTF-8 with BOM:                                       │
│ EF BB BF 43 52 45 41 54 45 20 54 41 42 4C 45...      │
│ (BOM 3바이트 추가!)                                     │
│                                                       │
│ CRC32 입력: 맨 앞에 EF BB BF 추가                      │
│ → 결과: 완전히 다른 체크섬!                             │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ 3. 공백/탭 변경                                          │
├─────────────────────────────────────────────────────────┤
│ CREATE TABLE users (           ← 4개 공백               │
│ CREATE TABLE users (     ← 2개 공백 또는 탭               │
│                                                       │
│ 보기에는 같지만 바이너리는 다름:                         │
│ 공백: 20 20 20 20                                     │
│ 탭:  09 09                                             │
│                                                       │
│ → 체크섬 불일치!                                      │
└─────────────────────────────────────────────────────────┘
```

### 2. flyway repair의 동작 원리

```java
// Flyway repair 명령어의 동작
public class FlywayRepair {
    
    public void repair() {
        // 1단계: 현재 마이그레이션 파일 읽기
        List<Migration> currentMigrations = loadMigrationFiles();
        
        // 2단계: 각 파일의 체크섬 재계산
        for (Migration m : currentMigrations) {
            int newChecksum = calculateChecksum(m.getContent());
            
            // 3단계: flyway_schema_history 업데이트
            // UPDATE flyway_schema_history 
            // SET checksum = newChecksum 
            // WHERE version = ?
            updateChecksum(m.getVersion(), newChecksum);
        }
    }
}

// 결과:
// Before repair:
// version | checksum | description
// 1       | 111111   | create table
// (파일 현재 체크섬: 222222 → 불일치!)
//
// After repair:
// version | checksum | description
// 1       | 222222   | create table
// (이제 일치!)

// 중요: repair는 데이터를 변경하지 않음!
// 단지 flyway_schema_history의 메타데이터만 업데이트
```

### 3. 체크섬 검증 과정

```java
public class ValidationProcess {
    
    public void validate() {
        // 1단계: flyway_schema_history 읽기
        List<HistoryRecord> records = readSchemaHistory();
        
        // 2단계: 마이그레이션 파일 스캔
        List<Migration> files = scanMigrationFiles();
        
        // 3단계: 각 파일별 검증
        for (Migration file : files) {
            // 이미 적용된 마이그레이션인가?
            HistoryRecord record = findRecord(file.getVersion());
            
            if (record == null) {
                // 적용 안 됨 → OK (새로운 마이그레이션)
                continue;
            }
            
            // 4단계: 체크섬 비교
            int currentChecksum = calculateChecksum(file.getContent());
            if (currentChecksum != record.checksum) {
                // 불일치!
                throw new ValidationFailedException(
                    "Migration checksum mismatch for version " + file.getVersion() +
                    "\n  Got: " + currentChecksum +
                    "\n  Expected: " + record.checksum);
            }
        }
        
        // 5단계: 파일이 삭제되지는 않았나?
        for (HistoryRecord record : records) {
            if (findFile(record.version) == null) {
                if (ignoreMissingMigrations) {
                    // 무시 (위험!)
                } else {
                    // 오류
                    throw new ValidationFailedException(
                        "Missing migration file for version " + record.version);
                }
            }
        }
    }
}
```

---

## 💻 실전 실험

### 실험 1: CRLF vs LF 체크섬 불일치 재현

```bash
# 파일 생성 (LF)
cat > db/migration/V1__create.sql << 'EOF'
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT
);
EOF

# 체크섬 계산
crc32 db/migration/V1__create.sql  # 예: 1234567890

# Flyway 실행
flyway migrate
# Successfully applied 1 migration

# 마이그레이션 상태 확인
mysql -uroot -proot mydb -e "SELECT version, checksum FROM flyway_schema_history;"
# version | checksum
# 1       | 1234567890

# 이제 파일을 CRLF로 변환 (Windows 줄바꿈)
unix2dos db/migration/V1__create.sql

# 또는
sed -i 's/$/\r/' db/migration/V1__create.sql

# 새로운 체크섬
crc32 db/migration/V1__create.sql  # 예: 9876543210 (다름!)

# Flyway validate 실행
flyway validate

# 출력:
# ERROR: Validate failed: Migrations have failed validation
# Migration checksum mismatch for migration version 1
#   Got:      9876543210
#   Expected: 1234567890
```

### 실험 2: .gitattributes로 CRLF 해결

```bash
# .gitattributes 생성
cat > .gitattributes << 'EOF'
*.sql text eol=lf
EOF

# 파일들이 현재 CRLF를 가지고 있음
file db/migration/*.sql
# db/migration/V1__create.sql: ASCII text, with CRLF line terminators

# Git 설정
git add .gitattributes
git commit -m "Add .gitattributes"

# 이미 있는 파일 정규화
git rm --cached -r .
git reset --hard

# 또는
git add -A
git commit -m "Normalize line endings"

# 확인
file db/migration/*.sql
# db/migration/V1__create.sql: ASCII text (CRLF 없음!)

# 이제 모든 팀원이 LF로 작업
```

### 실험 3: BOM 제거

```bash
# BOM 포함 파일 확인
hexdump -C db/migration/V1__create.sql | head
# 00000000  ef bb bf 43 52 45 41 54 45 20 54 41 42 4c 45...
#           ↑ BOM!

# BOM 제거 (sed)
sed -i '1s/^\xef\xbb\xbf//' db/migration/V1__create.sql

# 또는 (vim에서)
# :set nobomb
# :wq

# 확인
hexdump -C db/migration/V1__create.sql | head
# 00000000  43 52 45 41 54 45 20 54 41 42 4c 45...
# (BOM 없음!)
```

### 실험 4: flyway repair 안전하게 사용

```bash
# 현재 상태: 체크섬 불일치
flyway validate
# ERROR: Migration checksum mismatch

# Step 1: 원인 파악
git diff HEAD db/migration/V1__create.sql
# - ; (세미콜론 제거됨)

# Step 2: 파일 복구
git checkout db/migration/V1__create.sql

# Step 3: 검증
flyway validate
# Successfully validated 1 migration

# Step 4: 복구 불필요 (이미 파일 복구됨)

# 만약 파일이 정말 수정된 거라면?
# (예: 주석 추가, 포맷팅 변경 등)
# 그리고 파일 내용과 DB 스키마가 일치한다면?

flyway repair
# repair 실행
# (체크섬만 업데이트)

# 확인
flyway validate
# Successfully validated 1 migration
```

### 실험 5: CI/CD에서 검증

```bash
# GitHub Actions에서 실행
git push origin my-feature

# .github/workflows/validate.yml이 자동 실행:
# 1. 변경된 마이그레이션 파일 확인
# 2. 이미 적용된 파일이 수정됐나 확인
# 3. Flyway validate 실행
# 4. 실패하면 PR check 실패

# PR 화면에서:
# ❌ Validate Flyway Migrations
#    Applied migration V1__create.sql was modified
#    Run: git checkout db/migration/V1__create.sql
```

---

## 📊 성능/비용 비교

```
┌────────────────────────────────────────────────────────────────┐
│ 체크섬 검증 성능                                                │
├──────────────────┬──────────┬─────────────────────────────────┤
│ 작업              │ 시간     │ 특징                            │
├──────────────────┼──────────┼─────────────────────────────────┤
│ 파일 1개 체크섬   │ 1ms      │ CRC32 계산                      │
│ 파일 10개 체크섬  │ 10ms     │ O(n) 선형                       │
│ 파일 100개 체크섐 │ 100ms    │ 문제 없음                       │
│ 파일 1000개       │ 1000ms   │ 시작 시간 증가 주의             │
│ validate 전체     │ 50-100ms │ DB 쿼리 포함                    │
└──────────────────┴──────────┴─────────────────────────────────┘

repair 성능:
┌────────────────────────────────────────────────────────────────┐
│ 마이그레이션 수 │ repair 시간 │ 특징                           │
├─────────────────┼─────────────┼────────────────────────────────┤
│ 10개           │ 50ms        │ 각 파일 재계산                  │
│ 100개          │ 100ms       │ DB 업데이트                    │
│ 1000개         │ 1000ms      │ 느림                           │
└─────────────────┴─────────────┴────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
┌────────────────────────────────────────────────────────────────┐
│ validate-on-migrate 활성화                                    │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 모든 문제 미리 감지                            │
│              │ - 부분 적용 방지                                │
│              │ - 파일 수정 감지                                │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 시작 시간 증가 (10%~20%)                      │
│              │ - 마이그레이션 파일 많으면 느림                 │
│              │ - 개발 루프 시간 증가                           │
└──────────────┴─────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ .gitattributes로 줄바꿈 통일                                   │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 팀 전체 일관성                                │
│              │ - CRLF 문제 근본 해결                           │
│              │ - 한 번 설정으로 영구 해결                       │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 기존 파일들 재정규화 필요                      │
│              │ - Git 인덱스 변경 필요                          │
│              │ - 한 번 변경하면 git blame 스팸                 │
└──────────────┴─────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ flyway repair 사용                                              │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 빠른 복구 (메타데이터만 변경)                  │
│              │ - 파일 내용과 DB 일치하면 안전                 │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 잘못 사용하면 치명적 (데이터 불일치)          │
│              │ - 무작정 실행하면 위험                          │
│              │ - 운영진의 신뢰 손상 가능성                     │
└──────────────┴─────────────────────────────────────────────────┘
```

---

## 📌 핵심 정리

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 체크섬 불일치 원인 3가지                                    │
│    - 파일 내용 수정 (코드 변경)                                 │
│    - CRLF/LF 줄바꿈 차이 (OS별 다름)                           │
│    - BOM(Byte Order Mark) 포함 (인코딩)                        │
│                                                                 │
│ 2. 체크섬은 파일의 "지문"                                      │
│    - 1비트라도 다르면 다른 값                                   │
│    - CRC32로 계산 (매우 빠름)                                   │
│    - 프로덕션 스키마 보호의 핵심                                 │
│                                                                 │
│ 3. 적용된 마이그레이션은 절대 수정하지 말 것                    │
│    - V1이 적용되면 V1__*.sql 수정 금지                         │
│    - 수정하려면 새 V2 마이그레이션 작성                         │
│    - Git에서 파일 변경 감지 (PR review)                        │
│                                                                 │
│ 4. .gitattributes로 줄바꿈 통일                                │
│    - *.sql text eol=lf (모든 SQL 파일)                        │
│    - 팀 전체 일관성 확보                                       │
│    - CRLF 문제 근본 해결                                       │
│                                                                 │
│ 5. flyway repair는 신중하게                                   │
│    - 파일 내용과 DB 스키마 일치 확인 후 사용                    │
│    - 원인 파악 → 파일 복구 → validate 순서                     │
│    - 무작정 repair 사용 금지!                                  │
│                                                                 │
│ 6. CI/CD에서 자동 검증                                         │
│    - PR merge 전 flyway validate 실행                         │
│    - 적용된 파일 수정 감지                                     │
│    - 팀의 규칙 강제                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🤔 생각해볼 문제

**Q1.** 체크섬 불일치를 일으킬 수 있는 5가지 경우를 나열하고, 각각을 예방하는 방법은?

<details>
<summary>해설 보기</summary>

**5가지 원인과 예방:**

```
1. 파일 내용 수정 (의도적 또는 실수)
   원인: 세미콜론 제거, 주석 추가, 인덴트 변경 등
   예방: 
   - 적용된 파일은 read-only 권한 설정
   - Git pre-commit hook으로 체크
   - PR review에서 "applied migration 수정" 감지
   
   git hook (pre-commit):
   #!/bin/bash
   git diff --cached --name-only | grep "db/migration/V" | while read file; do
     # 이미 커밋된 파일인지 확인
     git log --oneline -- "$file" | head -1 | grep -q "." && {
       echo "ERROR: You're modifying an applied migration: $file"
       exit 1
     }
   done

2. CRLF/LF 줄바꿈 차이
   원인: Windows vs Linux 개발자
   예방:
   - .gitattributes 파일 생성: *.sql text eol=lf
   - Git 설정 통일: git config core.safecrlf true
   - EditorConfig: *.sql end_of_line = lf

3. BOM(Byte Order Mark) 포함
   원인: Windows 메모장 UTF-8 with BOM 저장
   예방:
   - IDE 설정 (VS Code): "[sql]": "files.encoding": "utf8"
   - 메모장 사용 금지 (VS Code, Sublime Text 등 권장)
   - EditorConfig: charset = utf-8-bom 금지

4. 화이트스페이스 변경 (공백 ↔ 탭)
   원인: IDE 자동 포맷팅
   예방:
   - EditorConfig: indent_style = space, indent_size = 2
   - Prettier/Spotless 자동 포맷팅
   - Git diff --check (trailing whitespace 감지)

5. 파일 인코딩 변경 (UTF-8 ↔ UTF-16)
   원인: IDE 설정 실수
   예방:
   - EditorConfig: charset = utf-8 (항상)
   - IDE 설정 강제: .idea/codeStyles/Project.xml
   - CI/CD 검증: file-type encoding 확인
```

**최종 방지:**

```yaml
# EditorConfig (.editorconfig)
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.sql]
indent_style = space
indent_size = 2

[*.java]
indent_style = space
indent_size = 4
```

</details>

---

**Q2.** `git diff` 명령어로 이미 적용된 V1 마이그레이션이 수정되었는지 감지하는 방법은?

<details>
<summary>해설 보기</summary>

**방법 1: 직접 diff 확인**

```bash
# V1이 포함된 모든 커밋 확인
git log --oneline -- src/main/resources/db/migration/V1__*.sql

# 출력:
# a1b2c3d V1 applied to production (1 month ago)
# 9f8e7d6 Initial commit of V1

# 이제부터의 모든 변경 확인
git diff a1b2c3d..HEAD -- src/main/resources/db/migration/V1__*.sql

# 출력이 있으면 = V1이 수정됨!
git diff a1b2c3d..HEAD -- src/main/resources/db/migration/V1__*.sql | head -20
# +
# +-- Added comment (수정됨!)
# ...
```

**방법 2: Pre-commit Hook으로 자동 감지**

```bash
# .git/hooks/pre-commit
#!/bin/bash

# 준비 커밋된 파일 중 마이그레이션 찾기
git diff --cached --name-only | grep "db/migration/V" | while read file; do
    # 이미 적용되었나? (origin/main에 있나?)
    git log --oneline origin/main -- "$file" | grep -q "." && {
        if git diff --cached -- "$file" | grep -q "^[-+]"; then
            echo "ERROR: Applied migration modified: $file"
            echo "Do not modify applied migrations!"
            exit 1
        fi
    }
done

exit 0
```

**방법 3: GitHub Actions에서 자동 검증**

```yaml
# .github/workflows/check-migrations.yml
name: Check Applied Migrations

on: [pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # 전체 히스토리 필요
      
      - name: Check for modified applied migrations
        run: |
          # main에 있는 마이그레이션 파일들
          APPLIED=$(git diff origin/main...HEAD --name-only \
                    | grep "db/migration/V")
          
          if [ -n "$APPLIED" ]; then
            echo "Modified migration files (should not happen):"
            echo "$APPLIED"
            
            # 내용이 정말 변경됐나?
            git diff origin/main...HEAD -- $APPLIED | grep -q "^[-+]" && {
              echo "ERROR: Applied migrations were modified!"
              exit 1
            }
          fi
```

**출력 예:**

```
$ git diff HEAD~2 src/main/resources/db/migration/V1__create.sql

diff --git a/src/main/resources/db/migration/V1__create.sql 
index abc123..def456 100644
--- a/src/main/resources/db/migration/V1__create.sql
+++ b/src/main/resources/db/migration/V1__create.sql
@@ -1,6 +1,7 @@
+-- Initial schema creation
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(100)
);

(세미콜론 삭제됨 또는 주석 추가됨이 감지됨!)
```

</details>

---

**Q3.** 로컬에서는 체크섬 검증이 통과하는데, CI/CD에서만 실패하는 경우는 왜 발생하고 해결책은?

<details>
<summary>해설 보기</summary>

**원인: 환경별 CRLF/LF 설정 다름**

```
로컬 (Windows):
$ git config core.autocrlf false
→ 파일이 CRLF로 저장됨
→ 체크섬: 111111

CI/CD (Linux):
$ git config core.autocrlf input (또는 true)
→ 파일이 LF로 변환됨
→ 체크섬: 222222

결과:
- 로컬 validate: PASS (111111 == 111111)
- CI validate: FAIL (111111 != 222222)
```

**해결책:**

```bash
# Step 1: .gitattributes 생성
cat > .gitattributes << 'EOF'
# 모든 SQL 파일: LF 통일
*.sql text eol=lf
# 모든 Java 파일: LF 통일
*.java text eol=lf
EOF

# Step 2: 기존 파일 정규화
git rm -rf . --cached  # 인덱스에서 제거
git reset --hard       # 작업 디렉토리 초기화
git add .              # 다시 추가 (자동 변환됨)
git commit -m "Normalize line endings (CRLF to LF)"

# Step 3: 모든 팀원 pull
# .gitattributes가 적용됨
# 로컬과 CI의 체크섬이 이제 일치

# Step 4: 로컬 git 설정 (옵션)
git config core.safecrlf true
# (실수로 CRLF를 커밋하려고 하면 경고)
```

**검증:**

```bash
# 로컬에서 확인
file db/migration/V1__create.sql
# V1__create.sql: ASCII text

# (CRLF 없음!)

# 체크섬 일치 확인
crc32 db/migration/V1__create.sql  # 111111

# CI에서도 같은 값:
# (Git에서 자동으로 LF 적용, 체크섬 동일)

# Flyway validate 실행
flyway validate
# 로컬: PASS
# CI:  PASS (이제 둘 다 통과!)
```

**예방 규칙:**

```
1. 프로젝트 시작 시 .gitattributes 설정
2. 모든 마이그레이션 파일: text eol=lf
3. CI/CD 설정:
   - Linux 환경 권장
   - git config core.safecrlf true
4. 개발 가이드:
   - "항상 LF 사용"
   - IDE 설정: .editorconfig
5. 정기 점검:
   - file db/migration/*.sql | grep -i crlf (없어야 함)
```

</details>

---

<div align="center">

**[⬅️ 이전: Flyway Callbacks](./05-flyway-callbacks.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — DDL이 Lock을 거는 원리 ➡️](../zero-downtime-migration/01-ddl-lock-internals.md)**

</div>
