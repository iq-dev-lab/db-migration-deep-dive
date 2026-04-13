# 인덱스 추가 — 대용량 테이블 전략

---

## 🎯 핵심 질문

- `CREATE INDEX`가 느린 이유는 무엇인가?
- MySQL Online DDL과 PostgreSQL의 `CONCURRENT` 인덱싱 차이는 무엇인가?
- 대용량 테이블(1억 행)에서 인덱스 추가의 최적 전략은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

인덱스 추가는:
- 조회 성능을 극적으로 향상 (1000배)
- 하지만 추가 과정이 느리고 시스템 부하 크게 증가
- 다른 DDL과 달리 쓰기 차단이 없는 Lock 수준이어도 성능 영향 큼

언제 어떤 인덱스를 추가할지 몰라서, 무분별한 인덱스 추가 또는 부족한 인덱스로 성능 문제가 발생합니다.

---

## 😱 흔한 실수 (Before — 인덱스 추가 전 검증 없음)

```sql
-- Before: 쿼리를 느리게 하는 인덱스를 무분별하게 추가
-- 원인: 성능 분석 부재, EXPLAIN 미사용

-- 성능 문제 발생
SELECT users.id, users.name, orders.id, orders.amount
FROM users
JOIN orders ON users.id = orders.user_id
WHERE orders.created_at > DATE_SUB(NOW(), INTERVAL 30 DAY);

-- 느린 쿼리 로그 분석 없이 추측으로 인덱스 추가
CREATE INDEX idx_created_at ON orders(created_at);
-- 그냥 기다리고 있는 동안 인덱스 빌드... 20분
-- 읽기: 가능하지만 지연
-- 쓰기: 변경 버퍼에 누적

-- 더 나은 인덱스를 몰라서 여러 개 추가
CREATE INDEX idx_user_created ON orders(user_id, created_at);
CREATE INDEX idx_amount ON orders(amount);
-- 불필요한 인덱스 3개 추가 → 저장소 낭비, INSERT 속도 저하
```

**결과**: 불필요한 인덱스, 저장소 낭비, 쓰기 성능 저하

---

## ✨ 올바른 접근 (After — EXPLAIN 분석 후 전략적 인덱스)

```sql
-- After: EXPLAIN 분석 후 필요한 인덱스만 추가

-- 1단계: 느린 쿼리 분석
EXPLAIN SELECT users.id, users.name, orders.id, orders.amount
FROM users
JOIN orders ON users.id = orders.user_id
WHERE orders.created_at > DATE_SUB(NOW(), INTERVAL 30 DAY);

-- 분석 결과:
-- ├─ users 테이블: index 사용 (id PRIMARY KEY)
-- ├─ orders 테이블: Full Table Scan (WHERE created_at에 인덱스 없음)
-- └─ 개선: orders.created_at에 인덱스 필요

-- 2단계: 복합 인덱스 설계 (created_at + user_id)
-- 이유: WHERE created_at와 JOIN user_id를 모두 만족
CREATE INDEX idx_orders_created_user 
ON orders(created_at, user_id);
-- ALGORITHM=INPLACE, LOCK=NONE

-- 3단계: EXPLAIN 다시 확인
EXPLAIN SELECT users.id, users.name, orders.id, orders.amount
FROM users
JOIN orders ON users.id = orders.user_id
WHERE orders.created_at > DATE_SUB(NOW(), INTERVAL 30 DAY);

-- 이제 INDEX RANGE SCAN 사용 (훨씬 빠름)
```

**결과**: 전략적 인덱스, 저장소 효율, 읽기-쓰기 균형

---

## 🔬 내부 동작 원리

### 1. CREATE INDEX의 실행 단계

