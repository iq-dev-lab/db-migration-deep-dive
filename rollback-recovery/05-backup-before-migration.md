# 05. 백업과 마이그레이션

---

## 🎯 핵심 질문

```
❓ 마이그레이션은 실패하면 ROLLBACK 불가능한데,
   그럼 데이터 손실은 어떻게 방지하나?
   백업은 언제, 어떻게 해야 하나?
```

---

## 🔍 왜 이 개념이 실무에서 중요한가

**마이그레이션의 최후의 보루**:

DDL 롤백 불가 → Forward-Only → repair → 그것도 실패 → **마지막 수단: 백업**

이를 모르면:

- ❌ 마이그레이션 실패 후 "데이터 다 날아갔습니다" → 복구 불가
- ❌ 백업은 있는데 복원 방법을 모름 → 대기 시간 증가
- ❌ 백업 때문에 서비스 다운타임 발생 → 사용자 불만
- ❌ "언제부터의 백업인가?" 불명확 → 데이터 불일치

**실제 사례**:
```
2024-01-15 09:00: 마이그레이션 시작
2024-01-15 09:05: 마이그레이션 실패 (제약 조건 위반)
2024-01-15 09:10: repair 시도
2024-01-15 09:15: 여전히 실패
2024-01-15 09:20: "백업에서 복원합시다"

→ 05:00 백업 ≠ 09:00 현재 상태 (4시간 차이)
→ 05:00~09:00 데이터 손실 가능
→ "이 정도면 낫다" 판단하고 복원
```

**따라서 마이그레이션은 "백업 → 실행 → 검증"**:

---

## 😱 흔한 실수 (Before — 백업 없이 시작)

### 실수 1: 백업 없이 바로 마이그레이션 실행

```bash
# 프로덕션에서
$ flyway migrate
# ❌ 마이그레이션 실패

# "아, 백업을..."
# → 이미 늦음. 스키마가 부분 적용된 상태
# → DDL은 이미 커밋되어 롤백 불가
```

### 실수 2: 수주 전의 백업에서 복원

```
백업 타이밍:
├─ 01-10: 백업 A (크기 500GB, 시간 1시간)
├─ 01-11~14: 백업 없음 (매일 복구 테스트만)
├─ 01-15: 마이그레이션 실패 (절대 백업 필요)
└─ "가장 최신 백업은 01-10입니다"

→ 01-10 이후의 데이터 손실 (5일치!)
```

### 실수 3: 백업 복원 후 불일치

```
1. 마이그레이션 실패 (01-15 09:10)
2. 백업 복원 (01-15 08:00 기준점으로 복원)
3. 다시 마이그레이션 시도

하지만:
├─ 08:00~09:10 사이의 데이터 변경사항은?
├─ 외부 API가 DB에 저장한 데이터?
├─ 캐시와의 불일치?
└─ 사용자가 만든 주문? (1시간 분량 손실)
```

---

## ✨ 올바른 접근 (After — 마이그레이션 전 체계적인 백업)

### 올바른 백업 전략

**3단계: 백업 → 마이그레이션 → 검증**

```
Phase 1: 마이그레이션 직전 백업
├─ RDS Snapshot (AWS) 또는 수동 백업
├─ 시간 기록: 09:00:00
├─ 백업 ID 저장: snap-12345678

Phase 2: 마이그레이션 실행
├─ 시작: 09:01:00
├─ 종료: 09:15:00 (또는 실패)
└─ 결과: 성공/실패 기록

Phase 3: 검증 (마이그레이션 성공 시)
├─ 스키마 확인: DESCRIBE, SHOW TABLES
├─ 행 수 확인: SELECT COUNT(*)
├─ 데이터 샘플링: 주요 데이터 일부 검증
└─ 헬스체크: 앱이 정상 동작 확인

Phase 4: 결정
├─ 성공 → 백업 보관 (선택: 삭제 또는 장기 보관)
├─ 실패 → 백업에서 복원 (시점 선택)
└─ 부분 실패 → Forward-Only 또는 복원
```

### 백업의 종류

#### 1. 논리 백업 (Logic Backup): mysqldump

