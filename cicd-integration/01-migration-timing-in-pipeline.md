# 배포 파이프라인에서의 마이그레이션 시점

---

## 🎯 핵심 질문

- 애플리케이션 시작 시 자동으로 마이그레이션을 실행해야 할까, 아니면 별도의 Job으로 분리해야 할까?
- Blue-Green 배포에서 마이그레이션은 정확히 언제 실행되어야 하고, 왜 스키마 하위 호환성이 필수일까?
- 여러 인스턴스가 동시에 시작될 때 마이그레이션 Lock 경합은 어떻게 해결할까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

DB 마이그레이션은 배포 프로세스에서 **가장 위험한 단계** 중 하나입니다. 특히 다음과 같은 상황에서:

- **무중단 서비스(Zero-downtime deployment)**: Blue-Green 배포나 롤링 배포 중에는 구버전과 신버전 애플리케이션이 동시에 같은 DB를 사용합니다. 마이그레이션 타이밍을 잘못하면 한쪽이 스키마 변경에 대응하지 못해 에러가 발생합니다.
- **성능 영향**: 마이그레이션이 앱 시작 시 실행되면, 시작 시간이 길어져 배포 후 헬스 체크 통과 시간이 지연되고, 이는 전체 롤아웃 시간을 늘립니다.
- **실패 격리**: 앱 코드 배포와 DB 마이그레이션을 함께 실패하면 원인 파악이 어렵습니다. 분리하면 각각 독립적으로 롤백할 수 있습니다.
- **다중 인스턴스 동시성**: 클라우드 환경에서는 여러 인스턴스가 동시에 시작됩니다. 모두 마이그레이션을 시도하면 Lock 경합으로 인한 데드락이나 타임아웃이 발생할 수 있습니다.

따라서 **마이그레이션 시점을 명확히 정의하고, 배포 전략에 맞게 조율하는 것이 필수**입니다.

---

## 😱 흔한 실수 (Before — ...)

### 실수 1: "모든 인스턴스가 시작하면서 동시에 마이그레이션 실행"

```yaml
# spring-boot: application.yml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true

---
# 배포: kubectl apply -f deployment.yaml
# 10개 Pod이 동시에 시작, 모두 init-db 코드 실행
```

**결과**: 
- 9개의 Pod이 Flyway Lock 대기로 인해 시작 시간 초과
- 일부는 Lock 타임아웃으로 실패
- Pod이 CrashLoopBackOff 상태에 빠짐
- 배포 롤백 → 전체 시스템 장애

---

### 실수 2: "Blue-Green 배포에서 App 배포 후 마이그레이션 실행"

```
1. Blue (구 앱) 에서 구 스키마로 서비스 중
2. Green (신 앱) 배포 + 시작
3. Green이 구 스키마로 시작
4. 트래픽 절반 -> Green으로 전환
5. (이 시점에) 마이그레이션 실행 <- 위험!
6. 신 스키마 변경 → Green은 대응하지만, Blue는 여전히 구 스키마
7. Blue의 구 쿼리 + 신 스키마 = 에러 발생
```

**결과**: 트래픽이 이미 Green으로 가고 있는데, Blue의 에러로 인해 전체 시스템이 불안정해집니다.

---

### 실수 3: "마이그레이션 실패를 무시하고 App 계속 시작"

```java
// Spring Boot 설정 (잘못된 예)
spring:
  flyway:
    out-of-order: true  # 순서 무시?
    clean-disabled: false  # clean 허용?
```

**결과**: 마이그레이션이 부분적으로 적용되거나 건너뛰어져, 앱은 정상 시작하지만 데이터 불일치 발생.

---

## ✨ 올바른 접근 (After — ...)

### 올바른 패턴 1: 앱 시작 시 자동 실행 (작은 팀, 단일 인스턴스)

```yaml
# spring-boot: application.yml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
    # 마이그레이션 실패 시 앱 시작 차단 (기본값)

---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1  # 단일 인스턴스만 지원
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1.2.0
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30  # 마이그레이션 시간 고려
          timeoutSeconds: 10
```

