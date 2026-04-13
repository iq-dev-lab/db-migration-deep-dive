# 마이그레이션 파일 명명 규칙

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Flyway가 파일명을 파싱하는 규칙은 무엇인가? `V`, `R`, `U` 접두사는 각각 무엇을 의미하는가?
- `V1__`, `V2__` 순차 번호 방식은 왜 팀 협업에서 충돌을 유발하는가?
- 타임스탬프(`yyyyMMddHHmmss`) 기반 버전이 충돌을 줄이는 원리는?
- 마이그레이션 파일 설명(description)을 잘 작성하는 기준은 무엇인가?
- Flyway가 파일명의 특수문자나 공백을 어떻게 처리하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이그레이션 파일의 이름은 단순한 파일명이 아니다. Flyway가 실행 순서를 결정하는 기준이고, 나중에 "언제 무엇을 바꿨는지" 이력을 읽는 문서이며, 팀 협업에서 충돌이 발생하는 지점이기도 하다. 명명 규칙을 팀 내에서 명확히 정하지 않으면, PR 머지 후 버전 충돌 오류로 배포가 막히는 상황이 반복된다.

---

## 😱 흔한 실수 (Before — 명명 규칙 없이 파일 생성)

```
시나리오: 순차 번호 방식의 충돌

  현재 상황: V1, V2가 메인 브랜치에 있음

  개발자 A (feature/payment 브랜치):
    V3__add_payment_table.sql 생성

  개발자 B (feature/notification 브랜치):
    V3__add_notification_table.sql 생성

  개발자 A 먼저 main에 머지:
    main: V1, V2, V3(payment)

  개발자 B PR 머지 시도:
    main: V1, V2, V3(payment) ← 이미 있음
    B의 브랜치: V3(notification) ← 충돌!

    결과:
      파일명을 V4__add_notification_table.sql로 수동 변경 필요
      그런데 이미 개발 서버에 V3(notification)이 적용됐다면?
      → flyway_schema_history에 V3로 기록됨
      → 파일명 변경하면 체크섬 불일치 오류
      → flyway repair 실행 + 수동 레코드 수정 필요

  이 상황이 팀 규모에 따라 매주, 매일 발생

---

다른 실수: 설명(description)이 무의미한 경우

  V3__update.sql           ← 뭘 업데이트한 건지?
  V4__fix.sql              ← 뭘 고친 건지?
  V5__change_users.sql     ← 어떤 변경인지?

  6개월 후 장애 상황에서:
    "이 시점 이후 users 테이블 뭔가 바뀐 것 같은데"
    → V5__change_users.sql 열어야만 내용 확인 가능
    → 파일명만 봐서는 전혀 알 수 없음
```

---

## ✨ 올바른 접근 (After — 명확한 명명 규칙)

```
Flyway 공식 파일명 패턴:

  {접두사}{버전}__{설명}.{확장자}

  접두사:
    V  → Versioned Migration (한 번만 실행)
    R  → Repeatable Migration (체크섬 변경 시마다 재실행)
    U  → Undo Migration (Flyway Teams 전용)

  구분자: __ (언더스코어 2개)

  예시:
    V20240315120000__add_phone_to_users.sql
    R__seed_test_data.sql
    U20240315120000__add_phone_to_users.sql

타임스탬프 버전 방식:
  형식: yyyyMMddHHmmss (14자리)
  V20240315120000__ → 2024년 3월 15일 12시 00분 00초

  충돌 확률:
    순차 번호: V3 충돌 → 두 명이 같은 번호를 사용할 확률 높음
    타임스탬프: V20240315120523 vs V20240315120537 → 초 단위 차이로 자연 분리

좋은 설명 작성 기준:
  형식: {동사}_{대상테이블}_{내용}
  예시:
    V20240315120000__add_phone_to_users.sql        ✓
    V20240315130000__create_orders_table.sql       ✓
    V20240316090000__add_index_on_orders_status.sql ✓
    V20240317100000__rename_amount_to_total_amount.sql ✓
    V20240318150000__drop_deprecated_temp_flag.sql ✓
```

---

## 🔬 내부 동작 원리

### 1. Flyway 파일명 파싱 규칙

```
파일명 파싱 정규식 (Flyway 내부):

  V{version}__{description}.sql
  
  version 파싱:
    V1__            → version = "1"       → 정렬 시 숫자로 비교
    V1.1__          → version = "1.1"     → 점(.)으로 구분된 세그먼트
    V20240315__     → version = "20240315"
    V2024.03.15__   → version = "2024.03.15"
    V2024_03_15__   → version = "2024_03_15" (언더스코어도 구분자)

  정렬 규칙:
    각 세그먼트를 숫자로 비교
    V1 < V2 < V10  (문자열 비교가 아닌 숫자 비교)
    V1.1 < V1.2 < V2.0
    V20240101 < V20240315 < V20241231

  description 파싱:
    __ 이후, .sql 이전의 문자열
    언더스코어(_)는 공백으로 치환되어 flyway_schema_history의 description 컬럼에 저장
    V20240315__add_phone_to_users → description = "add phone to users"

확장자:
  .sql  → SQL 마이그레이션
  .java → Java 마이그레이션 (BaseJavaMigration 구현체와 매핑)
```

