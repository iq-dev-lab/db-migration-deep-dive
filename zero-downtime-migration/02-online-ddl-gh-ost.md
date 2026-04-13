# Online DDL과 gh-ost

---

## 🎯 핵심 질문

- MySQL Online DDL의 `ALGORITHM=INSTANT`와 `ALGORITHM=INPLACE`는 정확히 어떻게 동작하는가?
- gh-ost는 왜 Lock이 짧고 안전한가?
- pt-online-schema-change와 gh-ost의 근본적인 차이는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

온라인 DDL 도구를 선택하는 것은 배포 전략을 결정하는 것입니다:
- **MySQL Online DDL**: 버전 8.0.29+, 간단, 서드파티 도구 불필요
- **gh-ost**: 대용량 테이블(>1GB), 진행 중 롤백 필요, GitHub에서 검증된 도구
- **pt-online-schema-change**: 오래된 MySQL 버전, 트리거 방식, 성능 저하 가능성

도구별 한계를 알면 올바른 선택과 예상되는 실행 시간을 계산할 수 있습니다.

---

## 😱 흔한 실수 (Before — 도구 없이 진행)

```sql
-- Before: MySQL Online DDL 지원 확인 없이 실행
-- 프로덕션 MySQL 5.7에서:
ALTER TABLE large_orders MODIFY COLUMN amount DECIMAL(12,2);
-- 결과: 
-- 1. ALGORITHM=COPY 선택됨 (MySQL 5.7은 INSTANT 미지원)
-- 2. 임시 테이블 생성, 전체 행 복사 (1억 건, 30분 소요)
-- 3. Lock=EXCLUSIVE 발생
-- 4. 서비스 마비

-- Before: 대용량 테이블에 Online DDL 적용
-- MySQL 8.0에서도 ALGORITHM=INPLACE는 읽기만 가능
ALTER TABLE orders ADD INDEX idx_user_date (user_id, created_at);
-- 읽기는 가능하지만 INSERT/UPDATE/DELETE는 변경 버퍼에 누적
-- 시간이 오래 걸리면 버퍼 메모리 부족 → 서버 영향
```

**결과**: 배포 실패, 긴급 롤백 필요

---

## ✨ 올바른 접근 (After — 도구 선택 및 검증)

```sql
-- After: MySQL 8.0.29+에서 ALGORITHM=INSTANT 활용
-- 1단계: 버전 확인
SELECT VERSION();
-- mysql-8.0.34 (INSTANT 지원)

-- 2단계: 변경 가능 여부 미리 확인 (Dry Run)
ALTER TABLE orders MODIFY COLUMN amount DECIMAL(12,2), ALGORITHM=INSTANT, LOCK=NONE;
-- 성공하면 실제 배포도 빠름

-- After: 대용량 테이블은 gh-ost 사용
-- $ gh-ost \
--   --user=root \
--   --password=password \
--   --host=prod-db.example.com \
--   --database=ecommerce \
--   --table=orders \
--   --alter="ADD INDEX idx_user_date (user_id, created_at)" \
--   --chunk-size=5000 \
--   --max-lag-millis=100 \
--   --execute

-- 진행 상황 모니터링
SELECT * FROM _orders_ghc WHERE id > 0 LIMIT 1;
-- OK: Ghost 테이블이 활동 중
```

**결과**: 배포 완료, Lock 무시할 수준, 롤백 가능

---

## 🔬 내부 동작 원리

### 1. ALGORITHM=INSTANT (MySQL 8.0.14+)

**메커니즘**: 메타데이터만 변경, 데이터 행 터치 없음