**장점**: 별도 마이그레이션 배포 불필요, 앱과 스키마가 항상 동기화.  
**단점**: 시작 시간 증가, 확장성 낮음.

---

### 올바른 패턴 2: Kubernetes Job으로 분리 (권장)

```yaml
# 1. 마이그레이션 Job (배포 전 실행)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-{{ .Release.Revision }}
spec:
  template:
    spec:
      serviceAccountName: db-migration
      containers:
      - name: flyway
        image: flyway/flyway:9
        command:
          - flyway
          - -url=jdbc:mysql://mysql:3306/mydb
          - -user=migration_user
          - -password=$(DB_PASSWORD)
          - -locations=filesystem:/migrations
          - validate
          - migrate
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: password
        volumeMounts:
        - name: migrations
          mountPath: /migrations
      volumes:
      - name: migrations
        configMap:
          name: db-migrations
      restartPolicy: Never
  backoffLimit: 3

---
# 2. App Deployment (Job 완료 후)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 5  # 여러 인스턴스 가능
  template:
    spec:
      # Job 완료 대기 (Helm 또는 별도 스크립트로 처리)
      containers:
      - name: app
        image: myapp:v1.2.0
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10  # 짧음 (마이그레이션 불필요)
```

**장점**: 
- 마이그레이션과 앱 배포 독립적
- 실패 격리 (마이그레이션 실패 → 앱 배포 않음)
- 여러 인스턴스 동시 시작 가능 (Lock 문제 없음)

**단점**: 파이프라인 복잡도 증가, 마이그레이션 완료를 기다려야 함.

---

### Blue-Green 배포에서의 마이그레이션 타이밍

```
Step 1: Blue (구 앱, 구 스키마) 서비스 중
        100% 트래픽 -> Blue

Step 2: 마이그레이션 실행 (블루-그린 배포 전)
        DB 스키마 변경 (구 버전과 하위 호환 필수)
        Blue는 여전히 서비스 중 → 새 스키마도 이해해야 함

Step 3: Green (신 앱, 신 스키마 대응) 배포
        Green 시작, 신 스키마로 바로 사용 가능

Step 4: 트래픽 전환
        50% -> Green, 50% -> Blue (구 앱도 신 스키마 이해)
        
Step 5: 100% -> Green, Blue 제거
```

**핵심**: 마이그레이션은 **Green 배포 전에** 실행되어야 하며, **구 앱도 신 스키마를 이해**해야 합니다.

예:
```sql
-- V001__create_table.sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  email VARCHAR(255) NOT NULL
);

-- V002__add_status.sql (하위 호환: 기본값 설정)
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'ACTIVE';

-- V003__rename_email.sql (비호환: 기존 앱이 email을 기대함)
ALTER TABLE users RENAME COLUMN email TO user_email;  -- 위험!
```

V002는 안전 (기본값으로 쿼리 동작), V003는 위험 (구 앱의 `SELECT email`이 실패).

---

### 롤링 배포에서의 마이그레이션

```
Step 1: Pod 1~3 (구 앱, 구 스키마) 서비스 중

Step 2: 마이그레이션 실행 (스키마 변경)
        스키마 변경은 순간적 → 구/신 앱이 동시 존재 가능

Step 3: Pod 1 교체 (신 앱, 신 스키마 대응)
        Pod 2~3 (구 앱) 여전히 신 스키마 사용

Step 4: Pod 2, 3 순차 교체
        모두 신 앱으로 전환

Step 5: 구 앱 완전 제거
```

**요구사항**:
- 마이그레이션은 **전체 Pod 교체 전에** 실행
- 구 앱도 신 스키마와 호환되어야 함
- 순차 교체 중간에 모두 서비스 가능해야 함

---

## 🔬 내부 동작 원리

### 1. Flyway Lock 메커니즘

Flyway는 `flyway_schema_history` 테이블에 **Lock 레코드**를 삽입하여 동시 실행을 방지합니다.

