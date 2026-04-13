# 스키마를 코드로 관리해야 하는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 수동 DDL 실행이 왜 팀 단위 프로젝트에서 반드시 문제를 일으키는가?
- "DB도 Git으로 관리한다"는 말이 실제로 어떤 의미인가?
- IaC(Infrastructure as Code)와 Schema as Code는 어떤 철학을 공유하는가?
- 마이그레이션 도구 없이 로컬/개발/프로덕션 환경의 스키마 불일치를 막을 수 있는가?
- 스키마 변경 이력이 없을 때 장애 상황에서 무슨 일이 벌어지는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

애플리케이션 코드는 Git으로 관리한다. 모든 변경은 커밋으로 남고, 누가 언제 무엇을 바꿨는지 추적할 수 있다. 배포는 자동화되어 있고, 문제가 생기면 이전 커밋으로 되돌릴 수 있다.

그런데 DB 스키마는 다르게 관리하는 경우가 많다. DBeaver를 열고, `ALTER TABLE`을 직접 실행하고, 팀원에게 Slack으로 공유한다. 이 방식은 팀이 2명일 때는 어떻게든 굴러가지만, 팀원이 늘어나고 환경이 많아지고 서비스가 성장하면 반드시 무너진다.

코드는 Git이 지켜주지만 스키마는 아무도 지켜주지 않는다. 그 간극이 장애로 이어진다.

---

## 😱 흔한 실수 (Before — 수동 DDL 실행)

```
팀 구성: 개발자 A, B, C / 환경: 로컬 3개 + 개발 서버 + 스테이징 + 프로덕션

시나리오 1: 팀원 누락
  개발자 A: 로컬에서 ALTER TABLE users ADD COLUMN phone VARCHAR(20) 실행
  개발자 A: Slack에 공유 — "users 테이블에 phone 컬럼 추가했어요"
  개발자 B: 공지를 못 봄 (휴가 중)
  개발자 B: 복귀 후 앱 실행 → "Unknown column 'phone'" 에러
  개발자 B: 30분 디버깅 후 원인 파악
  → 팀원이 늘어날수록, Slack 메시지가 많아질수록 이 문제는 더 자주 발생

시나리오 2: 환경별 불일치
  개발 서버에 배포 → 성공
  스테이징 서버에 배포 → 성공
  프로덕션 배포 → 실패: "Column 'status' cannot be null"
  원인: 개발 서버에만 ALTER TABLE이 적용됨
  스테이징에는 다른 개발자가 다른 방식으로 임시 패치
  프로덕션은 두 달 전 상태 그대로
  → 환경마다 스키마가 다른데 누구도 전체 상태를 모름

시나리오 3: 이력 없음
  프로덕션 장애 발생: 특정 쿼리에서 에러
  "이 컬럼이 언제 추가됐지?" → 아무도 모름
  "이 인덱스는 누가 추가한 거야?" → 아무도 모름
  Slack 히스토리를 3개월치 뒤지기 시작
  → 장애 중 원인 파악에 1시간 소요
```

---

## ✨ 올바른 접근 (After — 스키마를 코드로 관리)

```
마이그레이션 도구(Flyway) 도입 후:

1. 스키마 변경 = SQL 파일 생성
   src/main/resources/db/migration/V20240315120000__add_phone_to_users.sql
   ↓
   ALTER TABLE users ADD COLUMN phone VARCHAR(20) NULL;

2. SQL 파일 = Git 커밋
   git add db/migration/V20240315120000__add_phone_to_users.sql
   git commit -m "feat: users 테이블에 phone 컬럼 추가"
   git push → PR 생성 → 코드 리뷰

3. 배포 시 자동 적용
   Spring Boot 시작 → Flyway 실행 → flyway_schema_history 확인
   → 미적용 마이그레이션 발견 → 자동 실행 → 기록

결과:
  개발자 B가 복귀 후 git pull → 앱 실행 → Flyway 자동 적용 → 에러 없음
  개발/스테이징/프로덕션 모두 동일한 SQL 파일이 순서대로 적용됨
  "이 컬럼이 언제 추가됐지?" → git log db/migration/ → 즉시 확인
  "이 인덱스는 누가 추가한 거야?" → git blame → 즉시 확인
```

---

## 🔬 내부 동작 원리

### 1. IaC(Infrastructure as Code)와 동일한 철학

