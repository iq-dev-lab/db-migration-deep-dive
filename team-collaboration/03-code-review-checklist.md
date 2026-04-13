# 마이그레이션 코드 리뷰 체크리스트

---

## 🎯 핵심 질문

마이그레이션 PR을 리뷰할 때 어떤 항목을 반드시 확인해야 할까요? 어떤 위험한 DDL 패턴을 감지하고, 자동화로 어떻게 보완할까요?

## 🔍 왜 이 개념이 실무에서 중요한가

마이그레이션은 **일단 프로덕션에 배포되면 되돌리기 매우 어렵습니다**. 따라서 코드 리뷰가 배포 전 **마지막 안전장치**입니다. 프로덕션 데이터베이스 잠금, 데이터 손실, 성능 저하 같은 심각한 문제를 미리 차단해야 합니다. 리뷰 자동화와 수동 검토의 조합이 필수입니다.

---

## 😱 흔한 실수 (Before)

### 시나리오 1: DDL Lock으로 인한 서비스 중단

**상황:** 10억 건의 사용자 데이터가 있는 `users` 테이블

```sql
-- ❌ 위험한 마이그레이션
ALTER TABLE users ADD COLUMN status VARCHAR(50) NOT NULL;
```

**문제:**

```
MySQL에서 발생하는 순서:

1. ALTER TABLE 시작
   → users 테이블에 배타적 잠금 (exclusive lock) 설정

2. 전체 테이블 복사 (10억 건)
   → 시간: 5-10분
   → 이 동안 SELECT도 차단됨!

3. 복사 완료 후 원본 테이블과 교체
   → 원자적으로 이름 변경

4. 원본 테이블 삭제

결과:
┌─────────────────────────────────────┐
│ 5-10분간 users 테이블 접근 불가    │
│ → 앱에 timeout, 에러 발생           │
│ → 매출 손실, 알림 발생              │
└─────────────────────────────────────┘
```

**어떻게 감지할까?**

```sql
-- ❌ 감지 가능한 위험 패턴
ALTER TABLE {large_table} ADD COLUMN {non_nullable_column};
-- 또는
ALTER TABLE {large_table} MODIFY COLUMN ...;
-- 또는
ALTER TABLE {large_table} CHANGE COLUMN ...;
```

### 시나리오 2: NOT NULL 컬럼 추가 시 데이터 손실

```sql
-- ❌ 위험한 마이그레이션
ALTER TABLE orders ADD COLUMN discount_percent INT NOT NULL;
```

**문제:**

```
1. 새 컬럼 추가, DEFAULT 없음
   ↓
2. 기존 행들에 대해 값을 설정해야 함
   ↓
3. MySQL의 기본 동작: NULL이 아닌 기본값 설정
   ↓
4. INT 기본값은 0
   → 모든 주문이 0% 할인이 됨!
```

**실제 영향:**

```
Orders 테이블:
id | amount | discount_percent
1  | 100    | 0    (실제로는 값이 없어야 함)
2  | 200    | 0    (실제로는 값이 없어야 함)
3  | 150    | 0    (실제로는 값이 없어야 함)

재무 팀: "왜 할인이 100% 적용된 거야?"
→ 데이터 무결성 문제 발생
```

### 시나리오 3: 배치 처리 없이 대량 UPDATE

```sql
-- ❌ 위험한 마이그레이션 (10억 건 업데이트)
UPDATE users SET status = 'ACTIVE' WHERE created_at < '2024-01-01';
```

**문제:**

```
1. 단일 쿼리로 모든 행을 메모리에 로드 시도
   ↓
2. 메모리 부족 (OOM)
   ↓
3. 트랜잭션 롤백
   ↓
4. 마이그레이션 실패 (배포 실패)

또한:
- Replication lag 발생 (MySQL이 바이너리 로그 기록)
- Replica 따라잡기 위해 마이그레이션 대기
- 전체 배포 지연
```

### 시나리오 4: 하위 호환성 무시

```sql
-- ❌ 호환성 문제
ALTER TABLE users DROP COLUMN phone_number;
```

**배포 순서:**