### 2. 타임스탬프 방식의 충돌 방지 원리

```
순차 번호 방식의 충돌 시나리오:

  main: V1, V2
  브랜치 A: V3__payment.sql (생성 시각: 10:00)
  브랜치 B: V3__notification.sql (생성 시각: 10:05)

  A 먼저 머지 → main: V1, V2, V3(payment)
  B 머지 시 V3 충돌 → B가 V4로 수동 변경 필요

---

타임스탬프 방식의 자연 분리:

  main: V20240314, V20240315
  브랜치 A: V20240316100000__payment.sql    (10:00:00에 생성)
  브랜치 B: V20240316100523__notification.sql (10:05:23에 생성)

  A 먼저 머지 → main: V20240314, V20240315, V20240316100000
  B 머지 → main: V20240314, V20240315, V20240316100000, V20240316100523
  → 충돌 없음! 타임스탬프가 자연스럽게 순서를 결정

  같은 초(second)에 생성하는 경우:
    두 개발자가 정확히 같은 초에 파일을 만들 확률은 매우 낮음
    → 충돌 확률 ≈ 0 (완전히 없지는 않지만 실용적으로 무시 가능)

IntelliJ 파일 생성 타임스탬프 자동화:
  File Templates에서 마이그레이션 파일 템플릿 설정
  파일명: V${DATE}${TIME}__${NAME}.sql
  → 새 파일 생성 시 현재 타임스탬프 자동 입력
```

### 3. Repeatable Migration의 명명 규칙

```
Repeatable Migration (R__ 접두사):

  특징:
    버전 번호 없음 (R__로 시작)
    체크섬이 변경될 때마다 재실행
    모든 Versioned Migration 완료 후 실행

  사용 시나리오:
    뷰(View) 재생성: DDL 변경마다 DROP + CREATE
    저장 프로시저 재배포
    시드 데이터 갱신 (개발/테스트 환경)
    권한 재부여

  파일명 예시:
    R__create_view_monthly_sales.sql
    R__create_procedure_calculate_discount.sql
    R__seed_reference_data.sql

  R__create_view_monthly_sales.sql 내용:
    CREATE OR REPLACE VIEW monthly_sales AS
    SELECT
        DATE_FORMAT(created_at, '%Y-%m') AS month,
        SUM(amount) AS total
    FROM orders
    WHERE status = 'COMPLETED'
    GROUP BY DATE_FORMAT(created_at, '%Y-%m');
  
  체크섬 변경 시 자동 재실행:
    뷰 쿼리를 수정하고 git push → 배포 시 자동으로 뷰 재생성
    → 뷰 변경을 별도 마이그레이션 파일로 만들 필요 없음
```

### 4. Flyway 설정으로 명명 규칙 커스터마이징

```yaml
spring:
  flyway:
    # 기본값 변경 (필요한 경우만)
    sql-migration-prefix: V          # 기본값
    repeatable-sql-migration-prefix: R  # 기본값
    sql-migration-separator: __      # 기본값 (언더스코어 2개)
    sql-migration-suffixes: .sql     # 기본값
    
    # 마이그레이션 파일 위치 (여러 위치 지원)
    locations:
      - classpath:db/migration       # 공통 마이그레이션
      - classpath:db/migration/seed  # 시드 데이터 (환경별 활성화)
```

---

## 💻 실전 실험

### 실험 1: 파일명 파싱 동작 확인

```bash
# 다양한 버전 형식 테스트
mkdir -p test-migrations

# 순차 번호 방식
touch test-migrations/V1__create_users.sql
touch test-migrations/V2__add_email.sql
touch test-migrations/V10__add_index.sql  # 숫자로 정렬되는지 확인

# 타임스탬프 방식
touch test-migrations/V20240101000000__initial_schema.sql
touch test-migrations/V20240315120000__add_phone.sql

# flyway info로 파싱 결과 확인
docker run --rm \
  -v $(pwd)/test-migrations:/flyway/sql \
  flyway/flyway:9 \
  -url="jdbc:mysql://host.docker.internal:3306/test" \
  -user=root -password=root info

# 출력에서 Version 컬럼 정렬 순서 확인:
# V1 < V2 < V10 (문자열 정렬 시 V10 < V2가 되는 버그 없음)
# V20240101000000 < V20240315120000
```

### 실험 2: 순차 번호 충돌 재현 및 해결