```
Timeline:

T0: CREATE INDEX 시작
    메타데이터 Lock 획득 (EXCLUSIVE, 짧음)

T1: 인덱스 빌드 준비
    임시 인덱스 버퍼 할당
    메타데이터 Lock 해제 → 읽기 가능!

T2-T10: 인덱스 빌드 (주요 시간 소비)
    ├─ 전체 테이블 스캔 (1억 행)
    ├─ 각 행의 인덱스 컬럼 추출
    ├─ 정렬 (B+Tree 구조에 맞춰)
    ├─ 인덱스 블록 할당 및 저장
    │
    └─ 동시에 DML 발생하면:
       └─ 변경 버퍼(Change Buffer)에 누적
           ├─ INSERT: 버퍼에 기록
           ├─ UPDATE: 버퍼에 기록
           └─ DELETE: 버퍼에 기록
           
           (인덱스 빌드 완료 후 모두 merge)

T11: 변경 버퍼 merge
     빌드 완료된 인덱스와 버퍼 변경 병합
     시간: 몇 초~1분 (버퍼 크기 따라)

T12: 메타데이터 Lock 재획득 (EXCLUSIVE, 짧음)
     인덱스를 테이블에 등록

T13: 메타데이트 Lock 해제
     완료, INSERT/UPDATE/DELETE 즉시 영향

Timeline 다이어그램:

┌──────────────────────────────────────────────┐
│ T0-T1: 메타데이터 Lock 획득 및 해제         │
│        (밀리초)                              │
│                                             │
│ T1-T10: 인덱스 빌드                         │
│ SELECT ✓ (느림)                             │
│ INSERT/UPDATE/DELETE ✓ (변경 버퍼)          │
│                                             │
│ T11: 변경 버퍼 merge                        │
│ 쓰기 성능 저하 (버퍼 처리 중)               │
│                                             │
│ T12-T13: 최종 메타데이터 Lock              │
│ (밀리초, 무시할 수준)                       │
└──────────────────────────────────────────────┘
```

### 2. Lock 수준 분석

```
CREATE INDEX 시 Lock:
┌──────────────────────────────────────┐
│ ALGORITHM=INPLACE, LOCK=NONE         │
│                                      │
│ 읽기: ✓ (모든 SELECT 가능)            │
│ 쓰기: ✓ (모든 INSERT/UPDATE/DELETE) │
│ (하지만 성능 저하 가능)               │
└──────────────────────────────────────┘

실제 영향:
- LOCK 수준: NONE (명시적 Lock 없음)
- 실제 영향: 
  ├─ SELECT: 빌드 중 느림 (부분 인덱스 사용)
  ├─ INSERT/UPDATE/DELETE: 변경 버퍼 오버헤드
  │  └─ 일반적으로 5~30% 느려짐
  └─ 전체 처리량: 20~50% 저하 (변경 버퍼 크기에 따라)

Lock-free vs Performance:
Lock이 없으므로 기술적으로 모든 DML 가능
하지만 변경 버퍼 오버헤드로 실제 성능 저하
→ "다운타임 없음"과 "성능 영향 없음"은 다름
```

### 3. FULLTEXT와 SPATIAL 인덱스의 Lock

```
FULLTEXT INDEX:
CREATE FULLTEXT INDEX idx_content ON articles(content);
└─ ALGORITHM=INPLACE, LOCK=SHARED
   ├─ 읽기: ✓
   ├─ 쓰기: ✗ (INSERT/UPDATE/DELETE 차단)
   └─ 더 느림 (내용 분석 필요)

SPATIAL INDEX:
CREATE SPATIAL INDEX idx_location ON places(location);
└─ ALGORITHM=INPLACE, LOCK=SHARED
   ├─ 읽기: ✓
   ├─ 쓰기: ✗ (차단)
   └─ 더 느림

결론:
- 일반 INDEX: LOCK=NONE (쓰기 가능)
- FULLTEXT/SPATIAL: LOCK=SHARED (쓰기 차단)
- 대용량 테이블에서 FULLTEXT는 위험
```

### 4. PostgreSQL과의 비교

