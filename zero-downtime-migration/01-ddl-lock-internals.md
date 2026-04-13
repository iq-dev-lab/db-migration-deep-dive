# DDL이 Lock을 거는 원리

---

## 🎯 핵심 질문

- DDL(Data Definition Language) 실행 중 왜 테이블에 접근할 수 없는가?
- `ALTER TABLE`이 Lock을 거는 수준을 결정하는 것은 무엇인가?
- Metadata Lock(MDL)과 InnoDB Lock은 어떻게 다른가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

프로덕션 환경에서 테이블을 수정하려면 먼저 Lock의 동작을 이해해야 합니다. 같은 ALTER TABLE도 문제없이 완료되거나 시스템을 마비시킬 수 있습니다. 예를 들어, 야간(트래픽 최저)에 스키마 변경을 했는데도 아침 업무 시작 전에 테이블이 여전히 Locked 상태라면?

Lock 메커니즘을 알면:
- 변경에 얼마나 걸릴지 예측 가능
- 배포 전략 수립 가능
- 모니터링과 문제 해결이 효율적

---

## 😱 흔한 실수 (Before — Lock 무시)

```sql
-- Before: DDL Lock 무시하고 실행
-- 프로덕션 DB (주문 테이블 1억 건)
ALTER TABLE orders ADD COLUMN notes VARCHAR(255);
-- 실행 후: 앱 응답 불가, 모든 SELECT가 대기

-- 왜? 
-- 1. MDL(Metadata Lock)으로 테이블 구조 변경 중
-- 2. 기존 행들을 모두 새로운 구조로 재구성(ALGORITHM=COPY)
-- 3. SELECT 쿼리가 Lock 대기... timeout 발생
```

**결과**: 고객 조회 실패, SLA 위반

---

## ✨ 올바른 접근 (After — Lock 수준 확인 및 최소화)

```sql
-- After: Lock 수준을 INSTANT 또는 NONE으로 제한
-- 1단계: 기본값이 있는 NULL 컬럼이면 INSTANT 가능 (MySQL 8.0.29+)
ALTER TABLE orders 
ADD COLUMN notes VARCHAR(255) NULL, 
ALGORITHM=INSTANT;
-- 실행 시간: 밀리초 단위, Lock 거의 없음

-- 2단계: 대량 데이터 업데이트가 필요하면 배치 처리
UPDATE orders SET notes = '' WHERE notes IS NULL LIMIT 10000;
-- 이 쿼리를 5분마다 실행, 테이블 간 충돌 최소화

-- 3단계: 사전에 Lock 상태 모니터링
SELECT * FROM information_schema.innodb_trx;
SELECT * FROM performance_schema.metadata_locks;
```

**결과**: 변경 완료, 앱 무중단

---

## 🔬 내부 동작 원리

### 1. InnoDB Lock 3단계 (LOCK 레벨)

MySQL은 DDL 실행 중 Lock 수준을 3가지로 분류합니다.

| Lock 수준 | 읽기(SELECT) | 쓰기(INSERT/UPDATE/DELETE) | 설명 |
|---------|------------|--------------------------|------|
| **EXCLUSIVE** | ❌ | ❌ | 테이블 전체 잠금, DDL 실행 중 아무도 접근 불가 |
| **SHARED** | ✓ | ❌ | 읽기는 가능하지만 쓰기 불가 (많은 ALTER TABLE) |
| **NONE** | ✓ | ✓ | Lock 없음, 동시 DML 가능 (INSTANT) |

### 2. DDL별 Lock 수준 매트릭스 (MySQL 8.0 기준)

```
컬럼 추가:
  ├─ ADD COLUMN ... NULL
  │  └─ ALGORITHM=INSTANT, LOCK=NONE (메타데이터만 변경)
  ├─ ADD COLUMN ... NOT NULL DEFAULT ...
  │  └─ ALGORITHM=INSTANT, LOCK=NONE (MySQL 8.0.29+)
  └─ ADD COLUMN ... (기존 열과 이동 필요)
     └─ ALGORITHM=INPLACE, LOCK=SHARED (테이블 재구성, 읽기만 가능)

컬럼 삭제:
  └─ DROP COLUMN
     └─ ALGORITHM=INPLACE, LOCK=SHARED (전체 테이블 재구성 필요)

컬럼 이름 변경:
  └─ RENAME COLUMN old TO new
     └─ ALGORITHM=INSTANT, LOCK=NONE (MySQL 8.0.14+, 메타데이터만)

인덱스 추가:
  └─ CREATE INDEX
     └─ ALGORITHM=INPLACE, LOCK=NONE (대부분 경우)

인덱스 삭제:
  └─ DROP INDEX
     └─ ALGORITHM=INPLACE, LOCK=NONE (메타데이터만 변경)
```

