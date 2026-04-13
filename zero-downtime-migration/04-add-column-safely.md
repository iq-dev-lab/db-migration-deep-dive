# 안전한 컬럼 추가

---

## 🎯 핵심 질문

- `NOT NULL` 컬럼을 DEFAULT 없이 추가하면 왜 실패하는가?
- `NOT NULL` + `DEFAULT`를 추가할 때 MySQL 8.0과 그 이전의 차이는 무엇인가?
- 대용량 테이블에서 안전한 컬럼 추가의 최적 절차는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

컬럼 추가는 가장 빈번한 스키마 변경입니다. 개발 단계에서는:
```sql
ALTER TABLE users ADD COLUMN full_name VARCHAR(100) NOT NULL DEFAULT '';
```

이 한 줄이 대용량 프로덕션 DB에서는 수십 분을 소요할 수 있습니다. 또는 실패할 수 있습니다.

정확한 메커니즘을 알면:
- 어떤 추가 방식이 INSTANT인지 예측 가능
- 다단계 마이그레이션 계획 수립
- 배포 시간 예측

---

## 😱 흔한 실수 (Before — 무분별한 NOT NULL 추가)

```sql
-- Before: 프로덕션에서 한 번에 NOT NULL 추가
ALTER TABLE orders 
ADD COLUMN tracking_number VARCHAR(50) NOT NULL;
-- Error 1138: Invalid use of NULL in column definition

-- 왜 실패?
-- 1. 기존 1000만 행이 모두 NULL 상태
-- 2. NOT NULL 제약을 만족시킬 방법이 없음
-- 3. 어떤 기본값을 사용할지 정의되지 않음

-- Before 시도 2: DEFAULT를 지정하지만 COPY 방식 선택됨
ALTER TABLE orders 
ADD COLUMN tracking_number VARCHAR(50) NOT NULL DEFAULT 'PENDING';
-- ALGORITHM=COPY 선택 (MySQL 5.7)
-- 실행: 30분 (1000만 행, 배타적 Lock)
-- 결과: 서비스 중단, SLA 위반
```

**문제점**:
1. 1000만 행에 대해 DEFAULT 값 적용 (재구성 필요)
2. EXCLUSIVE Lock 발생
3. INSERT/UPDATE/DELETE 모두 차단

---

## ✨ 올바른 접근 (After — 3단계 안전 추가)

```sql
-- After: 3단계로 안전하게 추가 (Expand-Contract 변형)

-- 1단계: NULL 컬럼 추가 (INSTANT, 밀리초)
ALTER TABLE orders 
ADD COLUMN tracking_number VARCHAR(50) NULL,
ALGORITHM=INSTANT;
-- 실행 시간: < 100ms, Lock: NONE
-- 기존 1000만 행: tracking_number = NULL (메타데이터만)

-- 2단계: 배치 백필 (동시 DML 허용)
-- 배치 크기: 10000행씩, 5분마다 실행
UPDATE orders 
SET tracking_number = 'PENDING' 
WHERE tracking_number IS NULL 
LIMIT 10000;
-- 반복 (확인 쿼리로 NULL이 0이 될 때까지):
-- SELECT COUNT(*) FROM orders WHERE tracking_number IS NULL;

-- 3단계: NOT NULL 제약 추가 (INSTANT, 메타데이터만)
ALTER TABLE orders 
MODIFY COLUMN tracking_number VARCHAR(50) NOT NULL DEFAULT 'PENDING',
ALGORITHM=INSTANT;
-- 실행 시간: < 100ms, Lock: NONE
-- 이제 모든 행이 NULL이 아니므로 안전
```

**결과**: 무중단 추가, 총 소요 시간 ~15분 (배치 처리) + 메타데이터 시간 무시

---

## 🔬 내부 동작 원리

### 1. INSTANT와 INPLACE 결정 조건