```
PostgreSQL: CREATE INDEX CONCURRENTLY

CREATE INDEX CONCURRENTLY idx_orders_date 
ON orders(created_at);

특징:
1. 트랜잭션 격리 안 함 (다른 트랜잭션도 실행)
2. 2 단계 빌드:
   - Phase 1: 인덱스 빌드 (concurrent)
   - Phase 2: 기존 데이터와 병합 (잠깐 Lock)
3. 실패해도 안전 (INVALID 상태로 유지, 나중에 재시도)
4. DML 전혀 차단 안 함 (완전 Lock-free)
5. 시간이 더 걸림 (2배)

MySQL vs PostgreSQL:
┌─────────────────────────────────┬─────────────┬─────────────┐
│                                 │ MySQL       │ PostgreSQL  │
├─────────────────────────────────┼─────────────┼─────────────┤
│ LOCK 종류                       │ NONE        │ NONE        │
│ 읽기 가능                        │ ✓           │ ✓           │
│ 쓰기 가능                        │ ✓           │ ✓           │
│ 변경 버퍼 오버헤드               │ 있음        │ 없음        │
│ 실패 시 롤백                     │ 불가        │ INVALID     │
│ 소요 시간                        │ T           │ 2T          │
│ 구현 복잡도                      │ 단순        │ 복잡        │
└─────────────────────────────────┴─────────────┴─────────────┘
```

### 5. 복합 인덱스(Composite Index) 설계

```
쿼리:
SELECT * FROM orders 
WHERE user_id = 100 
  AND status = 'completed'
  AND created_at > '2024-01-01'
ORDER BY amount DESC;

인덱스 선택지:

Option 1: 단일 인덱스들
CREATE INDEX idx_user ON orders(user_id);
CREATE INDEX idx_status ON orders(status);
CREATE INDEX idx_created ON orders(created_at);
문제: 하나만 사용, 나머지는 filter
성능: 나쁨

Option 2: 복합 인덱스 (추천, ESR 규칙)
CREATE INDEX idx_orders_composite 
ON orders(user_id, status, created_at, amount);
         └─ E(Equality): user_id, status
         └─ S(Sort): created_at
         └─ R(Range): amount

쿼리 플랜:
1. 인덱스에서 user_id = 100 찾기 (Seek)
2. status = 'completed' 필터 (Filter)
3. created_at 순으로 정렬됨 (이미 정렬, Sort 불필요)
4. amount로 인덱스 스캔 (Index Range Scan)

성능: 매우 좋음

Option 3: 변형 (ORDER BY 컬럼 앞에 배치)
CREATE INDEX idx_orders_alt 
ON orders(user_id, status, amount, created_at);
문제: amount DESC (정렬 필요), created_at (Filter)
성능: 중간
```

---

## 💻 실전 실험

### 실험 1: EXPLAIN으로 인덱스 필요성 판단

```sql
USE testdb;

CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  status VARCHAR(20),
  amount DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 1000만 행 삽입 (시간이 걸림)
INSERT INTO orders (user_id, status, amount, created_at)
SELECT 
  FLOOR(RAND()*100000) + 1,
  ELT(FLOOR(RAND()*3) + 1, 'pending', 'completed', 'cancelled'),
  ROUND(RAND()*10000, 2),
  DATE_SUB(NOW(), INTERVAL FLOOR(RAND()*365) DAY)
FROM (
  SELECT 1 UNION SELECT 2 UNION SELECT 3
) t1
LIMIT 10000000;

-- 인덱스 없는 상태에서 EXPLAIN
EXPLAIN SELECT COUNT(*) FROM orders 
WHERE created_at > DATE_SUB(NOW(), INTERVAL 30 DAY);

-- Output:
-- type: ALL (Full Table Scan)
-- rows: 10000000 (1000만 행 모두 스캔)
-- Extra: Using where

-- 이제 인덱스 추가
SELECT NOW(6) as start_time;
CREATE INDEX idx_orders_created ON orders(created_at);
SELECT NOW(6) as end_time;

-- 인덱스 추가 후 EXPLAIN
EXPLAIN SELECT COUNT(*) FROM orders 
WHERE created_at > DATE_SUB(NOW(), INTERVAL 30 DAY);

-- Output:
-- type: RANGE (인덱스 범위 스캔)
-- rows: ~3000000 (30일 데이터 추정)
-- key: idx_orders_created

-- 성능 비교
-- Before: 전체 1000만 행 스캔
-- After: 약 300만 행만 스캔 (10배 향상)
```

