# Flyway vs Liquibase — 버전 기반 vs 변경 기반

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Flyway의 "버전 기반(Versioned)" 방식과 Liquibase의 "변경셋(ChangeSet)" 방식은 근본적으로 어떻게 다른가?
- Liquibase는 XML/YAML로 마이그레이션을 작성하는데, 왜 SQL보다 복잡한 방식을 택했는가?
- 두 도구에서 "롤백"은 어떻게 작동하고, 실제로 사용 가능한 수준인가?
- 팀 규모, DB 환경, 기술 스택에 따라 어떤 도구를 선택해야 하는가?
- Spring Boot 프로젝트에서 두 도구의 자동 설정 방식 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Flyway와 Liquibase는 각각 수백만 개의 프로젝트에서 사용되는 검증된 도구다. 하지만 설계 철학이 다르기 때문에 같은 문제를 다른 방식으로 해결한다. 도구를 선택하는 시점에 이 차이를 이해하지 못하면, 나중에 "롤백이 안 되네", "SQL을 직접 못 쓰네", "설정이 너무 복잡하네" 같은 문제로 도구를 바꾸는 비용이 발생한다. 처음부터 팀의 상황에 맞는 도구를 고르는 것이 중요하다.

---

## 😱 흔한 실수 (Before — 도구를 이해하지 못하고 선택)

```
실수 유형 1: "Liquibase가 더 기능이 많으니까" 선택
  → XML/YAML 문법 학습에 시간 소요
  → 팀원들이 SQL은 알지만 changeSet 구조를 몰라 생산성 저하
  → 간단한 컬럼 추가에도 XML 구조 필요
  → 결국 SQL runStatement로 SQL 직접 작성하게 됨

실수 유형 2: "Flyway가 더 심플하니까" 선택
  → 나중에 복잡한 데이터 변환 필요 → Java Migration으로 해결 가능하지만 처음엔 몰랐음
  → "롤백 기능이 없네?" → 처음부터 알았으면 Forward-Only 전략을 설계했을 것

실수 유형 3: "기본값으로 세팅된 걸 그냥 씀"
  → Spring Initializr로 프로젝트 생성 시 Flyway가 기본 포함됨
  → 왜 Flyway인지, Liquibase와 차이가 뭔지 모른 채 사용
  → 나중에 Liquibase 기능이 필요한 상황에서 뒤늦게 비교
```

---

## ✨ 올바른 접근 (After — 차이를 이해하고 상황에 맞게 선택)

```
선택 기준을 먼저 정의:

  Q1. 팀이 SQL에 익숙한가, 아니면 DB 추상화가 필요한가?
      SQL 익숙 → Flyway (SQL 파일 직접 작성)
      멀티 DB 지원 필요 → Liquibase (XML/YAML로 DB 독립 선언)

  Q2. 롤백이 실제로 필요한가?
      "프로덕션에서 마이그레이션을 즉시 롤백해야 한다" → Liquibase (제한적 지원)
      "Forward-Only 전략으로 설계한다" → Flyway

  Q3. 팀 규모와 학습 비용을 얼마나 허용하는가?
      빠르게 시작, 단순함 우선 → Flyway
      복잡한 마이그레이션 요구사항, 엔터프라이즈 → Liquibase

  Q4. CI/CD 통합과 감사(Audit) 요구사항이 있는가?
      기본적인 통합 → 둘 다 가능
      복잡한 감사, 컴플라이언스 → Liquibase의 태깅/레이블링 기능 유리
```

---

## 🔬 내부 동작 원리

### 1. 버전 기반 vs 변경셋 — 근본적 차이