```bash
# 프로덕션 마이그레이션 직전
$ mysqldump -u root -ppassword appdb \
  --single-transaction \
  --lock-tables=false \
  --dump-date \
  > backup_20240115_0900.sql

# 옵션 설명:
# --single-transaction: 일관된 스냅샷 생성 (InnoDB)
# --lock-tables=false: 테이블 잠금 없음 (서비스 중단 X)
# --dump-date: 백업 생성 시간 포함

# 백업 크기 확인
$ ls -lh backup_20240115_0900.sql
# 123G backup_20240115_0900.sql ← 대용량!
```

**장점**:
- ✅ 복원이 간단 (mysql < backup.sql)
- ✅ DB 간 이전 가능 (MySQL → PostgreSQL 가능)

**단점**:
- ❌ 시간 오래 걸림 (데이터 크기에 비례)
- ❌ 네트워크 I/O 증가
- ❌ 파일 크기 매우 큼

#### 2. 물리 백업 (Physical Backup): xtrabackup

```bash
# Percona XtraBackup 설치 필요
$ xtrabackup --backup \
  --target-dir=/backup/20240115_0900 \
  --user=root \
  --password=password

# 백업 완료 후 준비
$ xtrabackup --prepare \
  --target-dir=/backup/20240115_0900

# 복원
$ systemctl stop mysql
$ cp -r /backup/20240115_0900/* /var/lib/mysql/
$ chown -R mysql:mysql /var/lib/mysql
$ systemctl start mysql
```

**장점**:
- ✅ 매우 빠름 (mysqldump보다 10배 이상)
- ✅ 파일 크기 작음 (압축)
- ✅ 증분 백업 가능

**단점**:
- ❌ 복원이 복잡 (prepare, restore 단계)
- ❌ MySQL 버전 호환성 필요

#### 3. RDS Snapshot (AWS)

```bash
# AWS CLI로 스냅샷 생성
$ aws rds create-db-snapshot \
  --db-instance-identifier mydb \
  --db-snapshot-identifier mydb-20240115-0900

# 스냅샷 상태 확인
$ aws rds describe-db-snapshots \
  --db-snapshot-identifier mydb-20240115-0900 \
  --query 'DBSnapshots[0].[PercentProgress,AvailabilityZone]' \
  --output table

# 스냅샷에서 복원 (새 인스턴스로)
$ aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier mydb-restored \
  --db-snapshot-identifier mydb-20240115-0900

# 또는 제자리(in-place) 복원은 불가 (downtime 필요)
```

**장점**:
- ✅ GUI에서 간단 (AWS 콘솔)
- ✅ 빠른 복원
- ✅ Point-in-Time Recovery 가능

**단점**:
- ❌ 스냅샷 생성 중 성능 저하 (경미함)
- ❌ 스냅샷 저장 비용

---

## 🔬 내부 동작 원리

### 1. Point-in-Time Recovery (PITR)

```
Timeline:
├─ 09:00:00 Snapshot A 생성
├─ 09:00:01~09:15:00 Binlog 기록 (매초)
│  ├─ 09:01:00: INSERT user (user_id=100)
│  ├─ 09:05:00: UPDATE product SET price=99.99
│  ├─ 09:10:00: DELETE order WHERE id=5
│  └─ 09:15:00: 마이그레이션 실패
└─ 현재: 09:20:00 복원 결정

선택지:
├─ Option 1: 09:00:00 Snapshot A로 복원
│  ├─ 결과: Snapshot 생성 시점으로 돌아감
│  ├─ 손실: 09:00:01~09:20:00 모든 변경사항
│  └─ 위험: 이미 생성된 데이터 있는가?
│
├─ Option 2: 09:14:59 (마이그레이션 직전)로 복원
│  ├─ 결과: 마이그레이션 실패 직전 상태
│  ├─ 손실: 09:15:00~09:20:00 (거의 없음)
│  └─ 가장 좋음!
│
└─ Option 3: 09:10:00 (마이그레이션 중간)로 복원
   ├─ 결과: 마이그레이션 시작 전
   ├─ 손실: 09:10:00~09:20:00
   └─ 절충안
```