### 실험 2: 복합 인덱스와 쿼리 플랜

```sql
-- 쿼리 분석
EXPLAIN SELECT id, amount FROM orders
WHERE user_id = 100 AND status = 'completed'
ORDER BY amount DESC LIMIT 10;

-- Before 인덱스:
-- type: ALL
-- rows: 10000000

-- 복합 인덱스 추가
CREATE INDEX idx_orders_user_status_amount 
ON orders(user_id, status, amount DESC);

-- After 인덱스:
EXPLAIN SELECT id, amount FROM orders
WHERE user_id = 100 AND status = 'completed'
ORDER BY amount DESC LIMIT 10;

-- Output:
-- type: RANGE
-- key: idx_orders_user_status_amount
-- rows: ~100 (user_id=100 레코드만)
-- Extra: Using index condition
```

### 실험 3: 인덱스 빌드 진행 상황 모니터링

```bash
#!/bin/bash
# 터미널 1: 인덱스 추가 (오래 걸림)
mysql -h 127.0.0.1 -u root -proot testdb << EOF
CREATE INDEX idx_orders_large ON orders(created_at, user_id);
EOF

# 터미널 2: 모니터링 (동시에 실행)
watch -n 2 'mysql -h 127.0.0.1 -u root -proot testdb << EOF
SHOW PROCESSLIST;
EOF
'

# 또는 InnoDB 상태 확인
mysql -h 127.0.0.1 -u root -proot testdb << EOF
SHOW ENGINE INNODB STATUS\G
-- Output에서 "online DDL" 섹션 확인
-- 처리한 행 수, 전체 행 수, 진행률 표시
EOF

# 인덱스 빌드 중 DML 성능 측정
time mysql -h 127.0.0.1 -u root -proot testdb << EOF
INSERT INTO orders (user_id, status, amount, created_at)
VALUES (1, 'pending', 100.0, NOW());
EOF

# 결과: 인덱스 빌드 중 ~30% 느려짐
```

### 실험 4: 대용량 테이블 인덱스 전략

```sql
-- 상황: orders 테이블 1억 건, 인덱스 추가 필요
-- 전략: 야간 + gh-ost (또는 MySQL Online DDL)

-- 방법 1: MySQL Online DDL (MySQL 8.0+)
-- 장점: 간단, 서드파티 도구 불필요
-- 단점: Lock-free이지만 성능 영향
CREATE INDEX idx_orders_date_user 
ON orders(created_at, user_id);
-- 소요 시간: 20~30분
-- DML 영향: 20~50% 느려짐

-- 방법 2: gh-ost (완전한 Lock-free)
-- 장점: 롤백 가능, 성능 영향 최소
-- 단점: 추가 도구 필요
-- $ gh-ost \
--   --user=root \
--   --password=password \
--   --host=prod-db.example.com \
--   --database=testdb \
--   --table=orders \
--   --alter="ADD INDEX idx_orders_date_user (created_at, user_id)" \
--   --execute

-- 방법 3: 복제본에 먼저 적용 (테스트)
-- 1. 복제본 MySQL에서 인덱스 추가
-- 2. 복제 지연 모니터링
-- 3. 마스터로 승격
-- (복제 아키텍처 필요)
```

---

## 📊 성능/비용 비교

| 전략 | 소요 시간 | DML 영향 | Lock | 복잡도 |
|------|----------|--------|------|--------|
| MySQL Online DDL | 20~30분 | 20~50% | NONE | 낮음 |
| gh-ost | 20~30분 | 무시할 수준 | NONE | 중간 |
| 야간 + COPY | 5~10분 | 0% (중단) | EXCLUSIVE | 낮음 |
| PostgreSQL CONCURRENT | 40~60분 | 0% | NONE | 낮음 |

