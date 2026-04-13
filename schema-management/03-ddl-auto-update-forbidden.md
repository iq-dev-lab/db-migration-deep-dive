# `ddl-auto=update` 금지 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `spring.jpa.hibernate.ddl-auto=update`는 내부적으로 어떻게 동작하는가?
- 왜 개발 환경에서는 괜찮아 보이는데 프로덕션에서 재앙이 되는가?
- `update` 모드가 의도치 않게 컬럼을 삭제하거나 타입을 변경하는 시나리오는?
- 프로덕션에서 안전한 `ddl-auto` 값은 무엇인가?
- Flyway와 `ddl-auto`를 함께 사용할 때 올바른 설정은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`ddl-auto=update`는 "편리한 기능"처럼 보이지만, 프로덕션에서는 통제할 수 없는 스키마 변경을 유발하는 가장 위험한 설정 중 하나다. Hibernate가 Entity 클래스를 분석해 스키마를 자동으로 맞춰주는 것처럼 보이지만, "맞춘다"의 기준이 개발자의 의도와 다를 수 있다. 그리고 그 차이가 프로덕션 데이터 손실로 이어진다.

---

## 😱 흔한 실수 (Before — update 모드를 프로덕션에서 사용)

```
시나리오 1: 의도치 않은 컬럼 타입 변경

  Entity:
    @Column(name = "amount")
    private Integer amount;  // 처음엔 Integer로 설계

  운영 중 코드 변경:
    @Column(name = "amount")
    private Long amount;     // 범위가 부족해서 Long으로 변경

  ddl-auto=update 결과:
    MySQL: ALTER TABLE orders MODIFY COLUMN amount BIGINT
    → 기존 데이터는 보존되지만 Lock 발생 + 인덱스 재구성
    → 수천만 건이면 수십 분 다운타임

  ddl-auto=update가 위험한 이유:
    개발자가 DDL Lock을 전혀 인식하지 못한 채
    앱 배포만으로 프로덕션 스키마가 변경됨

시나리오 2: 컬럼 삭제가 반영 안 되는 위험

  Entity에서 필드 제거:
    // @Column(name = "old_field") 삭제
    // private String oldField;  ← 제거

  ddl-auto=update 결과:
    Hibernate는 기존 컬럼을 삭제하지 않음
    → DB에는 old_field 컬럼이 그대로 남음
    → 개발자는 "삭제됐겠지"라고 착각
    → 실제 DB에는 몇 년간 사용되지 않는 컬럼 누적

  나중에 다른 개발자가 old_field를 다른 용도로 사용하려 할 때:
    "이 컬럼은 뭐였지?" — 이력 없음

시나리오 3: 인덱스 관리 불가

  Entity:
    @Index(name = "idx_users_email", columnList = "email")
  
  ddl-auto=update:
    인덱스 추가는 되지만 인덱스 삭제는 안 됨
    → Entity에서 @Index를 제거해도 DB에 인덱스 잔존
    → 또는 인덱스 이름이 바뀌면 기존 인덱스를 유지한 채 새 인덱스 추가
    → 불필요한 인덱스가 쌓이며 쓰기 성능 저하

시나리오 4: 프로덕션 배포 중 예외

  여러 인스턴스가 동시에 배포될 때:
    인스턴스 A: 시작 → ddl-auto=update → ALTER TABLE 실행 (Lock!)
    인스턴스 B: 동시 시작 → ALTER TABLE 실행 시도 → Lock 대기 또는 타임아웃
    → 배포 중 서비스 불안정
```

---

## ✨ 올바른 접근 (After — validate 모드 + Flyway)

```
올바른 프로덕션 설정:

spring:
  jpa:
    hibernate:
      ddl-auto: validate   # 시작 시 Entity와 스키마 불일치 감지 (변경 X)
  flyway:
    enabled: true          # 스키마 변경은 Flyway가 담당

ddl-auto 값별 동작:
  create      → 시작 시 모든 테이블 DROP 후 재생성 (개발 초기에만)
  create-drop → create + 종료 시 DROP (테스트에만)
  update      → 시작 시 Entity와 스키마 비교 후 업데이트 (위험!)
  validate    → 시작 시 Entity와 스키마 비교 (불일치 시 예외, 변경 X)
  none        → 아무것도 하지 않음

환경별 권장 설정:
  로컬 개발:  create-drop 또는 update (빠른 프로토타이핑)
  개발 서버:  validate + flyway (팀 공유 환경부터 일관성 필요)
  스테이징:   validate + flyway
  프로덕션:   validate + flyway

validate 모드의 장점:
  - Entity와 실제 DB 스키마 불일치를 시작 시 즉시 감지
  - "배포 전에 마이그레이션 파일을 잊었다" → 앱 시작 실패 → 즉시 발견
  - 스키마 변경은 반드시 Flyway 마이그레이션 파일을 통해서만 가능
```