```
Flyway — 버전 기반 (SQL 파일):

  파일 구조:
  db/migration/
    V1__create_users.sql
    V2__add_email_to_users.sql
    V3__create_orders.sql

  V2__add_email_to_users.sql 내용:
    ALTER TABLE users ADD COLUMN email VARCHAR(200) NULL;
    CREATE INDEX idx_users_email ON users(email);

  동작 방식:
    - 파일명의 버전 번호(V2)로 실행 순서 결정
    - 한 번 실행된 파일은 다시 실행하지 않음 (체크섬으로 변경 감지)
    - 파일 내용 = 실행될 SQL 그대로

---

Liquibase — 변경셋(ChangeSet) 기반 (XML/YAML):

  파일 구조:
  db/changelog/
    db.changelog-master.xml  ← 진입점, 다른 파일을 include
    changes/
      001-create-users.xml
      002-add-email-to-users.xml

  002-add-email-to-users.xml 내용:
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="...">

    <changeSet id="2" author="developer-a">
      <addColumn tableName="users">
        <column name="email" type="VARCHAR(200)"/>
      </addColumn>
      <createIndex tableName="users" indexName="idx_users_email">
        <column name="email"/>
      </createIndex>
      <rollback>
        <dropIndex tableName="users" indexName="idx_users_email"/>
        <dropColumn tableName="users" columnName="email"/>
      </rollback>
    </changeSet>
  </databaseChangeLog>
  ```

  동작 방식:
    - id + author + 파일경로 조합으로 변경셋 식별
    - DATABASECHANGELOG 테이블에 실행 이력 기록
    - 각 changeSet에 rollback 태그로 롤백 SQL 명시 가능
    - addColumn, createIndex 같은 추상 태그 → DB별 SQL 자동 생성
```

### 2. 상태 추적 테이블 비교

```
Flyway — flyway_schema_history:

installed_rank | version | description        | checksum   | success
1              | 1       | create users       | 1234567890 | 1
2              | 2       | add email to users | -987654321 | 1

  - version: 파일명에서 추출 (V2__ → "2")
  - checksum: CRC32 (파일 변경 감지)
  - 단순하고 직관적

---

Liquibase — DATABASECHANGELOG:

ID  | AUTHOR      | FILENAME                      | DATEEXECUTED        | MD5SUM  | DESCRIPTION | ORDEREXECUTED
1   | developer-a | changes/001-create-users.xml  | 2024-01-01 09:00:00 | 8:abc.. | createTable | 1
2   | developer-a | changes/002-add-email.xml     | 2024-01-15 14:00:00 | 8:def.. | addColumn   | 2

  - ID + AUTHOR + FILENAME 조합으로 고유 식별
  - MD5SUM: 변경 감지
  - DESCRIPTION: changeSet의 작업 자동 요약
  - 더 많은 메타데이터 보존
```

### 3. 롤백 — 실제 사용 가능성

```
Flyway의 롤백:

  기본 Flyway (Community): 공식 롤백 없음
    → Forward-Only 전략: 실수 → 다음 버전에서 수정
    → 이것이 Flyway의 설계 철학 ("DDL은 되돌리지 않는다")

  Flyway Teams (유료): U{버전}__*.sql Undo 파일 지원
    U2__add_email_to_users.sql:
      ALTER TABLE users DROP COLUMN email;
    → flyway undo 명령어로 특정 버전까지 되돌리기 가능
    → 하지만 데이터가 이미 변환된 경우 Undo의 위험성 존재

---

Liquibase의 롤백:

  changeSet 내 <rollback> 태그로 명시적 롤백 SQL 정의
  liquibase rollback --tag=v1.0 → 해당 태그 이전으로 롤백
  liquibase rollbackCount 1 → 마지막 1개 changeSet 롤백

  실제 한계:
    ① 데이터 변환 후 롤백 → 데이터 손실 가능 (Flyway와 동일한 문제)
    ② rollback 태그가 없는 changeSet → 롤백 불가
    ③ 롤백 SQL을 직접 작성해야 함 → 개발자가 실수할 수 있음
    ④ 롤백 자체가 새로운 마이그레이션이 됨

  현실:
    많은 팀이 Liquibase를 써도 롤백 기능을 실제로는 거의 사용하지 않음
    프로덕션에서 DDL 롤백 = 매우 위험한 작업
    → Forward-Only가 더 안전한 선택인 경우가 많음