---

## ⚖️ 트레이드오프

| 선택지 | 장점 | 단점 |
|--------|------|------|
| **MySQL Online DDL** | 간단, 빠름, 서드파티 불필요 | DML 성능 저하, 롤백 불가 |
| **gh-ost** | 완전 Lock-free, 롤백 가능, 성능 영향 최소 | 추가 도구, 설정 복잡 |
| **야간 배포** | 성능 영향 0%, 간단 | 서비스 중단, 배포 시간 제약 |
| **불필요한 인덱스** | 추가 비용 없음 | 저장소 낭비, INSERT 느려짐 |

---

## 📌 핵심 정리

1. **CREATE INDEX 메커니즘**: 전체 테이블 스캔 → 정렬 → B+Tree 빌드 → 변경 버퍼 merge
2. **Lock 수준**: ALGORITHM=INPLACE, LOCK=NONE (하지만 성능 영향 있음)
3. **FULLTEXT/SPATIAL**: LOCK=SHARED (쓰기 차단)
4. **복합 인덱스**: ESR 규칙 (Equality, Sort, Range)
5. **대용량 선택**:
   - MySQL Online DDL: 간단하지만 성능 영향
   - gh-ost: 완벽하지만 복잡
6. **EXPLAIN**: 인덱스 추가 전 반드시 분석

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 인덱스 빌드 중 변경 버퍼가 가득 차면 어떻게 되는가?</strong></summary>

**답변**:

변경 버퍼가 메모리 제한에 도달하면, 자동으로 merge되거나 성능이 저하됩니다.

```
Scenario:

T1: CREATE INDEX 시작
    변경 버퍼 크기: innodb_change_buffer_max_size (기본 25%)
    할당 메모리: Pool 크기의 25%

T2: 빌드 진행 중 동시 INSERT/UPDATE 많음
    변경 버퍼 누적 시작

T3: 변경 버퍼가 거의 가득 찬 상태
    할당 메모리의 95%+ 사용

T4: 추가 변경 발생
    Option 1: 자동 merge (부분)
    Option 2: 디스크 spill (I/O 증가)
    Option 3: INSERT 일부 대기

결과:
- 메모리 부족: 디스크 I/O 증가
- 성능: 30~50% 추가 저하
- 최악: OOM (Out of Memory)

모니터링:
SELECT * FROM INFORMATION_SCHEMA.INNODB_CHANGE_BUFFER_PAGE;
-- 변경 버퍼 상태 조회

SELECT * FROM SHOW STATUS LIKE '%change_buffer%';
-- change_buffer_inserts, change_buffer_deletes 등

해결:
1. 변경 버퍼 크기 증가
   SET GLOBAL innodb_change_buffer_max_size = 50;
   (서버 재시작 필요)

2. 인덱스 빌드 시간 대기
   -- 다른 DML 줄이기
   -- gh-ost의 max-lag-millis로 속도 조절

3. 인덱스 빌드 중단 후 재시도
   KILL <connection_id>;
   -- 인덱스 빌드 취소
   -- 변경 버퍼 merge는 자동으로 처리됨
```

</details>

<details>
<summary><strong>Q2: 복합 인덱스 (a, b, c)가 있을 때, WHERE b = 1 쿼리는 이 인덱스를 사용할 수 있는가?</strong></summary>

**답변**:

NO. MySQL은 인덱스의 선행 컬럼(Leading Column)부터 사용해야 합니다.