### 2. Binlog 기반 복구

```sql
-- Binlog 위치 확인
SHOW MASTER STATUS;
-- File: mysql-bin.000010
-- Position: 1234567
-- Binlog_Do_DB: appdb

-- PITR 복원 명령
mysqlbinlog --start-datetime="2024-01-15 09:00:00" \
            --stop-datetime="2024-01-15 09:14:59" \
            mysql-bin.000010 | mysql -u root -ppassword appdb

-- 또는 Snapshot + Binlog 조합
# 1. Snapshot 복원 (09:00:00)
aws rds restore-db-instance-from-db-snapshot ...

# 2. Binlog 적용 (09:00:01 ~ 09:14:59)
# RDS는 자동으로 Binlog 적용
```

### 3. 백업 & 복원 타임라인

```
마이그레이션 프로세스:

T-1시간: 마이그레이션 계획
├─ 영향 범위 확인
├─ Rollback 계획 수립 (Forward-Only)
└─ 백업 생성 일정

T-15분: 마이그레이션 준비
├─ 모든 프로세스 정지 (또는 읽기 전용)
├─ 마지막 백업 생성
└─ 시간 동기화

T=0: 마이그레이션 시작
├─ Flyway migrate 또는 수동 SQL 실행
└─ 진행 상황 모니터링

T+30분: 마이그레이션 완료 (또는 실패)
├─ 성공 → 검증 후 정상화
├─ 실패 → 백업에서 복원 검토
└─ 결과 기록

T+1시간: 모니터링
├─ 장기 실행 쿼리 모니터링
├─ 성능 메트릭 확인
├─ 에러 로그 확인
└─ 추가 마이그레이션 필요 여부 판단
```

---

## 💻 실전 실험

### 실험 1: mysqldump 백업 & 복원

```bash
# MySQL 8.0 실행
docker run -d --name mysql-backup \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=appdb \
  mysql:8.0

sleep 10

# 초기 데이터 생성
docker exec mysql-backup mysql -u root -ppassword appdb << 'EOF'
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (email) VALUES ('user1@example.com');
INSERT INTO users (email) VALUES ('user2@example.com');
INSERT INTO users (email) VALUES ('user3@example.com');

SELECT COUNT(*) FROM users;
-- 3
EOF
```

```bash
# Step 1: 백업 생성 (마이그레이션 전)
docker exec mysql-backup mysqldump -u root -ppassword appdb \
  --single-transaction \
  --lock-tables=false \
  > /tmp/backup_20240115_before.sql

# 백업 파일 확인
ls -lh /tmp/backup_20240115_before.sql

# Step 2: 의도적으로 데이터 손상 (마이그레이션 실패 시뮬레이션)
docker exec mysql-backup mysql -u root -ppassword appdb << 'EOF'
DELETE FROM users WHERE id = 2;  -- 실수로 데이터 삭제

SELECT COUNT(*) FROM users;
-- 2 (user2가 사라짐)
EOF

# Step 3: 백업에서 복원
# 새 컨테이너에 복원하기 (원본 보존)
docker run -d --name mysql-restored \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=appdb \
  mysql:8.0

sleep 10

# 백업 복원
docker exec -i mysql-restored mysql -u root -ppassword appdb < /tmp/backup_20240115_before.sql

# 복원 확인
docker exec mysql-restored mysql -u root -ppassword appdb -e \
  "SELECT * FROM users;"

# id | email              | created_at
# 1  | user1@example.com  | 2024-01-15 10:00:00
# 2  | user2@example.com  | 2024-01-15 10:00:01  ← 복원됨!
# 3  | user3@example.com  | 2024-01-15 10:00:02
```

### 실험 2: RDS Snapshot으로 Point-in-Time Recovery