```
전통적 인프라 관리:
  서버에 직접 SSH 접속 → 설정 파일 수동 편집 → 변경 이력 없음
  → 서버 장애 시 "그 서버는 어떻게 설정돼 있었지?" 알 수 없음

IaC (Terraform, Ansible):
  인프라 상태를 코드(.tf, .yml)로 선언
  → Git으로 버전 관리 → PR로 리뷰 → CI/CD로 자동 적용
  → "서버가 어떤 상태인가"를 코드만 보면 알 수 있음

Schema as Code (Flyway, Liquibase):
  DB 스키마 변경을 SQL 파일로 선언
  → Git으로 버전 관리 → PR로 리뷰 → 배포 시 자동 적용
  → "DB가 어떤 상태인가"를 마이그레이션 파일 목록만 보면 알 수 있음

핵심 원칙: "시스템의 실제 상태는 코드(선언)에서 파생된다"
  코드가 진실의 원천(Source of Truth)
  코드가 없으면 상태를 알 수 없음
```

### 2. Flyway가 스키마 상태를 추적하는 방법

```
flyway_schema_history 테이블 (Flyway가 자동 생성):

installed_rank | version          | description           | type | script                                       | checksum    | installed_on        | success
1              | 20240101000000   | create users          | SQL  | V20240101000000__create_users.sql            | -1234567890 | 2024-01-01 09:00:00 | 1
2              | 20240115120000   | add email to users    | SQL  | V20240115120000__add_email_to_users.sql      | 987654321   | 2024-01-15 14:30:00 | 1
3              | 20240315120000   | add phone to users    | SQL  | V20240315120000__add_phone_to_users.sql      | 456789012   | 2024-03-15 12:05:00 | 1

이 테이블이 의미하는 것:
  - 어떤 마이그레이션이 적용됐는가 (version + script)
  - 언제 적용됐는가 (installed_on)
  - 성공했는가 (success)
  - 파일이 변경되지 않았는가 (checksum — CRC32)

Spring Boot 시작 시 Flyway의 동작:
  1. classpath:db/migration/*.sql 파일 스캔
  2. 각 파일의 CRC32 체크섬 계산
  3. flyway_schema_history와 비교
     ├── DB에 없는 버전 → 미적용 마이그레이션 → 실행
     ├── DB에 있고 체크섬 일치 → 이미 적용됨 → 건너뜀
     └── DB에 있는데 체크섬 다름 → 오류! (파일 변경 감지)
  4. 미적용 마이그레이션을 버전 순서대로 실행
  5. 각 실행 결과를 flyway_schema_history에 기록
```

### 3. 수동 DDL vs 마이그레이션 — 상태 일관성

```
수동 DDL 실행 시 환경별 상태:

환경          | users.phone | users.email | orders.status | 담당자 기억
로컬 A        | O (추가됨)  | O           | VARCHAR(20)   | "내가 추가함"
로컬 B        | X (누락)    | O           | VARCHAR(20)   | "몰랐음"
개발 서버     | O           | O           | INT (다름!)   | "누가 바꿨지?"
스테이징      | O           | X (누락)    | VARCHAR(20)   | "언제 빠졌지?"
프로덕션      | X           | X           | VARCHAR(20)   | "원래 상태"

마이그레이션 도구 사용 시:

모든 환경에서 flyway_schema_history가 동일
→ V1, V2, V3 순서대로 적용됨을 보장
→ 환경 간 스키마 불일치 원천 차단
→ 새 팀원 합류 시: git clone → 앱 실행 → Flyway 자동 동기화
```

---

## 💻 실전 실험

### 실험 1: 수동 DDL의 환경 불일치 재현

```bash
# Docker로 두 개의 독립적인 MySQL 인스턴스 실행 (로컬 A, 로컬 B 시뮬레이션)
docker run -d --name mysql-a -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=myapp -p 3306:3306 mysql:8.0
docker run -d --name mysql-b -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=myapp -p 3307:3306 mysql:8.0

# mysql-a: 수동으로 컬럼 추가 (개발자 A의 로컬)
docker exec -it mysql-a mysql -uroot -proot myapp -e "
  CREATE TABLE users (id BIGINT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(100));
  ALTER TABLE users ADD COLUMN email VARCHAR(200);
  ALTER TABLE users ADD COLUMN phone VARCHAR(20);
"

# mysql-b: 초기 테이블만 있음 (개발자 B의 로컬 — Slack 공지 못 봄)
docker exec -it mysql-b mysql -uroot -proot myapp -e "
  CREATE TABLE users (id BIGINT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(100));
"

# 두 환경의 스키마 비교
echo "=== mysql-a 스키마 ==="
docker exec mysql-a mysql -uroot -proot myapp -e "DESCRIBE users;"

echo "=== mysql-b 스키마 ==="
docker exec mysql-b mysql -uroot -proot myapp -e "DESCRIBE users;"

# 결과: 완전히 다른 스키마 — 아무도 이 불일치를 알지 못함
```