```sql
-- 복합 인덱스
CREATE INDEX idx_composite ON orders(user_id, status, amount);

-- 쿼리 1: 선행 컬럼부터 사용 ✓
SELECT * FROM orders WHERE user_id = 100;
-- Index: 사용 O (user_id로 seek)

-- 쿼리 2: 선행 컬럼부터 사용 ✓
SELECT * FROM orders 
WHERE user_id = 100 AND status = 'completed';
-- Index: 사용 O (user_id 먼저, status는 filter)

-- 쿼리 3: 선행 컬럼 건너뜀 ❌
SELECT * FROM orders WHERE status = 'completed';
-- Index: 사용 X (선행 컬럼 user_id 없음)
-- Full Table Scan 발생

-- 쿼리 4: 선행 컬럼 건너뜀 ❌
SELECT * FROM orders WHERE amount > 1000;
-- Index: 사용 X (선행 컬럼 user_id 없음)
-- Full Table Scan 발생

해결:
1. 인덱스 설계 시 쿼리 패턴 고려
   CREATE INDEX idx_status_user ON orders(status, user_id);
   -- status 먼저 오면 status 쿼리도 인덱스 사용 가능

2. 필요 시 추가 인덱스
   CREATE INDEX idx_status ON orders(status);
   CREATE INDEX idx_amount ON orders(amount);

3. 또는 커버링 인덱스
   CREATE INDEX idx_composite_covering 
   ON orders(user_id, status, amount, id);
   -- 필요한 모든 컬럼이 인덱스에 있으면 
   -- 테이블 접근 불필요 (Index-Only Scan)

EXPLAIN으로 확인:
EXPLAIN SELECT * FROM orders WHERE status = 'completed'\G
-- key: NULL (인덱스 미사용)
-- type: ALL

EXPLAIN SELECT * FROM orders WHERE user_id = 100\G
-- key: idx_composite
-- type: RANGE
```

</details>

<details>
<summary><strong>Q3: 생성 중인 인덱스를 취소할 수 있는가? 어떤 비용이 발생하는가?</strong></summary>

**답변**:

YES. 인덱스 빌드 중 연결을 종료하면 취소되지만, 정리 비용이 발생합니다.

```sql
-- 상황: CREATE INDEX 진행 중 (20분 예상)
-- 중간에 취소하려고 결정

-- 방법: 다른 터미널에서 KILL
SHOW PROCESSLIST;
-- id | user | command | state | info
--  42 | root | Query   | ... | CREATE INDEX idx_...

KILL 42;

-- 결과:
-- 인덱스 빌드 중단
-- 변경 버퍼 merge 시작 (자동)
-- 임시 인덱스 파일 삭제

취소 비용:
┌────────────────────────────────────────┐
│ 실행한 작업 수                          │
│ ├─ 빌드 완료: 500만 행 (5/10M)        │
│ ├─ 변경 버퍼: 100만 INSERT/UPDATE      │
│ └─ merge 필요: 100만 건 정리            │
│                                        │
│ 시간: 원래 예정 20분 중 10분 실행     │
│ 취소 후 정리: 2~3분 추가 소요          │
│                                        │
│ 손실: 10분 + 2분 정리비용 낭비         │
└────────────────────────────────────────┘

주의:
1. KILL은 우아한 종료
   - 변경 버퍼를 정리하지 않은 상태로 중단되지 않음
   - MySQL이 자동으로 merge 시작

2. 강제 종료 (KILL HARD) 또는 서버 재시작
   - 임시 인덱스 파일이 남을 수 있음
   - 서버 시작 시 정리 필요 (Crash Recovery)
   - 위험: 데이터 손상 가능

권장:
- 인덱스 빌드 중 취소는 가능하지만 비용이 있음
- 시간이 남아 있으면 끝까지 진행
- 취소할 경우 변경 버퍼 정리 시간 고려

별도 방법: 복제본에서 인덱스 추가
- 마스터: 인덱스 없이 계속 작동
- 복제본: 인덱스 추가 진행
- 복제본 상태 확인 후 다시 마스터로 전환
```

</details>

---

<div align="center">

**[⬅️ 이전: 컬럼 이름 변경](./05-rename-column.md)** | **[홈으로 🏠](../README.md)** | **[다음: 외래 키 제약 관리 ➡️](./07-foreign-key-management.md)**

</div>