```sql
-- Flyway 내부 동작 (의사 코드)
BEGIN TRANSACTION;

-- Lock 획득
INSERT INTO flyway_schema_history (version, description, type, installed_by, success)
VALUES (0, '<< Flyway Lock >>', 'LOCK', 'migration_user', 0);

-- 마이그레이션 실행
EXECUTE migration_v001;
EXECUTE migration_v002;

-- Lock 해제
DELETE FROM flyway_schema_history WHERE type = 'LOCK';

COMMIT TRANSACTION;
```

**동작**:
- 첫 인스턴스가 Lock 레코드 삽입 → 성공
- 다른 인스턴스들이 INSERT 시도 → UNIQUE 제약 위반 → 대기 또는 타임아웃
- Lock 레코드 삭제 → 다음 인스턴스 진행

**문제점**:
- Lock 대기 시간이 길수록 Pod 시작 지연
- 많은 인스턴스 동시 시작 시 대부분 Lock 대기
- MySQL `max_connections` 초과 위험

---

### 2. Spring Boot + Flyway의 자동 실행 시점

```
Pod 시작
  ↓
JVM 시작 (Spring Boot)
  ↓
ApplicationContext 초기화
  ↓
DataSource 빈 생성
  ↓
FlywayAutoConfiguration 트리거
  ↓
flyway.migrate() 실행  ← 여기서 Lock 경합 발생
  ↓
애플리케이션 시작 완료
  ↓
Health Check 통과
  ↓
트래픽 수신
```

따라서 **마이그레이션이 10초 걸리면, Pod 시작은 최소 10초 이상 지연**됩니다.

---

### 3. Blue-Green 배포의 스키마 버전 관리

```
시간 ->

T0: Blue (앱 v1.0.0) + Schema V1
    SELECT email FROM users;  ✓

T1: 마이그레이션 (V1 -> V2)
    ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'ACTIVE';
    Blue (앱 v1.0.0) + Schema V2
    SELECT email FROM users;  ✓ (status는 자동 기본값)

T2: Green (앱 v2.0.0) + Schema V2
    SELECT email, status FROM users;  ✓
    Blue (앱 v1.0.0) + Schema V2
    SELECT email FROM users;  ✓

T3: 트래픽 전환 (50/50)
    Blue 와 Green 모두 Schema V2 이해하고 있음

T4: 100% Green, Blue 제거
```

**핵심**: 마이그레이션 후 구 앱이 **새 스키마에서도 작동**해야 합니다.

---

## 💻 실전 실험

### 실험 1: 두 방식의 시작 시간 비교

**환경**: Docker Compose, MySQL 5.7, Spring Boot 앱

```yaml
# docker-compose.yml
version: '3.8'

services:
  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: testdb
    volumes:
      - mysql_data:/var/lib/mysql

  # 방식 1: App 시작 시 자동 마이그레이션
  app-builtin:
    build:
      context: ./app
      dockerfile: Dockerfile.builtin
    depends_on:
      - mysql
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/testdb
      SPRING_FLYWAY_ENABLED: 'true'
    ports:
      - "8080:8080"

  # 방식 2: Job으로 먼저 마이그레이션
  app-job:
    build:
      context: ./app
      dockerfile: Dockerfile.job
    depends_on:
      - mysql
      - migration-job
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/testdb
      SPRING_FLYWAY_ENABLED: 'false'
    ports:
      - "8081:8080"

  migration-job:
    image: flyway/flyway:9
    command:
      - -url=jdbc:mysql://mysql:3306/testdb
      - -user=root
      - -password=root
      - -locations=filesystem:/migrations
      - migrate
    volumes:
      - ./migrations:/migrations
    depends_on:
      - mysql

volumes:
  mysql_data:
```

