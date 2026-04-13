# 마이그레이션 환경 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 로컬/개발/스테이징/프로덕션 각 환경에서 마이그레이션을 어떻게 다르게 설정해야 하는가?
- `clean-disabled=true`는 무엇을 막고, 왜 프로덕션에서 필수인가?
- 환경별 시드 데이터를 Repeatable 마이그레이션으로 어떻게 분리하는가?
- 스테이징에서 검증 후 프로덕션에 적용하는 파이프라인 구조는?
- 테스트 환경에서 격리된 마이그레이션을 실행하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이그레이션 도구는 모든 환경에 동일한 SQL을 실행한다. 하지만 환경마다 목적이 다르다. 로컬은 빠른 개발이 목표이고, 프로덕션은 데이터 안전이 최우선이다. 이 차이를 설정에 반영하지 않으면, 개발자의 실수로 `flyway clean`이 프로덕션에서 실행되거나, 테스트용 시드 데이터가 프로덕션에 들어가는 사고가 발생한다. 환경별 전략은 마이그레이션 도구의 가장 실용적인 설계 결정이다.

---

## 😱 흔한 실수 (Before — 환경별 설정 없이 동일하게 적용)

```
사고 유형 1: flyway clean이 프로덕션에서 실행됨

  배경:
    개발자가 로컬에서 "처음부터 다시 시작"하려고 flyway clean 실행
    이 명령어가 CI/CD 파이프라인에도 포함됨 (복붙 실수)
    또는 spring.flyway.clean-on-validation-error=true 설정이 프로덕션에 적용됨

  결과:
    flyway clean = DB의 모든 테이블, 뷰, 프로시저 DROP
    프로덕션 데이터 전체 삭제
    복구: 가장 최근 백업으로 Point-In-Time Recovery
    → 수 시간의 다운타임, 일부 데이터 손실

  예방:
    spring.flyway.clean-disabled=true (프로덕션 필수)
    clean-on-validation-error=false (기본값이지만 명시적으로 설정)

사고 유형 2: 테스트 시드 데이터가 프로덕션에 삽입됨

  배경:
    R__seed_test_data.sql (Repeatable 마이그레이션)
    로컬/개발에서 더미 데이터를 자동 삽입하는 목적
    locations 설정에 실수로 시드 데이터 경로가 프로덕션에도 포함됨

  결과:
    사용자 테이블에 "테스트 유저", "admin@test.com" 계정 생성
    orders 테이블에 더미 주문 데이터 수천 건 삽입
    프로덕션 데이터와 혼합 → 정합성 깨짐
    수동으로 더미 데이터 식별 후 삭제 작업 수시간 소요

사고 유형 3: 스테이징 건너뛰고 바로 프로덕션 적용

  배경:
    "스테이징이랑 프로덕션 스키마가 어차피 같으니까" → 스테이징 생략
    대형 마이그레이션(수천만 건 데이터 변환)을 프로덕션에 처음 적용

  결과:
    스테이징에서 10분 예상이었는데 프로덕션(데이터 10배)에서 2시간 소요
    그동안 테이블 Lock → 서비스 중단
    스테이징에서 먼저 실행 시간을 측정했어야 함
```

---

## ✨ 올바른 접근 (After — 환경별 분리 설정)

```
환경별 Flyway 설정 전략:

로컬 개발:
  목표: 빠른 프로토타이핑, 자유로운 리셋
  ddl-auto: create-drop 또는 update
  flyway: 비활성화 (선택적)
  시드 데이터: R__seed_local_data.sql 포함
  clean: 허용 (개인 환경이므로 안전)

개발(팀 공유 서버):
  목표: 팀 통합 검증, 마이그레이션 첫 적용
  ddl-auto: validate
  flyway: 활성화
  시드 데이터: R__seed_dev_data.sql 포함 (더미 데이터 OK)
  clean: 비활성화 (clean-disabled=true)

스테이징:
  목표: 프로덕션과 동일한 환경 검증
  ddl-auto: validate
  flyway: 활성화
  시드 데이터: 없음 (프로덕션과 동일 조건)
  clean: 비활성화
  특이사항: 프로덕션 마이그레이션 전 타임 측정

프로덕션:
  목표: 데이터 안전, 서비스 가용성
  ddl-auto: validate
  flyway: 활성화
  시드 데이터: 없음
  clean: 반드시 비활성화 (clean-disabled=true)
  특이사항: 마이그레이션 전 스냅샷 필수
```