```
NOT NULL 컬럼 추가 시 결정 트리:

┌─ DEFAULT 값이 있는가?
│  ├─ YES
│  │  ├─ MySQL 8.0.29+인가?
│  │  │  ├─ YES → ALGORITHM=INSTANT (권장)
│  │  │  │  └─ 메타데이터만 변경
│  │  │  │  └─ 실행: 밀리초
│  │  │  │  └─ Lock: NONE
│  │  │  │
│  │  │  └─ NO → ALGORITHM=INPLACE
│  │  │     └─ 전체 테이블 재구성
│  │  │     └─ 실행: 분~시간
│  │  │     └─ Lock: SHARED (읽기만)
│  │  │
│  │  └─ 특수 경우: 구지정된 DEFAULT 값
│  │     ├─ 컬럼 기본값이 상수: INSTANT (8.0.29+)
│  │     ├─ 컬럼 기본값이 함수: INPLACE
│  │     │  (예: DEFAULT CURRENT_TIMESTAMP, DEFAULT UUID())
│  │     └─ 컬럼 기본값이 이전 컬럼: INPLACE
│  │        (예: DEFAULT (another_column))
│  │
│  └─ NO (DEFAULT 없음)
│     ├─ NOT NULL인가?
│     │  └─ YES → Error! 기본값 필수
│     │
│     └─ NULL 허용?
│        └─ YES → ALGORITHM=INSTANT
│           └─ 메타데이터만
│           └─ 실행: 밀리초
│           └─ Lock: NONE
```

### 2. MySQL 버전별 동작

#### MySQL 5.7 (INSTANT 미지원)

```sql
-- NOT NULL + DEFAULT 추가
ALTER TABLE orders 
ADD COLUMN tracking_number VARCHAR(50) NOT NULL DEFAULT 'PENDING';

내부 동작:
1. 임시 파일 생성 (_orders_new.ibd)
2. 모든 행 읽기 (1000만 행)
3. 각 행에 tracking_number = 'PENDING' 추가
4. 정렬 및 인덱스 재구성
5. 기존 테이블 삭제, 임시 파일로 교체
6. SHARED Lock 발생 (읽기만 가능, 쓰기 차단)

Timeline:
┌──────────────────────────┐
│ T0: ALTER 시작           │
│     MDL 대기 중          │
├──────────────────────────┤
│ T1-T30: 재구성           │
│ (모든 행 처리)           │
│ SELECT 가능 (느림)        │
│ INSERT/UPDATE 불가      │
├──────────────────────────┤
│ T31: Lock 해제           │
│ 완료                     │
└──────────────────────────┘

소요 시간: 30분 (1000만 행, 50바이트 평균)
```

#### MySQL 8.0.13~8.0.28 (부분 INSTANT)

```sql
-- NULL 컬럼 추가: INSTANT
ALTER TABLE orders 
ADD COLUMN notes VARCHAR(255) NULL;
-- INSTANT (메타데이터만)

-- NOT NULL + DEFAULT: INPLACE (여전히 느림)
ALTER TABLE orders 
ADD COLUMN tracking_number VARCHAR(50) NOT NULL DEFAULT 'PENDING';
-- INPLACE (전체 행 재구성)
-- 소요 시간: 20분 (약간 더 빠름)

내부 최적화:
- 임시 파일은 NEW 형식 (InnoDB Native)
- 병렬 처리 개선
- 하지만 여전히 모든 행을 터치해야 함
```

#### MySQL 8.0.29+ (완전 INSTANT)

```sql
-- NOT NULL + DEFAULT: INSTANT!
ALTER TABLE orders 
ADD COLUMN tracking_number VARCHAR(50) NOT NULL DEFAULT 'PENDING';
-- INSTANT (메타데이터만)

내부 동작:
1. 메타데이터에 새 컬럼 정의 추가
   ├─ 컬럼명: tracking_number
   ├─ 타입: VARCHAR(50)
   ├─ NOT NULL: YES
   ├─ DEFAULT: 'PENDING'
   └─ INSTANT: YES (플래그)
   
2. 기존 행 물리 저장소: 변경 없음
   ├─ 기존 레이아웃 유지 (tracking_number 필드 없음)
   └─ SELECT 시 런타임에 DEFAULT 'PENDING' 자동 삽입

3. 새로운 INSERT/UPDATE
   ├─ tracking_number 값 제공: 저장소에 저장
   └─ tracking_number 값 미제공: NULL이 아닌 'PENDING' 저장

Timeline:
┌──────────────────┐
│ T0: ALTER 시작   │
│ MDL 획득         │
├──────────────────┤
│ T0-T1: 메타 변경 │
│ (밀리초)         │
├──────────────────┤
│ T1: Lock 해제    │
│ 완료!            │
└──────────────────┘

소요 시간: < 100ms
Lock 시간: 밀리초 (감지 불가)
```