```
시간대별 변화:

T0 (시작 전):
┌─────────────────────────────┐
│ 테이블 메타데이터          │
│ ├─ id (컬럼 0, INT)         │
│ ├─ user_id (컬럼 1, INT)    │
│ ├─ amount (컬럼 2, DECIMAL) │
│ └─ 인스턴트 컬럼 없음       │
└─────────────────────────────┘
물리 저장소: [id][user_id][amount]... × 1억 건

T1 (ALTER 실행 중):
ALTER TABLE orders ADD COLUMN notes VARCHAR(255) NULL;

메타데이터 Lock 획득 (밀리초)
메타데이터 변경 (밀리초)
│
└─ 메타데이터만:
   ├─ id (컬럼 0, INT)
   ├─ user_id (컬럼 1, INT)
   ├─ amount (컬럼 2, DECIMAL)
   └─ notes (컬럼 3, VARCHAR(255), INSTANT, 기본값 NULL) ← 추가됨
   
   물리 저장소: [id][user_id][amount]... × 1억 건 (그대로)

T2 (완료):
메타데이터 Lock 해제 (밀리초)
SELECT 실행 가능 ← 런타임에 기본값(NULL) 자동 삽입
```

**지원되는 작업**:
- ADD COLUMN ... NULL (기본값 불필요)
- ADD COLUMN ... NOT NULL DEFAULT ... (MySQL 8.0.29+)
- DROP COLUMN
- RENAME COLUMN
- 제약 조건 변경

**지원되지 않는 작업**:
- 기존 컬럼의 DEFAULT 값 변경 (기존 행에 적용 필요)
- 컬럼 타입 변환
- ENUM 값 가운데 추가

### 2. ALGORITHM=INPLACE (MySQL 5.6+)

**메커니즘**: 임시 파일에 새 구조로 재구성, 기존 테이블 덮어쓰기

```
시간대별 변화:

T0 (시작):
테이블 orders
├─ 물리 저장소: [id][user_id][amount]... × 1억 건
└─ 인덱스들

T1 (ALGORITHM=INPLACE 실행):
1) 메타데이터 Lock 획득 (짧음)
2) 임시 파일 생성 (_orders_new_inplace.ibd 등)
3) 메타데이터 Lock 해제 → 읽기 허용!
4) 백그라운드에서 행 복사 (1억 건)
   └─ 변경 버퍼(Change Buffer)에 동시 DML 누적
5) 메타데이터 Lock 재획득 (짧음)
6) 변경 버퍼 merge
7) 기존 테이블과 임시 파일 교체
8) 메타데이터 Lock 해제

T2 (완료):
테이블 orders (새 구조로 재구성됨)
├─ 물리 저장소: [새로운 구조]... × 1억 건
└─ 인덱스들 (재구성됨)
```

**Lock 수준**:
- 시작/종료: EXCLUSIVE (짧음, 밀리초)
- 처리 중: SHARED → SELECT 가능, INSERT/UPDATE/DELETE 불가

**시간 추정**:
```
실행 시간 ≈ (행 수 × 평균 행 크기) / 디스크 처리 속도
         ≈ (100,000,000 × 50바이트) / (100MB/s)
         ≈ 50GB / 100MB/s
         ≈ 500초 ≈ 8분 20초
```

### 3. gh-ost (GitHub Online Schema Migrations)

**메커니즘**: 임시 테이블 생성, Binlog 스트리밍, cut-over 시에만 Lock