```bash
# 개발 서버에 V3__notification.sql이 이미 적용됨
# 이제 파일명을 V4로 변경해야 하는 상황

# 현재 flyway_schema_history 확인
mysql -uroot -proot myapp -e "
SELECT version, description, success FROM flyway_schema_history ORDER BY installed_rank;
"
# version=3, description=notification table, success=1 이 있음

# 파일명을 V4로 변경
mv db/migration/V3__notification_table.sql db/migration/V4__notification_table.sql

# flyway repair로 체크섬 재계산 (파일이 이미 적용된 경우)
flyway -url=jdbc:mysql://localhost:3306/myapp -user=root -password=root repair

# 하지만 version이 이미 3으로 기록됐으므로
# flyway_schema_history의 version 컬럼도 수동으로 수정 필요:
mysql -uroot -proot myapp -e "
UPDATE flyway_schema_history SET version='4' WHERE version='3' AND description='notification table';
"
# ← 이 복잡한 수동 작업이 타임스탬프 방식에서는 발생하지 않음
```

### 실험 3: 좋은 파일명 예시 모음

```bash
# 실제 프로젝트에서 사용할 수 있는 파일명 예시

# 테이블 생성
V20240101000000__create_users.sql
V20240102000000__create_orders.sql
V20240103000000__create_products.sql

# 컬럼 추가
V20240115120000__add_email_to_users.sql
V20240115130000__add_phone_to_users.sql

# 컬럼 변경
V20240201090000__rename_amount_to_total_amount_in_orders.sql

# 인덱스 추가
V20240210150000__add_index_on_orders_status.sql
V20240210160000__add_composite_index_on_users_email_status.sql

# 데이터 마이그레이션 (백필)
V20240215100000__backfill_order_total_amount.sql

# 컬럼 삭제 (이전 단계들이 완료된 후)
V20240301000000__drop_deprecated_amount_from_orders.sql

# 제약 추가
V20240310120000__add_not_null_constraint_to_total_amount.sql

# Repeatable: 뷰, 프로시저
R__view_monthly_sales.sql
R__view_active_users.sql
R__procedure_calculate_discount.sql
R__seed_category_data.sql
```

---

## 📊 비교

```
순차 번호 vs 타임스탬프 방식:

항목                    | 순차 번호 (V1, V2, ...)     | 타임스탬프 (V20240315120000)
───────────────────────┼────────────────────────────┼────────────────────────────
충돌 가능성             | 높음 (병렬 브랜치에서 자주)  | 매우 낮음 (초 단위 분리)
파일명 가독성           | 짧고 단순                   | 길지만 날짜 정보 포함
버전 순서 직관성        | V1 < V2 명확               | 날짜가 순서 역할
새 파일 생성 편의성     | 수동으로 최대 번호 확인 필요 | 현재 시각 사용 (자동화 가능)
기존 레포와 혼용        | 어려움                      | 독립적 (충돌 없음)
추천 팀 규모            | 1~3명 소규모               | 3명 이상 팀
```

---

## ⚖️ 트레이드오프

```
타임스탬프 방식의 단점:
  ① 파일명이 길다 (V20240315120000__ vs V3__)
     → IDE 자동완성으로 보완
  ② "이 마이그레이션이 몇 번째야?" 한눈에 안 보임
     → flyway info 명령어로 확인 (installed_rank 컬럼)
  ③ 정확히 같은 초에 두 파일이 생성될 수 있음
     → 실용적으로 거의 발생하지 않음

타임스탬프 방식의 장점:
  ① 충돌 없이 병렬 브랜치 작업 가능
  ② 파일명에서 언제 만든 마이그레이션인지 바로 파악
  ③ 자동화 스크립트로 현재 시각을 버전으로 자동 입력 가능

결론:
  소규모 팀(1-2명): 순차 번호로 시작해도 충분
  3명 이상 팀, 병렬 브랜치 작업: 타임스탬프 방식 권장
  이미 순차 번호를 쓰고 있다면: 충돌이 자주 발생할 때 마이그레이션 고려
```

---

## 📌 핵심 정리

```
마이그레이션 파일 명명 규칙:

파일명 형식:
  V{버전}__{설명}.sql
  R__{설명}.sql  (Repeatable)

버전 전략:
  소규모: V1, V2, V3 (단순하지만 충돌 위험)
  팀 작업: V20240315120000 (타임스탬프, 충돌 방지)

좋은 설명 작성:
  동사로 시작: add, create, drop, rename, backfill, add_index
  대상 명확히: _to_users, _on_orders, _from_products
  예: add_phone_to_users, create_index_on_orders_status

절대 하지 말아야 할 것:
  V3__fix.sql          → 뭘 고쳤는지 알 수 없음
  V3__update.sql       → 뭘 업데이트했는지 알 수 없음
  적용된 파일의 내용 수정 → 체크섬 불일치 오류
  적용된 파일 삭제      → missing migration 오류
```