### 실험 2: Flyway로 동일한 상황 → 자동 동기화

```bash
# migrations/V1__create_users.sql
cat > migrations/V1__create_users.sql << 'EOF'
CREATE TABLE users (
    id    BIGINT       NOT NULL AUTO_INCREMENT,
    name  VARCHAR(100) NOT NULL,
    PRIMARY KEY (id)
) ENGINE=InnoDB;
EOF

# migrations/V2__add_email_to_users.sql
cat > migrations/V2__add_email_to_users.sql << 'EOF'
ALTER TABLE users ADD COLUMN email VARCHAR(200) NULL;
EOF

# migrations/V3__add_phone_to_users.sql
cat > migrations/V3__add_phone_to_users.sql << 'EOF'
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NULL;
EOF

# mysql-a에 Flyway 적용
docker run --rm --network host \
  -v $(pwd)/migrations:/flyway/sql \
  flyway/flyway:9 \
  -url="jdbc:mysql://localhost:3306/myapp" \
  -user=root -password=root migrate

# mysql-b에도 동일하게 적용 (포트만 다름)
docker run --rm --network host \
  -v $(pwd)/migrations:/flyway/sql \
  flyway/flyway:9 \
  -url="jdbc:mysql://localhost:3307/myapp" \
  -user=root -password=root migrate

# 두 환경 모두 동일한 스키마 보장
# flyway info로 적용 상태 확인
docker run --rm --network host \
  -v $(pwd)/migrations:/flyway/sql \
  flyway/flyway:9 \
  -url="jdbc:mysql://localhost:3306/myapp" \
  -user=root -password=root info
# 출력:
# Version    Description         State
# 1          create users        Success
# 2          add email to users  Success
# 3          add phone to users  Success
```

### 실험 3: 스키마 이력 추적

```bash
# flyway_schema_history로 스키마 변경 이력 조회
docker exec mysql-a mysql -uroot -proot myapp -e "
SELECT
    version,
    description,
    installed_on,
    execution_time,
    success
FROM flyway_schema_history
ORDER BY installed_rank;
"

# git log로 누가 언제 마이그레이션을 추가했는지 확인
git log --oneline db/migration/
# a1b2c3d feat: phone 컬럼 추가 - 개발자A (2024-03-15)
# d4e5f6g feat: email 컬럼 추가 - 개발자B (2024-01-15)
# g7h8i9j feat: users 테이블 생성 - 개발자C (2024-01-01)
```

---

## 📊 비교

```
수동 DDL vs 마이그레이션 도구 비교:

항목                  | 수동 DDL 실행          | Flyway 마이그레이션
─────────────────────┼──────────────────────┼──────────────────────
환경 일관성           | 보장 불가             | 모든 환경 동일 보장
변경 이력             | Slack/기억에 의존      | Git + flyway_schema_history
새 팀원 온보딩        | 수동 스키마 동기화 필요  | git clone 후 자동 적용
배포 자동화           | 수동 DDL 실행 별도     | 배포 시 자동 포함
장애 원인 분석        | 언제 바뀌었는지 불명   | 커밋 + 타임스탬프로 추적
코드 리뷰             | 불가                  | PR로 스키마 변경 리뷰
롤백 계획             | 임기응변              | 구조화된 복구 절차
```

---

## ⚖️ 트레이드오프

```
Schema as Code 도입의 장단점:

장점:
  ① 환경 일관성 — 모든 환경이 동일한 스키마임을 보장
  ② 변경 이력 — 누가 언제 무엇을 왜 바꿨는지 Git으로 추적
  ③ 자동화 — 배포 시 마이그레이션 자동 적용, 수동 작업 제거
  ④ 코드 리뷰 — 스키마 변경도 PR로 리뷰 가능
  ⑤ 온보딩 — 새 팀원이 git clone 후 자동 동기화

단점/주의사항:
  ① 학습 비용 — 팀 전체가 마이그레이션 도구와 규칙을 숙지해야 함
  ② 파일 충돌 — 여러 브랜치에서 동시에 마이그레이션 작성 시 버전 충돌
     (→ Chapter 5에서 해결 방법 다룸)
  ③ DDL 자동 적용 위험 — 잘못된 마이그레이션이 자동으로 프로덕션에 적용될 수 있음
     (→ Chapter 6 CI/CD 통합에서 검증 단계 다룸)

결론:
  단점은 모두 관리 가능한 문제
  수동 DDL 실행의 위험(이력 없음, 불일치)은 관리 불가능
  → 팀 규모와 환경 수가 늘어날수록 도입 필요성이 커짐
```