### 3. Metadata Lock (MDL) vs InnoDB Lock

MySQL 5.7+에서 도입된 **Metadata Lock(MDL)**은 DDL과 DML의 충돌을 제어합니다.

```
Timeline:
┌─────────────────────────────────────────────────┐
│ Connection 1 (DDL)          Connection 2 (DML)  │
├─────────────────────────────────────────────────┤
│ ALTER TABLE orders          SELECT * FROM       │
│   ADD COLUMN notes          orders;             │
│   (MDL 대기 중...)          ← 기존 트랜잭션     │
│                              (READ LOCK 보유)    │
│                              COMMIT (트랜잭션   │
│                              종료)              │
│ MDL 획득 ✓                                       │
│ 구조 변경 시작                                   │
└─────────────────────────────────────────────────┘
```

**MDL 대기 원인**:
1. 진행 중인 SELECT 쿼리 (READ LOCK 보유)
2. INSERT/UPDATE/DELETE 쿼리 (WRITE LOCK 보유)
3. 다른 DDL 쿼리

---

## 💻 실전 실험

### 실험 1: INSTANT vs INPLACE 성능 비교

```bash
# Docker MySQL 컨테이너 시작
docker run -d \
  --name mysql80 \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=testdb \
  -p 3306:3306 \
  mysql:8.0

# 접속
mysql -h 127.0.0.1 -u root -ppassword testdb
```

```sql
-- 대용량 테이블 생성 (1억 건)
CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  amount DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 데이터 삽입 (시간이 걸림)
INSERT INTO orders (user_id, amount) 
SELECT 
  FLOOR(RAND() * 100000) + 1,
  ROUND(RAND() * 10000, 2)
FROM 
  (SELECT 1 UNION SELECT 2 UNION SELECT 3) t1,
  (SELECT 1 UNION SELECT 2 UNION SELECT 3) t2,
  (SELECT 1 UNION SELECT 2 UNION SELECT 3) t3
LIMIT 1000000; -- 먼저 작은 데이터로 테스트

-- 실험 1: INSTANT로 NULL 컬럼 추가
SET @start = NOW(6);
ALTER TABLE orders 
ADD COLUMN notes VARCHAR(255) NULL,
ALGORITHM=INSTANT;
SELECT TIMEDIFF(NOW(6), @start) AS execution_time;
-- 결과: 00:00:00.002000 (2ms)

-- 실험 2: INPLACE로 NOT NULL 컬럼 추가 (더 느림)
SET @start = NOW(6);
ALTER TABLE orders 
ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'pending',
ALGORITHM=INPLACE;
SELECT TIMEDIFF(NOW(6), @start) AS execution_time;
-- 결과: 00:00:15.234000 (15초, 데이터 크기에 따라 변동)
```

### 실험 2: Metadata Lock 재현

```bash
# Terminal 1 (Connection A)
mysql -h 127.0.0.1 -u root -ppassword testdb
```

```sql
-- Connection A: 트랜잭션 시작 (SELECT 실행, READ LOCK 획득)
BEGIN;
SELECT COUNT(*) FROM orders;
-- 트랜잭션 유지 (COMMIT하지 않음)
```

```bash
# Terminal 2 (Connection B)
mysql -h 127.0.0.1 -u root -ppassword testdb
```

```sql
-- Connection B: DDL 시도 (MDL 대기)
ALTER TABLE orders ADD COLUMN description TEXT;
-- 대기 중... (Connection A가 COMMIT할 때까지)

-- 별도 Terminal 3에서 Lock 상태 조회
SELECT * FROM performance_schema.metadata_locks 
WHERE OBJECT_SCHEMA = 'testdb' AND OBJECT_NAME = 'orders'\G
```