---

## 🔬 내부 동작 원리

### 1. Spring Profile별 Flyway 설정 분리

```yaml
# application.yml (공통 기본값)
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
    locations: classpath:db/migration
    clean-disabled: true          # 모든 환경 기본값
    validate-on-migrate: true
    out-of-order: false

---
# application-local.yml (로컬 개발)
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/myapp_dev
    username: root
    password: root
  jpa:
    hibernate:
      ddl-auto: create-drop        # 로컬만 허용
  flyway:
    enabled: false                 # 로컬은 JPA가 직접 관리
    # 또는 Flyway를 사용하되 시드 데이터 포함:
    # enabled: true
    # locations:
    #   - classpath:db/migration
    #   - classpath:db/seed/local
    # clean-disabled: false        # 로컬만 허용

---
# application-dev.yml (개발 서버)
spring:
  datasource:
    url: jdbc:mysql://dev-db:3306/myapp
  flyway:
    locations:
      - classpath:db/migration
      - classpath:db/seed/dev     # 개발 시드 데이터 포함
    clean-disabled: true

---
# application-staging.yml (스테이징)
spring:
  datasource:
    url: jdbc:mysql://staging-db:3306/myapp
  flyway:
    locations: classpath:db/migration   # 시드 데이터 없음
    clean-disabled: true
    # 프로덕션과 동일한 마이그레이션만 적용

---
# application-prod.yml (프로덕션)
spring:
  datasource:
    url: jdbc:mysql://prod-db:3306/myapp
  flyway:
    locations: classpath:db/migration
    clean-disabled: true              # 절대 데이터 삭제 불가
    validate-on-migrate: true
    out-of-order: false               # 순서 이탈 마이그레이션 불허
```

### 2. 환경별 시드 데이터 구조

```
디렉토리 구조:

src/main/resources/db/
├── migration/              ← 모든 환경에 적용
│   ├── V20240101000000__create_users.sql
│   ├── V20240102000000__create_orders.sql
│   └── V20240103000000__add_index_on_orders.sql
│
└── seed/
    ├── dev/                ← 개발 서버 전용
    │   └── R__seed_dev_users.sql
    ├── local/              ← 로컬 개발 전용
    │   └── R__seed_local_data.sql
    └── test/               ← 테스트 환경 전용
        └── R__seed_test_fixtures.sql

---
# db/seed/dev/R__seed_dev_users.sql (Repeatable)
-- 개발 서버용 테스트 계정
-- ⚠️ 이 파일은 절대 프로덕션에 적용되지 않음 (locations 분리)

DELETE FROM users WHERE email LIKE '%@dev.test';

INSERT INTO users (name, email, status, created_at) VALUES
    ('개발 관리자', 'admin@dev.test', 'ACTIVE', NOW()),
    ('테스트 유저1', 'user1@dev.test', 'ACTIVE', NOW()),
    ('테스트 유저2', 'user2@dev.test', 'INACTIVE', NOW());

---
# db/seed/local/R__seed_local_data.sql (로컬용)
-- 로컬 개발 시 빠른 테스트를 위한 더미 데이터

TRUNCATE TABLE orders;
TRUNCATE TABLE users;

INSERT INTO users (name, email, status) VALUES
    ('로컬 유저', 'local@test.com', 'ACTIVE');

INSERT INTO orders (user_id, status, total_amount) VALUES
    (1, 'COMPLETED', 50000),
    (1, 'PENDING', 30000);
```

### 3. clean-disabled의 내부 동작

```
flyway clean 명령어의 실제 동작:

  1. DB의 모든 스키마 오브젝트 목록 조회
     SELECT table_name FROM information_schema.tables WHERE table_schema = 'myapp';

  2. 의존성 역순으로 DROP:
     DROP TABLE order_items;     ← orders에 의존
     DROP TABLE orders;
     DROP TABLE users;
     DROP VIEW monthly_sales;
     DROP INDEX ...;
     DROP PROCEDURE ...;
     DROP SEQUENCE ...;  (PostgreSQL)

  3. flyway_schema_history 테이블도 DROP

  결과: 완전히 빈 DB
  → 다음 flyway migrate 실행 시 처음부터 재생성

clean-disabled=true 설정 효과:
  flyway clean 명령어 호출 시:
    "ERROR: Unable to execute clean as it has been disabled
     with the 'cleanDisabled' option."
  → 명령어 자체가 오류로 차단됨

clean-on-validation-error 위험:
  spring.flyway.clean-on-validation-error=true 설정 시:
    체크섬 불일치 오류 발생 → 자동으로 clean 후 재마이그레이션
    개발 환경에서 편리하지만 절대로 프로덕션에 적용하면 안 됨
    → 마이그레이션 파일 변경 한 번으로 프로덕션 DB 전체 삭제 가능
```