---

## 🔬 내부 동작 원리

### 1. ddl-auto=update의 실제 동작 분석

```
Hibernate SchemaUpdate 동작 순서:

1단계: 현재 DB 스키마 읽기
  DatabaseMetaData로 실제 테이블, 컬럼, 인덱스 정보 조회
  SELECT * FROM information_schema.COLUMNS WHERE TABLE_NAME = 'users';

2단계: Entity 클래스 분석
  @Entity, @Table, @Column, @Index 어노테이션 파싱
  JPA 메타모델 생성

3단계: 차이(diff) 계산
  DB에 없는 테이블 → CREATE TABLE 실행
  DB에 없는 컬럼 → ALTER TABLE ADD COLUMN 실행
  DB에 없는 인덱스 → CREATE INDEX 실행
  타입 변경 (일부) → ALTER TABLE MODIFY COLUMN 실행

4단계: update가 하지 않는 것들
  DB에 있지만 Entity에 없는 컬럼 → 삭제하지 않음 (데이터 손실 방지)
  DB에 있지만 Entity에 없는 테이블 → 삭제하지 않음
  DB에 있지만 Entity에 없는 인덱스 → 삭제하지 않음
  타입 변경 (일부) → 처리 못 하고 그냥 통과하기도 함

결론:
  "update"라는 이름과 달리 부분적인 단방향 동기화만 수행
  "DB → Entity" 방향의 변경은 처리하지 않음
  DDL Lock을 개발자가 제어할 수 없음
```

### 2. validate 모드 + Flyway의 동작

```
Spring Boot 시작 순서:

1. Flyway 마이그레이션 실행 (스키마 변경)
   └── flyway_schema_history 확인
   └── 미적용 마이그레이션 실행
   └── 완료

2. Hibernate SchemaValidator 실행 (검증만)
   └── Entity 클래스 분석 → JPA 메타모델 생성
   └── DB 실제 스키마 조회
   └── 비교:
       ├── Entity에 있는 컬럼이 DB에 없으면 → SchemaManagementException!
       │   "Missing column: users.phone"
       │   → 마이그레이션 파일을 추가하지 않은 것을 즉시 발견
       └── 일치하면 → 정상 시작

이 순서의 장점:
  Flyway가 먼저 스키마를 최신 상태로 맞추고
  Hibernate가 검증만 수행
  → 두 역할이 명확히 분리
  → 스키마 변경은 반드시 SQL 파일로만 가능

실제 오류 예시:
  org.hibernate.tool.schema.spi.SchemaManagementException:
    Schema-validation: missing column [phone] in table [users]
  
  → 이 오류가 뜨면 즉시 알 수 있음:
    "V{버전}__add_phone_to_users.sql 파일을 만들었어야 했는데 빠트렸구나"
```

### 3. 타입 불일치가 유발하는 실제 DDL

```
Entity 변경: Integer → Long (amount 컬럼)

Hibernate가 실행하는 DDL (ddl-auto=update 시):

MySQL:
  ALTER TABLE orders MODIFY COLUMN amount BIGINT NOT NULL DEFAULT 0;
  
  이 DDL의 실제 영향:
    - MySQL 8.0에서 INT → BIGINT 변환은 ALGORITHM=INPLACE 가능
    - 하지만 NOT NULL + DEFAULT 변경이 동시에 일어나면 COPY 알고리즘 필요
    - COPY: 임시 테이블 생성 → 전체 데이터 복사 → 원본 교체
    - 수천만 건이면 수십 분간 Write Lock 발생
    - 그동안 INSERT/UPDATE/DELETE 불가

개발자는 이 DDL이 실행되는지도 모름:
  로그에 Hibernate: alter table orders modify column amount bigint 한 줄
  Lock 시간 측정도 없음, 영향 분석도 없음
  → 그냥 배포했더니 10분간 서비스 장애
```

---

## 💻 실전 실험

### 실험 1: update 모드의 위험한 동작 확인

```java
// Step 1: 초기 Entity
@Entity
@Table(name = "products")
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "name")
    private String name;
    
    @Column(name = "price")
    private Integer price;  // Integer
}
```

```yaml
# application.yml
spring:
  jpa:
    hibernate:
      ddl-auto: update
  datasource:
    url: jdbc:mysql://localhost:3306/test
```