```bash
# AWS RDS 환경에서 (예시)

# Step 1: 마이그레이션 전 자동 백업 확인
aws rds describe-db-instances \
  --db-instance-identifier mydb \
  --query 'DBInstances[0].[BackupRetentionPeriod,LatestRestorableTime]' \
  --output table

# BackupRetentionPeriod: 30 (30일 보관)
# LatestRestorableTime: 2024-01-15T10:23:45.000Z

# Step 2: 마이그레이션 시작 전 스냅샷 생성
aws rds create-db-snapshot \
  --db-instance-identifier mydb \
  --db-snapshot-identifier mydb-before-migration-20240115

# 스냅샷 진행 상황 모니터링
aws rds describe-db-snapshots \
  --db-snapshot-identifier mydb-before-migration-20240115 \
  --query 'DBSnapshots[0].PercentProgress' \
  --output text
# 25
# 50
# 100 (완료)

# Step 3: 마이그레이션 실패 시 복원 (새 인스턴스)
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier mydb-restored-20240115 \
  --db-snapshot-identifier mydb-before-migration-20240115

# Step 4: 복원 완료 대기 (수 분)
aws rds wait db-instance-available \
  --db-instance-identifier mydb-restored-20240115

# Step 5: 데이터 검증
mysql -u admin -ppassword \
  -h mydb-restored-20240115.xxxxx.rds.amazonaws.com \
  appdb -e "SELECT COUNT(*) FROM users;"
```

### 실험 3: 마이그레이션 전 체계적인 백업 체크리스트

```bash
#!/bin/bash
# migration_backup_checklist.sh

set -e  # 오류 시 중단

DB_NAME="appdb"
DB_USER="root"
DB_PASSWORD="password"
BACKUP_DIR="/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_before_migration_$TIMESTAMP.sql"

echo "=== 마이그레이션 전 백업 체크리스트 ==="

# 1. 현재 상태 확인
echo "1️⃣ 현재 DB 상태 확인..."
mysql -u $DB_USER -p$DB_PASSWORD $DB_NAME -e \
  "SELECT COUNT(*) as total_tables FROM information_schema.tables 
   WHERE table_schema = '$DB_NAME';"

# 2. 디스크 용량 확인
echo "2️⃣ 디스크 용량 확인..."
AVAILABLE=$(df $BACKUP_DIR | tail -1 | awk '{print $4}')
DB_SIZE=$(du -sb /var/lib/mysql/$DB_NAME 2>/dev/null | awk '{print $1}' || echo "100GB")
echo "   사용 가능: $AVAILABLE, DB 크기: $DB_SIZE"

if [ $AVAILABLE -lt $DB_SIZE ]; then
    echo "❌ 디스크 용량 부족! 백업 불가"
    exit 1
fi

# 3. 백업 생성
echo "3️⃣ 백업 시작 ($BACKUP_FILE)..."
mysqldump -u $DB_USER -p$DB_PASSWORD $DB_NAME \
  --single-transaction \
  --lock-tables=false \
  > $BACKUP_FILE

echo "   ✅ 백업 완료"
ls -lh $BACKUP_FILE

# 4. 백업 무결성 확인
echo "4️⃣ 백업 무결성 확인..."
BACKUP_LINES=$(wc -l < $BACKUP_FILE)
if [ $BACKUP_LINES -lt 100 ]; then
    echo "❌ 백업 파일이 너무 작음!"
    exit 1
fi
echo "   ✅ 백업 유효 ($BACKUP_LINES 줄)"

# 5. 백업 복원 테스트 (스테이징에서)
echo "5️⃣ 백업 복원 테스트 (선택사항)..."
# mysql -u test -ppassword test_db < $BACKUP_FILE
# 복원 후 SELECT COUNT(*)로 행 수 검증

# 6. 백업 정보 기록
echo "6️⃣ 백업 정보 저장..."
cat > ${BACKUP_FILE}.meta << METADATA
backup_time: $(date)
backup_file: $BACKUP_FILE
backup_size: $(du -h $BACKUP_FILE | awk '{print $1}')
db_name: $DB_NAME
tables_included: $(mysql -u $DB_USER -p$DB_PASSWORD $DB_NAME -e "SELECT GROUP_CONCAT(table_name) FROM information_schema.tables WHERE table_schema = '$DB_NAME';" -N)
METADATA

echo "=== 백업 완료 ==="
echo "백업 파일: $BACKUP_FILE"
echo "메타데이터: ${BACKUP_FILE}.meta"
echo ""
echo "다음 단계: flyway migrate 실행"
```