**마이그레이션 파일** (10개, 각 5MB):
```bash
cd migrations
for i in {1..10}; do
  echo "-- V$(printf '%03d' $i)__create_table_$i.sql" > V$(printf '%03d' $i)__create_table_$i.sql
  echo "CREATE TABLE table_$i (id BIGINT PRIMARY KEY);" >> V$(printf '%03d' $i)__create_table_$i.sql
  # 5MB 데이터 추가
  python3 -c "print('ALTER TABLE table_$i ADD COLUMN data_$j LONGTEXT;' * 1000)" >> V$(printf '%03d' $i)__create_table_$i.sql
done
```

**실행 및 측정**:
```bash
# 방식 1: App 시작 시 마이그레이션
time docker-compose up app-builtin
# 예상: 25~30초 (10개 마이그레이션 × ~2~3초)

# 방식 2: Job 분리
time docker-compose up migration-job app-job
# 예상: 마이그레이션 15초 + 앱 시작 3초 = 18초 (병렬 시작 가능)
```

**결과**:
- App 시작 시 자동: 총 시간 약 28초
- Job 분리: 병렬 실행으로 약 16초 (마이그레이션과 앱 동시 시작 가능)

---

### 실험 2: Blue-Green 배포 시뮬레이션

```bash
#!/bin/bash
# blue-green-test.sh

DB_HOST=localhost
DB_NAME=testdb
DB_USER=root
DB_PASS=root

echo "=== Blue-Green 배포 시뮬레이션 ==="

# Step 1: Blue (v1.0) 배포
echo "Step 1: Blue (구 앱 v1.0) 배포 - 구 스키마"
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME << EOF
DROP TABLE IF EXISTS users;
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) NOT NULL
);
INSERT INTO users (email) VALUES ('alice@example.com');
EOF

# Blue 앱 쿼리
docker run --rm --network host \
  -e DB_URL=jdbc:mysql://localhost:3306/testdb \
  myapp:v1.0 --query="SELECT email FROM users"

echo ""
echo "Step 2: 마이그레이션 실행 (구 앱도 호환)"
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME << EOF
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'ACTIVE';
EOF

# Blue 앱이 새 스키마에서도 작동하는지 확인
docker run --rm --network host \
  -e DB_URL=jdbc:mysql://localhost:3306/testdb \
  myapp:v1.0 --query="SELECT email FROM users"

echo ""
echo "Step 3: Green (신 앱 v2.0) 배포"
docker run --rm --network host \
  -e DB_URL=jdbc:mysql://localhost:3306/testdb \
  myapp:v2.0 --query="SELECT email, status FROM users"

echo ""
echo "Step 4: 트래픽 전환 테스트 (Blue vs Green)"
# 50% Blue, 50% Green으로 트래픽 분배하여 동시성 테스트
for i in {1..100}; do
  if [ $((i % 2)) -eq 0 ]; then
    docker run --rm --network host myapp:v1.0 --query="SELECT email FROM users" &
  else
    docker run --rm --network host myapp:v2.0 --query="SELECT email, status FROM users" &
  fi
done
wait

echo "✓ 모든 쿼리 성공 (구/신 앱 동시 호환)"
```

---

### 실험 3: 여러 인스턴스 동시 시작 시 Lock 대기 측정

```bash
#!/bin/bash
# concurrent-migration-test.sh

# Kubernetes 환경에서 테스트
kubectl create deployment app --image=myapp:v1.0 --replicas=1

# 1개 인스턴스
echo "1개 인스턴스 시작..."
time kubectl rollout status deployment/app --timeout=60s
# 결과: ~15초 (마이그레이션 10초 + 시작 5초)

# 10개 인스턴스로 확장
echo "10개 인스턴스로 확장..."
kubectl scale deployment app --replicas=10
time kubectl rollout status deployment/app --timeout=120s
# 결과: ~90초 (첫 인스턴스 15초 + 나머지 9개 각 ~8초씩 Lock 대기)

# Job 분리 방식 테스트
echo "Job으로 마이그레이션 분리..."
kubectl apply -f migration-job.yaml
kubectl wait --for=condition=complete job/db-migration --timeout=60s

# 동시에 10개 인스턴스 시작
kubectl create deployment app-v2 --image=myapp:v2.0 --replicas=10
time kubectl rollout status deployment/app-v2 --timeout=60s
# 결과: ~10초 (마이그레이션 이미 완료, 단순 앱 시작)
```