### 4. 스테이징 → 프로덕션 검증 파이프라인

```
안전한 프로덕션 마이그레이션 절차:

단계 1: 스테이징 사전 검증
  스테이징 DB (프로덕션과 동일한 스키마)에 마이그레이션 적용
  측정 항목:
    ① 실행 시간: 스테이징에서 N초 → 프로덕션(데이터 M배)에서 N*M초 예상
    ② Lock 발생 여부: information_schema.innodb_trx 모니터링
    ③ 오류 여부: flyway_schema_history의 success=false 확인
    ④ 롤백 계획: 실패 시 복구 절차 검증

단계 2: 프로덕션 사전 준비
  DB 스냅샷 생성 (RDS: Manual Snapshot, 자체 MySQL: mysqldump)
  저트래픽 시간대 선택 (새벽 3-5시)
  모니터링 대시보드 열기 (Lock 모니터링 쿼리 준비)

단계 3: 프로덕션 적용
  flyway migrate 실행 (또는 Spring Boot 배포로 자동 실행)
  모니터링:
    SELECT * FROM performance_schema.metadata_locks WHERE OBJECT_TYPE='TABLE';
    SELECT * FROM information_schema.innodb_trx;

단계 4: 검증
  flyway info로 성공 확인
  애플리케이션 헬스체크
  주요 API 스모크 테스트
```

---

## 💻 실전 실험

### 실험 1: 환경별 설정 적용 확인

```bash
# 로컬에서 profiles 확인
./mvnw spring-boot:run -Dspring-boot.run.profiles=local

# 개발 서버 배포 시
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# 스테이징 배포
./mvnw spring-boot:run -Dspring-boot.run.profiles=staging

# 프로덕션 배포
./mvnw spring-boot:run -Dspring-boot.run.profiles=prod

# 각 환경에서 flyway info로 적용된 마이그레이션 확인
flyway -url=jdbc:mysql://dev-db:3306/myapp -user=${DB_USER} -password=${DB_PASS} info
```

### 실험 2: clean-disabled 동작 확인

```bash
# clean-disabled=true 설정 후 clean 시도
flyway -url=jdbc:mysql://localhost:3306/myapp \
  -user=root -password=root \
  -cleanDisabled=true \
  clean

# 출력:
# ERROR: Unable to execute clean as it has been disabled
# with the 'cleanDisabled' option. (set 'flyway.cleanDisabled = false' to enable)

# 로컬 환경에서만 clean 허용
flyway -url=jdbc:mysql://localhost:3306/myapp_local \
  -user=root -password=root \
  -cleanDisabled=false \
  clean
# 성공 — 로컬 DB 초기화
```

### 실험 3: 시드 데이터 환경별 분리 확인

```bash
# 개발 환경 (시드 포함)
flyway -url=jdbc:mysql://dev-db:3306/myapp \
  -user=root -password=root \
  -locations="filesystem:db/migration,filesystem:db/seed/dev" \
  migrate

# DB에서 시드 데이터 확인
mysql -h dev-db -uroot -proot myapp -e "SELECT email FROM users WHERE email LIKE '%@dev.test';"
# +-------------------+
# | email             |
# +-------------------+
# | admin@dev.test    |
# | user1@dev.test    |
# +-------------------+

# 프로덕션 환경 (시드 없음)
flyway -url=jdbc:mysql://prod-db:3306/myapp \
  -user=${PROD_DB_USER} -password=${PROD_DB_PASS} \
  -locations="filesystem:db/migration" \
  migrate
# 시드 마이그레이션이 locations에 없으므로 적용되지 않음
```

### 실험 4: 스테이징에서 마이그레이션 실행 시간 측정