```bash
# 앱 실행 → products 테이블 생성
# Hibernate 로그 확인
# create table products (id bigint not null auto_increment, name varchar(255), price integer, primary key (id))

# DB 확인
mysql -uroot -proot test -e "DESCRIBE products;"
# +-------+--------------+------+
# | Field | Type         | Null |
# +-------+--------------+------+
# | id    | bigint       | NO   |
# | name  | varchar(255) | YES  |
# | price | int          | YES  |
# +-------+--------------+------+
```

```java
// Step 2: price를 Long으로 변경
@Column(name = "price")
private Long price;  // Integer → Long
```

```bash
# 앱 재시작 → ddl-auto=update가 ALTER TABLE 자동 실행
# Hibernate 로그:
# alter table products modify column price bigint

# 이 DDL이 Lock을 걸었는지 확인 (별도 세션에서)
# SELECT * FROM performance_schema.metadata_locks WHERE OBJECT_NAME='products';

# DB 확인
mysql -uroot -proot test -e "DESCRIBE products;"
# price 컬럼이 int → bigint로 변경됨 (Lock 발생 여부는 로그에 없음)
```

### 실험 2: validate 모드에서 불일치 감지

```java
// Entity에 phone 컬럼 추가
@Column(name = "phone")
private String phone;
// 하지만 마이그레이션 파일을 작성하지 않음
```

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate  # validate 모드
  flyway:
    enabled: true
```

```bash
# 앱 시작 시도 → Flyway 실행 (phone 컬럼 추가 마이그레이션 없음)
# Hibernate 검증 시도 → 실패

# 로그:
# org.hibernate.tool.schema.spi.SchemaManagementException:
#   Schema-validation: missing column [phone] in table [users]
# APPLICATION FAILED TO START
#
# → 프로덕션 배포 전에 즉시 발견!
# → 마이그레이션 파일 추가 후 재배포
```

### 실험 3: 올바른 Flyway + validate 설정

```yaml
# application.yml (프로덕션)
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/myapp
    username: root
    password: root
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        format_sql: false
  flyway:
    enabled: true
    locations: classpath:db/migration
    clean-disabled: true      # 프로덕션 데이터 삭제 방지
    validate-on-migrate: true # 마이그레이션 전 체크섬 검증

---
# application-local.yml (로컬 개발)
spring:
  jpa:
    hibernate:
      ddl-auto: create-drop   # 로컬에서는 빠른 프로토타이핑
  flyway:
    enabled: false            # 로컬에서는 Flyway 비활성화
```

---

## 📊 비교

```
ddl-auto 옵션별 비교:

옵션          | 시작 시 동작               | 안전성   | 적합 환경
─────────────┼──────────────────────────┼─────────┼──────────────────
none         | 아무것도 하지 않음          | ★★★★★  | 프로덕션 (Flyway 사용 시)
validate     | 불일치 감지 → 예외 발생     | ★★★★★  | 프로덕션, 스테이징, 개발
update       | Entity 기준 부분 업데이트   | ★★☆☆☆  | 로컬 개발 (주의)
create       | DROP 후 재생성             | ★☆☆☆☆  | 로컬 초기 개발
create-drop  | create + 종료 시 DROP      | ★☆☆☆☆  | 단위 테스트

Flyway + ddl-auto 조합:
  Flyway enabled=true + ddl-auto=validate → 권장 (역할 분리)
  Flyway enabled=true + ddl-auto=none     → 가능 (validate 없음)
  Flyway enabled=true + ddl-auto=update   → 위험 (두 도구가 충돌)
  Flyway enabled=false + ddl-auto=update  → 위험 (프로덕션 절대 금지)
```

---

## ⚖️ 트레이드오프

```
ddl-auto=update 유지 주장 vs 금지 주장:

유지 주장:
  "개발 속도가 빠르다 — 마이그레이션 파일 작성 없이 Entity 변경만 하면 됨"
  "소규모 팀, 초기 단계에서는 충분하다"

금지 주장:
  "프로덕션에서 한 번 사고가 나면 모든 개발 속도 이점이 사라진다"
  "개발과 프로덕션에서 다른 도구를 쓰면 환경 불일치 발생"
  "DDL Lock 위험을 인식하지 못한 채 배포가 진행된다"

현실적 절충:
  로컬 개발: update 허용 (빠른 프로토타이핑, 데이터 손실 OK)
  개발 서버부터: validate + Flyway (팀 공유 환경은 일관성 필요)
  스테이징/프로덕션: validate + Flyway 필수

핵심 원칙:
  "개발에서 쓰는 방식이 프로덕션 동작을 설명하지 못하면, 개발은 의미가 없다"
  validate 모드는 "배포 전 마지막 안전망" 역할