**출력**:
```
LOCK_TYPE: TABLE
LOCK_DURATION: TRANSACTION
LOCK_STATUS: GRANTED (Connection A의 SELECT)
LOCK_TYPE: TABLE
LOCK_DURATION: TRANSACTION
LOCK_STATUS: WAITING (Connection B의 ALTER TABLE)
```

### 실험 3: `LOCK=NONE` 강제와 실패

```sql
-- ALGORITHM=INSTANT가 아닌 작업을 LOCK=NONE으로 강제 (실패)
ALTER TABLE orders 
DROP COLUMN description,
ALGORITHM=INPLACE,
LOCK=NONE;
-- 오류: Error 1846
-- LOCK=NONE이 지원되지 않는 작업입니다.

-- 지원되는 경우만 LOCK=NONE 적용
ALTER TABLE orders 
ADD COLUMN review_count INT DEFAULT 0,
ALGORITHM=INPLACE,
LOCK=NONE;
-- 성공 (읽고 쓰기 모두 가능)
```

---

## 📊 성능/비용 비교

| 작업 | Lock 수준 | 예상 시간 (1억 건) | 선택 기준 |
|------|---------|------------------|---------|
| NULL 컬럼 추가 | NONE | 밀리초 | 즉시 배포 가능 |
| NOT NULL + DEFAULT 추가 | NONE (8.0.29+) | 밀리초 | 최신 MySQL 사용 시 |
| ENUM 값 추가 | NONE (8.0.29+) | 밀리초 | 기존 값 뒤에만 추가 |
| 컬럼 이름 변경 | NONE | 밀리초 | Breaking Change 주의 |
| 컬럼 타입 변경 | SHARED | 10~30분 | gh-ost 고려 |
| 인덱스 추가 | NONE | 5~15분 | LOCK=NONE 보장 |
| 컬럼 삭제 | SHARED | 10~30분 | gh-ost 고려 |

**비용**: Lock이 높을수록 시스템 영향도 증가 → 서비스 중단 위험 → 야간/주말 배포 강요 → 배포 비용 증가

---

## ⚖️ 트레이드오프

| 선택지 | 장점 | 단점 |
|--------|------|------|
| **ALGORITHM=INSTANT** | 즉시 완료, 서비스 영향 최소 | MySQL 8.0.29+ 필요, 모든 DDL 지원 안 함 |
| **ALGORITHM=INPLACE** | 대부분 DDL 지원 | 시간 소요, LOCK=SHARED인 경우 쓰기 불가 |
| **gh-ost** | 진행 중 롤백 가능, 대용량 테이블 최적 | 추가 도구 필요, 복잡한 설정 |
| **pt-online-schema-change** | 오래되고 검증됨 | 트리거 오버헤드, MySQL 5.7+ 성능 저하 |
| **야간 배포 + ALGORITHM=COPY** | 간단, 오류 가능성 낮음 | 서비스 중단, 운영 부담, 빌드 시간 낭비 |

---

## 📌 핵심 정리

1. **MDL (Metadata Lock)**: 테이블 구조 변경 중 다른 DML 차단, 진행 중인 트랜잭션이 끝날 때까지 대기
2. **InnoDB Lock**: EXCLUSIVE (완전 차단) → SHARED (읽기만) → NONE (동시 접근)
3. **ALGORITHM=INSTANT** (MySQL 8.0.29+): 메타데이터만 변경, 거의 모든 컬럼 추가/삭제 가능, 권장
4. **성능 영향**: INSTANT < INPLACE < COPY, Lock 수준이 높을수록 배포 시기 제한
5. **모니터링**: `performance_schema.metadata_locks`, `information_schema.innodb_trx`로 Lock 상태 추적

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 왜 INSTANT 컬럼 추가가 메타데이터만 변경해도 되는가? 기존 행의 새 컬럼 값은 어떻게 되는가?</strong></summary>

**답변**:

INSTANT 추가 컬럼은 실제 물리적 저장소에 공간을 할당하지 않습니다. 대신 메타데이터에만 "이 컬럼이 추가됨"을 기록합니다.

```
메타데이터:
├─ id (컬럼 0)
├─ user_id (컬럼 1)
├─ amount (컬럼 2)
└─ notes (컬럼 3) ← 추가됨, 기본값 NULL

기존 행 물리 저장소: [id=1][user_id=100][amount=99.99]
                     (컬럼 3 공간 없음)

SELECT 실행 시:
SELECT id, user_id, notes FROM orders WHERE id = 1;

InnoDB는 런타임에:
1. 디스크에서 행 읽기: [1][100][99.99]
2. 메타데이터 확인: notes는 컬럼 3, 저장된 값 없음
3. 기본값(NULL)을 메모리에서 자동 할당
4. 결과 반환: [1, 100, NULL]
```