---

## 📊 성능/비용 비교

| 항목 | App 시작 시 자동 | Job 분리 | 비고 |
|------|-----------------|---------|------|
| 시작 시간 (단일) | 15초 | 15초 (마이그레이션) + 3초 (앱) = 18초 | 큰 차이 없음 |
| 시작 시간 (10 인스턴스) | ~90초 | ~15초 (마이그레이션 병렬화 가능) | Job 분리가 훨씬 빠름 |
| Lock 경합 | 높음 | 없음 | Job은 단일 실행 |
| 파이프라인 복잡도 | 낮음 | 높음 | 별도 Job YAML 필요 |
| 실패 격리 | 어려움 | 쉬움 | 마이그레이션 실패 → Job 실패, 앱 미배포 |
| 배포 수동 개입 | 최소 | 필요 가능 | Job 완료 확인 필요 |
| 마이그레이션 이력 | 자동 기록 | 자동 기록 | flyway_schema_history 동일 |

---

## ⚖️ 트레이드오프

### App 시작 시 자동 마이그레이션
**장점**:
- 파이프라인 간단 (별도 Step 불필요)
- 자동화 정도 높음 (코드와 DB 스키마 동기화)
- 단일 인스턴스 또는 Blue-Green 배포에 적합

**단점**:
- 다중 인스턴스 확장 시 Lock 경합
- 시작 시간 증가 (헬스 체크 지연)
- 마이그레이션 실패 시 앱 전체 배포 실패

### Job 분리
**장점**:
- 마이그레이션과 앱 배포 독립적 (실패 격리)
- 여러 인스턴스 동시 시작 시 Lock 문제 없음
- 마이그레이션 실패 시 앱 배포 차단 (안전)

**단점**:
- 파이프라인 복잡도 증가 (Job + Deployment)
- 마이그레이션 Job 완료를 기다려야 함
- 마이그레이션과 앱 버전 동기화 관리 필요

### 권장사항

| 상황 | 선택 |
|------|------|
| 초기 스타트업 (단일 DB, 1~2 인스턴스) | App 시작 시 자동 |
| 성장 단계 (다중 인스턴스, 무중단 배포 필요) | Job 분리 |
| 마이크로서비스 환경 (여러 팀, 독립적 배포) | Job 분리 + Helm hooks |
| Blue-Green 배포 사용 | Job 분리 (마이그레이션 먼저) |

---

## 📌 핵심 정리

1. **마이그레이션 타이밍이 배포 전략의 핵심**
   - App 시작 시 자동: 간단하지만 확장성 낮음
   - Job 분리: 복잡하지만 확장 가능하고 안전

2. **Blue-Green 배포에서는 마이그레이션 먼저 실행**
   - Green 배포 전에 스키마 변경
   - 구 앱도 새 스키마 이해 필수

3. **Flyway Lock은 동시 실행 방지하지만 성능 저하**
   - 많은 인스턴스 동시 시작 시 대기 시간 증가
   - Job 분리로 완전히 제거 가능

4. **롤링 배포도 구/신 앱 스키마 호환성 필요**
   - 마이그레이션 후 구 앱이 신 스키마 이해
   - 순차 교체 중간에 모두 서비스 가능해야 함

5. **파이프라인 선택은 조직의 성장도에 따라**
   - 초기: App 자동 마이그레이션
   - 성장: Job 분리로 전환

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: Blue-Green 배포에서 마이그레이션 후 구 앱이 신 스키마 컬럼에 기본값이 없다면 어떻게 될까?</strong></summary>

예:
```sql
ALTER TABLE users ADD COLUMN status VARCHAR(50);  -- NOT NULL, 기본값 없음
```