---

## 🤔 생각해볼 문제

**Q1.** 팀에서 타임스탬프 방식을 사용하기로 했다. 로컬 시간대가 다른 팀원(한국 UTC+9, 미국 UTC-8)이 같은 시각에 파일을 만들면 어떤 버전이 더 앞에 오는가?

<details>
<summary>해설 보기</summary>

파일명의 타임스탬프는 개발자의 **로컬 시간**을 기준으로 작성합니다. 따라서 한국 개발자가 `2024-03-15 12:00:00 (KST)`에 만든 파일과 미국 개발자가 `2024-03-14 20:00:00 (PST)`에 만든 파일은 실제로는 같은 UTC 시각이지만, 파일명에는 각자의 로컬 시간이 찍힙니다.

```
한국 개발자: V20240315120000__add_phone.sql
미국 개발자: V20240314200000__add_notification.sql
```

Flyway 정렬 결과: `V20240314200000` → `V20240315120000` 순서로 실행됩니다.

이것이 **의도한 동작**과 다를 수 있습니다. 실제로 두 파일이 동시에 만들어졌지만 로컬 시간 차이로 순서가 결정됩니다.

이를 방지하려면 팀 전체가 **UTC 기준으로 타임스탬프를 생성**하는 규칙을 정하거나, `date -u +"%Y%m%d%H%M%S"` 명령어로 UTC 시간을 사용하도록 공유하면 됩니다. 또는 `out-of-order=true` 설정으로 순서 불일치를 허용하는 방법도 있습니다.

</details>

---

**Q2.** Flyway의 `out-of-order=true` 설정은 무엇이며, 어떤 위험이 있는가?

<details>
<summary>해설 보기</summary>

`out-of-order=true`는 이미 적용된 마이그레이션보다 낮은 버전의 마이그레이션이 나타났을 때 실행을 허용합니다.

예를 들어 main에 V5까지 적용된 상태에서 V3를 머지하면, 기본 설정(`out-of-order=false`)에서는 V3가 "이미 지나간 버전"이라는 오류로 실행되지 않습니다. `out-of-order=true`이면 V3도 실행됩니다.

**위험**: V5가 V3의 변경 내용을 전제로 작성되어 있을 수 있습니다. 예를 들어 V5에서 V3이 추가할 컬럼을 참조하는 쿼리가 있다면, V3가 나중에 실행되어도 V5는 이미 그 컬럼 없이 실행됩니다. 이런 의존성 문제가 있을 때 `out-of-order`는 데이터 불일치를 만들 수 있습니다.

**권장 사용**: 장기 브랜치 없이 단기 Feature 브랜치만 운용하는 팀에서, 버전 충돌 발생 시 임시 해결책으로 허용하는 경우. 또는 `flyway info` 출력에서 경고로만 표시하되 오류는 발생시키지 않는 설정(`ignore-future-migrations`)과 조합하기도 합니다.

</details>

---

**Q3.** 마이그레이션 파일에서 여러 DDL을 하나의 파일에 넣어야 하는가, 아니면 DDL 하나당 파일 하나로 분리해야 하는가?

<details>
<summary>해설 보기</summary>

명확한 정답은 없지만, 다음 기준으로 판단합니다.

**하나의 파일에 여러 DDL을 넣는 경우 (관련 변경 묶음)**:
```sql
-- V20240315120000__create_order_system.sql
-- 주문 시스템 초기 테이블 전체를 하나의 마이그레이션으로
CREATE TABLE orders (...);
CREATE TABLE order_items (...);
CREATE TABLE order_payments (...);
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

여러 테이블이 하나의 도메인 기능을 위해 함께 생성되는 경우, 하나의 파일로 묶는 것이 논리적입니다. 하나가 실패하면 전체가 롤백(MySQL은 DDL 롤백이 안 되지만, 실패 시 다음 실행에서 전체를 재시도)되어야 하는 경우에도 묶습니다.

**파일을 분리하는 경우 (독립적 변경)**:
```
V20240315120000__add_phone_to_users.sql
V20240315130000__add_index_on_users_email.sql
```

독립적으로 실패할 수 있고, 각각의 실패 처리가 다를 때 분리합니다. 특히 데이터 백필(UPDATE)은 별도 파일로 분리하는 것이 권장됩니다. 백필이 실패했을 때 스키마 변경(ADD COLUMN)은 이미 적용됐고 백필만 재시도해야 하는 경우가 있기 때문입니다.

</details>

---

<div align="center">

**[⬅️ 이전: ddl-auto=update 금지 이유](./03-ddl-auto-update-forbidden.md)** | **[홈으로 🏠](../README.md)** | **[다음: 마이그레이션 환경 전략 ➡️](./05-environment-strategy.md)**

</div>