```
현황: App v1.0 (phone_number 컬럼 사용), DB v0

배포 순서:
1️⃣ 마이그레이션 적용 (phone_number 제거)
   → DB v1 (컬럼 없음)

2️⃣ App v1.1 배포 (phone_number 미사용)
   → 배포 중...

문제:
- 마이그레이션 완료 후 App 배포 전
- App v1.0 인스턴스가 여전히 실행 중
- phone_number 읽으려고 함
- → 컬럼 없음 에러!
```

### 시나리오 5: 인덱스 누락이나 중복

```sql
-- 마이그레이션 1
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255)
);

-- 마이그레이션 2
CREATE INDEX idx_users_email ON users(email);

-- 마이그레이션 3
CREATE INDEX idx_email ON users(email);  -- 중복!
```

**문제:**

```
1. 같은 인덱스가 두 번 생성됨
2. 불필요한 스토리지 사용
3. INSERT/UPDATE 성능 저하
4. 메모리 낭비

조회: "왜 같은 인덱스가 두 개야?"
     "누가 이렇게 했어?"
     → 리뷰 실패
```

---

## ✨ 올바른 접근 (After)

### 마이그레이션 코드 리뷰 체크리스트 (수동)

PR 리뷰 시 다음 항목을 순서대로 확인:

#### 1. 기본 정보 확인

- [ ] **버전 번호가 순차적인가?**
  ```sql
  -- ✅ 올바름
  V10__20240415_143000__Add_users.sql
  
  -- ❌ 건너뜀
  V10, V12 (V11 누락)
  ```

- [ ] **파일명이 명확한가?**
  ```sql
  -- ✅ 명확함
  V10__20240415_143000__Add_users_table.sql
  
  -- ❌ 애매함
  V10__Updates.sql
  ```

- [ ] **타임스탬프가 정렬 가능한가?**
  ```
  ✅ V10__20240415_143000__...
  ❌ V10__04-15-2024_14-30__...
  ```

#### 2. DDL Lock 위험 감지

**대용량 테이블 확인:**

```bash
# DB 접속해서 크기 확인
mysql> SELECT table_name, round(((data_length + index_length) / 1024 / 1024), 2) AS size_mb
       FROM information_schema.tables
       WHERE table_schema = 'production'
       ORDER BY size_mb DESC;

+-------------------+-----------+
| table_name        | size_mb   |
+-------------------+-----------+
| users             | 5000      | ← 5GB (위험!)
| orders            | 3000      | ← 3GB (위험!)
| order_items       | 2000      | ← 2GB (위험!)
+-------------------+-----------+
```

**위험한 패턴 감지 (마이그레이션 내용):**

```sql
-- ❌ 대용량 테이블에 NOT NULL 컬럼 추가
ALTER TABLE users ADD COLUMN status VARCHAR(50) NOT NULL DEFAULT 'ACTIVE';

-- ❌ 대용량 테이블 MODIFY (타입 변경)
ALTER TABLE orders MODIFY amount DECIMAL(12, 2);

-- ❌ 대용량 테이블 컬럼 추가 (기본값 없음)
ALTER TABLE users ADD COLUMN middle_name VARCHAR(100);

-- ✅ 안전한 방법: DEFAULT 값과 함께 추가
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'ACTIVE';
```

#### 3. 데이터 손실 위험 확인

**파괴적인 작업 감지:**

```sql
-- ❌ 데이터 손실 위험
DROP TABLE old_users;
TRUNCATE TABLE users;
ALTER TABLE users DROP COLUMN phone;
DELETE FROM users WHERE status = 'INACTIVE';

-- 리뷰: "왜 삭제해야 하나? 백업 계획이 있나?"
```

**확인 사항:**

- [ ] DROP COLUMN이 정말 필요한가?
  - 대안: 새 버전 앱에서 무시하고, 나중에 제거
- [ ] TRUNCATE/DELETE의 목적?
  - 데이터 정제 필요한가? 공식 기록이 있는가?
- [ ] 백업 계획이 있는가?
  - 실수했을 때 복구 방법?

#### 4. 배치 처리 확인