```
Architecture:

MySQL Master (소스)
│
├─ 테이블 orders
│  └─ Binlog (모든 변경 기록)
│
└─ Ghost 테이블 (_orders_gho)
   ├─ 새 스키마로 생성됨
   └─ 행 복사 진행 중

Timeline:

T0 (준비):
$ gh-ost \
  --user=root \
  --host=127.0.0.1 \
  --database=testdb \
  --table=orders \
  --alter="ADD COLUMN notes VARCHAR(255) NULL" \
  --chunk-size=5000 \
  --max-lag-millis=100

T1 (Ghost 테이블 생성):
CREATE TABLE _orders_gho LIKE orders;
ALTER TABLE _orders_gho ADD COLUMN notes VARCHAR(255) NULL;
-- Lock 없음

T2 (행 복사 시작):
INSERT INTO _orders_gho 
SELECT id, user_id, amount, NULL as notes FROM orders
WHERE id > 최근 복사한 ID
LIMIT 5000;
-- 청크 단위로 반복 (5000행씩)
-- max-lag-millis=100: 복제 지연이 100ms 초과하면 일시 정지

병렬로:
-- Binlog 스트리밍: 
-- orders에 INSERT/UPDATE/DELETE 발생하면 실시간으로 _orders_gho에 적용
INSERT INTO _orders_gho (id, user_id, amount, notes)
VALUES (1001, 100, 99.99, NULL);
-- Binlog에 기록된 변경이 gh-ost를 통해 _orders_gho에 적용됨

T3 (행 복사 완료):
마지막 청크 처리 완료
Binlog 변경 모두 적용됨

T4 (Cut-over - Lock 획득):
LOCK TABLES orders WRITE;
-- 이제 orders에 새 DML 거의 없음
-- 최종 변경분을 _orders_gho에 적용
RENAME TABLE orders TO _orders_old, _orders_gho TO orders;
-- 테이블 교체 (수 초)
UNLOCK TABLES;
-- Lock 해제

T5 (정리):
DROP TABLE _orders_old; -- 기존 테이블 삭제
```

**Lock 분석**:
- 청크 복사 중: Lock 없음
- Binlog 적용: Lock 없음
- Cut-over: LOCK TABLES (수 초, 매우 짧음)

**특징**:
```bash
# 진행 상황 모니터링 (실시간)
$ watch -n 1 'SELECT rows_copied, rows_total, ROUND(100*rows_copied/rows_total, 1) as pct FROM _orders_ghc'

# 언제든 중단 가능 (대기 중인 경우)
$ touch /tmp/ghost.abort.flag

# 진행 중이면 안전하게 중단
# Binlog를 추적했으므로 다음 실행 시 이어서 처리 가능 (거의)
```

### 4. pt-online-schema-change (Percona Toolkit)

**메커니즘**: 임시 테이블 + 트리거 (Binlog 방식 아님)

```
Architecture:

원본 테이블 orders
│
├─ 임시 테이블 _orders_new (새 스키마)
│
└─ 3개 트리거:
   ├─ AFTER INSERT: 새 테이블에도 INSERT
   ├─ AFTER UPDATE: 새 테이블에도 UPDATE
   └─ AFTER DELETE: 새 테이블에도 DELETE
   
Timeline:

T1: 임시 테이블 생성 (Lock 없음)
T2: 트리거 생성 (Lock 무시할 수준)
T3: 데이터 복사 (청크로)
   └─ 이 중간에 INSERT/UPDATE/DELETE는 트리거로 새 테이블에도 동시 적용
T4: 테이블 교체 (RENAME, 짧은 Lock)

문제점:
- 트리거 오버헤드: 모든 DML이 트리거를 실행 → 3배 느려질 수 있음
- MySQL 5.7+에서 성능 저하 더 심함
- 복합 트리거는 버그 가능성 높음
```

---

## 💻 실전 실험

### 실험 1: ALGORITHM=INSTANT vs INPLACE 성능

```bash
# Docker MySQL 8.0 시작
docker run -d \
  --name mysql80 \
  -e MYSQL_ROOT_PASSWORD=root \
  -p 3306:3306 \
  mysql:8.0 \
  --default-storage-engine=InnoDB \
  --innodb_file_per_table=1

# 접속
mysql -h 127.0.0.1 -u root -proot -e "CREATE DATABASE testdb;"
```