```bash
# 스크립트 실행
bash migration_backup_checklist.sh

# 출력:
# === 마이그레이션 전 백업 체크리스트 ===
# 1️⃣ 현재 DB 상태 확인...
#    total_tables: 15
# 2️⃣ 디스크 용량 확인...
#    사용 가능: 1000GB, DB 크기: 150GB
# 3️⃣ 백업 시작...
#    ✅ 백업 완료
#    -rw-r--r-- 1 root root 145G Jan 15 10:00 backup_20240115_1000.sql
# 4️⃣ 백업 무결성 확인...
#    ✅ 백업 유효 (1234567 줄)
# 5️⃣ 백업 복원 테스트 (선택사항)...
# 6️⃣ 백업 정보 저장...
# === 백업 완료 ===
# 백업 파일: /backups/backup_before_migration_20240115_1000.sql
# 메타데이터: /backups/backup_before_migration_20240115_1000.sql.meta
```

---

## 📊 성능/비용 비교

| 방식 | 속도 | 파일 크기 | 복원 속도 | 비용 | 용도 |
|------|------|---------|---------|------|------|
| **mysqldump** | 느림 (1시간+) | 매우 큼 | 중간 | 무료 | 소규모 마이그레이션 |
| **xtrabackup** | 빠름 (10분) | 중간 | 빠름 | 무료 | 대규모, 빈번한 백업 |
| **RDS Snapshot** | 중간 (20분) | 작음 | 매우 빠름 | $$ (저장소) | AWS RDS 운영 |
| **Cloud Backup** | 중간 | 작음 | 빠름 | $$$ | 엔터프라이즈 |

---

## ⚖️ 트레이드오프

### mysqldump vs xtrabackup

**mysqldump**:
- ✅ 간단 (한 줄 명령어)
- ✅ DB 독립적 (모든 DB 가능)
- ❌ 시간 오래 (100GB = 1시간+)
- ❌ 파일 크기 큼

**xtrabackup**:
- ✅ 빠름 (100GB = 10분)
- ✅ 파일 크기 작음 (압축)
- ❌ MySQL 종속 (PostgreSQL 불가)
- ❌ 복원이 복잡

---

## 📌 핵심 정리

1. **마이그레이션 전 백업은 필수**
   ```
   마이그레이션 실패 → DDL 롤백 불가 → Forward-Only/repair 
   → 그것도 실패 → 백업 복원 (최후의 수단)
   ```

2. **백업 타이밍**
   ```
   가장 좋음: 마이그레이션 직전 (5분 이내)
   좋음: 마이그레이션 당일 아침
   부족함: 전날 (시간이 경과하면서 데이터 변경)
   ```

3. **다양한 백업 방식**
   ```
   로컬 개발: mysqldump (간단)
   스테이징: xtrabackup 또는 RDS Snapshot (검증용)
   프로덕션: RDS Snapshot + 자동 백업 (장기 보관)
   ```

4. **Point-in-Time Recovery (PITR)**
   ```
   마이그레이션 실패 시점 바로 직전으로 복원 가능
   데이터 손실 최소화 (보통 1분 이내)
   RDS는 자동 지원, 온프레미스는 Binlog 활용
   ```

5. **백업 검증이 중요**
   ```
   - 백업 파일 크기 확인 (너무 작으면 오류)
   - 복원 테스트 (스테이징 환경에서)
   - 메타데이터 기록 (언제, 어느 버전인지)
   - 정기적 복원 훈련
   ```

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 마이그레이션이 1시간 걸리는데, 이 사이에 데이터가 바뀌면?</strong></summary>

**답**:

**장시간 마이그레이션의 문제**:

```
Timeline:
├─ 09:00: 마이그레이션 시작
├─ 09:00: 사용자 A가 주문 생성 (order_id=100)
├─ 09:30: 스키마 변경 진행 중 (테이블 잠금)
├─ 10:00: 마이그레이션 완료
└─ 결과: order_id=100은 어느 스키마 기준?
```

**상황 1: 읽기 전용 모드로 전환**