### 3. 배치 백필 메커니즘

```sql
-- 상황: NULL 컬럼 1000만 행, PENDING으로 변경해야 함

-- 나쁜 예: 한 번에 UPDATE (전체 Lock)
UPDATE orders SET tracking_number = 'PENDING';
-- 1000만 행 업데이트, Lock 발생, INSERT/UPDATE/DELETE 모두 차단
-- 시간: 5분, 그동안 서비스 영향

-- 좋은 예: 배치 백필 (청크 단위)
-- Iteration 1:
UPDATE orders 
SET tracking_number = 'PENDING' 
WHERE tracking_number IS NULL 
LIMIT 10000;
-- 처리: 10000행, Lock 시간: 몇 초
-- 나머지: 990만 행

-- Iteration 2 (5분 후):
UPDATE orders 
SET tracking_number = 'PENDING' 
WHERE tracking_number IS NULL 
LIMIT 10000;
-- 처리: 10000행

-- ... (Iteration 1000까지 반복)

-- 모니터링:
SELECT 
  COUNT(*) as total,
  SUM(IF(tracking_number IS NULL, 1, 0)) as pending_count,
  ROUND(100 * SUM(IF(tracking_number IS NOT NULL, 1, 0)) / COUNT(*), 1) as pct_complete
FROM orders;
```

**배치 크기 결정**:
```
배치 크기 = 평균 행 크기 × 쿼리 시간 한계

예시:
- 평균 행 크기: 1000바이트
- 쿼리 시간 한계: 1초 (Lock 시간 <= 1초)
- 디스크 처리 속도: 100MB/s

배치 크기 = (100MB/s × 1초) / 1000바이트
          = 100,000행

실제: 10000~50000행 권장 (I/O + Lock 시간 고려)
```

---

## 💻 실전 실험

### 실험 1: INSTANT vs INPLACE 비교

```bash
# Docker MySQL 8.0 시작
docker run -d \
  --name mysql80_test \
  -e MYSQL_ROOT_PASSWORD=root \
  -p 3306:3306 \
  mysql:8.0 \
  --default-storage-engine=InnoDB

# 접속
mysql -h 127.0.0.1 -u root -proot -e "CREATE DATABASE testdb;"
```

```sql
USE testdb;

-- 테스트 테이블 생성 (1000만 행)
CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT,
  amount DECIMAL(10,2),
  status VARCHAR(20),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user (user_id),
  INDEX idx_created (created_at)
);

-- 데이터 삽입
INSERT INTO orders (user_id, amount, status)
WITH RECURSIVE nums AS (
  SELECT 1 as n
  UNION ALL
  SELECT n+1 FROM nums WHERE n < 10000000
)
SELECT 
  FLOOR(RAND()*100000) + 1 as user_id,
  ROUND(RAND()*10000, 2) as amount,
  'completed' as status
FROM nums;

-- 확인
SELECT COUNT(*) FROM orders;  -- 10000000

-- 실험 1: NULL 컬럼 추가 (INSTANT)
SELECT NOW(6) as start_time;
ALTER TABLE orders 
ADD COLUMN notes VARCHAR(255) NULL,
ALGORITHM=INSTANT;
SELECT NOW(6) as end_time;
-- 예상: 밀리초 단위

-- 확인: 메타데이터에만 추가됨
DESCRIBE orders;  -- notes 컬럼 보임
SELECT notes FROM orders LIMIT 1;  -- NULL

-- 실험 2: NOT NULL + DEFAULT 추가 (MySQL 8.0.29+면 INSTANT, 아니면 INPLACE)
-- 먼저 버전 확인
SELECT VERSION();  -- mysql-8.0.35 이상이면 INSTANT

SELECT NOW(6) as start_time;
ALTER TABLE orders 
ADD COLUMN tracking_number VARCHAR(50) NOT NULL DEFAULT 'PENDING',
ALGORITHM=INSTANT;  -- 명시적으로 INSTANT 시도
SELECT NOW(6) as end_time;

-- MySQL 8.0.29 미만이면:
-- Error: ALGORITHM=INSTANT is not supported for this operation
-- 이 경우 INPLACE가 자동 선택됨 (시간 소요)

-- 실험 3: MODIFY로 NOT NULL 추가 (기존 DEFAULT 값)
-- (사전에 기본값이 있는 컬럼이 필요)
SELECT NOW(6) as start_time;
ALTER TABLE orders 
MODIFY COLUMN status VARCHAR(20) NOT NULL DEFAULT 'pending',
ALGORITHM=INSTANT;
SELECT NOW(6) as end_time;
-- 실행 시간: 밀리초 (기존 status는 모두 값이 있으므로 안전)
```