```bash
#!/bin/bash
# 스테이징 마이그레이션 시간 측정 스크립트

echo "=== 스테이징 마이그레이션 시작 ==="
START_TIME=$(date +%s)

flyway -url=jdbc:mysql://staging-db:3306/myapp \
  -user=${STAGING_DB_USER} \
  -password=${STAGING_DB_PASS} \
  migrate

END_TIME=$(date +%s)
ELAPSED=$((END_TIME - START_TIME))

echo "=== 마이그레이션 완료 ==="
echo "소요 시간: ${ELAPSED}초"
echo "프로덕션 예상 시간: $((ELAPSED * PROD_DATA_RATIO))초"
echo "(프로덕션 데이터가 스테이징의 ${PROD_DATA_RATIO}배인 경우)"

# Lock 모니터링 (별도 세션에서 실행)
while true; do
  LOCKS=$(mysql -h staging-db -u${STAGING_DB_USER} -p${STAGING_DB_PASS} \
    -e "SELECT COUNT(*) FROM performance_schema.metadata_locks WHERE LOCK_STATUS='WAITING';" \
    --skip-column-names 2>/dev/null)
  if [ "$LOCKS" -gt "0" ]; then
    echo "⚠️  Lock 대기 감지: ${LOCKS}개 세션 대기 중"
  fi
  sleep 1
done &
MONITOR_PID=$!

# 마이그레이션 완료 후 모니터링 종료
wait
kill $MONITOR_PID 2>/dev/null
```

---

## 📊 비교

```
환경별 Flyway 설정 요약:

설정 항목              | 로컬      | 개발 서버 | 스테이징  | 프로덕션
─────────────────────┼──────────┼──────────┼──────────┼──────────
flyway.enabled       | false/true| true     | true     | true
ddl-auto             | create-drop| validate | validate | validate
clean-disabled       | false    | true     | true     | true (필수!)
시드 데이터 위치       | db/seed/local| db/seed/dev| 없음  | 없음
out-of-order         | true     | true     | false    | false
validate-on-migrate  | false    | true     | true     | true
배포 전 스냅샷        | 불필요   | 선택     | 권장     | 필수
```

---

## ⚖️ 트레이드오프

```
환경별 설정 관리 비용 vs 안전성:

비용:
  환경별 application-{profile}.yml 파일 관리
  시드 데이터 파일 별도 관리 (DB 스키마 변경 시 시드도 함께 수정)
  CI/CD 파이프라인에서 환경별 profile 주입 설정

안전성:
  clean-disabled으로 프로덕션 데이터 삭제 사고 방지
  시드 데이터 격리로 더미 데이터 프로덕션 유입 방지
  스테이징 선 검증으로 프로덕션 적용 전 문제 발견

현실적 판단:
  환경이 2개(로컬 + 프로덕션)뿐이라도 clean-disabled는 필수
  시드 데이터 분리는 3개 이상 환경, 팀 작업 시 필요
  스테이징 검증은 대형 마이그레이션, 장기 운영 서비스에서 필수
```

---

## 📌 핵심 정리

```
환경별 마이그레이션 전략 핵심:

1. clean-disabled=true — 프로덕션 절대 원칙
   flyway clean = DB 전체 삭제
   프로덕션에서 clean 실행 = 데이터 전체 손실
   → clean-disabled=true로 명령어 자체를 차단

2. 시드 데이터 격리 — locations로 경로 분리
   db/migration/ → 모든 환경
   db/seed/dev/  → 개발만
   db/seed/local/→ 로컬만
   → locations 설정으로 환경별 포함 경로 제어

3. 스테이징 선 검증 — 프로덕션 전 타임 측정
   스테이징에서 실행 시간, Lock 여부 확인
   프로덕션 데이터 배수만큼 예상 시간 계산
   저트래픽 시간대에 프로덕션 적용

4. Spring Profile로 자동 분기
   application-{profile}.yml 파일로 환경별 설정
   CI/CD에서 SPRING_PROFILES_ACTIVE 환경변수 주입
```

---

## 🤔 생각해볼 문제

**Q1.** 로컬에서는 `flyway.enabled=false`로 설정하고 `ddl-auto=create-drop`을 사용한다. 그런데 팀에서 새로운 마이그레이션이 추가됐을 때 로컬에서는 어떻게 반영해야 하는가?

<details>
<summary>해설 보기</summary>

