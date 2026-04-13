<div align="center">

# 🗄️ DB Migration Deep Dive

**"SQL을 직접 실행하는 것과, 스키마 변경을 코드처럼 관리하고 안전하게 프로덕션에 적용하는 것은 다르다"**

<br/>

> *"`ALTER TABLE` 직접 실행하고 팀원한테 Slack으로 공유했어 — 와 — Flyway 마이그레이션이 CI/CD에 통합되어 배포 시 자동 적용되고, 프로덕션 Lock 없이 Online DDL로 실행되며, 실패 시 명확한 복구 절차가 있는 것의 차이를 만드는 레포"*

`flyway_schema_history`가 체크섬으로 파일 변경을 감지하는 원리, `ALTER TABLE`이 왜 프로덕션 테이블 전체에 Lock을 거는가, Expand-Contract 패턴으로 컬럼을 무중단으로 이름 변경하는 법, DDL은 롤백이 없다는 전제 아래 Forward-Only 전략을 설계하는 법까지  
**왜 이렇게 설계됐는가** 라는 질문으로 DB 마이그레이션의 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Flyway](https://img.shields.io/badge/Flyway-9.x-CC0200?style=flat-square&logo=flyway&logoColor=white)](https://documentation.red-gate.com/fd/)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?style=flat-square&logo=mysql&logoColor=white)](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl.html)
[![Spring](https://img.shields.io/badge/Spring_Boot-3.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://docs.spring.io/spring-boot/docs/current/reference/html/)
[![Docs](https://img.shields.io/badge/Docs-38개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

DB 마이그레이션에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "Flyway를 의존성에 추가하면 자동으로 마이그레이션됩니다" | `FlywayAutoConfiguration` → `flyway_schema_history` 조회 → CRC32 체크섬 비교 → 미적용 SQL 순서대로 실행 전 과정, 체크섬이 파일 변경을 어떻게 감지하는지 |
| "`spring.jpa.hibernate.ddl-auto=update`를 쓰면 됩니다" | Hibernate가 스키마를 자동 수정할 때 의도치 않은 컬럼 삭제, 인덱스 누락이 발생하는 이유, 프로덕션에서 `validate` 모드가 필수인 이유 |
| "`ALTER TABLE`로 컬럼을 추가하세요" | InnoDB Online DDL의 `ALGORITHM=COPY/INPLACE/INSTANT` 차이, 수억 건 테이블에서 `ALTER TABLE`이 Write Lock을 거는 조건, gh-ost가 프로덕션 Lock 없이 스키마를 변경하는 원리 |
| "마이그레이션 파일을 `V1__`, `V2__` 순서로 만드세요" | 두 개발자가 동시에 `V3__*.sql`을 만들 때 발생하는 충돌, `out-of-order` 설정의 허용 범위, 타임스탬프 기반 버전으로 충돌을 최소화하는 팀 전략 |
| "롤백 마이그레이션을 준비하세요" | MySQL DDL이 Auto-Commit인 이유, DDL 롤백이 원천적으로 불가능한 이유, Forward-Only 전략으로 앞으로만 나아가는 설계, PostgreSQL Transactional DDL과의 차이 |
| "CI/CD에 마이그레이션을 통합하면 됩니다" | 애플리케이션 시작 시 자동 실행 vs 별도 Job으로 실행의 실패 시나리오 비교, Kubernetes Init Container로 마이그레이션을 앱 시작 전에 완료하는 패턴 |
| 이론 나열 | 실행 가능한 SQL 마이그레이션 파일 + `flyway` CLI 실험 + `information_schema.innodb_trx`로 Lock 재현 + Docker Compose 환경 |

---

## 🔗 선행 학습 및 연결 레포

이 레포를 최대한 활용하려면 다음 레포를 먼저 학습하면 시너지가 큽니다.

| 레포 | 연결 지점 |
|------|----------|
| **database-internals** | InnoDB 내부 구조, B+Tree 인덱스, DDL Lock 원리 — Chapter 3 Zero-Downtime Migration의 이론적 기반 |
| **spring-data-transaction** | JPA Entity 매핑, `ddl-auto` 설정, Spring 통합 맥락 — Chapter 7 Spring 통합의 선행 지식 |
| **cicd-deep-dive** | 배포 파이프라인 구조, GitHub Actions, Kubernetes 배포 — Chapter 6 CI/CD 통합의 맥락 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Chapter1](https://img.shields.io/badge/🔹_Chapter1-스키마를_코드로_관리해야_하는_이유-CC0200?style=for-the-badge&logo=flyway&logoColor=white)](./schema-management/01-why-schema-as-code.md)
[![Chapter2](https://img.shields.io/badge/🔹_Chapter2-Flyway_내부_동작_원리-CC0200?style=for-the-badge&logo=flyway&logoColor=white)](./flyway-internals/01-flyway-schema-history.md)
[![Chapter3](https://img.shields.io/badge/🔹_Chapter3-DDL이_Lock을_거는_원리-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](./zero-downtime-migration/01-ddl-lock-internals.md)
[![Chapter4](https://img.shields.io/badge/🔹_Chapter4-DDL_롤백이_없는_이유-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](./rollback-recovery/01-why-ddl-no-rollback.md)
[![Chapter5](https://img.shields.io/badge/🔹_Chapter5-마이그레이션_버전_충돌-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./team-collaboration/01-version-conflict.md)
[![Chapter6](https://img.shields.io/badge/🔹_Chapter6-배포_파이프라인_통합-181717?style=for-the-badge&logo=github-actions&logoColor=white)](./cicd-integration/01-migration-timing-in-pipeline.md)
[![Chapter7](https://img.shields.io/badge/🔹_Chapter7-Spring_Boot_+_Flyway_자동_설정-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./spring-integration/01-spring-boot-flyway-autoconfigure.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: 스키마 관리의 필요성과 도구 비교

> **핵심 질문:** 왜 DDL을 직접 실행하면 안 되는가? Flyway와 Liquibase는 어떻게 다르고, `ddl-auto=update`는 왜 프로덕션에서 금지인가?

<details>
<summary><b>수동 DDL 실행의 문제부터 환경별 마이그레이션 전략까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 스키마를 코드로 관리해야 하는 이유](./schema-management/01-why-schema-as-code.md) | 수동 DDL 실행의 3가지 문제(팀원 누락, 환경별 불일치, 이력 없음), IaC(Infrastructure as Code)와 동일한 철학으로 DB 스키마를 Git으로 관리하는 원칙, "DB도 코드처럼 버전 관리한다"는 패러다임 전환 |
| [02. Flyway vs Liquibase — 버전 기반 vs 변경 기반](./schema-management/02-flyway-vs-liquibase.md) | Flyway의 버전 기반 SQL 파일 방식과 Liquibase의 변경셋(XML/YAML) 방식의 근본적 차이, 롤백 지원 여부, 학습 곡선, 대규모 팀에서의 선택 기준과 트레이드오프 |
| [03. ddl-auto=update 금지 이유](./schema-management/03-ddl-auto-update-forbidden.md) | Hibernate가 스키마를 자동 수정할 때 발생하는 위험(의도치 않은 컬럼 삭제, 인덱스 누락, 타입 불일치), 개발/프로덕션 환경 분리 원칙, `validate` 모드로 불일치를 감지하는 안전한 방법 |
| [04. 마이그레이션 파일 명명 규칙](./schema-management/04-naming-convention.md) | `V{버전}__{설명}.sql` 형식의 규칙과 파싱 원리, 순차 번호 방식의 충돌 취약성, 타임스탬프(`yyyyMMddHHmmss`) 기반 버전으로 충돌을 최소화하는 팀 협약, PR 머지 순서와 버전 번호의 관계 |
| [05. 마이그레이션 환경 전략](./schema-management/05-environment-strategy.md) | 로컬/개발/스테이징/프로덕션 각 환경별 마이그레이션 적용 방식, `clean-disabled` 설정으로 프로덕션 데이터 삭제를 방지하는 방법, 환경별 시드 데이터를 Repeatable 마이그레이션으로 분리하는 전략 |

</details>

<br/>

### 🔹 Chapter 2: Flyway 완전 분해

> **핵심 질문:** `flyway_schema_history`는 어떤 구조이고, 체크섬은 어떻게 파일 변경을 감지하는가? Versioned/Repeatable/Undo 마이그레이션은 언제 각각 사용하는가?

<details>
<summary><b>flyway_schema_history 내부부터 체크섬 불일치 복구까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Flyway 내부 동작 원리 — flyway_schema_history](./flyway-internals/01-flyway-schema-history.md) | `flyway_schema_history` 테이블의 `version`, `checksum`, `success` 필드 구조, Spring Boot 시작 시 Flyway가 수행하는 검사 순서(파일 스캔 → CRC32 체크섬 계산 → 테이블과 비교), 미적용 마이그레이션을 순서대로 실행하는 메커니즘 |
| [02. 마이그레이션 유형 — Versioned, Repeatable, Undo](./flyway-internals/02-migration-types.md) | `V`(버전 고정, 한 번만 실행), `R`(체크섬 변경 시마다 재실행), `U`(Undo, 롤백용) 마이그레이션의 실행 시점과 적합한 시나리오, Repeatable로 뷰·함수·시드 데이터를 자동 갱신하는 패턴 |
| [03. Flyway 설정 옵션 완전 분석](./flyway-internals/03-flyway-config-options.md) | `baseline-on-migrate`(기존 DB에 Flyway 도입 시), `out-of-order`(순서가 뒤바뀐 마이그레이션 허용), `ignore-missing-migrations`(삭제된 파일 무시), `clean-disabled`(프로덕션 데이터 삭제 방지) 각 옵션이 실제로 하는 일과 잘못 설정했을 때의 위험 |
| [04. Java 기반 마이그레이션](./flyway-internals/04-java-migration.md) | `BaseJavaMigration` 구현으로 SQL로 표현 못하는 복잡한 데이터 변환(JSON 파싱, 외부 API 호출, 조건부 변환) 처리, Java 마이그레이션의 트랜잭션 처리, SQL 마이그레이션과의 실행 순서 |
| [05. Flyway Callbacks — 마이그레이션 전후 자동화](./flyway-internals/05-flyway-callbacks.md) | `beforeMigrate`, `afterEachMigrate`, `afterMigrate`, `beforeClean` 콜백의 실행 시점, 마이그레이션 전후 권한 재부여·캐시 초기화·Slack 알림 자동화 구현, SQL Callback과 Java Callback 비교 |
| [06. 체크섬 불일치 오류 해결](./flyway-internals/06-checksum-mismatch.md) | 이미 적용된 마이그레이션 파일을 수정했을 때 발생하는 체크섬 불일치 오류, `flyway repair`로 `flyway_schema_history` 체크섬을 재계산하는 절차, 팀에서 충돌을 예방하는 규칙(커밋 전 검증, PR 리뷰 체크리스트) |

</details>

<br/>

### 🔹 Chapter 3: Zero-Downtime Migration 전략

> **핵심 질문:** `ALTER TABLE`이 프로덕션을 중단시키는 이유는 무엇인가? Lock 없이 컬럼을 추가·삭제·이름 변경하는 방법은?

<details>
<summary><b>InnoDB DDL Lock 원리부터 외래 키 없는 무결성 보장까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. DDL이 Lock을 거는 원리](./zero-downtime-migration/01-ddl-lock-internals.md) | InnoDB Online DDL의 Lock 수준(EXCLUSIVE/SHARED/NONE) 결정 기준, `ALTER TABLE`이 테이블 전체를 잠그는 조건, 수백만 건 테이블에서 DDL이 서비스를 중단시키는 시나리오, `information_schema.innodb_trx`와 `performance_schema.metadata_locks`로 Lock 상태 확인 |
| [02. Online DDL과 gh-ost](./zero-downtime-migration/02-online-ddl-gh-ost.md) | `ALGORITHM=INPLACE`/`COPY`/`INSTANT`의 차이와 각각의 Lock 수준, gh-ost(GitHub Online Schema Migrations)가 Shadow 테이블과 Binlog를 활용해 프로덕션 Lock 없이 스키마를 변경하는 원리, pt-online-schema-change와의 비교 |
| [03. Expand-Contract 패턴](./zero-downtime-migration/03-expand-contract-pattern.md) | Breaking Change 없이 컬럼을 변경하는 3단계(Expand: 새 컬럼 추가 → Migrate: 데이터 복사 → Contract: 구 컬럼 제거), 애플리케이션 배포와 마이그레이션을 분리하는 이유, 각 단계에서 하위 호환성을 유지하는 코드 패턴 |
| [04. 안전한 컬럼 추가](./zero-downtime-migration/04-add-column-safely.md) | `NOT NULL` 컬럼을 기본값 없이 추가하면 왜 안 되는가(기존 행 처리 문제), `NULL`로 추가 → 배치 백필 → `NOT NULL` 제약 추가의 3단계 Zero-Downtime 순서, `ALGORITHM=INSTANT`(MySQL 8.0.29+)로 즉시 추가 가능한 조건 |
| [05. 컬럼 이름 변경 — 직접 RENAME의 위험](./zero-downtime-migration/05-rename-column.md) | 직접 `RENAME COLUMN`이 Breaking Change인 이유(JPA 매핑, 쿼리 직접 참조), 새 컬럼 추가 → 두 컬럼 동시 쓰기 → 데이터 복사 → 새 컬럼만 읽기 → 구 컬럼 삭제의 단계적 전략, 각 단계별 배포와 마이그레이션 타이밍 |
| [06. 인덱스 추가 — 대용량 테이블 전략](./zero-downtime-migration/06-add-index-safely.md) | `CREATE INDEX`가 프로덕션에서 느린 이유(테이블 전체 스캔, 정렬), MySQL Online DDL의 `CREATE INDEX` Lock 수준, PostgreSQL의 `CREATE INDEX CONCURRENTLY`와 비교, 수억 건 테이블에서 인덱스를 추가하는 gh-ost 활용 전략 |
| [07. 외래 키 제약 관리](./zero-downtime-migration/07-foreign-key-management.md) | 외래 키 추가 시 기존 데이터 전체 검증으로 발생하는 Lock, 데이터 정합성 검증을 애플리케이션 레벨로 이동하는 전략, 외래 키 없이 참조 무결성을 보장하는 방법, MSA 환경에서 외래 키를 제거하는 이유 |

</details>

<br/>

### 🔹 Chapter 4: 롤백 전략과 장애 복구

> **핵심 질문:** DDL 롤백이 왜 불가능한가? 마이그레이션이 실패했을 때 어떻게 복구하는가?

<details>
<summary><b>DDL Auto-Commit 특성부터 백업 기반 복구까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. DDL 롤백이 없는 이유](./rollback-recovery/01-why-ddl-no-rollback.md) | MySQL DDL의 Auto-Commit 특성, 데이터 변경(DML)은 롤백 가능하지만 스키마 변경(DDL)은 불가능한 이유, PostgreSQL의 Transactional DDL이 DDL 롤백을 지원하는 원리와 MySQL과의 차이, 이 비대칭성이 마이그레이션 설계에 미치는 영향 |
| [02. Forward-Only 마이그레이션 전략](./rollback-recovery/02-forward-only-strategy.md) | "문제가 생기면 되돌리지 않고 앞으로 간다"는 전략의 실제 의미, 다음 버전에서 수정하는 패턴, 애플리케이션과 스키마의 하위 호환성 유지 방법, 각 배포 단계에서 구 버전 앱과 신 버전 스키마의 공존 전략 |
| [03. Flyway Undo 마이그레이션](./rollback-recovery/03-flyway-undo-migration.md) | `U{버전}__*.sql` Undo 파일로 롤백 SQL을 제공하는 방식, Undo가 유효한 실제 시나리오, 데이터가 이미 변경된 경우 Undo의 위험성(데이터 손실 가능), Flyway Teams 기능으로만 사용 가능한 이유 |
| [04. 마이그레이션 실패 시 복구 절차](./rollback-recovery/04-failure-recovery.md) | `flyway_schema_history`의 `success=false` 레코드가 생성되는 조건, `flyway repair`로 실패 레코드를 제거하고 재실행 가능한 상태로 복원하는 절차, 부분 적용된 마이그레이션의 수동 복구 체크리스트 |
| [05. 백업과 마이그레이션](./rollback-recovery/05-backup-before-migration.md) | 대형 마이그레이션 전 DB 스냅샷 절차(RDS 스냅샷, mysqldump), 백업 없는 마이그레이션의 위험, Point-in-Time Recovery(PITR)로 마이그레이션 실패 시 특정 시점으로 복원하는 시나리오, 백업 시간과 서비스 영향도 계산 |

</details>

<br/>

### 🔹 Chapter 5: 팀 협업과 충돌 해결

> **핵심 질문:** 두 개발자가 동시에 같은 버전 번호의 마이그레이션을 만들면? Feature 브랜치에서 마이그레이션 충돌은 어떻게 해결하는가?

<details>
<summary><b>버전 충돌부터 MSA 스키마 관리까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 마이그레이션 버전 충돌](./team-collaboration/01-version-conflict.md) | 두 개발자가 동시에 `V3__*.sql`을 만들었을 때의 시나리오, `out-of-order` 설정으로 순서가 뒤바뀐 마이그레이션을 허용하는 범위와 위험, `yyyyMMddHHmmss` 타임스탬프 기반 버전으로 충돌 확률을 낮추는 팀 규칙 |
| [02. 브랜치 전략과 마이그레이션](./team-collaboration/02-branch-strategy.md) | Feature 브랜치에서 마이그레이션 작성 시 main 병합 후 충돌 해결 절차, 장기 브랜치에서 마이그레이션이 누적될 때의 위험, main 브랜치 최신화 후 버전 번호 재조정 전략, Squash Merge와 마이그레이션 파일의 관계 |
| [03. 마이그레이션 코드 리뷰 체크리스트](./team-collaboration/03-code-review-checklist.md) | 마이그레이션 PR 리뷰 시 반드시 확인할 항목(DDL Lock 위험 여부, 대용량 테이블 처리, 롤백 가능성, 인덱스 적용 여부, 데이터 손실 가능성), 자동화 도구로 Lock 위험 마이그레이션을 PR 단계에서 차단하는 방법 |
| [04. 멀티 모듈 마이그레이션](./team-collaboration/04-multi-module-migration.md) | 여러 Spring 모듈이 같은 DB를 공유하는 경우 마이그레이션 파일 위치 전략(`db/migration/{module}/`), 모듈별 `flyway_schema_history` 테이블 분리, Flyway의 `table` 설정으로 스키마 히스토리를 모듈별로 관리하는 방법 |
| [05. MSA에서의 스키마 관리](./team-collaboration/05-msa-schema-management.md) | Database per Service 원칙에서 각 서비스가 독립적으로 마이그레이션을 소유하는 구조, 여러 서비스가 같은 DB를 공유하다가 스키마 변경 충돌이 나는 시나리오, 공유 테이블 제거 전략, Saga 패턴과 스키마 변경의 조율 |

</details>

<br/>

### 🔹 Chapter 6: CI/CD 파이프라인 통합

> **핵심 질문:** 마이그레이션은 배포 파이프라인 어디에 위치해야 하는가? 실패했을 때 자동으로 감지하고 차단하려면?

<details>
<summary><b>마이그레이션 타이밍부터 Kubernetes Init Container까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 배포 파이프라인에서의 마이그레이션 시점](./cicd-integration/01-migration-timing-in-pipeline.md) | 애플리케이션 시작 시 자동 실행(Spring Boot + Flyway)과 별도 Job으로 실행의 트레이드오프, 두 방식의 실패 시나리오 비교(앱 시작 실패 vs 마이그레이션 Job 실패), 블루-그린 배포에서 마이그레이션 타이밍의 중요성 |
| [02. GitHub Actions 통합](./cicd-integration/02-github-actions-integration.md) | 마이그레이션 전용 Job 구성, 테스트 DB에 마이그레이션 적용 후 통합 테스트 실행, 프로덕션 마이그레이션 수동 승인 게이트(`environment` + `required_reviewers`), 마이그레이션 실패 시 Slack 알림 자동화 |
| [03. 마이그레이션 검증 자동화](./cicd-integration/03-migration-validation.md) | `flyway validate`로 적용 전 체크섬 검증, Dry Run으로 실제 적용 전 실행될 SQL 미리 확인, 스테이징 환경 선 적용 후 프로덕션 적용하는 파이프라인 구성, Lock 위험 마이그레이션 자동 감지 스크립트 |
| [04. Kubernetes 배포와 마이그레이션](./cicd-integration/04-kubernetes-migration.md) | Init Container로 마이그레이션을 앱 시작 전에 완료하는 패턴(앱 컨테이너보다 먼저 실행), Kubernetes Job 리소스로 마이그레이션을 앱과 완전 분리 실행, 실패 시 Pod 재시작으로 인한 중복 실행 방지(`flyway_schema_history` 멱등성) |
| [05. 마이그레이션 감사(Audit)와 컴플라이언스](./cicd-integration/05-migration-audit.md) | `flyway_schema_history`가 자동으로 기록하는 감사 이력(누가/언제/어떤 마이그레이션), 프로덕션 스키마 변경 이력 보존 정책, 컴플라이언스 리포트 생성, `installed_by` 필드로 배포 주체 추적 |

</details>

<br/>

### 🔹 Chapter 7: 실전 시나리오와 Spring 통합

> **핵심 질문:** Spring Boot + Flyway 자동 설정은 어떻게 동작하는가? 테스트 환경에서 마이그레이션을 어떻게 격리하고, 대용량 데이터 변환은 어떻게 처리하는가?

<details>
<summary><b>Spring Boot 자동 설정부터 실전 케이스 스터디까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Spring Boot + Flyway 자동 설정](./spring-integration/01-spring-boot-flyway-autoconfigure.md) | `FlywayAutoConfiguration`이 활성화되는 조건, `FlywayMigrationStrategy` 커스터마이징으로 시작 시 동작 제어, `@FlywayTest`로 테스트 환경에서 클린 마이그레이션 실행, 멀티 DataSource 환경에서 Flyway 분리 구성 |
| [02. 테스트 데이터 관리](./spring-integration/02-test-data-management.md) | `R__test_data.sql` Repeatable 마이그레이션으로 환경별 시드 데이터 관리, Testcontainers + Flyway 조합으로 격리된 테스트 DB 구성, `@Sql` 어노테이션과 Flyway 마이그레이션의 실행 순서, 테스트 격리를 위한 `@Transactional` 롤백 전략 |
| [03. JPA Entity와 마이그레이션 동기화](./spring-integration/03-jpa-entity-sync.md) | Entity 변경과 마이그레이션 파일을 반드시 같은 커밋에 포함하는 이유, `ddl-auto=validate` 모드로 Entity와 실제 스키마 불일치를 시작 시 감지, 스키마 우선(Schema-First) vs Entity 우선(Entity-First) 워크플로우 비교 |
| [04. 대용량 데이터 마이그레이션](./spring-integration/04-large-data-migration.md) | 수천만 건 데이터를 단일 `UPDATE`로 변환하면 안 되는 이유(Lock 점유, 트랜잭션 로그 폭증), Chunk 단위 배치 처리 패턴, 백그라운드에서 점진적으로 마이그레이션하는 Dark Launch 패턴, 배치 진행 상황 모니터링 |
| [05. 실전 케이스 스터디](./spring-integration/05-real-world-case-study.md) | 컬럼 이름 변경 완전 가이드(Expand-Contract 전 과정 SQL + 코드), 복합 인덱스 추가 무중단 적용 시나리오, 단일 테이블 → 두 테이블로 분리하는 마이그레이션 실전, 각 단계별 배포·마이그레이션·앱 코드 변경의 타이밍 |

</details>

---

## 🧪 실험 환경

모든 챕터의 실험은 아래 Docker Compose 환경에서 실행합니다.

```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: migration_test
    ports:
      - "3306:3306"
    command: >
      --innodb_lock_wait_timeout=5
      --general_log=ON
      --general_log_file=/var/log/mysql/general.log

  flyway:
    image: flyway/flyway:9
    volumes:
      - ./migrations:/flyway/sql
    environment:
      FLYWAY_URL: jdbc:mysql://mysql:3306/migration_test
      FLYWAY_USER: root
      FLYWAY_PASSWORD: root
    command: migrate
    depends_on:
      - mysql

  spring-app:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/migration_test
      SPRING_FLYWAY_ENABLED: "true"
      SPRING_JPA_HIBERNATE_DDL_AUTO: validate
    depends_on:
      - mysql
    ports:
      - "8080:8080"
```

### Flyway CLI 핵심 명령어

```bash
# 마이그레이션 상태 확인
flyway -url=jdbc:mysql://localhost:3306/migration_test \
  -user=root -password=root info

# 마이그레이션 실행
flyway migrate

# 체크섬 불일치 수정
flyway repair

# 마이그레이션 검증 (적용 전 확인)
flyway validate

# DDL Lock 모니터링 (MySQL)
SELECT * FROM information_schema.innodb_trx;
SELECT * FROM performance_schema.metadata_locks WHERE OBJECT_TYPE='TABLE';

# gh-ost로 Online Schema Change
gh-ost \
  --host=localhost --user=root --password=root \
  --database=migration_test --table=orders \
  --alter="ADD COLUMN notes VARCHAR(255)" \
  --execute
```

---

## 🗺️ Flyway 내부 동작 흐름

```
Spring Boot 시작
  │
  ▼ FlywayAutoConfiguration 활성화
  │
  ▼ flyway_schema_history 테이블 존재 확인
  │   없으면 생성
  │
  ▼ classpath:db/migration/*.sql 파일 스캔
  │
  ▼ 각 파일의 체크섬 계산 (CRC32)
  │
  ▼ flyway_schema_history와 비교
  │   ├── version 없음 → 새 마이그레이션 → 실행
  │   ├── version 있고 체크섬 일치 → 이미 적용됨 → 건너뜀
  │   └── version 있고 체크섬 불일치 → 오류! 파일 변경 감지
  │
  ▼ 미적용 마이그레이션 순서대로 실행
  │
  ▼ 실행 후 flyway_schema_history에 기록
      (version, script, checksum, installed_on, success)
```

---

## 📌 핵심 원칙

```
이 레포의 3가지 핵심 원칙:

1. 스키마 변경은 코드다
   → SQL 파일로 작성 → Git으로 버전 관리 → PR로 리뷰 → CI/CD로 자동 적용

2. 프로덕션에서 ALTER TABLE은 위험하다
   → DDL Lock 이해 → Online DDL / gh-ost → Expand-Contract 패턴

3. DDL은 롤백할 수 없다
   → Forward-Only 전략 → 하위 호환성 유지 → 배포와 마이그레이션 분리
```

---

## 📚 참고 자료

- [Flyway 공식 문서](https://documentation.red-gate.com/fd/)
- [Liquibase 공식 문서](https://docs.liquibase.com/)
- [gh-ost — GitHub Online Schema Migrations](https://github.com/github/gh-ost)
- [Evolutionary Database Design — Martin Fowler](https://martinfowler.com/articles/evodb.html)
- [MySQL InnoDB Online DDL 공식 문서](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl.html)
- *Designing Data-Intensive Applications* — Martin Kleppmann (스키마 진화 챕터)

---

<div align="center">

**"SQL 직접 실행과 마이그레이션 코드 관리의 차이를 아는 것,**  
**그것이 프로덕션에서 서비스를 지키는 개발자의 차이다"**

</div>