### 실험 2: 배치 백필 자동화

```bash
#!/bin/bash
# batch-backfill.sh

DB_HOST="127.0.0.1"
DB_USER="root"
DB_PASS="root"
DB_NAME="testdb"
TABLE="orders"
COLUMN="tracking_number"
BATCH_SIZE=100000
INTERVAL=10  # 초

# 백필 시작 전 상태 확인
echo "백필 전 상태:"
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME << EOF
SELECT 
  COUNT(*) as total_rows,
  SUM(IF($COLUMN IS NULL, 1, 0)) as null_count
FROM $TABLE;
EOF

# 배치 반복
iteration=0
while true; do
  iteration=$((iteration + 1))
  
  # 현재 NULL 개수 확인
  null_count=$(mysql -h $DB_HOST -u $DB_USER -p$DB_PASS -se \
    "SELECT COUNT(*) FROM $DB_NAME.$TABLE WHERE $COLUMN IS NULL;")
  
  if [ "$null_count" -eq 0 ]; then
    echo "백필 완료! (총 $((iteration-1)) iteration)"
    break
  fi
  
  echo "Iteration $iteration: NULL $null_count행 처리 중..."
  
  # 배치 UPDATE
  mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME << EOF
UPDATE $TABLE 
SET $COLUMN = 'PENDING' 
WHERE $COLUMN IS NULL 
LIMIT $BATCH_SIZE;
EOF
  
  # 진행률 출력
  total=$(mysql -h $DB_HOST -u $DB_USER -p$DB_PASS -se \
    "SELECT COUNT(*) FROM $DB_NAME.$TABLE;")
  pct=$((100 * (total - null_count) / total))
  echo "  진행률: ${pct}%"
  
  # 다음 배치까지 대기
  sleep $INTERVAL
done

echo "최종 상태:"
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME << EOF
SELECT 
  COUNT(*) as total_rows,
  SUM(IF($COLUMN IS NOT NULL, 1, 0)) as non_null_count,
  ROUND(100 * SUM(IF($COLUMN IS NOT NULL, 1, 0)) / COUNT(*), 2) as pct_complete
FROM $TABLE;
EOF
```

### 실험 3: MODIFY로 NOT NULL 제약 추가

```sql
-- 사전 조건: tracking_number가 모두 NULL이 아님 (배치 완료)
-- NULL 개수 확인
SELECT COUNT(*) FROM orders WHERE tracking_number IS NULL;
-- 결과: 0

-- MODIFY로 NOT NULL 추가 (INSTANT)
SELECT NOW(6) as start_time;
ALTER TABLE orders 
MODIFY COLUMN tracking_number VARCHAR(50) NOT NULL DEFAULT 'PENDING',
ALGORITHM=INSTANT;
SELECT NOW(6) as end_time;
-- 실행 시간: 밀리초

-- 확인: NOT NULL 제약이 적용됨
-- 다음 쿼리가 오류 발생:
INSERT INTO orders (user_id, amount) VALUES (1, 100);
-- Error: Field 'tracking_number' doesn't have a default value
-- (tracking_number를 지정하지 않았으므로 오류)

-- 올바른 INSERT:
INSERT INTO orders (user_id, amount, tracking_number) 
VALUES (1, 100, 'PENDING');
-- 성공 (tracking_number 지정됨)
```

---

## 📊 성능/비용 비교

| 전략 | MySQL 버전 | Algorithm | 소요 시간 | Lock 시간 | DML 영향 |
|------|-----------|-----------|----------|---------|---------|
| NULL만 추가 | 모든 버전 | INSTANT | <100ms | 밀리초 | 없음 |
| NOT NULL+DEFAULT (3단계) | 모든 버전 | INSTANT+배치 | 15~30분 | 없음 | 없음 |
| NOT NULL+DEFAULT (동시) | 5.7 | COPY | 30~60분 | 분~시간 | 있음 |
| NOT NULL+DEFAULT (동시) | 8.0-8.0.28 | INPLACE | 15~30분 | 초~분 | 읽기만 |
| NOT NULL+DEFAULT (동시) | 8.0.29+ | INSTANT | <100ms | 밀리초 | 없음 |