```sql
-- ❌ 위험: 단일 쿼리로 대량 처리
UPDATE users SET status = 'ACTIVE' WHERE created_at < '2024-01-01';

-- ✅ 안전: 배치 처리
-- Flyway에서는 불가능하므로, 아래처럼 작성
UPDATE users SET status = 'ACTIVE' 
WHERE created_at < '2024-01-01' LIMIT 1000;
-- (필요시 여러 번 실행)

-- 또는 배치 마이그레이션 (Spring)으로 처리
-- db/migration/V10__20240415_143000__Add_status_column.sql
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'PENDING';

-- src/main/java/migration/BatchMigration.java
@Bean
public CommandLineRunner batchMigration(UserRepository repo) {
    return args -> {
        int batchSize = 1000;
        int page = 0;
        Page<User> users;
        do {
            users = repo.findAll(PageRequest.of(page++, batchSize));
            users.forEach(u -> u.setStatus("ACTIVE"));
            repo.saveAll(users);
        } while (users.hasNext());
    };
}
```

#### 5. 인덱스 검증

**확인 사항:**

- [ ] 필요한 인덱스가 모두 있는가?
  ```sql
  -- 포인트: WHERE, JOIN, ORDER BY에 사용되는 컬럼
  
  -- ✅ 필요한 인덱스
  SELECT * FROM orders WHERE user_id = ? AND status = ?;
  → CREATE INDEX idx_orders_user_status ON orders(user_id, status);
  
  -- ❌ 불필요한 인덱스
  CREATE INDEX idx_created_at ON orders(created_at);
  → 거의 쓰이지 않으면서 INSERT/UPDATE 성능 저하
  ```

- [ ] 중복 인덱스가 없는가?
  ```sql
  -- ❌ 중복
  CREATE INDEX idx_email ON users(email);
  CREATE INDEX idx_users_email ON users(email);
  
  -- ✅ 정리
  CREATE INDEX idx_users_email ON users(email);  -- 하나만
  ```

- [ ] 복합 인덱스 순서는 적절한가?
  ```sql
  -- ❌ 잘못된 순서
  CREATE INDEX idx ON orders(status, user_id);
  SELECT * FROM orders WHERE user_id = ?;  -- user_id 인덱스 미사용
  
  -- ✅ 올바른 순서
  CREATE INDEX idx ON orders(user_id, status);
  SELECT * FROM orders WHERE user_id = ?;  -- user_id 인덱스 사용!
  ```

#### 6. 하위 호환성 확인

```
현재 배포 상황:
App v1.0 (production) → DB v1 (production)
App v1.1 (staging)    → DB v2 (feature 브랜치)

마이그레이션 리뷰:
V2: DROP COLUMN phone_number;

문제:
1️⃣ DB를 v1 → v2로 마이그레이션
2️⃣ App v1.0이 여전히 실행 중 (phone_number 읽음)
3️⃣ 컬럼 없음 → 에러!
```

**체크리스트:**

- [ ] 기존 앱 버전에서 사용하는 컬럼/테이블을 제거하는가?
  ```sql
  -- 수행 전 확인
  SELECT COUNT(*) FROM information_schema.columns 
  WHERE table_name = 'users' AND column_name = 'phone_number';
  
  -- 앱 코드에서 phone_number 사용 여부 확인
  grep -r "phone_number" src/
  ```

- [ ] 다른 버전 앱과 호환성이 있는가?

#### 7. 성능 영향 분석

```sql
-- 마이그레이션 전 분석
EXPLAIN SELECT * FROM orders WHERE user_id = ? AND status = ?;

-- 마이그레이션 후 인덱스 추가 확인
SHOW INDEX FROM orders;
```

**리뷰 항목:**

- [ ] 새 인덱스로 인한 INSERT/UPDATE 성능 저하는 수용 가능한가?
- [ ] 대용량 데이터 마이그레이션이 예상 시간 내에 완료되는가?

---

## 🔬 내부 동작 원리

### 1. MySQL의 ALTER TABLE 알고리즘

MySQL 8.0에서 지원하는 3가지 알고리즘:

```sql
-- 알고리즘 1: COPY (가장 느림, 하지만 호환성 좋음)
ALTER TABLE users ADD COLUMN status VARCHAR(50), ALGORITHM=COPY;

동작:
1. 원본 테이블 전체 복사
2. 새 컬럼 추가
3. 인덱스 재구성
4. 원본과 교체

장점: 거의 모든 작업 가능
단점: 시간 오래 걸림, 테이블 잠금

-- 알고리즘 2: INPLACE (중간, MySQL 5.7+)
ALTER TABLE users ADD COLUMN status VARCHAR(50), ALGORITHM=INPLACE;

동작:
1. 메모리에서 변경
2. 원본 테이블 업데이트

장점: COPY보다 빠름
단점: 모든 작업 지원 안 함

-- 알고리즘 3: INSTANT (가장 빠름, MySQL 8.0.12+)
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'ACTIVE', ALGORITHM=INSTANT;

동작:
1. 메타데이터만 변경
2. 데이터 복사 안 함

장점: 거의 즉시 완료!
단점: 제한된 작업만 가능 (컬럼 추가 후 DEFAULT)
```

**Flyway에서 알고리즘 지정:**

```sql
-- 명시적으로 INPLACE 사용
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'ACTIVE', ALGORITHM=INPLACE, LOCK=NONE;

-- ❌ 지정하지 않으면
ALTER TABLE users ADD COLUMN status VARCHAR(50);
-- MySQL 옵션 기본값에 따라 결정됨 (버전마다 다름)
```

### 2. DDL Lock의 메커니즘

```
Timeline:
──────────────────────────────────────────────────────

시점 T1: ALTER TABLE 시작
         ├─ Metadata lock 획득 (배타적)
         ├─ 테이블 복사 시작

시점 T2~T5: 복사 진행 (5분)
         ├─ SELECT 시도 → BLOCKED (metadata lock 때문)
         ├─ INSERT 시도 → BLOCKED
         ├─ UPDATE 시도 → BLOCKED

시점 T6: 복사 완료
         ├─ 원본 테이블과 교체
         ├─ Lock 해제

결과:
┌────────────────────────────────┐
│ T1 ~ T6 동안 테이블 접근 불가  │
│ 이 시간 = 데이터베이스 다운     │
└────────────────────────────────┘
```

### 3. 체크섬 검증 메커니즘

Flyway는 다음 정보로 마이그레이션을 추적:

```sql
SELECT * FROM flyway_schema_history;

+-------+--------+----------------------------------+--------+
| id    | type   | script                           | checksum |
+-------+--------+----------------------------------+--------+
| 1     | SQL    | V1__Initial.sql                 | 12a3b4c5|
| 2     | SQL    | V2__Add_column.sql              | 56d7e8f9|
+-------+--------+----------------------------------+--------+

-- 마이그레이션 파일 변경 감지
파일 내용 변경:
V2__Add_column.sql (이전) → checksum: 56d7e8f9
V2__Add_column.sql (현재) → checksum: 99z8y7x6 (다름!)

→ Flyway validate 실패
```

---

## 💻 실전 실험

### 실험 1: DDL Lock 시뮬레이션

```bash
# 터미널 1: MySQL 모니터링
mysql -h localhost -u root -p

# 터미널 2: 테스트 환경 설정
docker run -d --name mysql-test \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=testdb \
  mysql:8.0

# 테스트 데이터 준비
mysql -h localhost -u root -ppassword testdb << 'EOF'
CREATE TABLE large_table (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    data VARCHAR(255)
);

-- 100만 건 데이터 삽입
INSERT INTO large_table (data) 
SELECT CONCAT('row_', id) FROM 
(SELECT @a:=@a+1 as id FROM 
 (SELECT 0 UNION SELECT 1) t1,
 (SELECT 0 UNION SELECT 1) t2,
 -- ... 여러 번 반복해서 100만 건 생성
) x LIMIT 1000000;

COMMIT;
EOF
```

**정렬 상황 1: ALGORITHM=COPY (위험)**

```bash
# 터미널 1: 모니터링
WATCH "SHOW PROCESSLIST;"

# 터미널 2: 위험한 ALTER
mysql -h localhost -u root -ppassword testdb << 'EOF'
ALTER TABLE large_table ADD COLUMN status VARCHAR(50), ALGORITHM=COPY;
EOF

# 터미널 3: 동시에 SELECT 시도
mysql -h localhost -u root -ppassword testdb << 'EOF'
SELECT COUNT(*) FROM large_table;  -- ← BLOCKED!
EOF

# 결과:
# ┌──────────────────────────────────────┐
# │ 10초 ~ 30초 대기 (COPY 진행 중)    │
# │ Query timeout 발생 가능             │
# └──────────────────────────────────────┘
```