**장점**: 1억 건이어도 행을 터치하지 않으므로 밀리초 단위
**단점**: NULL 기본값만 가능, NOT NULL은 다른 방식 필요

</details>

<details>
<summary><strong>Q2: `LOCK=NONE`을 지정했는데 "Lock=NONE이 지원되지 않는다"고 오류가 나면 어떻게 해야 하는가?</strong></summary>

**답변**:

LOCK=NONE은 특정 DDL과 MySQL 버전 조합에서만 가능합니다.

```sql
-- 실패하는 경우
ALTER TABLE orders DROP COLUMN notes, LOCK=NONE;
-- Error: LOCK=NONE is not supported

-- 해결책 1: LOCK 지정자 제거 (MySQL이 자동 결정)
ALTER TABLE orders DROP COLUMN notes;
-- ALGORITHM=INPLACE, LOCK=SHARED로 자동 선택

-- 해결책 2: gh-ost 사용
-- $ gh-ost \
--   --user=root \
--   --password=password \
--   --host=127.0.0.1 \
--   --database=testdb \
--   --table=orders \
--   --alter="DROP COLUMN notes" \
--   --execute

-- 해결책 3: 배치 + Expand-Contract 패턴
-- 1. 새 컬럼 추가 (LOCK=NONE)
ALTER TABLE orders ADD COLUMN notes_temp VARCHAR(255), LOCK=NONE;

-- 2. 기존 컬럼 삭제 (일단 보류)

-- 3. 앱 코드 수정 후 배포

-- 4. 야간 배포로 컬럼 완전 삭제
ALTER TABLE orders DROP COLUMN notes; -- 야간이므로 영향 최소
```

**핵심**: LOCK=NONE을 강제하지 말고, MySQL의 자동 선택이나 gh-ost 같은 도구에 의존하는 것이 안전합니다.

</details>

<details>
<summary><strong>Q3: Metadata Lock 대기 중에 다른 쿼리가 들어오면 어떻게 되는가? 큐처럼 순서대로 처리되는가?</strong></summary>

**답변**:

MySQL은 **Lock 우선순위 큐** 방식으로 처리합니다.

```
Timeline:

T1: Connection A - SELECT 시작 (READ LOCK 획득)
   └─ MDL: READ [A]
   
T2: Connection B - ALTER TABLE 시도 (WRITE LOCK 대기)
   └─ MDL: READ [A] + WRITE [B] (대기)
   
T3: Connection C - SELECT 시도 (READ LOCK 요청)
   └─ 해결책 1: 큐에 추가 대기
   └─ 해결책 2: 즉시 진행 (MySQL 5.7.10+부터 일부 경우)
   
   MySQL 우선순위:
   1. WRITE Lock (ALTER TABLE, TRUNCATE 등) 우선 획득
   2. 진행 중인 READ는 종료 대기
   3. 새로운 READ는 WRITE 뒤에 대기
```

**문제점**:
```
T1: Connection A - SELECT (READ LOCK)
T2: Connection B - ALTER TABLE (WRITE LOCK 대기) ← 여기서 대기!
T3: Connection C - SELECT (새 요청)
   → C는 B의 ALTER가 끝날 때까지 대기
   → A의 SELECT 성능 영향 없지만, 전체 처리량 저하
```

**모니터링**:
```sql
SELECT * FROM performance_schema.metadata_locks 
WHERE LOCK_STATUS IN ('GRANTED', 'WAITING')
ORDER BY LOCK_STATUS DESC;

-- WAITING 상태가 길어지면 문제
-- 이 경우 ALTER를 KILL하거나 connection A를 강제 종료 고려
```

</details>

---

<div align="center">

**[⬅️ 이전: Chapter 2 — 체크섬 불일치 오류 해결](../flyway-internals/06-checksum-mismatch.md)** | **[홈으로 🏠](../README.md)** | **[다음: Online DDL과 gh-ost ➡️](./02-online-ddl-gh-ost.md)**

</div>