---

## ⚖️ 트레이드오프

| 선택지 | 장점 | 단점 |
|--------|------|------|
| **3단계 (NULL→배치→NOT NULL)** | 모든 버전 지원, 무중단, 예측 가능 | 배포 횟수 증가, 복잡도 높음 |
| **동시 NOT NULL+DEFAULT** | 간단, 1회 배포 | 서비스 영향, MySQL 8.0.29+ 필수 |
| **응용층 기본값 처리** | DB 변경 불필요 | 코드 복잡도, 일관성 유지 어려움 |

---

## 📌 핵심 정리

1. **NOT NULL 컬럼 추가 실패**: 기본값 정의 필수
2. **3단계 안전 추가**:
   - 1단계: NULL 추가 (INSTANT, 밀리초)
   - 2단계: 배치 백필 (동시 DML 허용)
   - 3단계: NOT NULL 제약 (INSTANT, 밀리초)
3. **MySQL 8.0.29+ 특권**: NOT NULL+DEFAULT도 INSTANT 지원
4. **배치 크기**: 10000~50000행, 5분마다 실행 권장
5. **모니터링**: WHERE IS NULL 쿼리로 진행률 추적

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 배치 백필 중에 신규 INSERT가 발생하면, 그 행의 tracking_number는 NULL일까 아니면 'PENDING'일까?</strong></summary>

**답변**:

NOT NULL 제약이 아직 없으므로(3단계 아직 진행 중), DEFAULT 값이 없으면 NULL입니다.

```
Timeline:

배치 처리 중:
UPDATE orders SET tracking_number = 'PENDING'
WHERE tracking_number IS NULL LIMIT 100000;

이 시간에 신규 INSERT:
INSERT INTO orders (user_id, amount) VALUES (1, 100);
-- tracking_number를 지정하지 않음

결과:
- Column 정의: VARCHAR(50) NULL (NOT NULL 아님)
- Default: 없음 (아직 DEFAULT 'PENDING' 적용 안 됨)
- 실제 저장: NULL (explicit default 없으면 NULL)

해결:
1. INSERT 시 명시적 지정:
   INSERT INTO orders (user_id, amount, tracking_number)
   VALUES (1, 100, 'PENDING');

2. 또는 DEFAULT 값을 먼저 설정:
   ALTER TABLE orders 
   MODIFY COLUMN tracking_number VARCHAR(50) NULL DEFAULT 'PENDING'
   -- 3단계가 아닌 2.5단계로 진행
   -- 이후 배치 완료 시점에 NOT NULL 추가

3. 또는 Expand-Contract 패턴:
   - 앱 코드에서 tracking_number = 'PENDING' 명시
   - 배치는 기존 NULL만 처리
   - 신규 데이터는 앱이 보장
```

**권장**: 2.5단계 추가 (DEFAULT 사전 설정)
```sql
-- 1단계: NULL 컬럼 추가
ALTER TABLE orders ADD COLUMN tracking_number VARCHAR(50) NULL;

-- 2단계: DEFAULT 값 설정
ALTER TABLE orders 
MODIFY COLUMN tracking_number VARCHAR(50) NULL DEFAULT 'PENDING';
-- (이제 신규 INSERT가 명시하지 않으면 'PENDING')

-- 2.5단계: 배치 백필
UPDATE orders SET tracking_number = 'PENDING'
WHERE tracking_number IS NULL LIMIT 100000;

-- 3단계: NOT NULL 제약
ALTER TABLE orders 
MODIFY COLUMN tracking_number VARCHAR(50) NOT NULL DEFAULT 'PENDING';
```

</details>

<details>
<summary><strong>Q2: 배치 백필 중에 동일한 WHERE 조건의 SELECT를 실행하면 정확한 NULL 개수를 얻을 수 있는가?</strong></summary>

**답변**:

SELECT와 UPDATE의 시점 차이로 인해 정확한 개수를 보장할 수 없습니다.