---

## 📌 핵심 정리

```
Schema as Code의 3가지 핵심:

1. 스키마 변경 = SQL 파일 생성
   ALTER TABLE을 직접 실행하지 않고
   db/migration/V{버전}__{설명}.sql 파일로 선언

2. SQL 파일 = Git 커밋
   코드와 함께 버전 관리
   PR로 리뷰 → 코드 리뷰와 동일한 프로세스

3. 배포 = 자동 마이그레이션
   Flyway가 미적용 SQL을 자동 감지 후 실행
   flyway_schema_history에 기록 → 이력 보장

결과:
  환경 불일치 원천 차단
  누가 언제 무엇을 바꿨는지 항상 추적 가능
  새 팀원 온보딩 시 git clone 한 번으로 완료
```

---

## 🤔 생각해볼 문제

**Q1.** 팀 규모가 2명이고 환경이 로컬 + 프로덕션 둘뿐일 때도 마이그레이션 도구가 필요한가? 어느 시점부터 도입하는 것이 합리적인가?

<details>
<summary>해설 보기</summary>

팀이 작고 환경이 적어도 마이그레이션 도구를 초반부터 도입하는 것이 좋습니다. 이유는 "지금은 괜찮다"는 생각이 가장 위험한 시점이기 때문입니다.

초반에 도입하지 않으면 나중에 도입 비용이 훨씬 커집니다. 이미 운영 중인 DB에 Flyway를 도입하려면 `baseline-on-migrate` 설정으로 현재 상태를 기준점으로 잡는 작업이 필요하고, 기존 스키마 이력은 복구할 수 없습니다.

반면 처음부터 도입하면 비용이 거의 없습니다. `V1__create_initial_schema.sql` 파일 하나로 시작하면 됩니다.

실용적인 기준: **혼자 개발하더라도 환경이 2개 이상**(로컬 + 어딘가 배포)이면 도입을 권장합니다. 스키마 변경 이력은 나중에 만들 수 없으며, 미래의 자신이 "언제 이 컬럼을 추가했지?"라고 물을 때 Git 히스토리가 대답해줍니다.

</details>

---

**Q2.** 수동으로 프로덕션 DB에 `ALTER TABLE`을 실행한 후에 Flyway를 도입하려 한다. 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

이미 존재하는 DB에 Flyway를 도입할 때는 `baseline-on-migrate` 옵션을 사용합니다.

```yaml
spring:
  flyway:
    baseline-on-migrate: true
    baseline-version: 1
    baseline-description: "Existing schema baseline"
```

이 설정은 Flyway가 처음 실행될 때, 기존 DB 상태를 버전 1로 표시하고 `flyway_schema_history`에 baseline 레코드를 삽입합니다. 이후부터는 V2, V3... 마이그레이션 파일만 적용됩니다.

중요: baseline 이전의 스키마 이력은 복구되지 않습니다. `V1__initial_schema.sql`에 현재 스키마 전체를 `CREATE TABLE IF NOT EXISTS`로 문서화해두는 것을 권장합니다. 이 파일은 새 환경을 처음 셋업할 때도 활용됩니다.

</details>

---

**Q3.** "마이그레이션 파일은 Git에서 삭제하지 않는다"는 규칙이 있는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

Flyway는 `flyway_schema_history`에 적용된 마이그레이션의 버전과 파일명을 기록합니다. 파일을 삭제하면 다음 Flyway 실행 시 "DB에는 적용됐다고 기록되어 있는데, 파일이 없다"는 불일치를 감지합니다.

기본 설정에서는 이 상황이 오류로 처리됩니다(`ignore-missing-migrations=false`). 파일을 삭제한 환경(예: 새로 클론한 저장소)에서는 Flyway가 시작되지 않습니다.

**마이그레이션 내용을 취소하고 싶다면** 파일을 삭제하는 것이 아니라, 새 버전의 마이그레이션 파일에서 반대 작업을 수행합니다.

```sql
-- V5__add_column.sql (실수로 추가한 컬럼)
ALTER TABLE users ADD COLUMN temp_flag TINYINT DEFAULT 0;

-- V6__remove_column.sql (다음 버전에서 제거)
ALTER TABLE users DROP COLUMN temp_flag;
```

이것이 Forward-Only 전략의 핵심입니다. (Chapter 4에서 자세히 다룹니다)

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Flyway vs Liquibase ➡️](./02-flyway-vs-liquibase.md)**

</div>