```
09:00: 마이그레이션 시작 + 읽기 전용 설정
├─ SET GLOBAL read_only = ON;
├─ 진행 중인 트랜잭션 완료 대기
└─ 마이그레이션 시작

결과:
├─ 사용자는 데이터 읽기만 가능 (조회)
├─ 데이터 변경 불가 (INSERT/UPDATE/DELETE 오류)
└─ "이 시간에 업데이트 불가" 알림 필요

문제: 사용자 경험 저하 (트래픽이 많으면 더 심함)
```

**상황 2: 마이그레이션 중 계속 쓰기 허용**

```
09:00: 마이그레이션 시작 (쓰기 허용)
├─ 09:00~10:00: 사용자 계속 데이터 변경
├─ 스키마 변경과 동시에 데이터도 변경
└─ 결과: 불일치 또는 마이그레이션 실패 가능

예: 컬럼 추가 중에 새 주문이 들어오면?
   ├─ V1 스키마로 INSERT (phone 없음)
   ├─ V2 스키마로 UPDATE (phone 추가)
   └─ 불일치!
```

**해결책**:

```bash
# Option 1: 쓰기 차단 + 짧은 마이그레이션
# (1시간 마이그레이션을 10분으로 줄임)
└─ 병렬 처리, 인덱스 사전 생성, 배치 처리 등

# Option 2: Blue-Green 배포
1. Green 서버 준비 (V2 앱 + V1 DB)
2. 마이그레이션 (Green 서버에서 독립적으로)
3. 다중 마스터 또는 동기화 복제
4. Blue → Green 전환 (DNS, LB 변경)
5. 데이터 불일치 확인 및 보정

# Option 3: 논리적 마이그레이션 (앱 코드로)
Spring Boot Flyway Callback:
@Component
public class BigTableMigrationCallback implements FlywayCallback {
    @Override
    void handle(FlywayEvent event) {
        if (event.getVersion() == 10) {
            // 10분 단위로 배치 마이그레이션
            batchMigrateData(1000000);
        }
    }
}
```

**권장**:
- 마이그레이션 시간 < 5분 (읽기 전용 필요 없음)
- 마이그레이션 시간 5~30분 (저트래픽 시간대)
- 마이그레이션 시간 > 30분 (Blue-Green 배포)

</details>

<details>
<summary><strong>Q2: 백업 복원 후 애플리케이션의 캐시는? 외부 API와의 불일치는?</strong></summary>

**답**:

**백업 복원의 부작용**:

```
Before: 2024-01-15 09:00
├─ DB: user_id=1, balance=1000
├─ Cache: balance=1000
├─ 외부 결제 시스템: paid_order=100건

After: 복원 (2024-01-15 08:00)
├─ DB: user_id=1, balance=500 ← 1시간 전 상태
├─ Cache: balance=1000 ← 그대로 (무효화 안 됨)
├─ 외부 결제 시스템: paid_order=100건 (그대로)
└─ 불일치! (balance 500 vs 캐시 1000)
```

**복원 후 동기화 절차**:

```bash
#!/bin/bash
# restore_and_sync.sh

echo "1️⃣ 백업에서 복원"
mysql < backup_20240115_0800.sql

echo "2️⃣ 캐시 무효화"
redis-cli FLUSHALL  # 또는 특정 키만
# 또는 Redis Cluster에서
redis-cli -h cache-node-1 FLUSHALL
redis-cli -h cache-node-2 FLUSHALL

echo "3️⃣ 앱 서버 재시작"
systemctl restart app-server-1
systemctl restart app-server-2

echo "4️⃣ 외부 API와 상태 동기화"
# 각 외부 API별로:
# - Stripe: 결제 기록 확인, DB와 대조
# - 배송 시스템: 배송 상태 다시 조회
# - 분석 시스템: 이벤트 재전송 불필요한지 확인

echo "5️⃣ 사용자 알림"
# "2024-01-15 08:00 이후의 데이터가 손실되었습니다"
# 영향받은 사용자 목록 생성
SELECT * FROM orders WHERE created_at BETWEEN '2024-01-15 08:00' AND '2024-01-15 09:00';
```

**예방 방법**:

```java
// Spring Boot에서 백업 복원 후 자동 동기화

@Component
public class DataSyncAfterRestore {
    
    @EventListener
    public void onApplicationReady(ApplicationReadyEvent event) {
        // 데이터 일관성 확인
        syncCacheWithDB();
        syncExternalAPIs();
        notifyAffectedUsers();
    }
    
    private void syncCacheWithDB() {
        // 캐시 무효화
        cacheManager.getCacheNames().forEach(name -> 
            cacheManager.getCache(name).clear()
        );
    }
    
    private void syncExternalAPIs() {
        // 외부 API 재동기화
        paymentService.syncWithStripe();
        shippingService.syncWithCarrier();
    }
    
    private void notifyAffectedUsers() {
        // 영향받은 사용자 알림
        List<User> affectedUsers = findUsersWithChangesSince(
            restoreTimestamp
        );
        emailService.sendRestoreNotice(affectedUsers);
    }
}
```

</details>

<details>
<summary><strong>Q3: 일일 자동 백업으로 충분한가? 매시간 백업이 필요한가?</strong></summary>

**답**:

**백업 주기는 RPO(Recovery Point Objective)에 따름**:

```
RPO (목표 복구 지점):
├─ "최대 몇 시간 전 상태까지 복원 가능해야 하는가?"
└─ 이것이 백업 주기를 결정

RTO (목표 복구 시간):
├─ "복원에 몇 시간까지 걸려도 되는가?"
└─ 이것이 백업 방식을 결정
```

**시나리오별 권장 백업**:

```
전자상거래 (e-commerce):
├─ RPO: 15분 (시간당 수백 건 주문)
├─ RTO: 1시간 (사용자 손실 시 매출 손실)
├─ 권장: 15분마다 증분 백업 + 일일 전체 백업
└─ 도구: RDS 자동 백업 + xtrabackup 증분

은행/금융:
├─ RPO: 1분 (거래 손실 = 돈 손실)
├─ RTO: 30분 (규제 요구사항)
├─ 권장: 1분마다 Binlog 복제 (다중 마스터)
└─ 도구: MySQL Group Replication, Percona XtraDB Cluster

SaaS (중간 규모):
├─ RPO: 1시간 (사용자 수백 명)
├─ RTO: 4시간 (야간 장애는 덜 심함)
├─ 권장: 시간마다 스냅샷 + 일일 전체 백업
└─ 도구: RDS Snapshot (자동화)

개발/스테이징:
├─ RPO: 1일 (테스트 데이터)
├─ RTO: 1일 (비즈니스 영향 X)
├─ 권장: 일일 백업
└─ 도구: mysqldump (야간)
```

**일일 백업만으로 부족한 경우**:

```
예: 2024-01-15 13:00에 마이그레이션 실패

마지막 백업: 2024-01-15 00:00 (일일 백업)
│
├─ 00:00 ~ 13:00: 13시간의 데이터 손실
├─ 예: 1000건의 주문 손실
├─ 예: 100만 원의 매출 손실
└─ "일일 백업으로는 부족했다"

만약 매시간 백업:
├─ 마지막 백업: 2024-01-15 12:00
└─ 손실: 1시간만 (약 80건 주문)
```

**권장 최소 요구사항**:

```
프로덕션:
├─ 마이그레이션 당일: 마이그레이션 직전 + 직후 백업
├─ 평상시: 일일 자동 백업
├─ 중요 시스템: 시간마다 증분 백업 추가
└─ 매우 중요: 1분 단위 Binlog 복제 (RTO < 1시간)

테스트/스테이징:
├─ 마이그레이션 검증 직전: 백업
├─ 평상시: 주 1회 (또는 필요시)
└─ 데이터 초기화 쉬우면: 백업 불필요

개발:
├─ 거의 불필요 (로컬 DB)
├─ 팀 공유 개발 DB: 일일 백업
└─ 중요 데이터: 주 2회 백업
```

</details>

---

<div align="center">

**[⬅️ 이전: 마이그레이션 실패 시 복구 절차](./04-failure-recovery.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — 마이그레이션 버전 충돌 ➡️](../team-collaboration/01-version-conflict.md)**

</div>