`flyway.enabled=false` + `ddl-auto=create-drop` 설정에서 로컬 DB는 매번 앱 시작 시 Entity 기준으로 처음부터 재생성됩니다. 따라서 마이그레이션 파일이 추가됐을 때 로컬에서 별도 작업이 필요 없습니다. 앱을 재시작하면 Hibernate가 Entity를 다시 분석해 테이블을 재생성합니다.

단, 이 방식에는 두 가지 주의점이 있습니다.

첫째, `create-drop`은 매번 DB를 초기화하므로 로컬에서 수동으로 입력한 테스트 데이터가 사라집니다. 이 때문에 `R__seed_local_data.sql`(Repeatable)을 함께 사용하거나, 로컬도 `flyway.enabled=true`로 하고 `create-drop` 대신 `validate`를 사용하기도 합니다.

둘째, 로컬 DB의 스키마 상태가 마이그레이션 이력을 반영하지 않으므로, 로컬에서 "마이그레이션이 실제로 잘 작동하는가"를 검증할 수 없습니다. 개발 서버에서 첫 적용 시 발생하는 오류를 로컬에서 미리 발견하지 못할 수 있습니다.

이런 이유로 일부 팀은 로컬에서도 Flyway를 활성화하고, `flyway clean` + `flyway migrate` 조합으로 로컬을 초기화하는 방식을 택합니다.

</details>

---

**Q2.** 스테이징 DB는 프로덕션과 완전히 동일한 스키마를 유지해야 하는가? 그렇다면 프로덕션에만 있는 데이터(실제 사용자 데이터)를 스테이징에서 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

스테이징 DB의 **스키마**는 프로덕션과 동일해야 합니다. 마이그레이션이 프로덕션과 같은 순서로 적용돼야 "이 마이그레이션이 프로덕션에도 안전하게 적용된다"는 것을 검증할 수 있기 때문입니다.

**데이터**는 두 가지 전략으로 처리합니다.

1. **익명화된 복제 데이터**: 프로덕션 데이터를 개인정보를 마스킹(이메일 → `***@masked.com`, 이름 → 가명)해서 스테이징에 복제합니다. 프로덕션과 동일한 데이터 볼륨으로 마이그레이션 시간을 정확히 측정할 수 있습니다. 단, 주기적인 동기화 작업이 필요합니다.

2. **합성 데이터**: 실제 데이터와 유사한 볼륨의 생성된 더미 데이터를 사용합니다. 개인정보 리스크가 없지만 실제 데이터 분포와 다를 수 있어 마이그레이션 시간 예측이 부정확할 수 있습니다.

마이그레이션 시간 측정이 중요한 경우(대용량 테이블 DDL, 수천만 건 백필)에는 프로덕션과 비슷한 볼륨의 스테이징 DB가 필수입니다.

</details>

---

**Q3.** 프로덕션 마이그레이션 실패 시 가장 먼저 해야 할 행동은 무엇인가?

<details>
<summary>해설 보기</summary>

프로덕션 마이그레이션 실패 시 즉각적인 행동 순서는 다음과 같습니다.

**1. 현재 상태 파악** (1분 내)
```bash
flyway info  # 어느 버전에서 실패했는가?
# flyway_schema_history에서 success=false 레코드 확인
```

**2. 서비스 영향도 확인** (1분 내)
```bash
# 앱이 시작됐는가, 아니면 시작 실패인가?
# 사용자에게 영향이 있는가?
```

**3. 자동 복구 시도** (부분 실패인 경우)
```bash
flyway repair  # 실패 레코드 제거
flyway migrate  # 재시도
```

**4. 재시도 불가 시 — 수동 복구 또는 롤백**
```bash
# 마이그레이션이 일부만 적용됐다면 수동으로 역방향 SQL 실행
# 또는 사전에 생성한 DB 스냅샷으로 복원 (Point-In-Time Recovery)
```

**5. 사후 분석**
실패 원인(체크섬 불일치, SQL 오류, Lock 타임아웃 등)을 파악하고 재발 방지책 수립.

중요: 마이그레이션 전에 스냅샷을 찍지 않았다면 스텝 4가 불가능합니다. 이것이 "대형 마이그레이션 전 DB 스냅샷 필수" 원칙이 존재하는 이유입니다. (Chapter 4에서 자세히 다룹니다)

</details>

---

<div align="center">

**[⬅️ 이전: 마이그레이션 파일 명명 규칙](./04-naming-convention.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — Flyway 내부 동작 원리 ➡️](../flyway-internals/01-flyway-schema-history.md)**

</div>