**정렬 상황 2: ALGORITHM=INSTANT (권장)**

```bash
# 데이터 초기화
ALTER TABLE large_table DROP COLUMN status;

# 터미널 2: 안전한 ALTER (즉시 완료)
mysql -h localhost -u root -ppassword testdb << 'EOF'
ALTER TABLE large_table 
ADD COLUMN status VARCHAR(50) DEFAULT 'ACTIVE', 
ALGORITHM=INSTANT;
EOF

# 결과:
# ┌──────────────────────────────────────┐
# │ 거의 즉시 완료! (<100ms)            │
# │ SELECT 차단 없음                     │
# └──────────────────────────────────────┘
```

### 실험 2: 자동화 검사 도구 구현

**sqlfluff를 이용한 SQL 린팅:**

```bash
# 설치
pip install sqlfluff

# 마이그레이션 파일 검사
sqlfluff lint db/migration/V10__Add_users.sql

# 결과:
# ┌────────────────────────────────────────────┐
# │ L009: Unnecessary whitespace found        │
# │ L010: Unused alias                        │
# │ (마이그레이션 구조 검사는 별도)            │
# └────────────────────────────────────────────┘
```

**커스텀 검사 스크립트 (Bash):**

```bash
#!/bin/bash
# migration-checker.sh

FILE=$1
LARGE_TABLES=("users" "orders" "order_items" "payments")

# 1. 대용량 테이블에 NOT NULL 컬럼 추가 감지
for table in "${LARGE_TABLES[@]}"; do
    if grep -q "ALTER TABLE $table ADD COLUMN.*NOT NULL" "$FILE"; then
        echo "❌ ERROR: $table에 NOT NULL 컬럼 추가"
        echo "   권장: DEFAULT 값과 함께 추가"
        exit 1
    fi
done

# 2. DROP TABLE/COLUMN 감지
if grep -q "^DROP TABLE\|^ALTER TABLE.*DROP COLUMN" "$FILE"; then
    echo "⚠️  WARNING: DROP 작업 감지"
    echo "   정말 필요한가? 백업 계획이 있는가?"
    exit 2
fi

# 3. 배치 없는 대량 UPDATE 감지
if grep -q "^UPDATE.*WHERE.*LIMIT" "$FILE"; then
    echo "✅ 배치 처리 감지"
else
    if grep -q "^UPDATE.*WHERE" "$FILE"; then
        echo "⚠️  WARNING: LIMIT 없는 UPDATE 감지"
        echo "   대량 업데이트 시 배치 처리 권장"
    fi
fi

# 4. 중복 인덱스 감지
INDEX_COUNT=$(grep -c "^CREATE INDEX" "$FILE")
if [ "$INDEX_COUNT" -gt 1 ]; then
    echo "⚠️  WARNING: 여러 인덱스 생성"
    grep "^CREATE INDEX" "$FILE"
fi

echo "✅ 검사 완료"
```

**사용:**

```bash
chmod +x migration-checker.sh
./migration-checker.sh db/migration/V10__Add_users.sql
```

**GitHub Actions로 자동화:**

```yaml
# .github/workflows/migration-check.yml
name: Migration Validation

on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # 전체 히스토리 필요
      
      - name: Check for migration files
        run: |
          if git diff origin/main HEAD --name-only | grep -q "db/migration/"; then
            echo "마이그레이션 파일 감지"
            
            # 1. Flyway 검증
            - name: Flyway validation
              run: |
                mvn flyway:validate \
                  -Dflyway.url=jdbc:mysql://localhost:3306/testdb \
                  -Dflyway.user=root \
                  -Dflyway.password=password
            
            # 2. SQL Lint
            - name: SQL Lint
              run: |
                pip install sqlfluff
                sqlfluff lint db/migration/
            
            # 3. 커스텀 검사
            - name: Custom checks
              run: |
                ./scripts/migration-checker.sh db/migration/
          fi
```

### 실험 3: 리뷰 체크리스트 검증

**리뷰 템플릿 (GitHub PR):**