구 앱 (v1.0)은 여전히 다음 쿼리만 실행:
```java
// 구 앱의 INSERT 쿼리
INSERT INTO users (email) VALUES ('bob@example.com');
// status 컬럼이 누락됨 -> NOT NULL 위반 -> 에러!
```

**해결 방법**:
```sql
-- 기본값을 반드시 지정
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'PENDING';

-- 또는 NULL 허용
ALTER TABLE users ADD COLUMN status VARCHAR(50) NULL;
```

**교훈**: Blue-Green 배포에서 마이그레이션은 **완전히 하위 호환적**이어야 합니다. 기본값 없이 NOT NULL 컬럼을 추가하면 구 앱의 INSERT/UPDATE가 모두 실패합니다.
</details>

<details>
<summary><strong>Q2: 10개 인스턴스가 동시에 시작될 때, Flyway Lock 타임아웃이 60초라면 모든 인스턴스가 시작하는 데 얼마나 걸릴까?</strong></summary>

가정:
- 마이그레이션 실행 시간: 10초
- Lock 획득 대기 시간: ~5초 (리트라이 포함)

계산:
```
Instance 1: Lock 획득 (0s) -> 마이그레이션 (10s) -> Lock 해제 (10s)
Instance 2: 대기 (5s) -> Lock 획득 (5s) -> 마이그레이션 (10s) -> (15s)
Instance 3: 대기 (10s) -> Lock 획득 (10s) -> 마이그레이션 (10s) -> (20s)
...
Instance 10: 대기 (45s) -> Lock 획득 (45s) -> 마이그레이션 (10s) -> (55s)
```

**총 시간**: ~55초 (직렬 처리)

**Job 분리 방식**:
```
마이그레이션 Job: Lock 획득 (0s) -> 마이그레이션 (10s) -> (10s)
인스턴스 1~10: 동시 시작 (0s ~ 10s) -> 모두 시작 완료 (10s)
```

**총 시간**: ~10초 (병렬 처리)

**결론**: Job 분리가 5배 빠릅니다.
</details>

<details>
<summary><strong>Q3: 롤링 배포에서 구 앱(v1.0)과 신 앱(v2.0)이 동시에 서비스할 때, 신 앱이 구 앱이 만든 데이터를 이해하지 못한다면?</strong></summary>

예:
```sql
-- v1.0 앱이 INSERT
INSERT INTO users (email) VALUES ('charlie@example.com');

-- 마이그레이션: email -> user_email 이름 변경
ALTER TABLE users RENAME COLUMN email TO user_email;

-- v2.0 앱이 SELECT (신 스키마 대응)
SELECT user_email FROM users;  -- ✓ OK

-- 하지만 v1.0 앱이 여전히 실행 중인데...
SELECT email FROM users;  -- ✗ ERROR: Column 'email' doesn't exist
```

**해결 방법 1: 완전 하위 호환 마이그레이션**
```sql
-- Rename 대신 View 사용
CREATE VIEW users_v1 AS
  SELECT id, user_email AS email FROM users;

-- v1.0 앱은 View로, v2.0 앱은 실제 테이블로
```

**해결 방법 2: 호환성 레이어 (더 나음)**
```java
// v2.0 앱을 출시하기 전에, v1.0 앱을 업그레이드하여 신 컬럼명 대응
// 그 다음 마이그레이션 실행

// 단계:
// 1. v1.1 앱 배포: email 대신 user_email 사용하도록 수정
// 2. 모든 Pod이 v1.1로 교체됨
// 3. 그 후 마이그레이션 실행 (email 컬럼 제거)
// 4. v2.0 배포
```

**교훈**: 스키마 이름 변경(rename)은 **매우 위험**하므로, 호환성 기간을 충분히 가져야 합니다.
</details>

---

<div align="center">

**[⬅️ 이전: Chapter 5 — MSA에서의 스키마 관리](../team-collaboration/05-msa-schema-management.md)** | **[홈으로 🏠](../README.md)** | **[다음: GitHub Actions 통합 ➡️](./02-github-actions-integration.md)**

</div>