```sql
USE testdb;

-- 테이블 생성 (1천만 건)
CREATE TABLE big_orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT,
  amount DECIMAL(10,2),
  status VARCHAR(20),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user (user_id),
  INDEX idx_created (created_at)
);

-- 데이터 삽입 (몇 분 걸림)
INSERT INTO big_orders (user_id, amount, status)
WITH RECURSIVE cte AS (
  SELECT 1 as n
  UNION ALL
  SELECT n+1 FROM cte WHERE n < 10000000
)
SELECT 
  FLOOR(RAND()*100000) + 1,
  ROUND(RAND()*10000, 2),
  'completed'
FROM cte;

-- 실험 1: ALGORITHM=INSTANT
SELECT NOW(6) as start_time;
ALTER TABLE big_orders 
ADD COLUMN notes VARCHAR(255) NULL,
ALGORITHM=INSTANT;
SELECT NOW(6) as end_time;
-- 예상: 밀리초 단위

-- 확인: 메타데이터에만 추가됨
DESCRIBE big_orders;
-- notes 컬럼 보임

-- 실험 2: ALGORITHM=INPLACE (DELETE는 기존 구조 변경 필요)
-- (대신 DEFAULT 값 추가로 시뮬레이션)
SELECT NOW(6) as start_time;
ALTER TABLE big_orders 
ADD COLUMN review_count INT DEFAULT 0,
ALGORITHM=INPLACE;
SELECT NOW(6) as end_time;
-- 예상: 수십 초~수분

-- 성능 비교
SHOW ENGINE INNODB STATUS\G
-- Rows_inserted, Rows_updated 확인 (INPLACE의 변경 버퍼 활동)
```

### 실험 2: gh-ost 실행 및 모니터링

```bash
# gh-ost 설치 (또는 Docker 이미지 사용)
# macOS: brew install gh-ost
# Linux: https://github.com/github/gh-ost/releases

# Docker MySQL에서 바이너리 로깅 활성화 필요
# 컨테이너 중단 후 재시작:
docker run -d \
  --name mysql80 \
  -e MYSQL_ROOT_PASSWORD=root \
  -p 3306:3306 \
  mysql:8.0 \
  --log-bin=mysql-bin \
  --server-id=1 \
  --binlog-format=ROW

# gh-ost 실행 (dry-run 먼저)
gh-ost \
  --user=root \
  --password=root \
  --host=127.0.0.1 \
  --port=3306 \
  --database=testdb \
  --table=big_orders \
  --alter="ADD COLUMN archived_at TIMESTAMP NULL" \
  --chunk-size=10000 \
  --max-lag-millis=100 \
  --verbose \
  --timestamp-old-table \
  --execute  # --execute 빼면 dry-run

# 별도 터미널에서 모니터링
watch -n 1 'mysql -h 127.0.0.1 -u root -proot testdb \
  -e "SELECT 
    rows_copied, 
    rows_total, 
    ROUND(100*rows_copied/rows_total,1) as pct,
    is_running
  FROM _big_orders_ghc WHERE id IS NULL LIMIT 1;"'

# 또는 소켓을 통한 인터랙티브 상태 확인
# (gh-ost가 /tmp/gh-ost.testdb.big_orders.sock 생성)
echo "status" | nc -U /tmp/gh-ost.testdb.big_orders.sock

# 출력 예:
# Migrating testdb.big_orders
# 100000 rows copied (10%)
```

### 실험 3: pt-online-schema-change와 성능 비교

```bash
# pt-online-schema-change 설치
# percona-toolkit 패키지에 포함

pt-online-schema-change \
  --user=root \
  --password=root \
  --host=127.0.0.1 \
  D=testdb,t=big_orders \
  --alter="ADD COLUMN backup_at TIMESTAMP NULL" \
  --progress=time,30s \
  --chunk-size=5000 \
  --max-lag-time=5s \
  --execute

# 실행 중 트리거 확인
SHOW TRIGGERS FROM testdb;
-- _big_orders_del, _big_orders_ins, _big_orders_upd 생성됨

# 성능 모니터링: DML 속도 저하 관찰
# INSERT/UPDATE/DELETE가 일반적인 3배 느림
```

---

## 📊 성능/비용 비교

| 도구 | 테이블 크기 | Lock 시간 | 총 소요 시간 | 롤백 | 비고 |
|------|----------|---------|-----------|------|------|
| INSTANT | 모든 크기 | 밀리초 | 초 | 불가 | MySQL 8.0.29+ 필수, 가장 빠름 |
| INPLACE | <10GB | 초 | 분 | 불가 | 읽기는 가능, 쓰기 불가 |
| gh-ost | 10GB~500GB | 수 초 | 분~시간 | 가능 | 대용량 최적, 롤백 안전 |
| pt-online-schema-change | <10GB | 초 | 분~시간 | 가능 | 오버헤드 큼, 지원 모드 제한 |
| COPY (야간) | 모든 크기 | 분~시간 | 시간~일 | 불가 | 서비스 중단 |