```markdown
# 마이그레이션 PR 리뷰 체크리스트

## 필수 검사

### 기본 정보
- [ ] 버전 번호가 순차적인가? (V10 이후 V11?)
- [ ] 파일명이 명확한가? (버전__날짜_시간__설명)
- [ ] 타임스탐프 형식이 정렬 가능한가?

### DDL Lock 위험
- [ ] 대용량 테이블(>1GB)에 NOT NULL 컬럼 추가하는가?
  - [ ] YES → DEFAULT 값 포함했는가?
  - [ ] NO → 테이블 크기 확인
- [ ] ALGORITHM=INSTANT 또는 =INPLACE 지정했는가?

### 데이터 손실 위험
- [ ] DROP COLUMN 또는 DROP TABLE이 있는가?
  - [ ] YES → 정말 필요한가? 백업 계획이 있는가?
  - [ ] NO → 계속

### 배치 처리
- [ ] 대량 UPDATE (>100,000건)가 있는가?
  - [ ] YES → LIMIT 또는 WHERE 조건으로 배치 처리되는가?
  - [ ] NO → 계속

### 인덱스
- [ ] 새 인덱스가 추가되는가?
  - [ ] YES → 쿼리에서 실제 사용되는가? (EXPLAIN)
  - [ ] NO → 계속
- [ ] 중복 인덱스가 있는가?

### 호환성
- [ ] 기존 앱(v1.0)에서 사용하는 컬럼/테이블을 삭제하는가?
  - [ ] YES → 위험! 배포 순서를 재검토해야 함
  - [ ] NO → 계속

## 승인 조건

- [ ] 모든 필수 검사 통과
- [ ] CI/CD 검증 통과 (flyway validate, SQL lint)
- [ ] 팀 리뷰 완료
```

---

## 📊 성능/비용 비교

| 리뷰 방법 | 감지율 | 소요시간 | 자동화 | 비용 |
|---------|------|---------|-------|------|
| **수동 리뷰만** | 70% | 15분/PR | 없음 | 높음 |
| **Flyway validate** | 30% | 1분 | 자동 | 낮음 |
| **SQL lint** | 40% | 2분 | 자동 | 낮음 |
| **커스텀 검사** | 60% | 3분 | 자동 | 중간 |
| **수동 + 자동 조합** | 95% | 10분 | 혼합 | 중간 |

**권장:** 수동 리뷰 + GitHub Actions 자동화

---

## ⚖️ 트레이드오프

### 1. 보안 vs 개발 속도

```
매우 엄격한 리뷰
├─ 장점: 거의 모든 문제 포착
├─ 단점: 리뷰 시간 30분 이상
└─ 결과: 배포 지연 (병목)

자동화만 의존
├─ 장점: 빠른 피드백 (1분)
├─ 단점: 놓칠 수 있는 문제 (40%)
└─ 결과: 프로덕션 이슈 증가

균형:
자동화 (5분) + 빠른 수동 리뷰 (5분) = 10분
감지율: 95% + 배포 지연 최소
```

### 2. 정확성 vs 유연성

```
엄격한 정책: "DROP 명령 절대 금지"
├─ 유연성 낮음
├─ 하지만 데이터 손실 사고 zero

유연한 정책: "필요하면 DROP 허용"
├─ 유연성 높음
├─ 하지만 사고 위험 증가

권장:
DROP은 드물게, 하지만 필요시 명확한 이유와 함께 검토
```

---

## 📌 핵심 정리

1. **마이그레이션은 배포 후 되돌리기 어려움** → 리뷰가 마지막 안전장치
2. **DDL Lock은 서비스 중단 초래** → 대용량 테이블에 ALGORITHM=INSTANT 사용
3. **NOT NULL 컬럼 추가 시 DEFAULT 필수** → 기존 행 보호
4. **대량 데이터 처리는 배치로** → 메모리 부족 방지
5. **하위 호환성 확인** → 앱 배포 순서 고려
6. **자동화 도구 활용** → GitHub Actions + sqlfluff + 커스텀 검사
7. **체크리스트 사용** → 일관된 리뷰 기준

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 이미 프로덕션에 배포된 마이그레이션에 문제가 있으면 어떻게 할까요?</strong></summary>

**시나리오:**
```
V10__Add_users_status.sql (이미 배포됨)

내용:
ALTER TABLE users ADD COLUMN status VARCHAR(50);
-- ❌ 문제: 기존 1억 건 행에 NULL이 들어감
```

**대응책:**