```

### 4. 멀티 DB 지원

```
Flyway:
  SQL을 직접 작성하므로 DB별 SQL 문법 차이를 개발자가 직접 처리
  MySQL 전용 SQL: ALTER TABLE ... ENGINE=InnoDB
  PostgreSQL 전용: CREATE INDEX CONCURRENTLY
  
  멀티 DB 지원 방법:
    db/migration/mysql/V1__create_users.sql
    db/migration/postgresql/V1__create_users.sql
  → spring.flyway.locations=classpath:db/migration/{vendor}
  → 번거롭지만 SQL을 정확하게 제어 가능

---

Liquibase:
  추상 태그가 DB별 SQL을 자동 생성:
  <createTable tableName="users">
    <column name="id" type="BIGINT" autoIncrement="true">
      <constraints primaryKey="true"/>
    </column>
  </createTable>
  → MySQL: CREATE TABLE users (id BIGINT AUTO_INCREMENT PRIMARY KEY)
  → PostgreSQL: CREATE TABLE users (id BIGSERIAL PRIMARY KEY)
  → Oracle: CREATE TABLE users (id NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY)

  장점: 하나의 changeSet 파일로 여러 DB 지원
  단점: 추상화로 인해 DB별 최적화 옵션을 사용하기 어려움
```

---

## 💻 실전 실험

### 실험 1: Spring Boot에서 두 도구 설정 비교

```yaml
# Flyway 설정 (application.yml)
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/myapp
    username: root
    password: root
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    out-of-order: false
    clean-disabled: true  # 프로덕션 데이터 삭제 방지

# 마이그레이션 파일 위치: src/main/resources/db/migration/V1__*.sql
```

```yaml
# Liquibase 설정 (application.yml)
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/myapp
    username: root
    password: root
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.xml
    drop-first: false  # 프로덕션 데이터 삭제 방지