**비용 계산**:
```
시간당 손실 = 서비스 중단 시 분당 손실 × 중단 분 × 시간당 중단 회수
            = $50(분당) × 30분 × 2회/주
            = $3,000/주 → $156,000/년
            
반면, INSTANT는:
시간당 손실 = 0 (중단 없음)
```

---

## ⚖️ 트레이드오프

| 도구 | 장점 | 단점 |
|------|------|------|
| **INSTANT** | 즉시 완료, 거의 모든 변경 지원 | MySQL 8.0.29+ 필수, 특정 작업 불가 |
| **INPLACE** | 대부분 MySQL 버전에서 지원 | DML 차단 시간 (읽기만 가능), 느림 |
| **gh-ost** | 진행 중 롤백 가능, 대용량 최적 | 추가 도구, Binlog 필수, 설정 복잡 |
| **pt-online-schema-change** | 오래 검증됨, 간단 | 트리거 오버헤드, MySQL 5.7+ 성능 저하 |

---

## 📌 핵심 정리

1. **ALGORITHM=INSTANT** (권장, MySQL 8.0.29+):
   - 메타데이터만 변경
   - 모든 DML 허용
   - 밀리초 단위 완료
   - 가장 빠르고 안전

2. **ALGORITHM=INPLACE** (MySQL 5.6+):
   - 임시 파일 재구성
   - 읽기는 가능, 쓰기 차단
   - 시간 소요 (몇 분~수십 분)
   - 변경 버퍼로 동시 DML 누적

3. **gh-ost** (대용량 + 롤백 필요):
   - Binlog 스트리밍
   - Cut-over 시에만 짧은 Lock
   - 진행 중 중단 및 롤백 가능
   - 가장 안전하고 검증됨

4. **선택 기준**:
   - MySQL 8.0.29+? → INSTANT
   - 대용량 + 롤백 필요? → gh-ost
   - 오래된 MySQL? → pt-online-schema-change (또는 upgr ade)

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: ALGORITHM=INPLACE 실행 중 변경 버퍼가 가득 차면 어떻게 되는가?</strong></summary>

**답변**:

변경 버퍼(Change Buffer)는 InnoDB의 메모리 또는 디스크 공간에 누적됩니다.

```
시나리오:

T1: ALTER TABLE ... ALGORITHM=INPLACE 시작
    변경 버퍼 초기화 (크기: innodb_change_buffer_max_size)
    
T2: 동시에 INSERT/UPDATE/DELETE 발생
    변경 버퍼에 쌓임 (메모리 효율적)
    
T3: 변경 버퍼가 가득 참
    └─ 옵션 1: 자동 merge (메모리 부족 시)
       ├─ 임시 테이블에 변경 적용
       └─ 속도 저하
    └─ 옵션 2: 디스크에 spill (변경 버퍼를 디스크 페이지로)
       └─ I/O 증가, 성능 저하
    └─ 옵션 3: INSERT/UPDATE/DELETE 일부 실패
       └─ 앱 오류 발생 가능
```

**모니터링 및 조치**:
```sql
-- 변경 버퍼 상태 확인
SHOW ENGINE INNODB STATUS\G
-- Output에서 "change buffer" 섹션 확인

-- 변경 버퍼 크기 조정 (서버 재시작 필요)
-- my.cnf에 추가:
-- innodb_change_buffer_max_size=50 (기본값)

-- ALTER 중 변경 버퍼 모니터링
SHOW STATUS LIKE '%change_buffer%';

-- 변경 버퍼 충분히 크면 성능 유지
-- 부족하면 ALTER 속도를 늦춤 (--chunk-size 감소, gh-ost 권장)
```