```sql
-- 1️⃣ 즉시 조치 (롤백은 피함)
-- 새로운 마이그레이션으로 보정

-- V11__20240415_160000__Fix_users_status_null_values.sql
UPDATE users SET status = 'ACTIVE' WHERE status IS NULL;

-- 또는 상황에 따라
ALTER TABLE users MODIFY COLUMN status VARCHAR(50) NOT NULL DEFAULT 'ACTIVE';

-- 2️⃣ 장기 조치
-- 왜 이런 일이 발생했는가?
-- → 리뷰 프로세스 개선
-- → 자동화 검사 추가
```

**교훈:** 문제가 발생한 후에 수정하는 것보다, 리뷰로 미리 방지하는 것이 훨씬 저렴.

</details>

<details>
<summary><strong>Q2: 마이그레이션 리뷰를 위해 얼마나 많은 DB 지식이 필요한가요?</strong></summary>

**기본 필수 지식:**

```
1. SQL 문법 (ALTER, CREATE, INDEX)
   ↓
2. 인덱스의 목적 (WHERE, JOIN, ORDER BY에 사용)
   ↓
3. DDL Lock의 개념 (대용량 테이블 위험)
   ↓
4. 데이터 타입 (VARCHAR vs INT vs DECIMAL)
   ↓
5. 제약 조건 (NOT NULL, UNIQUE, FOREIGN KEY)
```

**학습 경로:**

```
개발자 중급 레벨이면 충분함.

- 1주: SQL 기본 문법 (CREATE, ALTER)
- 1주: 인덱스와 성능 (EXPLAIN 읽기)
- 1주: 마이그레이션 위험 패턴 (10가지)
- 1주: 자동화 도구 (Flyway, sqlfluff)

→ 합계: 4주 학습으로 효과적인 리뷰 가능
```

**도움이 될 리소스:**

- MySQL 공식 문서: ALTER TABLE, Index 섹션
- 실습: 테스트 DB에서 EXPLAIN 분석
- 코드리뷰: 먼저 받은 후 피드백 관찰

</details>

<details>
<summary><strong>Q3: 런타임에 마이그레이션이 실패하면 자동으로 롤백할 수 있나요?</strong></summary>

**상황:**

```
배포 중:
App v1.1 배포
  ↓
마이그레이션 V10 시작
  ↓
중간에 실패 (예: 외래키 위반)
  ↓
???자동 롤백???
```

**Flyway의 동작:**

```sql
-- 마이그레이션 실패 시
SELECT * FROM flyway_schema_history 
WHERE version = 10;

+--------+--------+---------+--------+
| version| type   | status  |checksum|
+--------+--------+---------+--------+
| 10     | SQL    | FAILED  | ...    |
+--------+--------+---------+--------+

-- ✅ 상태: FAILED (자동 롤백 안 함!)
```

**원인:**

```
Flyway는 원자적(Atomic) 마이그레이션을 보장하지 않음

실패 원인별:
1. SQL 에러 → 해당 SQL만 롤백, 이전 SQL은 커밋됨
2. 트랜잭션 (--mode=auto) → 전체 롤백 가능
```

**해결책:**

```bash
# 1. 마이그레이션을 트랜잭션으로 감싸기 (선택사항)
-- db/migration/V10__Add_column.sql
BEGIN TRANSACTION;
ALTER TABLE users ADD COLUMN status VARCHAR(50);
CREATE INDEX idx_status ON users(status);
COMMIT;

# 2. 실패 후 수동 확인
mvn flyway:validate
# 상태 확인

# 3. 수동 수정 후 재실행
# 또는 new migration
cat > db/migration/V11__Fix_failed_migration.sql << EOF
-- V10 실패 이유 분석 후 작성
-- 예: 외래키 제약 추가
ALTER TABLE users ADD CONSTRAINT fk_... FOREIGN KEY ...;
EOF

mvn flyway:migrate
```

**교훈:** 자동 롤백에 의존하지 말고, 처음부터 실패 없는 마이그레이션 작성하기 (좋은 리뷰로 가능).

</details>

---

<div align="center">

**[⬅️ 이전: 브랜치 전략과 마이그레이션](./02-branch-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: 멀티 모듈 마이그레이션 ➡️](./04-multi-module-migration.md)**

</div>