```
Timeline:

T1: SELECT 실행
    SELECT COUNT(*) FROM orders WHERE tracking_number IS NULL;
    결과: 500000 (Snapshot A)
    
T2: UPDATE 실행 (배치 1)
    UPDATE orders SET tracking_number = 'PENDING'
    WHERE tracking_number IS NULL LIMIT 100000;
    처리: 100000행 UPDATE
    
T3: 새로운 INSERT (다른 연결)
    INSERT INTO orders (user_id, amount) VALUES (2, 200);
    tracking_number = NULL (DEFAULT 없음, 아직)
    
T4: SELECT 재실행
    SELECT COUNT(*) FROM orders WHERE tracking_number IS NULL;
    결과: 400001 (Snapshot B)
    └─ 기대: 500000 - 100000 = 400000
    └─ 실제: 400001 (신규 INSERT 때문에)

원인:
- SNAPSHOT 격리 수준에도 불구하고
- 배치 UPDATE와 신규 INSERT의 순서 보장 불가
- 모니터링이 완벽하지 않음

해결:
1. 신뢰도 높은 모니터링 (정확하지 않아도 진행률만 확인)
   SELECT COUNT(*) FROM orders WHERE tracking_number IS NULL;
   
2. 배치 시작 후 신규 레코드 차단 (좋은 방법 아님)
   SET GLOBAL read_only = ON;  # 읽기 전용 모드 (위험)
   
3. 배치 UPDATE와 동시 INSERT 무시 (권장)
   # 백필 100%는 100일 수 없음
   # 99.9% 정도면 충분
   # 나머지는 MIGRATE 단계에서 처리

4. 또는 배치 시작 후 DEFAULT 설정
   # 신규 INSERT는 기본값으로 처리
   # 기존 NULL만 배치로 처리
   ALTER TABLE orders 
   MODIFY COLUMN tracking_number DEFAULT 'PENDING';
```

</details>

<details>
<summary><strong>Q3: 대용량 테이블에서 배치 LIMIT 10000은 얼마나 시간이 걸리는가? 시간 추정 공식은?</strong></summary>

**답변**:

배치 UPDATE 시간은 I/O, 행 크기, 인덱스 수에 따라 결정됩니다.

```
시간 추정 공식:

배치 시간 (초) = (행 수 × 행 크기) / 스캔 속도 + 버퍼 플러시 시간

예시:
- 행 수: 10000
- 평균 행 크기: 500바이트
- 디스크 스캔 속도: 50MB/s (순차 읽기)
- WHERE 조건: tracking_number IS NULL (인덱스 없음)
  └─ 전체 테이블 스캔 필요

계산:
데이터량 = 10000행 × 500바이트= 5MB
스캔 시간 = 5MB / 50MB/s = 0.1초

UPDATE 적용 시간 = 0.1초
버퍼 플러시 = 0.1~0.5초 (InnoDB 동작)

총 시간 = 0.1 + 0.5 = 0.6초

실제 예시 (Monitoring):

배치 크기별 실행 시간:
- LIMIT 10000:  ~1초  (컨택트 스캔)
- LIMIT 50000:  ~4초  (대량 스캔)
- LIMIT 100000: ~8초  (메모리 부족, 디스크 I/O 증가)

배치 크기 결정 기준:
배치 시간이 1~5초 범위 유지 권장
(이 시간 동안 Lock이 발생하고, 다른 쿼리 대기)

1000만 행 → 1000배치 필요
총 시간 = 1000 × 1초 = 1000초 ≈ 17분

반복 간격: 배치 시간 + 대기 시간
예) 1초 배치 + 4초 대기 = 5초 간격
(다음 배치 시작 전에 1행 INSERT라도 처리 가능)

최적화:
1. 인덱스 활용 (WHERE 조건)
   CREATE INDEX idx_tracking_null 
   ON orders((tracking_number IS NULL));
   # MySQL 8.0.13+부터 가능 (생성된 컬럼)
   # 스캔 속도 10배 향상 가능

2. Parallel 배치 (MySQL 8.0+)
   # 다중 스레드로 배치 병렬 처리
   # 하지만 Lock 경합 주의

3. gh-ost 사용
   # MySQL Online DDL 도구로 배치 자동화
   # 내부적으로 청크 단위로 처리하고 Lock 최소화
```

</details>

---

<div align="center">

**[⬅️ 이전: Expand-Contract 패턴](./03-expand-contract-pattern.md)** | **[홈으로 🏠](../README.md)** | **[다음: 컬럼 이름 변경 ➡️](./05-rename-column.md)**

</div>