```

---

## 📌 핵심 정리

```
ddl-auto=update 금지 이유 3가지:

1. 통제 불가능한 DDL 실행
   Entity 변경이 곧 프로덕션 ALTER TABLE
   DDL Lock 시간, 영향 분석 없이 자동 실행

2. 부분적 동기화
   컬럼 삭제, 인덱스 삭제 반영 안 됨
   DB 상태가 Entity와 실제로 다를 수 있음

3. 환경별 불일치
   로컬에서만 잘 작동하다가 프로덕션에서 예상치 못한 동작

올바른 설정:
  개발: ddl-auto=create-drop (빠른 프로토타이핑)
  프로덕션: ddl-auto=validate + Flyway enabled=true

validate의 효과:
  Entity와 스키마 불일치 → 앱 시작 실패 → 마이그레이션 누락 즉시 발견
  스키마 변경은 반드시 Flyway 마이그레이션 파일로만 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `ddl-auto=validate` 모드에서 `@Column(nullable = false)`로 설정된 필드가 DB에는 NULL 허용으로 되어 있다면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Hibernate의 validate 모드는 컬럼 존재 여부와 타입 호환성을 검사하지만, `nullable` 제약의 불일치에 대해서는 버전과 설정에 따라 다르게 동작합니다.

일반적으로 Hibernate validate는 다음을 검사합니다. 컬럼 존재 여부(`missing column`), 기본적인 타입 호환성 등입니다.

반면 `nullable` 제약, 컬럼 길이(`length`), 정밀도(`precision`) 등의 세부 제약은 검사하지 않거나 경고만 출력하는 경우가 많습니다. 이는 Hibernate validate의 한계입니다.

이런 세부 불일치까지 검사하려면 Flyway의 `validate` 명령어와 조합하거나, 별도 스키마 비교 도구(예: SchemaCrawler)를 활용해야 합니다. 또는 통합 테스트에서 실제 INSERT/UPDATE 시도를 통해 제약 위반을 발견하는 방법도 있습니다.

</details>

---

**Q2.** JPA Entity에 `@Column(name = "created_at", updatable = false, insertable = false)`로 설정한 컬럼을 마이그레이션으로 추가했다. validate 모드에서 이 컬럼은 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

`updatable = false, insertable = false`는 JPA 레벨에서 이 컬럼을 INSERT/UPDATE SQL에 포함하지 않겠다는 의미입니다. DB 스키마에는 영향을 주지 않습니다.

validate 모드에서는 해당 컬럼이 DB에 실제로 존재하는지만 검사합니다. 컬럼이 DB에 있으면 통과이고, 없으면 `missing column` 오류가 발생합니다.

`insertable = false`로 설정된 컬럼은 보통 DB의 `DEFAULT CURRENT_TIMESTAMP`처럼 DB가 자동으로 값을 채워주는 경우에 사용합니다. 이런 컬럼은 마이그레이션 파일에서 `DEFAULT CURRENT_TIMESTAMP(6)`을 명시해야 하며, 애플리케이션 코드에서는 해당 컬럼에 값을 직접 넣지 않습니다.

</details>

---

**Q3.** `ddl-auto=none`과 `ddl-auto=validate`의 차이는? 언제 `none`을 선택해야 하는가?

<details>
<summary>해설 보기</summary>

`ddl-auto=none`은 Hibernate가 스키마와 관련된 어떤 작업도 하지 않습니다. 시작 시 스키마 검증 자체를 수행하지 않으므로, Entity와 DB 스키마가 불일치해도 앱이 정상 시작됩니다.

`ddl-auto=validate`는 시작 시 Entity와 DB 스키마를 비교하고, 불일치가 있으면 `SchemaManagementException`을 던져 앱 시작을 실패시킵니다.

`none`을 선택해야 하는 경우는 다음과 같습니다. Entity가 DB 전체 스키마의 일부만 매핑하는 경우(레거시 DB에 JPA를 덧씌운 경우), 스키마 검증 자체가 시작 시간을 지연시키는 경우(테이블이 매우 많을 때), 외부 마이그레이션 도구(Flyway)가 이미 스키마를 완전히 관리하고 validate가 불필요한 중복 검사라고 판단할 때입니다.

일반적으로 `validate`가 더 안전합니다. 마이그레이션 파일 누락을 시작 시 즉시 발견할 수 있기 때문입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Flyway vs Liquibase](./02-flyway-vs-liquibase.md)** | **[홈으로 🏠](../README.md)** | **[다음: 마이그레이션 파일 명명 규칙 ➡️](./04-naming-convention.md)**

</div>