</details>

<details>
<summary><strong>Q2: gh-ost 실행 중에 원본 테이블 스키마가 변경되면(다른 ALTER)어떻게 되는가?</strong></summary>

**답변**:

gh-ost는 Binlog를 추적하므로, 원본 테이블의 다른 스키마 변경도 감지합니다.

```
Timeline:

T1: gh-ost 시작 (orders → _orders_gho로 복사 중)

T2: 누군가 다른 ALTER 실행
    ALTER TABLE orders ADD COLUMN rating INT;
    
T3: gh-ost 응답:
    └─ 옵션 1: 오류 발생 및 중단
       Error: Detected schema change on original table
    └─ 옵션 2: Binlog에서 감지하고 _orders_gho에도 적용
       (gh-ost 버전에 따라)
    
만약 gh-ost가 감지하지 못하면:
_orders_gho와 원본 orders의 스키마가 달라짐
Cut-over 실패
```

**안전한 관행**:
```bash
# gh-ost 실행 중 다른 ALTER 금지
# gh-ost 진행 상태 확인:
SELECT * FROM _orders_ghc WHERE id IS NULL;
-- is_running = 1이면 진행 중

# 완료 시까지 대기:
while [ 1 ]; do
  mysql -e "SELECT * FROM _orders_ghc" | grep 'is_running = 1' || break
  sleep 10
done
# 이제 다른 ALTER 가능

# 또는 gh-ost에 --postpone-cut-over 옵션
# (수동으로 cut-over 시작)
```

</details>

<details>
<summary><strong>Q3: INSTANT와 INPLACE 중 어느 것을 선택할 때 INSTANT가 실패할 수 있는가?</strong></summary>

**답변**:

MySQL은 내부적으로 ALGORITHM 호환성을 확인합니다.

```sql
-- 실패하는 경우:

-- Case 1: 기존 컬럼 DEFAULT 값 변경
ALTER TABLE orders 
MODIFY COLUMN status VARCHAR(20) NOT NULL DEFAULT 'new_status',
ALGORITHM=INSTANT;
-- 오류: ALGORITHM=INSTANT is not supported for this operation

-- Case 2: 컬럼 타입 변환
ALTER TABLE orders 
MODIFY COLUMN amount DECIMAL(12,2),
ALGORITHM=INSTANT;
-- (amount가 이미 DECIMAL(10,2)인 경우)
-- 오류: ALGORITHM=INSTANT is not supported

-- Case 3: 기존 데이터와 호환되지 않는 변경
ALTER TABLE orders 
MODIFY COLUMN amount INT,
ALGORITHM=INSTANT;
-- 오류: ALGORITHM=INSTANT is not supported

해결책 1: ALGORITHM 지정 제거 (MySQL이 자동 선택)
ALTER TABLE orders 
MODIFY COLUMN status VARCHAR(20) NOT NULL DEFAULT 'new_status';
-- MySQL이 INPLACE로 자동 선택

해결책 2: 데이터를 2단계로 변경
-- Step 1: 새 컬럼 추가 (INSTANT)
ALTER TABLE orders 
ADD COLUMN status_new VARCHAR(20) NOT NULL DEFAULT 'new_status',
ALGORITHM=INSTANT;

-- Step 2: 배치 업데이트
UPDATE orders SET status_new = status LIMIT 10000;
-- (반복)

-- Step 3: 컬럼 이름 변경 (INSTANT)
ALTER TABLE orders RENAME COLUMN status TO status_old;
ALTER TABLE orders RENAME COLUMN status_new TO status;

-- Step 4: 오래된 컬럼 삭제
ALTER TABLE orders DROP COLUMN status_old;

이 과정에서 모든 단계가 INSTANT이므로 전체 시간도 빠름
```

</details>

---

<div align="center">

**[⬅️ 이전: DDL이 Lock을 거는 원리](./01-ddl-lock-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Expand-Contract 패턴 ➡️](./03-expand-contract-pattern.md)**

</div>