# 마이그레이션 파일 위치: src/main/resources/db/changelog/
```

### 실험 2: 동일한 마이그레이션을 두 방식으로 작성

```sql
-- Flyway: V1__create_orders.sql
CREATE TABLE orders (
    id         BIGINT       NOT NULL AUTO_INCREMENT,
    user_id    BIGINT       NOT NULL,
    status     VARCHAR(20)  NOT NULL DEFAULT 'PENDING',
    amount     BIGINT       NOT NULL DEFAULT 0,
    created_at DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    PRIMARY KEY (id),
    INDEX idx_orders_user_id (user_id),
    INDEX idx_orders_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

```xml
<!-- Liquibase: changes/001-create-orders.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.0.xsd">

  <changeSet id="1" author="developer">
    <createTable tableName="orders">
      <column name="id" type="BIGINT" autoIncrement="true">
        <constraints primaryKey="true" nullable="false"/>
      </column>
      <column name="user_id" type="BIGINT">
        <constraints nullable="false"/>
      </column>
      <column name="status" type="VARCHAR(20)" defaultValue="PENDING">
        <constraints nullable="false"/>
      </column>
      <column name="amount" type="BIGINT" defaultValueNumeric="0">
        <constraints nullable="false"/>
      </column>
      <column name="created_at" type="DATETIME(6)"
               defaultValueComputed="CURRENT_TIMESTAMP(6)">
        <constraints nullable="false"/>
      </column>
    </createTable>

    <createIndex tableName="orders" indexName="idx_orders_user_id">
      <column name="user_id"/>
    </createIndex>
    <createIndex tableName="orders" indexName="idx_orders_status">
      <column name="status"/>
    </createIndex>

    <rollback>
      <dropTable tableName="orders"/>
    </rollback>
  </changeSet>
</databaseChangeLog>
```

```bash
# Flyway 마이그레이션 상태 확인
flyway -url=jdbc:mysql://localhost:3306/myapp -user=root -password=root info

# Liquibase 마이그레이션 상태 확인
liquibase --url=jdbc:mysql://localhost:3306/myapp \
  --username=root --password=root \
  --changeLogFile=db/changelog/db.changelog-master.xml \
  status
```

---

## 📊 비교

```
Flyway vs Liquibase 종합 비교:

항목                    | Flyway                      | Liquibase
───────────────────────┼────────────────────────────┼────────────────────────────
마이그레이션 형식        | SQL 파일 (직관적)            | XML/YAML/JSON/SQL (유연)
학습 곡선               | 낮음 (SQL만 알면 됨)         | 중간~높음 (changeSet 구조 학습)
멀티 DB 지원            | 수동 (파일 분기)             | 자동 (추상 태그)
롤백                    | 없음 (유료: Undo 파일)       | 있음 (rollback 태그, 제한적)
Spring Boot 기본 통합   | O (spring-boot-starter-data)| O
버전 충돌 관리          | 파일명 버전으로 단순 관리    | id+author 조합으로 관리
감사(Audit) 기능        | 기본적                      | 풍부 (태깅, 컨텍스트, 레이블)
커뮤니티/생태계         | 대규모                      | 대규모
라이선스                | Community(무료)/Teams(유료)  | Community(무료)/Pro(유료)
```

---

## ⚖️ 트레이드오프

```
상황별 선택 가이드:

Flyway를 선택해야 할 때:
  ✓ 팀이 SQL에 익숙하고 별도 학습 비용을 최소화하고 싶을 때
  ✓ MySQL 또는 PostgreSQL 단일 DB 환경
  ✓ 심플한 버전 관리가 우선일 때
  ✓ Spring Boot 프로젝트 (자동 설정 완벽 지원)
  ✓ Forward-Only 전략을 수용할 수 있을 때
  → 대부분의 스타트업, 중소 팀에 적합

Liquibase를 선택해야 할 때:
  ✓ 여러 종류의 DB를 동일한 코드베이스로 지원해야 할 때 (멀티테넌시)
  ✓ 롤백 기능이 실제로 필요한 엔터프라이즈 환경
  ✓ 복잡한 감사(Audit), 컴플라이언스 요구사항이 있을 때
  ✓ 마이그레이션 태깅, 컨텍스트 분리가 필요할 때
  → 대규모 엔터프라이즘, 멀티 DB SaaS에 적합

현실적 조언:
  이 레포는 Flyway를 기준으로 설명
  이유: SQL 직접 작성, Spring Boot 친화적, 학습 비용 낮음
  하지만 위 기준에 해당한다면 Liquibase도 좋은 선택
```

---

## 📌 핵심 정리

```
Flyway vs Liquibase 핵심 차이:

Flyway:
  SQL 파일 직접 작성 → 버전 번호로 순서 관리
  flyway_schema_history에 CRC32 체크섬으로 변경 감지
  롤백 없음 (Free) → Forward-Only 전략 필수
  심플함, 낮은 학습 비용

Liquibase:
  changeSet(XML/YAML)으로 변경 선언 → DB 독립 추상화
  DATABASECHANGELOG에 MD5로 변경 감지
  rollback 태그로 제한적 롤백 가능
  복잡하지만 강력한 기능 (태깅, 멀티 DB, 감사)

공통점:
  마이그레이션을 코드로 관리
  상태 추적 테이블로 이력 보존
  Spring Boot 자동 설정 지원
  CI/CD 통합 가능
```

---

## 🤔 생각해볼 문제

**Q1.** Liquibase에서 `<rollback>` 태그를 작성할 때 모든 changeSet에 대해 정확한 롤백 SQL을 작성할 수 있는가? 어떤 경우에 롤백이 불가능한가?

<details>
<summary>해설 보기</summary>

구조적 변경(컬럼 추가, 테이블 생성, 인덱스 추가)은 롤백 SQL 작성이 가능합니다. 예를 들어 `addColumn`의 롤백은 `dropColumn`입니다.

하지만 다음 경우에는 롤백이 위험하거나 불가능합니다.

**데이터 변환을 포함한 경우**: `UPDATE users SET status = 'ACTIVE' WHERE status IS NULL`과 같은 데이터 변환은 롤백 시 원래 `NULL`이었는지 `'INACTIVE'`였는지 알 수 없습니다. 롤백 SQL을 작성해도 데이터 손실이 발생합니다.

**`DROP` 작업**: 컬럼이나 테이블을 삭제했을 때 이미 데이터가 삭제됩니다. 롤백으로 컬럼을 다시 추가해도 삭제된 데이터는 복구되지 않습니다.

**복잡한 비즈니스 로직**: 여러 테이블에 걸친 데이터 마이그레이션은 순서가 중요하기 때문에 단순한 역방향 SQL로는 올바른 롤백이 보장되지 않습니다.

결론적으로, 롤백 기능이 있다고 해서 안전한 롤백이 보장되는 것은 아닙니다. 이것이 Forward-Only 전략이 더 현실적인 이유입니다.

</details>

---

**Q2.** 팀이 이미 Flyway를 사용하고 있는데, 멀티 테넌시 요구사항이 생겨 여러 종류의 DB(MySQL, PostgreSQL)를 지원해야 한다. Flyway로 해결할 수 있는가?

<details>
<summary>해설 보기</summary>

Flyway로도 멀티 DB를 지원할 수 있지만, 추가적인 구조 설계가 필요합니다.

```yaml
# application.yml
spring:
  flyway:
    locations: classpath:db/migration/common,classpath:db/migration/{vendor}
```

```
db/migration/
  common/         # MySQL, PostgreSQL 공통 SQL
    V1__create_users.sql
  mysql/          # MySQL 전용 SQL
    V2__add_fulltext_index.sql
  postgresql/     # PostgreSQL 전용 SQL
    V2__add_fulltext_index.sql
```

`{vendor}` 플레이스홀더는 런타임에 DB 종류(mysql, postgresql)로 대체됩니다.

이 방식의 단점은 공통 파일과 DB별 파일의 버전 번호를 관리하는 것이 복잡해진다는 점입니다. 또한 DB별 SQL 문법 차이를 개발자가 직접 관리해야 합니다.

멀티 DB 지원이 핵심 요구사항이라면 Liquibase의 추상 태그 방식이 더 적합할 수 있습니다. 이미 Flyway로 많은 마이그레이션이 쌓인 상태라면 마이그레이션 비용을 먼저 평가해야 합니다.

</details>

---

**Q3.** Flyway Community에서 Teams(유료)로 업그레이드할 때 기존 마이그레이션 파일에 영향이 있는가?

<details>
<summary>해설 보기</summary>

기존 마이그레이션 파일(`V{버전}__*.sql`)에는 전혀 영향이 없습니다. Flyway Teams는 Community 위에 추가 기능을 얹은 것이기 때문에 기존 `flyway_schema_history` 테이블과 마이그레이션 파일은 그대로 사용됩니다.

Teams에서 추가되는 주요 기능은 다음과 같습니다. `U{버전}__*.sql` Undo 마이그레이션 지원, `flyway undo` 명령어, Dry Run(실제 실행 없이 SQL 미리 확인), Cherry Pick(특정 버전만 선택 적용), Report 생성 등입니다.

업그레이드 방식은 의존성 변경만으로 가능합니다.

```xml
<!-- Community -->
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-core</artifactId>
</dependency>

<!-- Teams -->
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-core</artifactId>
  <version>9.x.x</version>  <!-- Teams 라이선스 키 필요 -->
</dependency>
```

대부분의 팀에서 Community 버전으로 충분하며, Undo 마이그레이션이 꼭 필요한 경우에만 Teams 도입을 검토하는 것이 좋습니다.

</details>

---

<div align="center">

**[⬅️ 이전: 스키마를 코드로 관리해야 하는 이유](./01-why-schema-as-code.md)** | **[홈으로 🏠](../README.md)** | **[다음: ddl-auto=update 금지 이유 ➡️](./03-ddl-auto-update-forbidden.md)**

</div>
